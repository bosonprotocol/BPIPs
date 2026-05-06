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
3. it preserves the account history even if the wallet addresses change.  

While this are a good features, the seller creation requires a setup, which presents a friction and might be unnecessary for certain sellers.  

In this BPIP we propose to change the account management system to allow the sellers to create an offer and execute exchange without an explicit account creation. 

BPIP-9 (Buyer Initiated Exchange) and BPIP-10 (Off-chain Listing Phase) significantly expanded the ways the protocol can be used. BPIP-9 lets buyers post offers and sellers commit to them without ever publishing a seller-side listing; BPIP-10 lets both parties agree on terms entirely off-chain and enter the protocol only at the commitment point. Together, these open the protocol to flows where a party may participate in only a single exchange - a pattern that is especially prevalent in agentic commerce, where an AI agent acts on behalf of a user and has no reason to maintain a persistent on-chain identity.

It is expected the combination of these new flows and the rise of agentic commerce to substantially increase the number of participants who need to use the protocol without going through an explicit account-creation step. Requiring them to do so before their first offer or commit is a barrier that provides little benefit and risks excluding entire classes of automated and lightweight participants.

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
