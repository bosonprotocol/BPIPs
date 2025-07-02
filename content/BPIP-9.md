---
bpip: 9
title: Buyer initiated exchange
authors: Klemen Zajc
discussions-to: https://github.com/bosonprotocol/BPIPs/discussions/36
status: Draft
created: 2025-07-02
---

## Abstract
This proposal describes a new way to create Boson protocol offers, which allows buyers to initiate the exchange.

## Motivation
In Boson, the seller always first creates an offer, which can then be discovered by the buyer. When a buyer commits to an offer, they pay the price, which remains encumbered until the exchange is finalised (either the item is delivered and funds are released to the seller, or the dispute is raised and resolved via one of Boson mechanisms)

It doesn’t matter if the offer is with a fixed price or it’s a price discovery offer; it’s always the seller who creates the first step in the protocol.

That could be extended - instead of the seller offering a service or product and asking for a payment, the buyer could offer the payment and ask for a service. Once seller commits to an offer, the Boson Protocol user flows continue as usual. Therefore, all protocol guarantees are preserved.  

This change will extend the protocol usefulness. Often the buyers are seeking for a item or service and the sellers are not aware of the demand. In the era of agentic commerce this is expected to become even more crucial than it is now.

## Specification
#### IBosonOfferHandler

```diff solidity
/**
     * @notice Creates an offer.
     *
     * Emits an OfferCreated event if successful.
     *
     * Reverts if:
     * - The offers region of protocol is paused
-    * - Caller is not an assistant
+    * - Caller is not an assistant or a buyer
+    * - sellerId is not 0 when offer is created by the buyer
     * - Valid from date is greater than valid until date
     * - Valid until date is not in the future
     * - Both voucher expiration date and voucher expiration period are defined
     * - Neither of voucher expiration date and voucher expiration period are defined
     * - Voucher redeemable period is fixed, but it ends before it starts
     * - Voucher redeemable period is fixed, but it ends before offer expires
     * - Dispute period is less than minimum dispute period
     * - Resolution period is not between the minimum and the maximum resolution period
     * - Voided is set to true
     * - Available quantity is set to zero
     * - Dispute resolver wallet is not registered, except for absolute zero offers with unspecified dispute resolver
     * - Dispute resolver is not active, except for absolute zero offers with unspecified dispute resolver
-    * - Seller is not on dispute resolver's seller allow list
+    * - Seller is not on dispute resolver's seller allow list if offer is created by the seller
     * - Dispute resolver does not accept fees in the exchange token
     * - Buyer cancel penalty is greater than price
-    * - Collection does not exist
+    * - Collection does not exist if offer is created by the seller
+    * - Collection id os different from 0 if offer is created by the buyer
     * - When agent id is non zero:
     *   - If Agent does not exist
     * - If the sum of agent fee amount and protocol fee amount is greater than the offer fee limit determined by the protocol
-    * - If the sum of agent fee amount and protocol fee amount is greater than fee limit set by seller
+    * - If the sum of agent fee amount and protocol fee amount is greater than fee limit set by caller
-    * - Royalty recipient is not on seller's allow list
-    * - Royalty percentage is less than the value decided by the admin
-    * - Total royalty percentage is more than max royalty percentage
+    * If the seller is creating an offer and:
+      * - Royalty recipient is not on seller's allow list
+      * - Royalty percentage is less than the value decided by the admin
+      * - Total royalty percentage is more than max royalty percentage
+    * If the buyer is creating an offer and roylties are set.
     *
     * @param _offer - the fully populated struct with offer id set to 0x0 and voided set to false
     * @param _offerDates - the fully populated offer dates struct
     * @param _offerDurations - the fully populated offer durations struct
     * @param _drParameters - the id of chosen dispute resolver (can be 0) and mutualizer address (0 for self-mutualization)
     * @param _agentId - the id of agent
     * @param _feeLimit - the maximum fee that seller is willing to pay per exchange (for static offers)
     */
    function createOffer(
        BosonTypes.Offer memory _offer,
        BosonTypes.OfferDates calldata _offerDates,
        BosonTypes.OfferDurations calldata _offerDurations,
        BosonTypes.DRParameters calldata _drParameters,
        uint256 _agentId,
        uint256 _feeLimit
    ) external;
```

#### IBosonFundsHandler
`depositFunds` must allow deposits to buyers.

```diff solidity
    /**
-    * @notice Receives funds from the caller, maps funds to the seller id and stores them so they can be used during the commitToOffer.
+    * @notice Receives funds from the caller, maps funds to the entity id and stores them so they can be used during the commitToOffer.
     *
     * Emits FundsDeposited event if successful.
     *
     * Reverts if:
     * - The funds region of protocol is paused
     * - Amount to deposit is zero
-    * - Seller id does not exist
+    * - Entity id does not exist
     * - It receives some native currency (e.g. ETH), but token address is not zero
     * - It receives some native currency (e.g. ETH), and the amount does not match msg.value
     * - It receives no native currency, but token address is zero
     * - Contract at token address does not support ERC20 function transferFrom
     * - Calling transferFrom on token fails for some reason (e.g. protocol is not approved to transfer)
     * - Received ERC20 token amount differs from the expected value
     *
-    * @param _sellerId - id of the seller that will be credited
+    * @param _entityId - id of the entity that will be credited
     * @param _tokenAddress - contract address of token that is being deposited (0 for native currency)
     * @param _amount - amount to be credited
     */
    function depositFunds(
-       uint256 _sellerId,
+       uint256 _entityId,
        address _tokenAddress,
        uint256 _amount
    ) external payable;
```   

