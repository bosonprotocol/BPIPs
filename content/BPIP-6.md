---
bpip: 6
title: Multiple collections per seller
author: Klemen Zajc
discussions-to: https://github.com/bosonprotocol/BPIPs/discussions/15
status: Draft
created: 2023-03-13
---

## Abstract
This proposal describes the possibility of having multiple collections per offer.

## Motivation
When a seller creates multiple offers, all vouchers are issued on a single Boson Voucher contract. Boson Vouchers are ERC721 tokens and external observers typically consider all tokens on the same contract as one collection.
This has an implication on marketplaces and AMM pools, where all Boson Vouchers are then considered equivalent even if they belong to different offers.

For example, OpenSea currently supports multiple recipients per collection, but they cannot be set per individual offer. If a seller sells multiple brands and would like to send royalties based on sold items, they just don't have a way to do it on a marketplace. [Proposal BPIP#5](https://github.com/bosonprotocol/BPIPs/pull/12) addresses basic changes that need to be done in the protocol to even support multiple and offer-based recipients, but that is still not enough for the marketplaces that enforce single royalties for a whole collection.

Sellers could always create multiple "subSeller", where they would have a seller account for every brand they want to sell. But with the raising number of brands, this quickly becomes infeasible.

In this BPIP we propose to allow creating multiple collections for a single seller, which would immediately allow better control of offers on 3rd party services.

## Specification
#### BosonTypes
`Offer` is extended with an additional field

```diff solidity
struct Offer {
    uint256 id;
    uint256 sellerId;
    uint256 price;
    uint256 sellerDeposit;
    uint256 buyerCancelPenalty;
    uint256 quantityAvailable;
    address exchangeToken;
    string metadataUri;
    string metadataHash;
    bool voided;
+   uint256 collectionIndex;
}
```

#### IBosonAccountEvents
The following event is added.
```solidity
event CollectionCreated(
    uint256 indexed sellerId,
    uint256 collectionIndex,
    address collectionAddress,
    string indexed externalId
    address indexed executedBy
);
```

#### IBosonAccountHandler
The following methods are added.
```solidity
/**
 * @notice Creates a new seller collection.
 *
 * Emits a CollectionCreated event if successful.
 *
 *  Reverts if:
 *  - The offers region of protocol is paused
 *  - Caller is not the seller assistant
 *
 * @param _externalId - external collection id
 */
    function createNewCollection(string calldata _externalId, string calldata _contractURI) external;
```

#### IBosonOfferHandler and IBosonOrchestrationHandler
Although definitions of methods that create an offer do not change, the type of input parameter `Offer offer` changes. As the consequence, the signatures of all methods that accept `Offer` also change. Affected methods are

- createOffer
- createOfferBatch
- createSellerAndPremintedOffer
- createOfferWithCondition and createPremintedOfferWithCondition
- createOfferAddToGroup and createPremintedOfferAddToGroup
- createOfferAndTwinWithBundle and createPremintedOfferAndTwinWithBundle
- createOfferWithConditionAndTwinAndBundle and createPremintedOfferWithConditionAndTwinAndBundle
- createSellerAndOfferWithCondition and createSellerAndPremintedOfferWithCondition
- createSellerAndOfferAndTwinWithBundle and createSellerAndPremintedOfferAndTwinWithBundle
- createSellerAndOfferWithConditionAndTwinAndBundle and createSellerAndPremintedOfferWithConditionAndTwinAndBundle

All these methods must revert if, for any Offer:
```solidity
  - CollectionIndex is out of bounds
```

## Rationale
Boson Vouchers already work as a minimal clone of Beacon Proxy Contract which points to Boson Voucher implementation. Protocol already stores mapping `sellerId => cloneAddress`.
Adding more clones to the existing seller therefore should not pose an extreme change to the protocol, since the majority of the logic already exists.

All methods that interact with vouchers (`commitToOffer`, `redeemVoucher`, `cancelVoucher`, `revokeVoucher`, `onVoucherTransferred`) would just have to look at different mapping that they currently do. The only change is at offer creation time, when assistant must specify which collection the offer belongs to, while other methods don't require any additional parameters, since information about the collection is available to the protocol.

## Backward compatibility
This specification has an impact on previous protocol versions. 
- All [methods that create an offer](#ibosonofferhandler-and-ibosonorchestrationhandler) have a different input parameter, so old methods won't work anymore.
- Need to investigate how existing collection data will be affected.

## Implementation
Implementation details are not known yet.
  
## Copyright waiver & license
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
