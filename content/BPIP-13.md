---
bpip: 13
title: Atomic Commit-and-Redeem
authors: Klemen Zajc
discussions-to: https://github.com/bosonprotocol/BPIPs/discussions/45
status: Draft
created: 2026-05-04
---

## Abstract
This proposal introduces three new orchestration methods that bundle the commit and redeem steps into a single atomic transaction, enabling buyers to acquire and immediately redeem a voucher without waiting for the voucher to be issued and then redeemed in separate transactions.

## Motivation
Currently, completing an exchange requires at least two separate on-chain transactions: committing to an offer (which issues a voucher) and subsequently redeeming that voucher. This two-step flow exists because the voucher can be traded between commitment and redemption, and because the buyer may want to delay redemption.

However, in many practical scenarios — particularly in agentic or automated commerce — the buyer has no intention of trading the voucher and wants the item delivered as quickly as possible. Requiring two transactions adds latency, increases gas costs, and complicates the buyer experience.

Adding atomic commit-and-redeem orchestration methods removes this friction: the buyer pays and simultaneously signals readiness to redeem in one transaction. This is especially valuable for:

- Agent-to-agent commerce where the buyer agent needs the item immediately.
- Scenarios where the seller creates and signs an offer off-chain (using the `FullOffer` mechanism from BPIP-10) and the buyer wants to complete the entire flow in one call.
- Applications that want to minimise the number of user-facing wallet interactions.

## Specification

### BosonErrors

A new error is introduced for cases where an atomic commit-and-redeem is attempted on a buyer-created offer, which is not supported.

```solidity
error OfferCreatorMustBeSeller();
```

### IBosonOrchestrationHandler

Three new methods are added. In all three, the committer (and therefore the voucher holder at redemption time) is `_msgSender()`, which guarantees that the redeem authorisation check always passes atomically.

```solidity
/**
 * @notice Commits to a static offer and immediately redeems the issued voucher.
 * The caller is the buyer. The buyer must send the offer price in the exchange token.
 *
 * Emits BuyerCommitted and VoucherRedeemed events if successful.
 *
 * Reverts if:
 * - The exchanges region of protocol is paused
 * - The buyers region of protocol is paused
 * - OfferId is invalid
 * - Offer has been voided
 * - Offer has expired
 * - Offer is not yet available for commits
 * - Offer's quantity available is zero
 * - Offer is not a static offer
 * - Offer has a condition (use commitToConditionalOfferAndRedeemVoucher instead)
 * - Offer price is in native token and caller does not send enough
 * - Offer price is in some ERC20 token and caller also sends native currency
 * - Contract at token address does not support ERC20 function transferFrom
 * - Calling transferFrom on token fails for some reason (e.g. protocol is not approved to transfer)
 * - Received ERC20 token amount differs from the expected value
 * - Seller has less funds available than sellerDeposit
 *
 * @param _offerId - the id of the offer to commit to and redeem
 */
function commitToOfferAndRedeemVoucher(uint256 _offerId) external payable;

/**
 * @notice Commits to a conditional offer and immediately redeems the issued voucher.
 * The caller is the buyer. The buyer must send the offer price in the exchange token.
 *
 * Emits BuyerCommitted, ConditionalCommitAuthorized, and VoucherRedeemed events if successful.
 *
 * Reverts if:
 * - The exchanges region of protocol is paused
 * - The buyers region of protocol is paused
 * - OfferId is invalid
 * - Offer has been voided
 * - Offer has expired
 * - Offer is not yet available for commits
 * - Offer's quantity available is zero
 * - Offer creator is not the seller (buyer-created offers are not supported)
 * - Buyer is token-gated (conditional commit requirements not met or already used)
 * - Offer price is in native token and caller does not send enough
 * - Offer price is in some ERC20 token and caller also sends native currency
 * - Contract at token address does not support ERC20 function transferFrom
 * - Calling transferFrom on token fails for some reason (e.g. protocol is not approved to transfer)
 * - Received ERC20 token amount differs from the expected value
 * - Seller has less funds available than sellerDeposit
 *
 * @param _offerId - the id of the offer to commit to and redeem
 * @param _conditionalTokenId - the token id to use for the conditional commit, if applicable
 */
function commitToConditionalOfferAndRedeemVoucher(uint256 _offerId, uint256 _conditionalTokenId) external payable;

/**
 * @notice Creates an offer from a seller-signed FullOffer, commits to it, and immediately redeems the issued voucher.
 * The caller is the buyer. The buyer must send the offer price in the exchange token.
 * Only seller-created offers are supported; buyer-created offers revert with OfferCreatorMustBeSeller.
 *
 * Emits OfferCreated, BuyerCommitted, and VoucherRedeemed events if successful.
 * Emits ConditionalCommitAuthorized if the offer has a condition.
 *
 * Reverts if:
 * - The offers region of protocol is paused
 * - The exchanges region of protocol is paused
 * - The buyers region of protocol is paused
 * - Offer creator is not the seller (OfferCreatorMustBeSeller)
 * - Signature from offerCreator is invalid
 * - Any offer validation that applies to createOffer
 * - Any commit validation that applies to commitToOffer
 * - Offer price is in native token and caller does not send enough
 * - Offer price is in some ERC20 token and caller also sends native currency
 * - Contract at token address does not support ERC20 function transferFrom
 * - Calling transferFrom on token fails for some reason (e.g. protocol is not approved to transfer)
 * - Received ERC20 token amount differs from the expected value
 * - Seller has less funds available than sellerDeposit
 *
 * @param _fullOffer - the fully populated struct containing offer, offer dates, offer durations, dispute resolution parameters, condition, agent id and fee limit
 * @param _offerCreator - the address of the offer creator (seller assistant)
 * @param _signature - EIP-712 signature from the offer creator
 * @param _conditionalTokenId - the token id to use for the conditional commit, if applicable
 */
function createOfferCommitAndRedeem(
    BosonTypes.FullOffer calldata _fullOffer,
    address _offerCreator,
    bytes calldata _signature,
    uint256 _conditionalTokenId
) external payable;
```

