---
bpip: 4
title:  Support price discovery 
discussions-to: https://github.com/bosonprotocol/BPIPs/discussions/9
status: Last Call 
created: 2023-01-25
---

## Abstract

This proposal defines an Interface that extends the protocol to enable price discovery. The changes proposed will allow Sellers to pick their price discovery method of choice so that they can leverage external Price Discovery mechanisms.

## Motivation

Currently, Boson Protocol (BP) doesn't support price discovery. The price is currently represented as a `uint256` within the Offer struct which is set by the Seller at Offer creation time. We see a need for Sellers to be able to use the protocol to leverage a price discovery system that will in turn discover the best price for their Offers. We believe this would be a key driver for adoption. There are several well-established price discovery mechanisms available both on-chain and off-chain on the market, such as Automated Market Makers (AMMs), auction systems, and order books. Making BP compatible with these mechanisms would open up an entirely new world of opportunity, such as allowing sellers to utilise AMMs for market-driven pricing, create auctions for exclusive items, and support 24/7 orderbooks for physical items.


## Specification 

With respect to Price Discovery and Boson Protocol Offers it is important to note the differences between single Offers (i.e. Offer with a quantity set to 1) and with Offers that have multiple instances (i.e. quantity > 1). We believe that these two classes of Offers are most likely going to employ different Price Discovery methods, e.g. Single offers will tend to be auctioned, whereas Multi-instance offers may employ a bonding curve for example. Furthermore, we believe that the Price Discovery Interface must address the needs of both of these two distinct Offer use cases. 

In the proposed changes, the price for items in an offer can either be established at the time of offer creation which applies to all items within the offer (maintaining compatibility with existing offers), or it can be dynamically determined upon buyer commitment.

The proposal is to introduce a new struct `priceDiscovery`:

```solidity
   struct PriceDiscovery {
        uint256 price;
        Side side;
        address priceDiscoveryContract;
        address conduit;
        bytes priceDiscoveryData;
    }
```
The struct contains all the information to execute a call on an external price discovery contract and to infer the price for which the item was sold. This struct is used as an input parameter in the following methods:
- `IBosonPriceDiscoveryHandler.commitToPriceDiscoveryOffer` defined later in this proposal.
- `IBosonSequentialCommitHandler.sequentialCommitToOffer` defined in [BPIP-7](BPIP-7.md)

Struct fields represent:
- *price*: the expected price of an item, denominated in the exchange token as defined for the corresponding offer in Boson Protocol. The actual price executed on the external price discovery can be different. Depending on the outcome, protocol either returns the surplus or reverts the transaction.
- *side*: defines who is fulfilling the order. Can be
  - *Ask*: a buyer (or someone on their behalf) is fulfilling the order
  - *Bid*: the seller is fulfilling the order
  - *Wrapper*: the order was already fulfilled using an external wrapper contract and the must only be finalized in the protocol. Anyone can call it.
- *priceDiscoveryContract*: external price discovery contract (e.g. Seaport)
- *conduit*: external contract that is approved to transfer funds and/or the Boson voucher
- *priceDiscoveryData*: instructions forwarded to `priceDiscoveryContract`, which should execute the price discovery mechanism.

### Boson Price Discovery Handler
We introduce a new handler for new price discovery methods.

```solidity
/**
 * @title IBosonPriceDiscoveryHandler
 *
 * @notice Handles exchanges associated with offers within the protocol.
 *
 * The ERC-165 identifier for this interface is: 0xdec319c9
 */
interface IBosonPriceDiscoveryHandler {
    /**
     * @notice Commits to a price discovery offer (first step of an exchange).
     *
     * Emits a BuyerCommitted event if successful.
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
     * - Buyer address is zero
     * - Buyer account is inactive
     * - Buyer is token-gated (conditional commit requirements not met or already used)
     * - Any reason that PriceDiscoveryBase fulfilOrder reverts. See PriceDiscoveryBase.fulfilOrder
     * - Any reason that ExchangeHandler onPremintedVoucherTransfer reverts. See ExchangeHandler.onPremintedVoucherTransfer
     *
     * @param _buyer - the buyer's address (caller can commit on behalf of a buyer)
     * @param _tokenIdOrOfferId - the id of the offer to commit to or the id of the voucher (if pre-minted)
     * @param _priceDiscovery - price discovery data (if applicable). See BosonTypes.PriceDiscovery
     */
    function commitToPriceDiscoveryOffer(
        address payable _buyer,
        uint256 _tokenIdOrOfferId,
        BosonTypes.PriceDiscovery calldata _priceDiscovery
    ) external payable;
}

```

