---
bpip: 4
title:  Support price discovery 
discussions-to: https://github.com/bosonprotocol/BPIPs/discussions/9
status: Draft 
created: 2023-01-25
---

## Abstract

This proposal defines an Interface that extends the protocol to enable price discovery. The changes proposed will allow Sellers to pick their price discovery method of choice so that they can leverage external price discovery mechanisms. .

## Motivation

Currently, BP doesn't support price discovery. The price is currently represented as a uint256 within the Offer struct which is set by the Seller at Offer creation time. We see a need for Sellers to be able to use the protocol to leverage a price discovery system that will in turn discover the best price for their Offers. We believe this would be a key driver for adoption. There are several well-established price discovery mechanisms available both on-chain and off-chain on the market, such as Automated Market Makers (AMMs), auction systems, and order books. Making BP compatible with these mechanisms would open up an entirely new world of opportunity, such as allowing sellers to utilise AMMs for market-driven pricing, create auctions for exclusive items, and support 24/7 orderbooks for physical items.


## Specification 

With respect to Price Discovery and Boson Protocol Offers it is important to note the differences between single Offers (i.e. Offer with a quantity set to 1) and with Offers that have multiple instances (i.e. quantity > 1). We believe that these two classes of Offers are most likely going to employ different Price Discovery methods, e.g. Single offers will tend to be auctioned, whereas Multi-instance offers may employ a bonding curve for example. Furthermore, we believe that the Price Discovery Interface must address the needs of both of these two distinct Offer use cases. 

In the proposed changes, the price for items in an offer can either be established at the time of offer creation which applies to all items within the offer (maintaining compatibility with existing offers), or it can be dynamically determined upon buyer commitment.

The proposal is to add an additional input field in the form of a new struct `priceDiscovery` to `commitToOffer` and `sequentialCommitToOffer` 

  ```solidity
  struct PriceDiscovery {
    uint256 price
    address priceDiscoveryContract
    bytes priceDiscoveryData
  }
```

The intention is that `priceDiscoveryContract` is an external contract and the `priceDiscoveryData` is a function that the protocol will call on the `priceDiscoveryContract` before allowing a buyer to commit to an Offer. For example, the `priceDiscoveryContract`
could be an AMM router, and the buyer would have to provide a function call encoded as bytes (`priceDiscoveryData` parameter) which exists on the AMM router contract (`priceDiscoveryContract`). The protocol then calls this function internally, 
if the call succeeds, then `commitToOffer` would also succeed, otherwise, it would revert. . 

### Examples: 

#### Single Offer example <br>
*Seller wants to create an auction for an exclusive item.*

**Steps:**
  1. Seller creates an Offer in BP (with quantity = 1) [tx]
  2. Seller funds the Seller Pool [tx]
  3. Seller preMints the offer voucher [tx] <br>
       *Note that we will adapt BV to not call commitToPreMintedOffer when transferring the voucher if offer has price discovery enabled (transfer will happen internally when a buyer calls commitToOffer)*
  4. Seller chooses an external auction protocol of their preference, External Protocol (EP)
  5. Seller creates an auction on the EP.<br>
  6. Seller gives permission for EP to transfer the voucher  
  7. buyer approves BP to transfer exchange token
  8. Buyer calls `commitToOffer` passing the bid price, the EP address as priceDiscoveryContract and the EP bid function encoded as the priceDiscoveryData.<br>
      * BP call priceDiscoveryData on EP contract (priceDiscoveryContract address)
      * Check if protocol received the BV, otherwise reverts (the bid failed)
      * BP transfers the received voucher to the buyer 
      * BP checks its own balance to see if the priceDiscoveryContract returned some funds beyond BV and forwards extra funds to the buyer
      * Transfer minimal required funds from the seller into the escrow

#### An example of a multi-instance Offer (i.e, quantity > 1). <br>
*Seller wants to use a bonding curve to let the market discover the price*

