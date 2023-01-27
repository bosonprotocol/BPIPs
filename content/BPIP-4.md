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

1. **Turn Boson Vouchers compatible.** 
    Boson Vouchers are ERC721 and therefore it's easier to make them compatible with price discovery features such as AMMs and Auctions mechanism. 
    NFT pools work just ERC20 tools. Pools that are willing to buy or sell NFTs will return the same price no matter which NFT is sent in or out from the collection. 
    That makes the current implementation of Boson Vouchers incompatible with NFT pools because Vouchers can represent different offers with different initial prices.
    To make it compatible with AMMs, we must turn Boson Vouchers collections individual to offers - instead of sellers, so the initial price for the entire vouchers in the pool is the same (the offer price). 

    **How do we communicate the price paid for the voucher to the protocol?**
    The price is a crutial info for the protocol as we still want that protocol guarantees remain unbroken, so when a sale happens outside the protocol (no matter how this sale happens, can be seller selling preminted
    vouchers on marketplaces, an auction, an AMM pool, a secundary sale using a settlement protocol, etc) we need somehow inform protocol about the order price.

    In order to make this happens we would have to request both parties (buyer and seller) to confirm the exchange. 
    Just as proposed for [sequential commit](https://docs.google.com/document/d/1lb6ERrGtOg2iPjf17CfHJRuKWwPdkvQRsLQ3yV_0nh0/edit#heading=h.e4aw2z4amzsr),
    both seller and buyer signs the exchange and then somebody submit to the protocol.

    Open questions:
     - How imperment loss can influence seller misbehavior?
     - Seller has any advantage in deciding not confirm the exchange?
         
    **Pros**:
     - ✅ Seller can pre-mint vouchers (Shell NFTs) to deposit into an AMM pool, e.g:
        1. Seller creates an offer with the price set to 10 MATIC and quantity available set to 100.
        2. Seller pre-mint 100 Vouchers from this new offer. Offer quantity available becomes 0. There is no commit available to be done directly in the protocol.
        3. Seller deposits these 100 vouchers into a Voucher/Matic pool. 
        4. Now the price for the vouchers will be discovered by the bonding curve selected for the pool.
     - ✅ Works with secondary market as well. Any voucher owner can deposit their voucher in the correspondent offer pool, or even create their own pool. 
     - ✅ Easy to implement. We can reuse Boson Vouchers instead of creating a hole new thing.

    **Cons**:
     - ❌ Incompatible with existing Boson Vouchers. Would be a breaking change upgrade.

2. **Turn price an generic interface instead of a immutable value.**
    Price is a generic interface that can be:
      - An immutable value (as currently works)
      - An ERC721 token, e.g:
        1. Seller creates an Offer O and the price is any token from an ERC721 collection C.
        2. Buyers needs to acquire a token from this collection C to be able to commit to Offer 0.
           Buyer B acquired this "gate" (need to find a better name to avoid confusion with gated offers) token from a AMMs pool, an auction, any marketplace, etc.
        3. Buyer B now has a token X and can commit to Offer 0.
        4. Buyer B commits to offer O and transfer token X to protocol. 

    **Pros**:
      - ✅ No breaking changes. - need to confirm

    **Cons**:
      - ❌ Doesn't fully support secondary market. Boson Vouchers remain incompatible with AMMs and therefore secondary market can only make use of the auction mechanism.

## Rationale
/

## Backward compatibility

/

## Implementation
/

## Copyright waiver & license
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
