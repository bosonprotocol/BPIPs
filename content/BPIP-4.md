---
bpip: 4
title: Support price discovery
discussions-to: https://github.com/bosonprotocol/BPIPs/discussions/9
status: Draft
created: 2023-01-25
---

## Abstract

This proposal defines an Interface that extends the protocol to enable price discovery. The changes proposed will allow Sellers to pick their price discovery method of choice so that they can leverage external Price Discovery mechanisms.

## Motivation

Currently, Boson Protocol (BP) doesn't support price discovery. The price is currently represented as a `uint256` within the Offer struct which is set by the Seller at Offer creation time. We see a need for Sellers to be able to use the protocol to leverage a price discovery system that will in turn discover the best price for their Offers. We believe this would be a key driver for adoption. There are several well-established price discovery mechanisms available both on-chain and off-chain on the market, such as Automated Market Makers (AMMs), auction systems, and order books. Making BP compatible with these mechanisms would open up an entirely new world of opportunity, such as allowing sellers to utilise AMMs for market-driven pricing, create auctions for exclusive items, and support 24/7 orderbooks for physical items.

## Specification

With respect to Price Discovery and Boson Protocol Offers it is important to note the differences between single Offers (i.e. Offer with a quantity set to 1) and with Offers that have multiple instances (i.e. quantity > 1). We believe that these two classes of Offers are most likely going to employ different Price Discovery methods, e.g. Single offers will tend to be auctioned, whereas Multi-instance offers may employ a bonding curve for example. Furthermore, we believe that the Price Discovery Interface must address the needs of both of these two distinct Offer use cases.

In the proposed changes, the price for items in an offer can either be established at the time of offer creation which applies to all items within the offer (maintaining compatibility with existing offers), or it can be dynamically determined upon buyer commitment.

The proposal is to create an function `commitToPriceDiscoveryOffer` with an extra field `priceDiscovery`

```
enum Side {
    Ask,
    Bid
}

struct PriceDiscovery {
  uint256 price
  address priceDiscoveryContract
  bytes priceDiscoveryData
  side Side
}

```

- `priceDiscoveryContract` This is an external contract capable of performing price discovery for ERC721 tokens.
- `priceDiscoveryData` This is a bytes-encoded function which BP will call on the `priceDiscoveryContract` before allowing a buyer to commit to an offer. For example, the caller can provide Seaport address in `priceDiscoveryContract` field and a call to the `fulfillAdvancedOrder` function, encoded as bytes in `priceDiscoveryData`. BP then calls this function internally and acts as an intermediary.

BP acts as the seller when the side is Bid and as the buyer when the side is Ask. However, this is not the case when the price discovery mechanism requires pre-holding the NFT before the trade (see Wrappers in implementation section).

If the call to `priceDiscoveryContract` is successful, the protocol checks its balance to ensure that PD has sent the expected price set in price. For Ask side, the caller must provide the maximum acceptable price, while for Bid side, the caller must provide the minimum acceptable price.

### Examples:

### Single Offer example

A seller wants to auction off an exclusive item. The following steps describe the process:

Seller Flow:

1. Creates an offer in BP (with a quantity of 1).
2. Funds the seller pool with a deposit.
3. Pre-mints the offer voucher. Note that the function `commitToPreMintedOffer` will be deprecated in favor of a new function, `onPremintedVoucherTransferred`, which will properly handle transfers based on the price type. See the implementation section for more details.
4. Chooses an external auction protocol of their preference, called External Protocol (EP).
5. Creates an auction on the EP. Note that the EP must allow the seller to define receiver of the funds after the auction ends since protocol must be the receiver to allow buyer protection.
6. Gives permission for the EP to transfer the voucher.

Buyer flow:

1. Create a bid on the auction for EP.
2. After the auction's valid period has elapsed, anyone can call `commitToPriceDiscoveryOffer`, passing the winner's bid price, the EP address as the `priceDiscoveryContract`, and EP's finalize auction function encoded as `priceDiscoveryData`. The following actions will occur:
   - BP calls `priceDiscoveryData` on `priceDiscoveryContract`
   - BP checks if the protocol balance has increased by at least the expected price.
   - BP checks if the buyer passed on the `commitToPremintedOffer` call is the new owner of the voucher.
   - BP transfers the minimal required funds from the seller into the escrow.

### Multi-instance Offer (i.e, quantity > 1)

The flow of the exchange depends on how the PD contract works. If PD requires custody of the NFTs, BP cannot act as an intermediary in the exchange. To protect buyers, sellers must use an intermediate contract, which we call Wrappers.