**Steps:**
  1. Seller creates the BP Offer (with quantity > 1) [tx]
  2.  Seller funds the Seller Pool [tx]
  3. Seller preMint the vouchers [tx] <br>
     *Note that we will adapt BV to not call commitToPreMintedOffer when transferring the voucher if offer has price discovery enabled (transfer will happen internally when a buyer calls commitToOffer)*
  4. Seller chooses an external AMM protocol of their preference, External Protocol (EP)
  5. Seller creates a sell pool on the EP with the bonding curve of their preference
  6. Seller deposits BVs into the pool. 
  7. Buyer approves BP to transfer exchange token 
  8. Buyer calls `commitToOffer` passing the price (UI can simulate the price base on the chosen bonding curve), the EP address as priceDiscoveryContract and the EP buy function encoded as the priceDiscoveryData. <br>
      * BP calls priceDiscoveryData on EP contract (priceDiscoveryContract address)
      * Check if BP received the BV, otherwise reverts (the bid failed)
      * BP transfers the received voucher to the buyer 
      * BP checks its own balance to see if the priceDiscoveryContract returned some funds beyond BV and forwards extra funds to the buyer
      * Transfer minimal required funds from the seller into the escrow <br>

#### Seller creates a multi-instance Offer (i.e, quantity > 1) and price discovery happens in an orderbook protocol (like Seaport for example)

The flow of the protocol depends on how the `priceDiscoveryContract` works. If it always sends the voucher to msg.sender (Boson Protocol), additional steps are necessary. 
We refer to this type of orderbook as a "Fulfill Orderbook" (FO). If the `priceDiscoveryContract` allows order matching (or similar mechanism) between addresses that are not the msg.sender,
we call it an "Match Orderbook" (MO). Note that the idea of MO was inspired by Seaport's matching feature.

**Ask flow**
  1. Seller creates the BP Offer (with quantity > 1) [tx]
  2. Seller preMints the vouchers [tx] <br>
   *Note that we will adapt BV to not call commitToPreMintedOffer when transferring the voucher if the offer has price discovery enabled (transfer will happen internally when a buyer calls commitToOffer)*
  3. Seller funds the Seller Pool [tx]
  4. Buyer approves [tx] <br>
      * protocol to transfer the funds if `priceDiscoveryData` is FO
      * priceDiscoveryContract to transfer the funds if `priceDiscoveryData` is MO
  5. buyer calls `commitToOffer`/`sequentialCommitToOffer, the protocol then`: [tx]  
      * if the `priceDiscoveryData` is FO: (skip these steps otherwise) <br>
           * transfers buyers' funds to protocol 
           * approves `priceDiscoveryContract` to transfer protocol's funds 
      * stores information about the price in the protocol
      * calculates the amount to go in the escrow (full price in case of initial commit; and minimal escrow in case of sequential commit)
      * makes a call to `priceDiscoveryContract` with `priceDiscoveryData` as calldata. Continue only if it succeeds.
      * if `priceDiscoveryData` is FO:
          * transfers voucher to buyer
      * if `priceDiscoveryData` is MO:
         * makes sure buyer is the owner of the voucher, otherwise revert
         * checks BP balance to see if `priceDiscoveryContract` returned some funds (both with FO and MO)
         * transfer minimal required funds from the seller into the escrow

