---
bpip: 5
title: Flexible royalties
author: Klemen Zajc
discussions-to: https://github.com/bosonprotocol/BPIPs/discussions/13
status: Review
created: 2023-03-13
---

## Abstract
This proposal describes the change of royalty handling to support more granular control, which enables multiple recipients and setting royalties per individual offer instead of a collection.

## Motivation
Currently, a seller can only set royalties on a voucher contract level, which means that all offers from one seller get the same recipient (seller's treasury) and the same royalty percentage. 
A seller might want to specify multiple recipients and/or set different royalty percentages per offer. A big motivation to do it is that sellers can sell items of multiple brands and they want to pay out per-brand royalties.

A seller could always do it manually, but that presents a bad user experience and require an off-chain management process. Alternatively, they could use a splitting contract (such as https://www.0xsplits.xyz/) to have multiple recipients.
But that requires them to set the splitting contract as the treasury, which means that all proceeds (not only royalties) will eventually end up in a splitter, which is not desired. And it also does not allow setting royalties per offer.

Additional motivation to change royalties is compatibility with https://royaltyregistry.xyz/. Although Boson voucher contracts already implement the EIP2981 royalty standard, which is compatible with the registry, it imposes additional limitation which is currently not met by the voucher contract.

To achieve this we propose to move royalties management from the Boson voucher into the Boson protocol, which will enable both more flexible royalties and full compatibility with Royalty Registry.

## Specification
#### Glossary
**bps**: basis points. A basis point is a unit of measure for percentages and equals 0.01%. Since EVM does not have a float or decimal type, percentages are stored in bps, for example, 23.55% is stored as 2355. The bps range is between 0 and 10,000, representing 0 and 100% respectively.

#### BosonTypes
The following structs are added.
```solidity
    struct RoyaltyRecipient {
        address wallet;
        uint256 minRoyaltyPercentage;
        string externalId;
    }

    struct RoyaltyInfo {
        address payable[] recipients;
        uint256[] bps; // basis points
    }
```
And `Offer` is extended with an additional field
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
+       RoyaltyInfo royaltyInfo;
    }
```


#### IBosonVoucher
The following event is changed.
```diff
- event VoucherInitialized(uint256 indexed sellerId, uint256 indexed royaltyPercentage, string indexed contractURI);
+ event VoucherInitialized(uint256 indexed sellerId, string indexed contractURI);
```

The following methods are removed.
```diff
- function setRoyaltyPercentage(uint256 _newRoyaltyPercentage) external;
- function getRoyaltyPercentage() external view returns (uint256);
```

#### IBosonAccountEvents
The following event is added.
```solidity
    event RoyaltyRecipientsChanged(
        uint256 indexed sellerId,
        BosonTypes.RoyaltyRecipient[] royaltyRecipients,
        address indexed executedBy
    );
```

#### IBosonAccountHandler
The following methods are added.
```solidity
/**
     * @notice Adds royalty recipients to a seller.
     *
     * Emits a RoyalRecipientsUpdated event if successful.
     *
     *  Reverts if:
     *  - The sellers region of protocol is paused
     *  - Seller does not exist
     *  - Caller is not the seller admin
     *  - Caller does not own auth token
     *  - Some recipient is not unique
     *  - some royalty percentage is above the limit
     *
     * @param _sellerId - seller id
     * @param _royaltyRecipients - list of royalty recipients to add
     */
    function addRoyaltyRecipients(uint256 _sellerId, BosonTypes.RoyaltyRecipient[] calldata _royaltyRecipients)
        external;

    /**
     * @notice Updates seller's royalty recipients.
     *
     * Emits a RoyalRecipientsUpdated event if successful.
     *
     *  Reverts if:
     *  - The sellers region of protocol is paused
     *  - Seller does not exist
     *  - Caller is not the seller admin
     *  - Caller does not own auth token
     *  - Length of ids to change does not match length of new values
     *  - Id to update does not exist
     *  - Seller tries to update the address of default recipient
     *  - Some recipient is not unique
     *  - Some royalty percentage is above the limit
     *
     * @param _sellerId - seller id
     * @param _royaltyRecipientIds - list of royalty recipient ids to update
     * @param _royaltyRecipients - list of new royalty recipients corresponding to ids
     */
    function updateRoyaltyRecipients(
        uint256 _sellerId,
        uint256[] calldata _royaltyRecipientIds,
        BosonTypes.RoyaltyRecipient[] calldata _royaltyRecipients
    ) external;

    /**
     * @notice Removes seller's royalty recipients.
     *
     * Emits a RoyalRecipientsUpdated event if successful.
     *
     *  Reverts if:
     *  - The sellers region of protocol is paused
     *  - Seller does not exist
     *  - Caller is not the seller admin
     *  - Caller does not own auth token
     *  - List of ids to remove is not sorted in ascending order
     *  - Id to remove does not exist
     *  - Seller tries to remove the default recipient
     *
     * @param _sellerId - seller id
     * @param _royaltyRecipientIds - list of royalty recipient ids to remove
     */
    function removeRoyaltyRecipients(uint256 _sellerId, uint256[] calldata _royaltyRecipientIds) external;
```
#### IBosonExchangeHandler

```solidity
    /**
     * @notice Gets EIP2981 style royalty information for a chosen exchange.
     *
     * EIP2981 supports only 1 recipient, there fore this method defaults to treasury address.
     * This method is not exactly compliant with EIP2981, since it does not accept `salePrice` and does not return `royaltyAmount,
     * but it rather returns `royaltyPercentage` which is the sum of all bps (exchange can have multiple royalty recipients).
     *
     * This function is meant to be primarily used by boson voucher client, which implements EIP2981.
     *
     * Reverts if exchange does not exist.
     *
     * @param _queryId - if _isPreminted this is offer id, else is the exchange id
     * @param _isPreminted - indicates if the query is for preminted voucher
     * @return receiver - the address of the royalty receiver (seller's treasury address)
     * @return royaltyPercentage - the royalty percentage in bps
     */
    function getExchangeEIP2981Royalties(uint256 _queryId, bool _isPreminted)
        external
        view
        returns (address receiver, uint256 royaltyPercentage);

    /**
     * @notice Gets royalty information for a chosen exchange.
     *
     * Returns a list of royalty recipients and corresponding bps. Format is compatible with Manifold and Foundation royalties
     * and can be directly used by royalty registry.
     *
     * Reverts if exchange does not exist.
     *
     * @param _queryId - if _isPreminted this is offer id, else is the exchange id
     * @param _isPreminted - indicates if the query is for preminted voucher
     * @return royaltyInfo - list of royalty recipients and corresponding bps
     */
    function getExchangeRoyalties(uint256 _queryId, bool _isPreminted)
        external
        view
        returns (BosonTypes.RoyaltyInfo memory royaltyInfo);
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
  - Royalty recipient is not on seller's allow list
  - Royalty percentage is less that the value decided by the admin
  - Total royalty percentage is more than max royalty percentage
```

## Rationale
Moving royalty management from the Boson voucher contract to the protocol contract is sensible since all other offer-related actions happen in the protocol. This then allows royalties to be set at the same time that the offer is created and does not require an additional external or internal transaction. Royalties are fixed at offer creation time and cannot be changed later to prevent potential malicious seller actions.  

Offers are created by the assistant, which should not have the right to specify an arbitrary recipient and/or royalty percentage since that would allow them to set the recipients that should not be entitled to get the royalties. Admin must approve recipients beforehand and the assistant can then choose any subset of recipients (including an empty set). Admin also has the option to set a minimum royalty percentage that a recipient should receive. When the assistant creates an offer, they must set percentages at least equal to or higher than the minimum percentage set by the admin.
The seller's treasury is the default royalty recipient and cannot be removed from the list of approved recipients. It is created automatically when a seller is created and is updated automatically if the treasury address is updated.

New methods in `IBosonExchangeHandler` (`getExchangeEIP2981Royalties` and `getExchangeRoyalties`) are helper functions that are intended to be mainly used by the Boson voucher itself. It is expected that end users or clients will interact with Boson vouchers directly and these `ExchangeHandler` methods will help to efficiently get the royalty recipients. `getExchangeRoyalties` returns the full list of recipients and royalty percentage, while `getExchangeEIP2981Royalties` reduces it to be closer to the `EIP2981` royalties format, which supports only one recipient. `getExchangeEIP2981Royalties` sums up all individual royalty percentages and makes the first recipient in the list the recipient of all royalties. 

## Backward compatibility
This specification has an impact on previous protocol versions. 
- All [methods that create an offer](#ibosonofferhandler-and-ibosonorchestrationhandler) have a different input parameter, so old methods won't work anymore.
- Existing sellers do not have a default royalty recipient set, so they will have to be initialized during the upgrade

## Implementation
### BosonVoucher

* Internal storage
  * `uint256 private _royaltyPercentage;` is not used anymore. But since it affects storage layout we must keep a placeholder in to not affect other data
* Removed methods
  * `setRoyaltyPercentage` and `getRoyaltyPercentage` are totally removed.
  * Previously royalty percentage was also set during the initialization. This step is now omitted.
* Royalty info
Check if the token exists or is preminted. Otherwise, return zero address and zero royalty amount.
  * If is preminted, query the protocol for royalties with `offerId` as an identifier.
  * If it is a normal voucher, query the protocol for royalties with `tokenId` as an identifier.
  * Apply returned percentage to input `_salePrice`.

* Implemented [here](https://github.com/bosonprotocol/boson-protocol-contracts/pull/579).

### Protocol

* Royalty recipients management
  * When a seller is created, a default royalty recipient is created (treasury address), which cannot be removed.
  * If the treasury address is updated via `updateSeller`, the default recipient is updated as well.
  * Other royalty recipients can be
    * added with `addRoyaltyRecipients`,
    * updated with `updateRoyaltyRecipients` and
    * removed with `removeRoyaltyRecipients`.
  * Royalty recipients should be unique within a seller but can duplicate across sellers.
* Offer creation
  * `Offer` has an additional field - struct `RoyaltyInfo` which includes a list of recipients with corresponding royalty percentages.
  * Royalties are fixed when an offer is created and cannot be changed later.
  * If some recipient is not approved by the admin or the specified royalty percentage is too low, offer creation fails.
* Retrieving royalty info
  * Use `getExchangeRoyalties` to get a full list of royalty recipients.
  * Use `getExchangeEIP2981Royalties` to get a reduced list of royalty recipients. The returned recipient is the first address from the recipient list and returned royalty percentage is a sum of all individual royalty percentages.
  * Both methods can be queried with `tokenId` for preminted offers and `exchangeId` for actual vouchers.

* Implemented [here](https://github.com/bosonprotocol/boson-protocol-contracts/pull/579).

## Compatibility with external marketplaces

* External marketplaces that honor the royalties from https://royaltyregistry.xyz/ will send royalties to all recipients correctly.
Marketplaces that follow EIP2981 will send royalties only to the first recipient.

To support both options in a single offer, we suggest using a splitter contract (for example https://www.0xsplits.xyz/) and setting the RoyaltyInfo as follows:

```
RoyaltyInfo{
  recipients: [splitterAddress, recipient1, recipient2, recipient3, ...]
  bps: [0, bps1, bps2, bps3, ...]
}
```
  
## Copyright waiver & license
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
