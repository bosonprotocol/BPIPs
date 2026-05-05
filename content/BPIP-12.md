---
bpip: 12
title: Token-Transfer Authorization in Metatransactions
authors: Klemen Zajc
discussions-to: https://github.com/bosonprotocol/BPIPs/discussions/43
status: Draft
created: 2026-05-04
---

## Abstract
This proposal introduces a new metatransaction entry point that lets a relayer submit a single transaction that both authorises the metatransaction and pulls the required token funds — eliminating the need for any prior on-chain `approve` call. Three pull strategies are supported in a mixed, per-slot queue: ERC-3009 (`receiveWithAuthorization`), EIP-2612 (`permit` + `transferFrom`), and Permit2.

## Motivation
The existing `executeMetaTransaction` entry point allows a relayer to submit a signed function call on behalf of a user. However, functions that move ERC-20 tokens (e.g. `commitToOffer`, `createOfferAndCommit`, `depositFunds`) still require the user to issue an on-chain `approve` before the metatransaction can pull funds. This creates a two-step UX that partially defeats the purpose of metatransactions.

Several token standards provide signature-based authorisation that proves the user's intent and moves funds in the same call:

- **ERC-3009** (`receiveWithAuthorization`) — used by USDC and many other tokens; combines auth and pull in one external call.
- **EIP-2612** (`permit`) — token-native permit; widely supported by modern stablecoins.
- **Permit2** — Uniswap's universal gateway; works for any ERC-20 after a one-time `approve(Permit2, MaxUint)`.

By accepting a queue of such authorisations alongside the metatransaction payload, the protocol can route fund pulls through whichever strategy the user's token supports — all in one relayer-submitted transaction.

## Specification

### BosonTypes

A new enum is added to identify the pull strategy for each slot in the authorization queue.

```solidity
enum TokenTransferAuthorizationStrategy {
    None,
    ERC3009,
    EIP2612,
    Permit2
}
```

### IBosonMetaTransactionsHandler

A new entry point is added alongside the existing `executeMetaTransaction`.

```solidity
/**
 * @notice Executes a metatransaction with a queue of per-transfer token-pull authorizations.
 *
 * Each entry in _tokenTransferAuthorization is consumed in order by transferFundsIn calls
 * made during execution of _functionSignature. An empty-bytes entry ("0x") is a
 * fallback marker that causes that transfer slot to fall back to safeTransferFrom
 * (requires a prior on-chain approve). Entries beyond the number of transferFundsIn
 * calls are ignored; the queue is cleared from transient storage at transaction end.
 *
 * Emits MetaTransactionExecuted event if successful.
 *
 * Reverts if:
 * - The metatransactions region of protocol is paused
 * - Nonce has already been used by the signer for this function
 * - Function signature is for a function that cannot be called via metatransaction
 * - _functionSignature cannot be decoded
 * - Any reason the encoded function itself reverts
 * - Any entry's strategy tag is unrecognised
 * - An ERC-3009 receiveWithAuthorization call fails (e.g. invalid signature, expired)
 * - An EIP-2612 permit call fails and the current allowance does not equal the required amount
 * - A Permit2 permitTransferFrom call fails
 *
 * @param _userAddress - the address of the user on whose behalf the transaction is being executed
 * @param _functionSignature - the encoded function call to execute
 * @param _nonce - nonce of the transaction, used to prevent replay attacks
 * @param _sigR - r part of the metatransaction signer's signature
 * @param _sigS - s part of the metatransaction signer's signature
 * @param _sigV - v part of the metatransaction signer's signature
 * @param _tokenTransferAuthorization - ABI-encoded queue of (TokenTransferAuthorizationStrategy, bytes) pairs,
 *   one per transferFundsIn call in execution order. Each non-empty entry is
 *   abi.encode(TokenTransferAuthorizationStrategy strategy, bytes data) where data
 *   holds the strategy-specific fields described below.
 */
function executeMetaTransactionWithTokenTransferAuthorization(
    address _userAddress,
    bytes calldata _functionSignature,
    bytes32 _nonce,
    bytes32 _sigR,
    bytes32 _sigS,
    uint8 _sigV,
    bytes[] calldata _tokenTransferAuthorization
) external payable;
```

#### Per-strategy entry payload (`data` field)

| Strategy | `data` encoding |
|----------|-----------------|
| `ERC3009` | `abi.encode(address from, address to, uint256 value, uint256 validAfter, uint256 validBefore, bytes32 nonce, uint8 v, bytes32 r, bytes32 s)` |
| `EIP2612` | `abi.encode(uint256 deadline, uint8 v, bytes32 r, bytes32 s)` |
| `Permit2` | `abi.encode(uint256 nonce, uint256 deadline, bytes signature)` |
| `None` / `"0x"` | empty — falls back to `safeTransferFrom` |

#### Queue layout rules

The queue length must match the number of `transferFundsIn` positions in the called function. A position that is unconditionally skipped at runtime (zero amount, `useDepositedFunds=true`, native currency) still occupies a slot; the library's `discardNext()` advances the queue head without performing any transfer. This means relayers can build queues from a static template per function, without knowing runtime values like `sellerDeposit`, `price`, or flag values.