**Bid flow**
  1. Seller creates the BP Offer (with quantity > 1) [tx]
  2. Seller preMints the vouchers  [tx] <br>
  *Note that we will adapt BV to not call commitToPreMintedOffer when transferring the voucher if the offer has price discovery enabled (transfer will happen internally when a buyer calls commitToOffer)*
  3. Buyer creates a bid offer on `priceDiscoveryContract`
     * with protocol as the seller, if `priceDiscoveryData` is FO
     * normally, if `priceDiscoveryData` is MO
  4. if `priceDiscoveryData` is FO: 
     * seller approves protocol to transfer the voucher (this step could potentially be skipped since we could use the access controller contract to give the protocol privilege to transfer voucher without owner's approval [this is still up for discussion]) [tx]
  5. if `priceDiscoveryData` is MO:
     * seller approves `priceDiscoveryContract` to transfer the voucher [tx]
  6. Seller funds the Seller Pool
  7. Seller (or somebody authorized) calls `commitToOffer`/`sequentialCommitToOffer` with the bid price, the protocol then: [tx] 
      * stores information about the price within the protocol
      * calculates amount to go in the escrow (full price in case of initial commit; and minimal escrow in case of sequential commit)
      * makes a call to the `priceDiscoveryContract` with `priceDiscoveryData` as calldata. Continue only if it succeeds.
      * if `priceDiscoveryData` is FO:
         * keep the minimum amount in the escrow and transfer the remainder to seller
      * if `priceDiscoveryData` is MO:
          * make sure buyer is owner of the voucher, otherwise, revert
      * checks BP balance to see if `priceDiscoveryContract` returned some funds (both with FO and MO)
      * transfers minimal required funds from the seller into the escrow

Note that to make this bid flow work, we must allow third-parties the ability to call `commitToOffer` on behalf of the buyer.

## Rationale
The aforementioned price discovery techniques are widely accepted within the industry as current standards. Our interface has the capability to incorporate any external protocols that utilize these established techniques. As an example, we have described a minimal integration between Boson Protocol and Seaport.

1. Seller creates a BP Offer. Let's say the offer has quantity = 1000, token is USDC, and the price here is set to 0 (0 being a magic number, as the protocol must know this is not a free offer) because the price discovery will happen via Seaport.

  ```javascript
  offer {
    ...
    exchangeToken: USDC address
    quantityAvailable: 1000
    type: OfferType.PriceDiscovery
    price: 0
  }
  offerDates: {
    ...
    validFrom: 02/06/2023
    validUntil: 02/06/2024
  }
  ```
2. Seller preMints 1000 vouchers. 
3. Seller approves protocol to transfer USDC
4. Seller approves Seaport to transfer preMinted BVs
5. Seller creates an Order on Seaport with the following fields:

  ```javascript
    {
      offer:
        {
          itemType: 2 // ERC721
          token: BV address
          identifierOrCriteria: the voucher id // check if Seaport has a batch function
          startAmount: 1
          endAmount:1
        }
      consideration: {
        itemType: 1 // ERC20
        token: USDC address
        identifierOrCriteria: 0
        startAmount: 10
        endAmount: 100 // The initial price will be 10 and 100 is the price at the moment the 
                       // offer expires, realised amount is calculated linearly based on the time 
                       // elapsed since the order became active.
      }
      startTime: protocol offerDates.validFrom
      endtime: protocol offerDates.validUntil
      ...
    }
  ```
6. Buyer approves protocol to transfer USDC
7. Buyer calls `commitToOffer` which the following fields:

```javascript
const basicOrderParameters = {
 considerationToken: USDC,
 considerationAmount: startAmount * t,
 offerer: seller address,
 offerToken: BV,
 offerAmount: 1, 
 ...
}

const priceDiscovery = {
  price: startAmount * t
  priceDiscoveryContract: seaport address
  priceDiscoveryData: encodeFunctionData("fulfillBasicOrder", basicOrderParameters)
}

commitToOffer(..., priceDiscovery)
```

### Notes
Note that 2-side AMM pools are out of the scope of this BPIP as they would either require us to make changes to the Boson Voucher, or they would require us to develop an intermediate token. Although this is interesting, it is out of the scope of this proposal. 

## Backward compatibility

This proposed improvement to the protocol is fully backward compatible, we would still support fixed price offers and maintain the existing interfaces

All of the methods described here would be compatible with the current Boson Vouchers. 

## Implementation

### Security considerations

Given the nature of smart contracts we might want gate which price discovery contracts can be used

## Copyright waiver & license
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).