#### IBosonExchangeEvents
A new event is introduced. It is emitted when the seller commits to a buyer-created offer.
````diff solidity
+    event SellerCommitted(
+        uint256 indexed offerId,
+        uint256 indexed sellerId,
+        uint256 indexed exchangeId,
+        BosonTypes.Exchange exchange,
+        BosonTypes.Voucher voucher,
+        address executedBy
+    );

````
Note: instead of having `SellerCommitted` and `BuyerCommitted` events, it could be merged into a single event containing both buyer and seller id.

#### IBosonExchangeHandler

The commit to offer method must allow the seller to call it and encumber the `sellerDeposit` at commitment time. The buyer's payment must already be in the escrow at commitment time.  

Note: `commitToConditionalOffer` does is not changed in this proposal, since conditional offers are meant to restrict the buyers, not the sellers. It makes no sense for a buyer to put an arbitrary restrictions in the offer.

```diff solidity
    /**
     * @notice Commits to a static offer (first step of an exchange).
     *
-    * Emits a BuyerCommitted event if successful.
+    * Emits a BuyerCommitted or SellerCommitted event if successful.
     * Issues a voucher to the buyer address.
     *
     * Reverts if:
     * - The exchanges region of protocol is paused
     * - The buyers region of protocol is paused
+    * - The sellers region of protocol is paused
     * - OfferId is invalid
     * - Offer has been voided
     * - Offer has expired
     * - Offer is not yet available for commits
     * - Offer's quantity available is zero
-    * - Buyer address is zero
-    * - Buyer account is inactive
+    * - Committer address is zero
+    * - Committer account is inactive
     * - Buyer is token-gated (conditional commit requirements not met or already used)
-    * - Offer price is in native token and caller does not send enough
-    * - Offer price is in some ERC20 token and caller also sends native currency
+    * - Offer exchange token is in native token and caller does not send enough
+    * - Offer exchange token is in some ERC20 token and caller also sends native currency
     * - Contract at token address does not support ERC20 function transferFrom
     * - Calling transferFrom on token fails for some reason (e.g. protocol is not approved to transfer)
     * - Received ERC20 token amount differs from the expected value
-    * - Seller has less funds available than sellerDeposit
+    * - Seller has less funds available than sellerDeposit if offer was created by the seller
+    * - Buyer has less funds available than price if offer was created by the buyer
     *
-    * @param _buyer - the buyer's address (caller can commit on behalf of a buyer)
+    * @param _committer - the seller's or the buyer's address. The caller can commit on behalf of a buyer or a seller.
     * @param _offerId - the id of the offer to commit to
     */
-   function commitToOffer(address payable _buyer, uint256 _offerId) external payable;
+   function commitToOffer(address payable _committer, uint256 _offerId) external payable;
```

#### IBosonPriceDiscoveryHandler
```diff solidity
    /**
     * @notice Commits to a price discovery offer (first step of an exchange).
     *
-    * Emits a BuyerCommitted event if successful.
+    * Emits a BuyerCommitted or SellerCommitted event if successful.
     * Issues a voucher to the buyer address.
     *
     * Reverts if:
     * - Offer price type is not price discovery. See BosonTypes.PriceType
     * - Price discovery contract address is zero
     * - Price discovery calldata is empty
     * - Exchange exists already
     * - Offer has been voided
     * - Offer has expired
     * - Offer is not yet available for commits
+    * - Committer address is zero
+    * - Committer account is inactive
     * - Buyer is token-gated (conditional commit requirements not met or already used)
     * - Any reason that PriceDiscoveryBase fulfilOrder reverts. See PriceDiscoveryBase.fulfilOrder
     * - Any reason that ExchangeHandler onPremintedVoucherTransfer reverts. See ExchangeHandler.onPremintedVoucherTransfer
     *
-    * @param _buyer - the buyer's address (caller can commit on behalf of a buyer)
+    * @param _committer - the seller's or the buyer's address. The caller can commit on behalf of a buyer or a seller.
     * @param _tokenIdOrOfferId - the id of the offer to commit to or the id of the voucher (if pre-minted)
     * @param _priceDiscovery - price discovery data (if applicable). See BosonTypes.PriceDiscovery
     */
    function commitToPriceDiscoveryOffer(
-       address payable _buyer,
+       address payable _committer,
        uint256 _tokenIdOrOfferId,
        BosonTypes.PriceDiscovery calldata _priceDiscovery
    ) external payable;
```

## Rationale
The offer creation phase in the protocol serves as an offer discovery mechanism. Parameters for a successful exchange should not depend on the offer creator (seller or buyer). For the protocol it is only relevant that the correct amount is encumbered at commitment time. Afterwards, the flows should be unaffected.

![Buyer Initiated Exchange Diagram](./assets/bpip-9/buyer-initiated-exchange.png "Buyer Initiated Exchange Diagram")

## Backward compatibility
This specification does not break backward compatibility. It should work for both static and price-discovery offers.

## Implementation
* Create offer
  * Store the information who created the offer. The existing offers should default to "seller"
* Commit to offer
  * Accept committer address, which can be either a buyer or a seller
  * If seller-created offer:
    * Check that seller has at least the `sellerDeposit` in available funds end encumber them
    * Validate that the buyer paid the `price`
  * If buyer-created offer:
    * Check that buyer has at least the `price` in available funds end encumber them
    * Validate that the seller paid the `sellerDeposit`
    * Set `sellerId` parameter of the offer
  * All existing validations in commit to offer should remain in place
  * If any validation fails, revert
* Deposit funds
  * Release the restriction that funds can only be deposited to the seller's available funds
  * All other existing validations should remain the same

### Security considerations

None.
  
## Copyright waiver & license
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
