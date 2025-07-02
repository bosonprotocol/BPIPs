---
bpip: 7
title: Sequential Commit
authors: Jonas Seiferth, Justin Banon, Aditya Asgaonkar, Mischa Tuffield, Klemen Zajc
discussions-to: https://github.com/bosonprotocol/BPIPs/discussions/17
status: Final
created: 2023-03-21
---

## Abstract
This proposal describes a new feature that provides buyer protection on secondary market sales and enforces perpetual on-chain royalties.

## Motivation
When a buyer commits to an offer in Boson Protocol, they get buyer protection - either they get the purchased item or they get financially compensated if the seller does not deliver. The maximum compensation they can get is the sum of the item price and the seller's deposit.
Buyer is also free to resell the voucher on a secondary marketplace, presumably for a higher price. When that happens, the protocol is not aware of it and no additional funds are put in the escrow. If the original seller fails to deliver, the maximum financial compensation is still only the original item price plus the seller deposit. If the secondary price was higher than that, the last buyer cannot get everything back. In general, we can say that secondary buyers do not get the same protection as primary buyers.  

To overcome this we propose a change to the protocol, where during each sequential commit, additional funds get locked in the escrow. Only when the exchange is finalized do resellers get fully paid out depending on the final exchange state. For example, if an exchange is normally completed, every reseller gets their net profit at the end. However, if the original seller revokes the voucher, resellers don't get any profit, but they also don't lose anything. This kind of system ensures protection for all involved parties. 

This system should also work if the secondary price is equal to or lower than the previous price. In this case, no additional funds are locked during the sequential commit, reseller just gets paid out the secondary price. If the exchange is completed normally, the reseller does not get any additional payout, but if it ends in a revoked or cancelled state, the reseller is reimbursed for their loss (their net profit is 0).  

Sequential commit is not limited to one secondary sale but can be performed multiple times for the same voucher. In any subsequent sale, the last price is used as the reference point to determine how much must be put into the escrow based on a new price.  

The proposed system is also designed to be time-capital efficient, i.e. amount kept in escrow is the minimal needed to cover any final payouts. Since multiple different scenarios exist, which have different outcomes, we prepared a detailed explanation of the system with examples for all possible scenarios. A PDF document is [available here](./assets/../assets/bpip-7/Sequential%20Commit.pdf).

Since sequential commit in fact represents secondary market exchange, it will at the same time implement two other features, proposed for the Boson protocol:
- [Price discovery](BPIP-4.md), which will allow compatibility with the generic price discovery mechanism
- Perpetual royalties, based on [BPIP-5](BPIP-5.md). At finalization time, all royalties will be paid out to recipients assigned to the offer.

## Specification
#### SequentialCommitFacet
A new facet `SequentialCommitFacet` is added. It implements the following method

```solidity
/**
 * @title ISequentialCommitHandler
 *
 * @notice Handles sequential commits.
 *
 * The ERC-165 identifier for this interface is: 0x34780cc6
 */
interface IBosonSequentialCommitHandler is BosonErrors, IBosonExchangeEvents, IBosonFundsLibEvents {
    /**
     * @notice Commits to an existing exchange. Price discovery is offloaded to external contract.
     *
     * Emits a BuyerCommitted event if successful.
     * Transfers voucher to the buyer address.
     *
     * Reverts if:
     * - The exchanges region of protocol is paused
     * - The buyers region of protocol is paused
     * - Buyer address is zero
     * - Exchange does not exist
     * - Exchange is not in Committed state
     * - Voucher has expired
     * - It is a bid order and:
     *   - Caller is not the voucher holder
     *   - Voucher owner did not approve protocol to transfer the voucher
     *   - Price received from price discovery is lower than the expected price
     * - It is a ask order and:
     *   - Offer price is in native token and caller does not send enough
     *   - Offer price is in some ERC20 token and caller also sends native currency
     *   - Calling transferFrom on token fails for some reason (e.g. protocol is not approved to transfer)
     *   - Received ERC20 token amount differs from the expected value
     *   - Protocol does not receive the voucher
     *   - Transfer of voucher to the buyer fails for some reason (e.g. buyer is contract that doesn't accept voucher)
     *   - Reseller did not approve protocol to transfer exchange token in escrow
     * - Call to price discovery contract fails
     * - Protocol fee and royalties combined exceed the secondary price
     * - The secondary price cannot cover the buyer's cancellation penalty
     * - Transfer of exchange token fails
     *
     * @param _buyer - the buyer's address (caller can commit on behalf of a buyer)
     * @param _exchangeId - the id of the exchange to commit to
     * @param _priceDiscovery - the fully populated BosonTypes.PriceDiscovery struct
     */
    function sequentialCommitToOffer(
        address payable _buyer,
        uint256 _exchangeId,
        BosonTypes.PriceDiscovery calldata _priceDiscovery
    ) external payable;
}
```

#### Changes in other facets
No other interfaces are changed. Reference implementation might include other changes, but they are the result of changes proposed in other BPIPs ([Price discovery](BPIP-4.md), [Royalties](BPIP-5.md))

## Rationale
A new method is needed to distinguish between the initial commit (when only `offerId` is relevant) and the sequential commit where `exchangeId` is the relevant indicator. Sequential commit's main effects are voucher ownership transfer and locking up additional funds. This does not affect other actions such as redeeming, canceling or revoking the voucher or raising the dispute. It does affect how funds are released at finalization time, but it does not affect public functions. Withdrawal of released funds remains the same as it was.

Additional information about the sequential commit is [available here](./assets/../assets/bpip-7/Sequential%20Commit.pdf).

## Backward compatibility
This specification does not break backward compatibility.

## Implementation
* Commit to sequential offer
  * Accept exchangeId as an identifier
Similar to commit to offer, sequential commit to offer can be done on the buyer's behalf
  * Validate that the voucher exists and has not expired
  * Pass price discovery data to price discovery contract client.
The success of a call is not enough to assume the exchange actually happened. The client must verify that the voucher and exchange token were sent to the correct addresses.
The protocol must calculate the minimal amount to go into the escrow and ensure it has been received.
  * Release the difference between the price and the minimal escrow amount to the reseller.
  * The protocol must send any outstanding assets to the buyer or seller, depending on the offer type.
The protocol stores information about the exchange (buyer id, price, fees and royalties) so they can be used during the next sequential commit or exchange finalization.
* Releasing the funds
  * Releasing funds for the original seller remains as it was.
  * The formulae are adapted to use the last price.
  * Additionally, loop over all sequential commits and release the difference between what was already released (during the commit) and the total payout.

The current implementation is available in [v2.4.0 release candidate](https://github.com/bosonprotocol/boson-protocol-contracts/releases/tag/v2.4.0-rc.3).

### Security considerations

Security considerations are available in [this document](./assets/../assets/bpip-7/Sequential%20Commit.pdf).
  
## Copyright waiver & license
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
