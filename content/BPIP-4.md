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
    address validator
    bytes proof 
  }
```

The intention is that `validator` is an external price discovery contract and the proof is a function that the protocol will call on the validator contract before allowing a buyer to commit to an Offer. For example, the validator could be an AMM router, and the buyer would have to provide a proof to the validator, if the proof call succeeds, then `commitToOffer` would also succeed, otherwise, it would revert. . 
    
### Examples: 

#### Single Offer example <br>
*Seller wants to create an auction for an exclusive item.*

**Steps:**
  1. Seller creates an Offer in BP (with quantity = 1) [tx]
  2. Seller Funds the Seller Pool [tx]
  3. Seller preMints the offer voucher [tx] <br>
       *Note that we will adapt BV to not call commitToPreMintedOffer when transferring the voucher if offer has price discovery enabled (transfer will happen internally when a buyer calls commitToOffer)*
  4. Seller chooses an external auction protocol of their preference, External Protocol (EP)
  5. Seller creates an auction on the EP.<br>
  6. Seller gives permission for EP to transfer the voucher  
  7. buyer approves BP to transfer exchange token
  8. Buyer calls `commitToOffer` passing the bid price, the EP address as validator and the EP bid function encoded as the proof.<br>
      * BP call proof on EP contract (validator address)
      * Check if protocol received the BV, otherwise reverts (the bid failed)
      * BP transfers the received voucher to the buyer 
      * BP checks its own balance to see if the validator returned some funds beyond BV and forwards extra funds to the buyer
      * Transfer minimal required funds from the seller into the escrow

#### An example of a multi-instance Offer (i.e, quantity > 1). <br>
*Seller wants to use a bonding curve to let the market discover the price*

**Steps:**
  1. Seller creates the BP Offer (with quantity > 1) [tx]
  2.  Seller Funds the Seller Pool [tx]
  3. Seller preMint the vouchers [tx] <br>
     *Note that we will adapt BV to not call commitToPreMintedOffer when transferring the voucher if offer has price discovery enabled (transfer will happen internally when a buyer calls commitToOffer)*
  4. Seller chooses an external AMM protocol of their preference, External Protocol (EP)
  5. Seller creates a sell pool on the EP with the bonding curve of their preference
  6. Seller deposits BVs into the pool. 
  7. Buyer approves BP to transfer exchange token 
  8. Buyer calls `commitToOffer` passing the price (UI can simulate the price base on the chosen bonding curve), the EP address as validator and the EP buy function encoded as the proof.
    * BP calls proof on EP contract (validator address)
    * Check if BP received the BV, otherwise reverts (the bid failed)
    * BP transfers the received voucher to the buyer 
    * BP checks its own balance to see if the validator returned some funds beyond BV and forwards extra funds to the buyer
    * Transfer minimal required funds from the seller into the escrow <br>

#### Seller creates a multi-instance Offer (i.e, quantity > 1) and price discovery happens in an orderbook protocol (like Seaport for example)

This flow depends on how the validator works. If the validator always sends the voucher to `msg.sender` (Boson Protocol) a few additional steps are needed.  we will refer to this kind of validator as a "simple validator" (SV). If the validator allows order matching (or similar mechanism), between addresses that are not the `msg.sender` we will refer to them as an "advanced validator" (AV).

**Ask flow**
  1. Seller creates the BP Offer (with quantity > 1) [tx]
  2. Seller preMints the vouchers [tx] <br>
   *Note that we will adapt BV to not call commitToPreMintedOffer when transferring the voucher if the offer has price discovery enabled (transfer will happen internally when a buyer calls commitToOffer)*
  3. Seller Funds the Seller Pool [tx]
  4. Buyer approves [tx] <br>
      * protocol to transfer the funds if validator is SV
      * validator to transfer the funds if validator is AV
  5. buyer calls `commitToOffer`/`sequentialCommitToOffer, the protocol then`: [tx]  
      * if the validator is SV: (skip these steps otherwise) <br>
           * transfers buyers' funds to protocol 
           * approves validator to transfer protocol's funds 
      * stores information about the price in the protocol
      * calculates the amount to go in the escrow (full price in case of initial commit; and    minimal escrow in case of sequential commit)
      * makes a call to `validator` with `proof` as calldata. Continue only if it succeeds.
      * if validator is SV:
          * transfers voucher to buyer
      * if validator is AV:
         * makes sure buyer is the owner of the voucher, otherwise revert
         * checks BP balance to see if validator returned some funds (both with SV and AV)
         * transfer minimal required funds from the seller into the escrow

**Bid flow**
  1. Seller creates the BP Offer (with quantity > 1) [tx]
  2. Seller preMints the vouchers  [tx] <br>
  *Note that we will adapt BV to not call commitToPreMintedOffer when transferring the voucher if the offer has price discovery enabled (transfer will happen internally when a buyer calls commitToOffer)*
  3. Buyer creates a bid offer on validator 
     * with protocol as the seller, if validator is SV
     * normally, if validator is AV
  4. if validator is SV: 
     * seller approves protocol to transfer the voucher (this step could potentially be skipped since we could use the access controller contract to give the protocol privilege to transfer voucher without owner's approval [this is still up for discussion]) [tx]
  5. if validator is AV:
     * seller approves validator to transfer the voucher [tx]
  6. Seller funds the Seller Pool
  7. Seller (or somebody authorized) calls `commitToOffer`/`sequentialCommitToOffer` with the bid price, the protocol then: [tx] 
      * stores information about the price within the protocol
      * calculates amount to go in the escrow (full price in case of initial commit; and minimal escrow in case of sequential commit)
      * makes a call to the validator with proof as calldata. Continue only if it succeeds.
      * if validator is SV:
         * keep the minimum amount in the escrow and transfer the remainder to seller
      * if validator is AV:
          * make sure buyer is owner of the voucher, otherwise, revert
      * checks BP balance to see if validator returned some funds (both with SV and AV)
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
  validator: seaport address
  proof: encodeFunctionData("fulfillBasicOrder", basicOrderParameters)
}

commitToOffer(..., priceDiscovery)
```

### Notes
Note that 2-side AMM pools are out of the scope of this BPIP as they would either require us to make changes to the Boson Voucher, or they would require us to develop an intermediate token. Although this is interesting, it is out of the scope of this proposal. 

## Backward compatibility

This proposed improvement to the protocol is fully backward compatible, we would still support fixed price offers and maintain the existing interfaces

All of the methods described here would be compatible with the current Boson Vouchers. 

## Implementation

/

## Copyright waiver & license
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).