### Metatransaction support

All three methods are callable via the metatransaction with authorized token transfer mechanism ([BPIP12](BPIP-12.md)). The authorization queue layouts are:

| Method | Auth queue |
|--------|-----------|
| `commitToOfferAndRedeemVoucher` | `[buyer_auth(offer.price)]` |
| `commitToConditionalOfferAndRedeemVoucher` | `[buyer_auth(offer.price)]` |
| `createOfferCommitAndRedeem` (seller offer only) | `[seller_auth(sellerDeposit), buyer_auth(price)]` |

The redeem leg performs no additional `transferFundsIn` calls, so each queue is identical in shape to its non-redeem counterpart. The `useDepositedFunds=true` discard-slot rule applies to `createOfferCommitAndRedeem` in the same way it applies to `createOfferAndCommit`. The buyer is always hardcoded to `_msgSender()` in the orchestration variants, so there is no separate `_committer` parameter.

## Rationale

Combining commit and redeem into a single transaction is safe because the committer is always `_msgSender()`. At the moment the redeem check runs, the voucher holder is guaranteed to be the caller, so the authorisation check passes without any additional state. Bundled twins are transferred to `_msgSender()` during the redeem leg, mirroring the behaviour of standalone `redeemVoucher`.

To share the internal commit and redeem logic across the new orchestration methods without duplicating code or exceeding the 24 KB facet size limit, two new abstract base contracts are introduced:

- **ExchangeRedeemBase** — contains `redeemVoucherInternal`, `burnVoucher`, `transferTwins`, and `updateNFTRanges`. Voucher token-id backward compatibility with pre-2.2.0 exchanges is handled directly inside `burnVoucher` using an immutable versioning threshold (`EXCHANGE_ID_2_2_0`): for exchange ids at or above the threshold the token id is `exchangeId | (offerId << 128)`; older ids are passed through unchanged.
- **ExchangeCommitBase** — contains `commitToOfferInternal`, `verifyOffer`, `authorizeCommit`, `holdsThreshold`/`holdsSpecificToken`, `validateConditionRange`, `handleDRFeeCollection`, and `addSellerParametersToBuyerOffer`. `ExchangeCommitFacet` now inherits it instead of implementing these directly.

`OrchestrationHandlerFacet2` inherits both bases and the three new methods delegate to the shared internals.

Buyer-created offers are explicitly rejected in all three methods (`OfferCreatorMustBeSeller`). The atomic flow assumes the seller has already deposited the `sellerDeposit`; there is no mechanism in a single transaction to obtain both the buyer's payment and the seller's deposit from two different parties simultaneously.

## Backward compatibility
This proposal does not break backward compatibility. All existing methods continue to work unchanged. The refactoring of `ExchangeHandlerFacet` and `ExchangeCommitFacet` into base contracts is an implementation detail with no observable interface change.

## Implementation
* Introduce `ExchangeRedeemBase` with the internal redeem logic extracted from `ExchangeHandlerFacet`.
* Introduce `ExchangeCommitBase` with the internal commit logic extracted from `ExchangeCommitFacet`.
* `ExchangeHandlerFacet` inherits `ExchangeRedeemBase`. Backward-compatible voucher token-id handling is implemented directly in `ExchangeRedeemBase.burnVoucher` via `EXCHANGE_ID_2_2_0` — no override is needed.
* `ExchangeCommitFacet` inherits `ExchangeCommitBase`.
* `OrchestrationHandlerFacet2` inherits both `ExchangeCommitBase` and `ExchangeRedeemBase` and adds the three new public methods.
* Each new method:
  1. Validates and processes the commit step using shared internals (including offer validation, condition check, fund encumbrance). The commit is called with `_skipVoucher = true`, so no NFT is minted and `voucherCount` is not incremented.
  2. Immediately calls `redeemVoucherInternal(_exchangeId, _skipVoucher: true)`, which skips the burn call and buyer-ownership check (no voucher NFT was ever issued), and transfers any twins to `_msgSender()`.
* Add `OfferCreatorMustBeSeller` to `BosonErrors.sol`.


### Security considerations
The only party that can initiate an atomic commit-and-redeem is `_msgSender()`. Since `_msgSender()` is also the voucher holder at the moment of redemption, there is no way for a third party to intercept the voucher between issuance and redemption within the same transaction. All existing commit and redeem validations remain in place.

## Copyright waiver & license
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