### Boson Price Discovery Client
Since the proposed mechanism includes unrestricted external calls, we want to decouple it from the Boson Protocol for the following security reasons:
- Allowing unrestricted calls directly from the protocol could result in asset theft.
- The malicious actor could call other contracts which would see the Boson Protocol as the `msg.sender`, so it would look like the Boson Protocol is making those actions. This could negatively impact the Boson Protocol reputation.

To reduce the negative impact, we move the part of price discovery logic to an external client. This client does not hold any funds, so that attack vector is removed. The malicious actors could still make bad calls on the client's behalf, but since it's a separate contract, it has no negative impact on the Boson Protocol itself.

Boson Price Discovery Client is a stateless contract, so it's easy to replace it at any given time if necessary. It implements the method to handle different price discovery flows (Ask, Bid and Wrapper) and all method must be restricted, so only Boson Protocol can call them.

```solidity
/**
 * @title BosonPriceDiscovery
 *
 * @notice This is the interface for the Boson Price Discovery contract.
 *
 * The ERC-165 identifier for this interface is: 0x8bcce417
 */
interface IBosonPriceDiscovery is IERC721Receiver {
    /**
     * @notice Fulfils an ask order on external contract.
     *
     * Reverts if:
     * - Call to price discovery contract fails
     * - The implied price is negative
     * - Any external calls to erc20 contract fail
     *
     * @param _exchangeToken - the address of the exchange contract
     * @param _priceDiscovery - the fully populated BosonTypes.PriceDiscovery struct
     * @param _bosonVoucher - the boson voucher contract
     * @param _msgSender - the address of the caller, as seen in boson protocol
     * @return actualPrice - the actual price of the order
     */
    function fulfilAskOrder(
        address _exchangeToken,
        BosonTypes.PriceDiscovery calldata _priceDiscovery,
        IBosonVoucher _bosonVoucher,
        address payable _msgSender
    ) external returns (uint256 actualPrice);

    /**
     * @notice Fulfils a bid order on external contract.
     *
     * Reverts if:
     * - Call to price discovery contract fails
     * - Protocol balance change after price discovery call is lower than the expected price
     * - This contract is still owner of the voucher
     * - Token id sent to buyer and token id set by the caller don't match
     * - The implied price is negative
     * - Any external calls to erc20 contract fail
     *
     * @param _tokenId - the id of the token
     * @param _exchangeToken - the address of the exchange token
     * @param _priceDiscovery - the fully populated BosonTypes.PriceDiscovery struct
     * @param _seller - the seller's address
     * @param _bosonVoucher - the boson voucher contract
     * @return actualPrice - the actual price of the order
     */
    function fulfilBidOrder(
        uint256 _tokenId,
        address _exchangeToken,
        BosonTypes.PriceDiscovery calldata _priceDiscovery,
        address _seller,
        IBosonVoucher _bosonVoucher
    ) external payable returns (uint256 actualPrice);

    /**
     * @notice Call `unwrap` (or equivalent) function on the price discovery contract.
     *
     * Reverts if:
     * - Protocol balance doesn't increase by the expected amount.
     * - Token id sent to buyer and token id set by the caller don't match
     * - The wrapper contract sends back the native currency
     * - The implied price is negative
     *
     * @param _exchangeToken - the address of the exchange contract
     * @param _priceDiscovery - the fully populated BosonTypes.PriceDiscovery struct
     * @return actualPrice - the actual price of the order
     */
    function handleWrapper(
        address _exchangeToken,
        BosonTypes.PriceDiscovery calldata _priceDiscovery
    ) external payable returns (uint256 actualPrice);
}
```

## Examples: 

In all examples, the seller must first create an offer and premint the desired number of vouchers. These can be then plugged into a price discovery mechanism of seller's choice.

1. A seller creates an Offer in Boson Protocol (BP)
2. The seller funds the Seller Pool
3. The seller preMints the offer voucher
4. The seller chooses an external auction protocol of their preference, External Protocol (EP)

#### Ask flow
Example: *The seller puts the vouchers in AMM to use a bonding curve*.

**Steps:**

  5. The seller transfers the preminted vouchers into AMM and chooses the bonding curve.
  6. A buyer approves BP to transfer the exchange tokens.
  7. The seller approves BP to transfer the exchange tokens. This is needed to encumber the funds at the end. Even if the seller does not have them before the order is fulfilled, they can still approve the transfer - the commit should still succeed, since the seller is expected to receive them during the order fulfillment.
  8. The buyer calls `commitToPriceDiscoveryOffer` with the price and other price discovery data.
     1. BP validates the incoming payment and forwards it to the price discovery client.
     2. BP forwards the instructions to Boson Price Discovery Client which:
        1. Tracks the native token balance and exchange token balance.
        2. Makes a call to `priceDiscoveryContract` with `priceDiscoveryData`.
        3. Calculates the actual price.
           1. If the price is negative, it reverts.
           2. If the actual price is less than the price supplied by the buyer, the surplus is returned to the caller.
        4. Assumes it received the Boson Voucher and approves BP to transfer it.
        5. Returns the actual price to BP.
     3. BP verifies that the correct voucher was transferred and that Boson Price Discovery owns it.
     4. BP transfers the full price from the seller and puts it into escrow.
     5. If the exchange token is native, the protocol unwraps it.


