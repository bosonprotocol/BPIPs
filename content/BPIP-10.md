---
bpip: 10
title: Off-chain Listing Phase
authors: Klemen Zajc
discussions-to: https://github.com/bosonprotocol/BPIPs/discussions/38
status: Draft
created: 2025-07-02
---

## Abstract
This proposal describes an option to do the whole listing phase (including price discovery) off-chain and only submit the agreement to Boson Protocol.

## Motivation
Currently, the exchange starts on-chain, by offer creation. This makes the offer visible to buyers, and anyone can commit to it:
- If it’s a fixed price offer, the buyer can do it without restrictions
- If it’s a price discovery offer, it depends on the discovery mechanism. In some cases, the buyer can buy it immediately, but in some cases (e.g. if the item is on auction), the seller must explicitly accept the bid, so the seller can choose who they are selling to.

The flow includes two main steps - “create offer” and “commit to the offer”. Even if the seller and the buyer decide on an exchange off-chain, they must still perform these two steps.

The protocol could be changed to also facilitate the exchanges where buyers and sellers fully agree on exchange terms before any information is stored in the protocol. Any of the involved parties could submit an agreement to the protocol, which would result in the same state as if one created an offer and the other committed to it.
The protocol role in this scenario is to validate the agreement, and do all the other actions that happen in the protocol after “commit to offer”, for example, Boson issues a tradeable rNFT, encumbers funds until the exchange finalisation and handles the disputes.

The protocol's guarantees are preserved, since the buyer still gets the guarantee to either get the item or the money back. The only part that is left out compared to the existing protocol is the discovery phase.

![Off-chain Listing and Negotiation](./assets/bpip-10/create-offer-and-commit.png "Off-chain Listing and Negotiation")

## Specification
#### IBosonExchangeHandler

A new method that combines `createOffer` and `commitToOffer` is added to the protocol.
There is no distinction between static offer and price-discovery offer - it is irrelevant since all funds are encumbered at the same time the offer is created. 
When this offer is created the buyer gets `quantity` of rNFTs.

```diff solidity
     /**
     * @notice Creates an offer.
     *
     * Emits an OfferCreated, FundsEncumbered, BuyerCommitted and SellerCommitted event if successful.
     *
     * Reverts if:
     * - The offers region of protocol is paused
     * - Valid from date is greater than valid until date
     * - Valid until date is not in the future
     * - Both voucher expiration date and voucher expiration period are defined
     * - Neither of voucher expiration date and voucher expiration period are defined
     * - Voucher redeemable period is fixed, but it ends before it starts
     * - Voucher redeemable period is fixed, but it ends before offer expires
     * - Dispute period is less than minimum dispute period
     * - Resolution period is not between the minimum and the maximum resolution period
     * - Voided is set to true
     * - Available quantity is 0
     * - Dispute resolver wallet is not registered, except for absolute zero offers with unspecified dispute resolver
     * - Dispute resolver is not active, except for absolute zero offers with unspecified dispute resolver
     * - Seller is not on dispute resolver's seller allow list
     * - Dispute resolver does not accept fees in the exchange token
     * - Buyer cancel penalty is greater than price
     * - Collection does not exist
     * - When agent id is non zero:
     *   - If Agent does not exist
     * - If the sum of agent fee amount and protocol fee amount is greater than the offer fee limit determined by the protocol
     * - If the sum of agent fee amount and protocol fee amount is greater than fee limit set by seller
     * - Royalty recipient is not on seller's allow list
     * - Royalty percentage is less than the value decided by the admin
     * - Total royalty percentage is more than max royalty percentage
     * - Not enough funds can be encumbered
     *
     * @param _offer - the fully populated struct with offer id set to 0x0 and voided set to false
     * @param _offerDates - the fully populated offer dates struct
     * @param _offerDurations - the fully populated offer durations struct
     * @param _drParameters - the id of chosen dispute resolver (can be 0) and mutualizer address (0 for self-mutualization)
     * @param _agentId - the id of agent
     * @param _feeLimit - the maximum fee that seller is willing to pay per exchange (for static offers)
     * @param _otherCommitter - the address of the other party
     * @param _signature - signature of the other party. If the signer is EOA, it must be ECDSA signature in the format of (r,s,v) struct, otherwise, it must be a valid ERC1271 signature.
     */
    function createOfferAndCommit(
        BosonTypes.Offer memory _offer,
        BosonTypes.OfferDates calldata _offerDates,
        BosonTypes.OfferDurations calldata _offerDurations,
        BosonTypes.DRParameters calldata _drParameters,
        uint256 _agentId,
        uint256 _feeLimit,
        address _otherCommitter,
        bytes calldata _signature
    ) external;

     /**
     * @notice Voids an offer.
     *
     * Emits an OfferVoided, FundsEncumbered, BuyerCommitted and SellerCommitted event if successful.
     *
     * Reverts if:
     * - The offers region of protocol is paused
     * - The caller is not one of the committers
     *
     * @param _offer - the fully populated struct with offer id set to 0x0 and voided set to false
     * @param _offerDates - the fully populated offer dates struct
     * @param _offerDurations - the fully populated offer durations struct
     * @param _drParameters - the id of chosen dispute resolver (can be 0) and mutualizer address (0 for self-mutualization)
     * @param _agentId - the id of agent
     * @param _feeLimit - the maximum fee that seller is willing to pay per exchange (for static offers)
     * @param _otherCommitter - the address of the other party
     */
    function voidOffer(
        BosonTypes.Offer memory _offer,
        BosonTypes.OfferDates calldata _offerDates,
        BosonTypes.OfferDurations calldata _offerDurations,
        BosonTypes.DRParameters calldata _drParameters,
        uint256 _agentId,
        uint256 _feeLimit,
        address _otherCommitter
    ) external;
```

## Rationale
The buyer and seller can agree on an exchange off-chain, but if they want the protection offered by Boson, they must enter the protocol at the commitment point. All offer data must still be provided, since the real world exchange does not happen atomically, and the protocol needs all the information that is normally needed in the case of (escalated) dispute.  

This approach adds the efficiency to the exchange flow since it does not require from seller to create an offer, that might never be fulfilled. Additionally, the participants do not have to provide the funds upfront, since they are all encumbered at the commitment time.  

If one of the committers already agreed on an exchange (i.e. they signed the agreement), they can opt-out without any penalty before it is submitted on-chain. The protocol must provide a method, where the offer can be voided before it is even created.

## Backward compatibility
This specification does not break backward compatibility.

## Implementation
* Add another storage variable `mapping(bytes32=>bool) isOfferVoided`
* Create offer and commit
  * Perform all the validations done in `createOffer`
  * Perform all the validations done in `commitToOffer`
  * Calculate hash of offer parameters and caller's address. Validate signature validity (ECSDA signature for EOA or EIP1271 contract signature if not EOA). If validation fails, revert.
  * If `isOfferVoided[hash]`, revert.
  * In `commitToOffer` the party that created the offer needs to provide its payment upfront. In this `createOfferAndCommit` all payments are encumbered at the same time. Both parties must approve protocol to transfer the exchange token. If the exchange token is native, the other party must approved a wrapped version of the native token.
* Void offer
  * Calculate hash of offer parameters and other committer's address. Validate signature validity (ECSDA signature for EOA or EIP1271 contract signature if not EOA). If validation fails, revert.
  * Set `isOfferVoided[hash]=true`

### Security considerations

None.
  
## Copyright waiver & license
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