| Function | Queue layout |
|----------|-------------|
| `depositFunds` | `[auth(amount)]` |
| `commitToOffer` | `[buyer_auth(price)]` |
| `commitToConditionalOffer` | `[buyer_auth(price)]` |
| `commitToBuyerOffer` | `[seller_auth(sellerDeposit)]` |
| `createOfferAndCommit` (seller offer) | `[creator_auth(sellerDeposit), buyer_auth(price)]` |
| `createOfferAndCommit` (buyer offer) | `[creator_auth(price), seller_auth(sellerDeposit)]` |
| `commitToPriceDiscoveryOffer` | `[buyer_auth(price)]` |
| `escalateDispute` | `[buyer_auth(escalationFee)]` |

### TokenTransferAuthorizationLib

A new internal library manages the queue in transient storage (EIP-1153 / Cancun). Key design points:

- **Transient storage** — the queue is stored with `tstore` / `tload`, so it is automatically cleared at the end of each transaction with no explicit cleanup required.
- **ERC-7201-style slot masking** — the last byte of each entry's base slot is zeroed, giving every entry a 256-sub-slot range that cannot overlap with any other entry for typical ERC-3009 payloads (which use 7 sub-slots).
- **`discardNext()`** — advances the queue head by one without performing any work. Called by `transferFundsIn` for zero-amount transfers and by `createOfferAndCommit` when `useDepositedFunds=true` bypasses the creator pull.
- **`consumeForTransfer(token, from, amount)`** — pops the next entry, decodes the `(strategy, data)` envelope, and dispatches to the appropriate per-strategy helper. Returns `false` for `None` / empty-bytes entries, causing the caller to fall through to `safeTransferFrom`.

### EIP-2612 diversion guard

The EIP-2612 path is the only strategy that splits signature verification (`permit`) from the transfer (`safeTransferFrom`), which opens a cross-permit allowance diversion window. The library guards against this as follows:

- If the current on-chain allowance already equals `_amount`, the `permit` call is skipped and `safeTransferFrom` proceeds directly. This handles the benign case where another party has already replayed the same permit.
- Otherwise, `permit` is called. If a frontrunner has already consumed the nonce with a *different* permit, the `permit` call reverts, causing the entire metatransaction to revert. No funds move.

ERC-3009 and Permit2 are unaffected — both verify the signature inside the same external call that performs the transfer, leaving no window between authorisation and pull.

## Rationale

Storing the authorization queue in transient storage rather than calldata threading avoids changes to every internal function that calls `transferFundsIn`. The queue is invisible to the internal call graph; only the entry point and `transferFundsIn` interact with it.

Supporting three strategies in a single queue — with per-slot strategy tags — lets a single metatransaction cover flows that require two transfers from different parties (e.g. `createOfferAndCommit`), each potentially using a different token standard. New strategies can be added by appending an enum value and a private dispatch helper without changing the queue encoding or any existing call site.

Choosing transient storage (EIP-1153) instead of regular storage avoids the gas cost of explicit cleanup and prevents slot-squatting attacks where a caller pre-fills the queue before calling the protocol.

## Backward compatibility
This proposal does not break backward compatibility. The existing `executeMetaTransaction` entry point is unchanged. Functions that do not use the new entry point are entirely unaffected.

Requires a network that supports EIP-1153 (Cancun hardfork, active on Ethereum mainnet since March 2024) and Solidity ≥ 0.8.24.

## Implementation

* Add `TokenTransferAuthorizationStrategy` enum to `BosonTypes`.
* Implement `TokenTransferAuthorizationLib` with transient-storage queue management, per-strategy helpers (`_consumeERC3009`, `_consumeEIP2612`, `_consumePermit2`), `discardNext()`, and `consumeForTransfer()`.
* Add interfaces `IERC3009`, `IERC2612`, and `IPermit2` (the `permitTransferFrom` flow).
* Extend `FundsBase.transferFundsIn` to call `TokenTransferAuthorizationLib.consumeForTransfer` before falling back to `safeTransferFrom`; call `discardNext()` in the zero-amount branch.
* Extend `ExchangeCommitFacet.createOfferAndCommit` to call `discardNext()` when `useDepositedFunds=true`.
* Add `executeMetaTransactionWithTokenTransferAuthorization` to `MetaTransactionsHandlerFacet` — loads the queue into transient storage then delegates to the existing metatransaction execution logic.

### Security considerations

* **Replay protection** — the metatransaction nonce prevents the outer call from being replayed. Each per-strategy entry carries its own replay protection (ERC-3009 nonce, EIP-2612 nonce, Permit2 nonce).
* **Cross-permit diversion (EIP-2612)** — mitigated by the allowance-equality guard described above.
* **Queue overflow** — excess entries are silently ignored; the queue head never reads past the number of `transferFundsIn` calls in the executed function.
* **Transient slot isolation** — ERC-7201-style masking ensures entries cannot overlap for queues of up to 256 sub-slots per entry.
* **Reentrancy** — transient storage persists within a transaction but does not survive across calls at the EVM level; a reentrant call would consume from the same queue, which is safe because the protocol's existing reentrancy guards prevent reentrant execution of state-mutating functions.

## Copyright waiver & license
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