#### Bid flow
Example: *The seller wants to find the price through off-chain auction*.

  5. The seller creates an auction.
  6. A buyer approves the EP to transfer the exchange tokens.
  7. The seller approves BP to transfer the voucher  
  8. The seller calls `commitToPriceDiscoveryOffer` with the price and other price discovery data.
     1. BP transfers the voucher to itself
     2. BP forwards the instructions to Boson Price Discovery Client which:
        1. Tracks the native token balance and exchange token balance.
        2. Makes a call to `priceDiscoveryContract` with `priceDiscoveryData`.
        3. Calculates the actual price.
           1. If the price is negative, it reverts.
           2. If the actual price is less than the price supplied by the seller it reverts.
        4. Transfers the received funds to BP.
     3. BP puts the funds into escrow (no need for additional transfer, since all funds are in the protocol).
     4. If the exchange token is native, the protocol unwraps it.

#### Wrapper
Example: *The seller wants to find the price through on-chain auction, which requires them to deposit the voucher*.

**Steps:**

  5. Seller transfers the voucher to the wrapper contract, which issues a wrapped voucher. The wrapper becomes the owner of the true voucher.
  6. The seller uses the wrapper to put the voucher into the auction, effectively making the wrapper contracts the seller.
  7. Buyers place bids.
  8. Auction is finalized and the proceeds are transferred to the wrapper, while the wrapped voucher is transferred to the auction winner.
  9. Either the seller or the buyer calls `commitToPriceDiscoveryOffer` with the price and other price discovery data. In this case, the `priceDiscoveryContract` is the wrapper contract and the `priceDiscoveryData` is the data that unwraps the voucher (i.e. transfer the true voucher to the buyer) and sends the funds to the protocol.
     1. BP forwards the instructions to Boson Price Discovery Client which:
        1. Tracks the native token balance and exchange token balance.
        2. Makes a call to `priceDiscoveryContract` with `priceDiscoveryData`.
        3. Calculates the actual price.
           1. If the price does not match the actual price, it revert.
        4. Transfers the received funds to BP.
     3. BP puts the funds into escrow (no need for additional transfer, since all funds are in the protocol).

### Rationale

Supporting the ask and bid side is a standard in virtually all existing price discovery mechanisms, so it was natural for us to support it as well. The only challenge was to get the correct price information since the existing price mechanisms do not provide it. If the actors used them directly, Boson Protocol would have no knowledge of the price and could not encumber the appropriate amount in the escrow.

To solve this, we require that the final step of price discovery is executed via Boson Protocol, since this allows us to track the transfer of the funds and vouchers. Boson protocol is able to infer the actual price and encumber the funds, which can then be used as in normal operations.

Another challenge was discovered since some price discovery mechanisms allow the finalization of the exchange in a way that circumvents the protocol, which could be bad for both the seller and the buyer. To solve this, we introduce the concept of wrappers. A wrapper contract accepts the true voucher and issues a wrapped voucher to the seller. The wrapped voucher is then put into the price discovery mechanism. The main difference is only that the wrapper contract itself is posing as the seller, meaning that all proceeds will eventually come to it. This then allows full control over the asset exchange and gives the guarantee that the funds will come to the Boson Protocol at the end. We do not define the wrapper as part of this BPIP, since different use cases might require different wrappers.

### Implementation

The current implementation is available in [v2.4.0 release candidate](https://github.com/bosonprotocol/boson-protocol-contracts/releases/tag/v2.4.0-rc.3).

### Notes
Note that 2-side AMM pools are out of the scope of this BPIP as they would either require us to make changes to the Boson Voucher, or they would require us to develop an intermediate token. Although this is interesting, it is out of the scope of this proposal. 

## Backward compatibility

This proposed improvement to the protocol is fully backward compatible, we would still support fixed-price offers and maintain the existing interfaces

All of the methods described here would be compatible with the current Boson Vouchers. 

### Security considerations

Security considerations were originally described for [BPIP-7](BPIP-7.md), but they apply to general price discovery as well. They are available in [this document](./assets/../assets/bpip-7/Sequential%20Commit.pdf).

## Copyright waiver & license
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).



