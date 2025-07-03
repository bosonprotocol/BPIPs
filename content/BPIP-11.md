---
bpip: 11
title: Account-free exchanges
authors: Klemen Zajc
discussions-to: https://github.com/bosonprotocol/BPIPs/discussions/40
status: Draft
created: 2025-07-03
---

## Abstract
This proposal describes an option to do the use the Boson Protocol without an explicit account creation.

## Motivation
Before the seller can create an offer, they must create a protocol account. This has some features that are used by some sellers:
1. it allows better role management via admin, assistant and treasury,
2. it allows the seller to attach metadata URI,
3. it preserves the account history even if the wallet addresses are change.  

While this are a good features, the seller creation requires a setup, which presents a friction and might be unnecessary for certain sellers.  

In this BPIP we propose to change the account management system to allow the sellers to create an offer and execute exchange without an explicit account creation. When BPIP9 and BPIP10 are implemented, they will even increase the demand for account-free exchanges. 

The protocol can still use internal ids for tracking; however, all account are created implicitly, similarly as it is already done for buyer accounts.

Similarly, the dispute resolvers account could be implicitly created when they are added to the offer. 

This BPIP does not remove the existing account management. Sellers that want to use the advanced accounts can continue using their existing accounts and new sellers can create new accounts.

## Specification
TBD

## Rationale
TBD

## Backward compatibility
This specification does not break backward compatibility.

## Implementation
* Create offer
  * If seller id is provided, check that the caller is the assistant
  * If seller id is not provided, create an account with all roles managed by the caller
  * Allow providing dispute resolver address instead of dispute resolver id.
    * If address belongs to an existing DR account, use its id
    * If address does not belong to an existing DR, create a new DR account with all roles managed by the address

### Security considerations

This does not break protocol guarantees or lower the trust, since already in the existing system, both seller and buyer had to agree on all offer parameters in order to go into exchange.
  
## Copyright waiver & license
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
