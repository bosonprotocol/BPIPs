---
bpip: 4
title:  Support price discovery 
discussions-to: https://github.com/bosonprotocol/BPIPs/discussions/9
status: Draft 
created: 2023-01-25
---

## Abstract

This proposal describe necessary changes on protocol in order to support price discovery. 

## Motivation

Currently, BP doesn't support price discovery because the price is an immutable value set on offer creation.
There are some well-validated price discovery mechanisms on-chain and off-chain on the market, such as Automated Market Markers (AMMs), 
auction systems, order books. Making BP compatible with these mechanisms opens up an entirely new world of opportunity.
Allowing sellers to let the market discover the price for a product with AMMs, creating auctions for exclusive items, 
and what about a 24x7 order book of any physical item?

## Specification 

Turn price an generic interface instead of a immutable value.
Price is a generic interface that can be a uint256 immutable value set on offer creation or:

### Options:

1. **Price discovery happens only off-chain:** <br>
    A signed message passed on `commitToOffer`/`sequentialCommitToOffer` proves that both seller and buyer agree with the price. 
    `commitToOffer`/`sequentialCommitToOffer` accepts 2 new parameters, a uint256 `price` and a bytes `signature` using EIP712 
    format which needs to be signed by the seller (or the contrary when is a bid order) and that matches offerId and price passed 
    as parameter by the buyer.

    The amount of signatures would depend upon on the choosed price discovery solution.<br>
    Example of an simple offchain orderbook:
      - UX Ask:
        * Seller sign a EIP712 ask order with offer and price 
        * Only if exchange token is ERC20: Buyer approve exchangeToken transfer [tx]
        * Buyer calls `commitToOffer` passing price and seller signature [tx]
      - UX Bid: 
        * Buyer sign a EIP721 bid order with offer and price and submit it to orderbook 
        * Buyer approve exchangeToken transfer if ERC20 or buyer deposit the bid price into protocol [tx] <br>
          Must adapt `depositFunds` method to accept buyer funds as well 
        * Seller accepts the offer and commit to protocol [tx] <br>
          Must create a new commit to offer function which can be called by seller and accepts buyer signature

    The problem here is that signature must match the exactly price that the order was fullfilled.  
    This works for simple orders, but becomes a UX problem for limit, stop-limit and market orders.
    The same UX problem occurs for others price discovery system like auctions.
      
2. **Price discovery can happen both on-chain/off-chain and transaction start by calling protocol:**<br>
    `commitToOffer` and `sequentialCommitToOffer` has additional input field - an struct `priceDiscovery`.

    ```solidity
    struct PriceDiscovery {
      uint256 price
      address validator
      bytes encodedFunction 
    }
    ```
    Validator is an external contract responsible for checking if there is a consensus on the price between the seller and buyer.
    The protocol will call encodedFunction on the validator contract before commit to the offer. 
    For example, the validator could try to buy a Voucher on an AMM (this AMMs system would need to be compatible with BV, and existing 
    solutions aren't, see #notes section) with the price set by the buyer (UI can simulate the price for the buyer before trying to commit 
    and validator can allow slippage config) and then forward the voucher to the respected buyer if the transaction succeeds. 
    If the validator transaction reverts, then `commitToOffer` also reverts. Price could also be a parameter encoded directly on `encondedFunction`
    
    Validator could also check for off-chain consensus using the same signature approach proposed for option 1. 

3. **Price discovery happens on-chain and transaction start externally to protocol:**

|  | compatible with existing solutions | development complexity | UX |
|---|---|---|---|
| Opt 1:  | only off-chain solutions |  | (x sign + 2 tx) |
| Opt 2 | except with AMMs |  |  |

## Rationale
/

## Backward compatibility

/

## Implementation


/

## Copyright waiver & license
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