If Boson Protocol can act as the intermediary between the trade, meaning it can take custody of the seller's voucher before calling the PD protocol when on the bid side, and take custody of the buyer's funds when on the ask side, we can use the same flow as single offer on the bid side, and a similar flow on the ask side, which we describe below.

Seller flow:

1. Creates an offer in BP (with quantity = 1).
2. Funds the seller pool with seller deposit.
3. Pre-mints the offer voucher. Note that the function `commitToPreMintedOffer` will be deprecated in favor of a new function, `onPremintedVoucherTransferred`, which will properly handle transfers based on price type. See implementation for more details.
4. Chooses an external auction protocol of their preference, External Protocol (EP).
5. Creates an offer in EP

Buyer flow:

1. Approves protocol to transfer offer exchange token when is not native token
2. Call `commitToPriceDiscoveryOffer`/`sequentialCommitToOffer`, protocol then:
   - transfers buyers' funds to protocol
   - approves `priceDiscoveryContract` to transfer protocol's funds (exchange token)
   - makes a call to `priceDiscoveryContract` with `priceDiscoveryData` as calldata. Continue only if it succeeds.
   - check protocol balance before and after the call to `priceDiscoveryContract` to see if it has decrease by at max the price set by caller
   - calculates the buyer amount to go in the escrow (full price in case of initial commit; and minimal escrow in case of sequential commit)
   - transfers voucher to buyer
   - transfer minimal required funds from the seller into the escrow

## Rationale

The aforementioned price discovery techniques are widely accepted within the industry as current standards. Our interface has the capability to incorporate any external protocols that utilize these established techniques. As an example, we have described a minimal integration between Boson Protocol and Seaport.

1. Seller creates a BP Offer. Let's say the offer has quantity = 1000, token is USDC, and the price here is set to 0 (0 being a magic number, as the protocol must know this is not a free offer) because the price discovery will happen via Seaport.

```
offer {
  ...
  exchangeToken: USDC address
  quantityAvailable: 1000
  type: PriceType.Discovery
  price: 0
}

```

1. Seller preMints 1000 vouchers.
2. Seller approves protocol to transfer USDC
3. Seller approves Seaport to transfer preMinted BVs
4. Seller creates an order on Seaport with the following fields:

```
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

1. Buyer approves protocol to transfer USDC
2. Buyer calls `commitToPriceDiscoveryOffer` which the following fields:

```
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

Note that 2-side and token AMM pools are out of the scope of this BPIP as they would either require us to make changes to the Boson Voucher, or they would require us to develop an intermediate token. Although this is interesting, it is out of the scope of this proposal.

## Backward compatibility

This proposed improvement to the protocol is fully backward compatible, we would still support fixed price offers and maintain the existing interfaces

All of the methods described here would be compatible with the current Boson Vouchers.

## Implementation

We built some POCs to validate the interface and the flow. You can check the code in [this PR](https://github.com/bosonprotocol/boson-protocol-contracts/pull/578).

The POCs have shown that the interface is flexible enough to support different price discovery protocols. However, it is only safe to use vouchers directly in the price discovery protocol if the protocol is designed to not take custody of NFTs beforehand. Otherwise, we need to use a wrapper contract that will work with price discovery but would at the same time allow the protocol to get the information about the price that was agreed upon. Check [Wrappers](<[https://docs.google.com/document/d/1A8z5ojcUZRMlfkBUQiJjy1APUOpJrm7LBSPeSHU-RZQ/edit](https://docs.google.com/document/d/1A8z5ojcUZRMlfkBUQiJjy1APUOpJrm7LBSPeSHU-RZQ/edit#)>)

### Boson Voucher changes

Normal vouchers:

1. Always treat as true token, i.e. onVoucherTransferred

Pre-minted voucher:

1. if `offer.priceType == PriceType.Static` initial transfer should start an exchange
2. if `offer.priceType == PriceType.Discovery`, initial transfer:
   1. If the transaction is initiated from BP, start an exchange.
   2. If the transaction is initiated from the seller or voucher contract, it is equivalent to depositing tokens into a PD pool and the transaction succeeds.
   3. If the transaction is initiated outside of BP and the "from" address is not the seller assistant or BV, then the transaction reverts, since this would bypass the buyer protection.

### Security considerations

Given the nature of smart contracts we might want gate which price discovery contracts can be used

## Copyright waiver & license

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
