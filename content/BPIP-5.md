---
bpip: 3
title:  Auction mechanism inside protocol
discussions-to: 
status: Living
created: 2023-01-25
---

## Abstract
This proposal describe a new feature to allow Boson Vouchers auctions on the protocol

## Motivation


## Specification 

- An Boson Voucher owner ~or approved operator~ can create an auction.
- The auction timer will start when a bid is greater than or equal to the price set by the voucher owner.
- The Boson Voucher is only taken into custody after the first valid bid is submitted.
- The timer will be set back to 15 minutes if a bid is submitted within the last 15 minutes.
- The amount for a bid is held in custody by protocol until either the auction ends or a higher bid is placed.
- The percentage difference for a new bid must be greater than or equal to x% of the previous bid. 
- Royalties are paid out when an auction ends.

## Rationale

/

## Backward compatibility

/

## Implementation
/

## Copyright waiver & license
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
