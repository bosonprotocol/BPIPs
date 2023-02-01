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

### Notes

Usually, AMMs pools are design to return the same price no matter which NFT is sent in or out from the collection. 
Thats make currently implementation of Boson Vouchers incompatible with existing NFT AMMs because Vouchers can represent different offers with different initial prices.

### Options:

1. **transaction start by calling protocol**<br>

    1.1 **off-chain with in protocol validation:** <br>
      A signed message passed on `commitToOffer`/`sequentialCommitToOffer` proves that both seller and buyer agree with the price. 
      `commitToOffer`/`sequentialCommitToOffer` accepts 2 new parameters, a uint256 `price` and a bytes `signature` using EIP712 
      format which needs to be signed by the seller (or the contrary when is a bid order) and that matches offerId and price passed 
      as parameter by the buyer.

      Example of an simple offchain orderbook:
        - UX Ask:
          * Seller sign a EIP712 ask order with offer and price 
          * Only if exchange token is ERC20: Buyer approve exchangeToken transfer [tx]
          * Buyer calls `commitToOffer` passing price and seller signature [tx]
        - UX Bid: 
          * Buyer sign a EIP712 bid order with offer and price and submit it to orderbook 
          * Buyer approve exchangeToken transfer if ERC20 or approves WETH if is native token [tx] <br>
          * Seller accepts the offer and commit to protocol [tx] <br>


      Must create a new commit to offer function (or adapt existing commitToOffer) which can be called by seller and accepts buyer signature

    - **Pros:** 
        - simpler 
    - **Cons:**
        - works only for off-chain price discovery
        - the EIP712 message structure will have to be dinamically enough to be compatible with different existing price discovery solutions, or we would have to 
          design something completely new and propose a new interface for price discovery communication - and consequently not compatible with existing solutions.

          
    1.2 **Price discovery happens both on-chain/off-chain**<br>
      `commitToOffer` and `sequentialCommitToOffer` has additional input field - an struct `priceDiscovery`.

      ```solidity
      struct PriceDiscovery {
        uint256 price
        address validator
        bytes proof 
      }
      ```

    Validator is an external price discovery contract and proof is a function that the protocol will call on the validator contract before commit to the offer. 
    For example, the validator could be an AMM router (this AMMs system would need to be compatible with BV, and existing 
    solutions aren't, see notes section) and try to buy a voucher with the config passed by buyer on proof function
    (UI can simulate the price for the buyer before trying to commit and validator can allow slippage config) and weither send the voucher directly for buyer or 
    send it to protocol (msg.sender) and then protocol forward to buyer. If the proof call reverts, then `commitToOffer` also reverts. 
    
    Flow depends on how validator works. If validator always sends voucher to `msg.sender` (Boson Protocol) a few additional steps are needed. 
    Let's call this validator "simple". If validator allows order matching (or some similar mechanism) it means that voucher and exchange token can be exchanged 
    directly even if `msg.sender` is none of them. Let's call this kind of validator "advanced"
  
    - Ask flow: 
      - seller approves protocol to transfer exchange tokens. If offer is native, they must approve WETH. This will be needed in the last step. [tx]
      - buyer approves [tx]
          * protocol to transfer the funds, if validator is simple
          * validator to transfer the funds, if validator is advanced
      - buyer calls `commitToOffer`/`sequentialCommitToOffer`: [tx]  
        - if validator is simple: (skip these steps otherwise)
           * transfer buyers funds to protocol 
           * approve validator to transfer protocol's funds 
        - store information about the price in the protocol
        - calculate amount to go in the escrow (full price in case of initial commit; and minimal escrow in case of sequential commit)
        - make a call to `validator` with `proof` as calldata. Continue only if it succeeds.
        - if validator is simple:
            * transfer voucher to buyer
        - if validator is advanced:
            * make sure buyer is owner of the voucher, otherwise revert
        - check own balance to see if validator returned some funds (possible both with simple and advanced validators)
        - Transfer minimal required funds from seller into the escrow

    - Bid flow:
      - buyer create a bid offer on validator 
        * with protocol as the seller, if validator is simple
        * normally, if validator is avanced
      - if validator is simple:
        * seller approves protocol to transfer the voucher (potentially this can be skipped, since we could use access controller contract to give protocol privilege to transfer voucher without owner's approval) [tx]
      - is validator is advanced: 
        * seller approves protocol to transfer exchange tokens. If offer is native, they must approve WETH. This will be needed in the last step. [tx]
        * seller approves validator to transfer the voucher
      - Seller (or somebody authorized) calls `commitToOffer`/`sequentialCommitToOffer` with the bid price: [tx] 
        * store information about the price in the protocol
        * calculate amount to go in the escrow (full price in case of initial commit; and minimal escrow in case of sequential commit)
        * make a call to validator with proof as calldata. Continue only if it succeeds.
        * if validator is simple:
          - keep minimum amount in the escrow and transfer the remainder to seller
        * if validator is advanced:
          - make sure buyer is owner of the voucher, otherwise revert
          - check own balance to see if validator returned some funds (possible both with simple and advanced validators)
          - transfer minimal required funds from seller into the escrow

    Must create a new commit to offer function (or adapt existing commitToOffer) which can be called by seller or by some allowed party.

    - **Pros:**
      - it enables to use existing protocols such as 0x, SeaPort so it could support whatever price discovery mechanisms they support
      - if later new price discovery mechanism emerge, it should be ready without modification<br>
    - **Cons:**
      - incompatible with existing AMMs solutions

    1.3. **Price discovery happens both on-chain/off-chain with in protocol validation or off-chain validation:**<br>
      Combinantion of option 1 and option 2. `commitToOffer` and `sequentialCommitToOffer` has additional input field - an struct `priceDiscovery`.

      ```solidity
        struct PriceDiscovery {
          uint256 price
          address validator // zero address or protocol address when type is Offchain
          bytes proof // proof is validator calldata (type Onchain) or an EIP712 signature (type Offchain)
          PriceDiscoveryType type
        }
      ```

    1.4 **Price discovery happens both on-chain/off-chain and we make BosonVoucher compatible with AMMs**

      As I’ve explained in the notes, existing NFT AMMs aren’t compatible with Boson Vouchers. 
      To make it compatible with AMMs we must turn Boson Vouchers collections individual to offers - instead of sellers,
      so the initial price for the entire vouchers in the pool is the same (the offer price).
      This option is a combination necessary changes to Boson Vouchers and option 1.2.

      - **Pros**:
        - Support existing AMMs
      - **Cons**:
        - breaking change for existing BVs

2. **Price discovery happens on-chain and transaction start externally to protocol:**

    `commitToOffer`/`sequentialCommitToOffer` accepts a new parameter `price`. Externally price discovery solutions must 
    call protocol `commitToOffer`/`sequentialCommitToOffer` with the price.
    Protocol must check if the buyer received the voucher and the seller receives the exchange tokens (reverts otherwise).
    
    The UX depends on the PD solution mechanism, as the protocol call happens internally. 

  - **Pros**:
    * no need to start the transaction on protocol
    * no development complexity on protocol side
  - **Cons**:
    * incompatible with all existing PD solutions, must create new solutions compatible with protocol


### Options comparation

|  | compatible with existing solutions | development complexity | UX |
|---|---|---|---|
| Opt 1.1  | only off-chain solutions | 🟢 | 1 sign + 2 tx |
| Opt 1.2 | except existing AMMs | 🟡 | 3 tx |
| Opt 1.3 | except existing AMMs | 🟡 | 3 tx or 1 sign + 2 tx |
| Opt 1.4 | compatible | 🔴 | 3 tx
| Opt 2 | incompatible | 🟢 | depends on the PD solution design

## Rationale
/

## Backward compatibility

/

## Implementation


/

## Copyright waiver & license
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
