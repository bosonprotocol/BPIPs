---
bpip: 3
title: Pre-minted Vouchers (Shell rNFTs)
author: Cliff Hall
discussions-to: https://github.com/bosonprotocol/BPIPs/discussions
status: Living
created: 2022-11-22
---

## Abstract
This proposal describes a new feature which allows sellers to pre-mint vouchers, making them available for primary purchase on NFT marketplaces.

## Motivation
Ordinarily, sellers on Boson Protocol create offers, which buyers then commit to, resulting in the issuance of a voucher (aka, a redeemable NFT or rNFT). Vouchers can be resold on secondary markets or redeemed to receive the merchandise.

Since the vouchers don’t exist until buyers commit to the protocol, traffic must be driven to the seller’s site or the Boson Dapp for primary market purchases. Meanwhile, popular marketplaces and destinations such as metaverses, already have a high traffic volume.

In order to help sellers capture that potential market, we propose to solve this "chicken and egg" problem with a new feature called pre-minted vouchers.

## Specification
#### IBosonVoucher
The following methods are added.
```solidity
    /**
     * @notice Reserves a range of vouchers to be associated with an offer
     *
     * Must happen prior to calling preMint
     * Caller must have PROTOCOL role.
     *
     * Reverts if:
     * - Start id is not greater than zero
     * - Offer id is already associated with a range
     *
     * @param _offerId - the id of the offer
     * @param _start - the first id of the token range
     * @param _length - the length of the range
     */
    function reserveRange(
        uint256 _offerId,
        uint256 _start,
        uint256 _length
    ) external;

    /**
     * @notice Pre-mints all or part of an offer's reserved vouchers.
     *
     * For small offer quantities, this method may only need to be
     * called once.
     *
     * But, if the range is large, e.g., 10k vouchers, block gas limit
     * could cause the transaction to fail. Thus, in order to support
     * a batched approach to pre-minting an offer's vouchers,
     * this method can be called multiple times, until the whole
     * range is minted.
     *
     * A benefit to the batched approach is that the entire reserved
     * range for an offer need not be pre-minted at one time. A seller
     * could just mint batches periodically, controlling the amount
     * that are available on the market at any given time, e.g.,
     * creating a pre-minted offer with a validity period of one year,
     * causing the token range to be reserved, but only pre-minting
     * a certain amount monthly.
     *
     * Caller must be contract owner (seller operator address).
     *
     * Reverts if:
     * - Offer id is not associated with a range
     * - Amount to mint is more than remaining un-minted in range
     * - Too many to mint in a single transaction, given current block gas limit
     *
     * @param _offerId - the id of the offer
     * @param _amount - the amount to mint
     */
    function preMint(uint256 _offerId, uint256 _amount) external;

    /**
     * @notice Burn all or part of an offer's preminted vouchers.
     * If offer expires or it's voided, the seller can burn the preminted vouchers that were not transferred yet.
     * This way they will not show in seller's wallet and marketplaces anymore.
     *
     * For small offer quantities, this method may only need to be
     * called once.
     *
     * But, if the range is large, e.g., 10k vouchers, block gas limit
     * could cause the transaction to fail. Thus, in order to support
     * a batched approach to pre-minting an offer's vouchers,
     * this method can be called multiple times, until the whole
     * range is burned.
     *
     * Caller must be contract owner (seller operator address).
     *
     * Reverts if:
     * - Offer id is not associated with a range
     * - Offer is not expired or voided
     * - There is nothing to burn
     *
     * @param _offerId - the id of the offer
     */
    function burnPremintedVouchers(uint256 _offerId) external;

    /**
     * @notice Gets the number of vouchers available to be pre-minted for an offer.
     *
     * @param _offerId - the id of the offer
     * @return count - the count of vouchers in reserved range available to be pre-minted
     */
    function getAvailablePreMints(uint256 _offerId) external view returns (uint256 count);

    /**
     * @notice Gets the range for an offer.
     *
     * @param _offerId - the id of the offer
     * @return range - range struct with information about range start, length and already minted tokens
     */
    function getRangeByOfferId(uint256 _offerId) external view returns (Range memory range);
```

#### IBosonExchangeHandler
The following method is added.
```solidity
    /**
     * @notice Commits to a preminted offer (first step of an exchange).
     *
     * Emits a BuyerCommitted event if successful.
     *
     * Reverts if:
     * - The exchanges region of protocol is paused
     * - The buyers region of protocol is paused
     * - Caller is not the voucher contract, owned by the seller
     * - Exchange exists already
     * - Offer has been voided
     * - Offer has expired
     * - Offer is not yet available for commits
     * - Buyer account is inactive
     * - Buyer is token-gated (conditional commit requirements not met or already used)
     * - Seller has less funds available than sellerDeposit for non preminted offers
     * - Seller has less funds available than sellerDeposit and price for preminted offers
     *
     * @param _buyer - the buyer's address (caller can commit on behalf of a buyer)
     * @param _offerId - the id of the offer to commit to
     * @param _exchangeId - the id of the exchange
     */
    function commitToPreMintedOffer(
        address payable _buyer,
        uint256 _offerId,
        uint256 _exchangeId
    ) external;
```

## Interactions 
![Pre-minted Vouchers Interactions Diagram](./assets/bpip-3/Preminted-Vouchers-Diagram.png "Pre-minted Voucher Sequences Diagram")

## Rationale
Several other potential implementations were considered and options analysis performed across quite a few important dimensions. This was the least objectionable approach.

## Backward compatibility
This specification does not break backward compatibility.

## Implementation
### BosonVoucher
* Support the tracking of reserved ranges so that exchange id and token id can remain the same.
* Emit events that look as if vouchers have been minted, but don't store the owner address to conserve the bulk of an actual minting's gas. 
* In the ownerOf method, if the token id is in a reserved range and has no stored owner, report the contract owner (the seller) as the owner of the voucher. 
* Add a hook transfer hook that will detect if the voucher is being transferred to the first real owner, and if so, call the commitToPremintedOffer protocol method, which does not require payment.
* Implemented [here](https://github.com/bosonprotocol/boson-protocol-contracts/pull/483).

### Protocol
* Implemented [here](https://github.com/bosonprotocol/boson-protocol-contracts/pull/490).

### Discussion
* Discussed [here](https://github.com/bosonprotocol/BPIPs/issues/3).

## Copyright waiver & license
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
