---
bpip: 4
title:  Support price discovery 
discussions-to: 
status: Living
created: 2023-01-25
---

## Abstract

This proposal describe necessary changes on protocol in order to support price discovery. 

## Motivation

Currently protocol doens't supports price discovery because price is a immutable value set on offer creation.

## Specification 

### Options:

1. **Turn Boson Vouchers compatible.**<br>
    Boson Vouchers are ERC721 and therefore it's easier to make them compatible with price discovery features such as AMMs and Auctions mechanism. 
    NFT pools work just ERC20 tools. Pools that are willing to buy or sell NFTs will return the same price no matter which NFT is sent in or out from the collection. 
    That makes the current implementation of Boson Vouchers incompatible with NFT pools because Vouchers can represent different offers with different initial prices.
    To make it compatible with AMMs, we must turn Boson Vouchers collections individual to offers - instead of sellers, so the initial price for the entire vouchers in the pool is the same (the offer price). 

    **How do we communicate the price paid for the voucher to the protocol?**
    The price is a crutial info for the protocol as we still want that protocol guarantees remain unbroken, so when a sale happens outside the protocol (no matter how this sale happens, can be seller selling preminted
    vouchers on marketplaces, an auction, an AMM pool, a secundary sale using a settlement protocol, etc) we need inform protocol about the exchange price.

    In order to make this happens we would have to request both parties (buyer and seller) to confirm the exchange. 
    Just as proposed for [sequential commit](https://docs.google.com/document/d/1lb6ERrGtOg2iPjf17CfHJRuKWwPdkvQRsLQ3yV_0nh0/edit#heading=h.e4aw2z4amzsr),
    both seller and buyer signs the exchange and then somebody submit to the protocol.

    Open questions:
     - How imperment loss can influence seller misbehavior?
     - Seller has any incentive to not confirm the exchange?
     - Buyer has any incetive to not confirm the exchange?
     - Secondary saller needs to confirm the exchange? 
         
    **Pros**:
     - ✅ Seller can use pre-mint vouchers (Shell NFTs) to deposit into an AMM pool, e.g:
        1. Seller creates an offer with the price set to 10 MATIC and quantity available set to 100.
        2. Seller pre-mint 100 Vouchers from this new offer. Offer quantity available becomes 0. There is no commit available to be done directly in the protocol.
        3. Seller deposits these 100 vouchers into a Voucher/Matic pool. 
        4. Now the price for the vouchers will be discovered by the bonding curve selected for the pool.
     - ✅ Works with secondary market as well. Any voucher owner can deposit their voucher in the correspondent offer pool, or even create their own pool. 

    **Cons**:
     - ❌ Incompatible with existing Boson Vouchers. Would be a breaking change upgrade.

2. **Turn price an generic interface instead of a immutable value.**<br>
    Price is a generic interface that can be a immutable value set on offer creation or:<br>
      2.1 price discovery happens off-chain: A signed message with prove both parties are ok with price (same as proposed on option 1)<br>
      2.2 price discovery happens on-chain: whenever buyer buys it there, it needs to invoke our commitToOffer, or sequentialCommitToOffer together with price that was determined in some trusted way.<br>
      2.3 price discovery can happen both on and off chain: both parties need to sign a message (same as 2.1) 

    **Pros**:
      - ✅ No breaking changes. - need to confirm

    **Cons**:
      - ❌ Boson Vouchers remain incompatible with AMMs.

3. **Build custom price discovery mechanism inside the protocol**<br>
    We build or own price discovery solution

    **Pros**:
      - ✅ We can customize as we want

    **Cons**:
      - ❌ Incompatible with existing price discovery solution 

|                                          | compatible with existing<br>price discovery solutions | supports secondary market | development complexity |   
|------------------------------------------|-------------------------------------------------------|---------------------------|------------------------|
| **Opt1: turn boson vouchers compatible**    | ✅                                                   | ✅                       | BV breaking change     |   
| **Opt 2.1: generic interface + off-chain**   | ✅                                                   | ✅                       |                        |   
| **Opt 2.2: generic interface + on-chain**    | ❌                                                    | ✅                       |                        |   
| **Opt 2.3: generic interface + on/off-chain**| ✅                                                   | ✅                       |                        |
| **Opt 3: build custom solution on protocol** | ❌                                                    | ✅                           |                        |   


## Rationale
/

## Backward compatibility

/

## Implementation
/

## Copyright waiver & license
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
