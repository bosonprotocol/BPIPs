---
bpip: 3
title: Pre-minted Vouchers (Shell rNFTs)
discussions-to: 
status: Living
created: 2022-11-22
---

## Abstract
This proposal describes a new feature which allows sellers to pre-mint vouchers, making them available for primary purchase on NFT marketplaces.

## Motivation
Ordinarily, sellers on Boson Protocol create offers, which buyers then commit to, resulting in the issuance of a voucher (aka, a redeemable NFT or rNFT). Vouchers can be resold on secondary markets or redeemed to receive the merchandise.

Since the vouchers don’t exist until buyers commit to the protocol, traffic must be driven to the seller’s site or the Boson Dapp for primary market purchases. Meanwhile, popular marketplaces and destinations such as metaverses, already have a high traffic volume.

In order to help sellers capture that potential market, we propose to solve this "chicken and egg" problem with a new feature called pre-minted vouchers.

## Specification
#### IBosonVoucher
The following methods are added.
```solidity
    /**
     * @notice Reserves a range of vouchers to be associated with an offer
     *
     * Must happen prior to calling preMint
     * Caller must have PROTOCOL role.
     *
     * Reverts if:
     * - Start id is not greater than zero for the first range
     * - Start id is not greater than the end id of the previous range for subsequent ranges
     * - Range length is zero
     * - Range length is too large, i.e., would cause an overflow
     * - Offer id is already associated with a range
     *
     * @param _offerId - the id of the offer
     * @param _start - the first id of the token range
     * @param _length - the length of the range
     */
    function reserveRange(
        uint256 _offerId,
        uint256 _start,
        uint256 _length
    ) external;

    /**
     * @notice Pre-mints all or part of an offer's reserved vouchers.
     *
     * For small offer quantities, this method may only need to be
     * called once.
     *
     * But, if the range is large, e.g., 10k vouchers, block gas limit
     * could cause the transaction to fail. Thus, in order to support
     * a batched approach to pre-minting an offer's vouchers,
     * this method can be called multiple times, until the whole
     * range is minted.
     *
     * A benefit to the batched approach is that the entire reserved
     * range for an offer need not be pre-minted at one time. A seller
     * could just mint batches periodically, controlling the amount
     * that are available on the market at any given time, e.g.,
     * creating a pre-minted offer with a validity period of one year,
     * causing the token range to be reserved, but only pre-minting
     * a certain amount monthly.
     *
     * Caller must be contract owner (seller operator address).
     *
     * Reverts if:
     * - Offer id is not associated with a range
     * - Amount to mint is more than remaining un-minted in range
     * - Too many to mint in a single transaction, given current block gas limit
     *
     * @param _offerId - the id of the offer
     * @param _amount - the amount to mint
     */
    function preMint(uint256 _offerId, uint256 _amount) external;

    /**
     * @notice Burn all or part of an offer's preminted vouchers.
     * If offer expires or it's voided, the seller can burn the preminted vouchers that were not transferred yet.
     * This way they will not show in seller's wallet and marketplaces anymore.
     *
     * For small offer quantities, this method may only need to be
     * called once.
     *
     * But, if the range is large, e.g., 10k vouchers, block gas limit
     * could cause the transaction to fail. Thus, in order to support
     * a batched approach to pre-minting an offer's vouchers,
     * this method can be called multiple times, until the whole
     * range is burned.
     *
     * Caller must be contract owner (seller operator address).
     *
     * Reverts if:
     * - Offer id is not associated with a range
     * - Offer is not expired or voided
     * - There is nothing to burn
     *
     * @param _offerId - the id of the offer
     */
    function burnPremintedVouchers(uint256 _offerId) external;

    /**
     * @notice Gets the number of vouchers available to be pre-minted for an offer.
     *
     * @param _offerId - the id of the offer
     * @return count - the count of vouchers in reserved range available to be pre-minted
     */
    function getAvailablePreMints(uint256 _offerId) external view returns (uint256 count);

    /**
     * @notice Gets the range for an offer.
     *
     * @param _offerId - the id of the offer
     * @return range - range struct with information about range start, length and already minted tokens
     */
    function getRangeByOfferId(uint256 _offerId) external view returns (Range memory range);
```

#### IBosonExchangeHandler
The following method is added.
```solidity
    /**
     * @notice Commits to a preminted offer (first step of an exchange).
     *
     * Emits a BuyerCommitted event if successful.
     *
     * Reverts if:
     * - The exchanges region of protocol is paused
     * - The buyers region of protocol is paused
     * - Caller is not the voucher contract, owned by the seller
     * - Exchange exists already
     * - Offer has been voided
     * - Offer has expired
     * - Offer is not yet available for commits
     * - Buyer account is inactive
     * - Buyer is token-gated (conditional commit requirements not met or already used)
     * - Seller has less funds available than sellerDeposit for non preminted offers
     * - Seller has less funds available than sellerDeposit and price for preminted offers
     *
     * @param _buyer - the buyer's address (caller can commit on behalf of a buyer)
     * @param _offerId - the id of the offer to commit to
     * @param _exchangeId - the id of the exchange
     */
    function commitToPreMintedOffer(
        address payable _buyer,
        uint256 _offerId,
        uint256 _exchangeId
    ) external;
```

#### IBosonOfferHandler
The following method is added.
```solidity
    /**
     * @notice Reserves a range of vouchers to be associated with an offer
     *
     *
     * Reverts if:
     * - The offers region of protocol is paused
     * - The exchanges region of protocol is paused
     * - Offer does not exist
     * - Offer already voided
     * - Caller is not the seller
     * - Range length is zero
     * - Range length is greater than quantity available
     * - Range length is greater than maximum allowed range length
     * - Call to BosonVoucher.reserveRange() reverts
     *
     * @param _offerId - the id of the offer
     * @param _length - the length of the range
     */
    function reserveRange(uint256 _offerId, uint256 _length) external;
```
#### IBosonOrchestrationHandler
For convenience, all orchestration methods that allow offer creation, get a counterpart method that allow range reservation at the same time. Seller needs to provide an additional input which tells how many vouchers can be preminted. New methods are
These methods are not essential for this BPIP, so full specification is not provided. Revert reasons, events emitted and input parameters are union of revert reasons, events emitted and input parameters of base functions, combined in given orchestration function. List of suggested orchestration methods:
- createSellerAndPremintedOffer
- createPremintedOfferWithCondition
- createPremintedOfferAddToGroup
- createPremintedOfferAndTwinWithBundle
- createPremintedOfferWithConditionAndTwinAndBundle
- createSellerAndPremintedOfferWithCondition
- createSellerAndPremintedOfferAndTwinWithBundle
- createSellerAndPremintedOfferWithConditionAndTwinAndBundle


## Interactions 
![Pre-minted Vouchers Interactions Diagram](./assets/bpip-3/Preminted-Vouchers-Diagram.png "Pre-minted Voucher Sequences Diagram")

## Rationale
Vouchers, which are ERC721 NFTs show on marketplaces as soon as they are minted. Currently vouchers are minted at the same time when exchange is created, which means that whenever they are resold on secondary market, new buyer does not get full dispute period (i.e. transfer does not reset the timers in the protocol). In order to allow buyers on any marketplace to get full dispute period when they are the first owner of the voucher, we need a different solution. 

One possible approach is to create a dedicated bridge contract, where sellers could create NFTs which would show on marketplaces. When buyers on marketplace bought them, this bridge NFT would be burnt and buyers would get redeemable voucher instead. However this approach breaks user flow in some marketplace and might be questionable since buyer gets a different NFT that was actually bought.

Another approach was to enable mint on demand directly on voucher contract. Although it does not suffer from the same limitations than previous approach, mint is not standard ERC721 function, so compatibility with all marketplaces would be hard to achieve.

This leads to this proposals, where vouchers are preminted directly on existing voucher contract. Their existence automatically means that they can be transferred as regular ERC721 NFTs, so they can be sold on any marketplace. Compared to first approach, voucher here is not burned, but simply transferred to a new owner (i.e. first buyer). This first transfer invokes `commitToPremintedOffer` on the protocol, which starts the exchange and effectively converts preminted voucher into a true voucher. After that, voucher behaves as other vouchers, i.e. dispute period starts at the time of the purchase, voucher can be redeemed, transferred etc.
Important to note here is that we now have two types of voucher in the voucher contract:
- preminted vouchers, with no exchange associated and
- true vouchers, with exactly one exchange associated.
  
In current protocol, all vouchers are issued when protocol invokes `issueVoucher` method. In this proposal we add additional method `preMint` which allows seller (operator) to issue a desired number of vouchers.
Since currently it holds that exchange id (in protocol) always matches token id (in voucher contract), it's desired that this stays even with premint. To achieve it, we propose a method to reserve range (`reserveRange`), which effectively reserves a desired number of exchange ids in the protocol. Then whenever a preminted voucher is converted into a true voucher, it keeps its token id and gets matching exchange id in the protocol.

Although range reservation ultimately happens on voucher contract, method must be invoked true the protocol. That way it can be ensured that the amount of preminted vouchers never exceeds quantity available, set in the offer.

If voucher expires or is voided, it currently means that buyers cannot commit to it anymore. However, with preminted vouchers it's possible that some of them were not bought before that happen. In this case protocol should also prevent committing to the offer, which then means that some of unusable vouchers are stuck in the contract and possibly on marketplaces. To delist them we suggest method to burn all preminted vouchers (`burnPremintedVouchers`) that cannot be redeemed anymore.

Given that now buyer does not directly interact with the protocol, this opens the question, how to ensure protocol exchange guarantees (i.e. possibility to get funds back if something goes wrong). The proposed solution is than now seller needs to provide enough funds to cover both item price and seller deposit. This ensures buyer protection straight away, but it increases a seller's capital requirement a bit. However, this drawback is not to big, since as soon as marketplace exchange happens, seller receives the funds, which can be directly used to cover item price in next sales. Moreover, as soon as some exchange are finalized and funds are released back into seller's pool, those funds can be automatically used to cover future sales.

This proposal might rise some efficiency concerns about the solution efficiency, since minting ERC721 is relatively costly operation. ERC1155 was considered as alternative, but it turned out that with it it's impossible to maintain nice exchange id and token id equivalence. To overcome the problem of ERC721 inefficiency we propose another solution. Since premint always assign token ownership to voucher contract owner, there is no need to explicitly store it's address during the premint. It's enough that only correct transfer events are emitted, while general consistency is achieved by overriding ERC721 `ownerOf`, `transferFrom` and `safeTransferFrom` methods to correctly manage all tokens that were preminted.

## Backward compatibility
This specification does not break backward compatibility.

Still it is important that when upgrade is done voucher contracts are upgraded before the protocol is upgraded.

## Implementation
### BosonVoucher

* Range reservation
  * Must be done before preminting can happen.
  * Ranges must be strictly increasing, i.e. start of new range must be greater than current highest range.
  * Store information about associated offer id, range start and range length.
  * The tracking of reserved ranges enables that exchange id (in protocol) and token id (voucher contract) can remain the same.
* Preminting
  * Can be done in batches. This allows to premint the quantities that could otherwise not be possible because of the block gas limit. Additionally it allows seller, to only partially release vouchers to the market.
  * Preminting is always done to contract owner's address. It emits events that look as if vouchers have been minted, but don't store the owner address to conserve the bulk of an actual minting's gas. 
  * Preminting always mints consecutive token ids, starting with lowest non-minted id in the range.
* Token ownership
  * Change ERC721 method `ownerOf`, so it properly reports the owner.
  * If token id is in a reserved range, has already been preminted, but not yet transferred or burned, report the contract owner (the seller) as the owner of the voucher.
  * In other cases return true owner if exists, or revert otherwise.
* Preminted voucher transfer
  * When preminted voucher is transferred for the first time, detect it in before transfer hook it and call the commitToPremintedOffer protocol method, which does not require payment.
  * In all subsequent transfers, treat voucher as normal, i.e. call onVoucherTransferred protocol method.
  * Since all inhered transfer methods (transferFrom and safeTranferFrom) revert if token id has no owner, it needs to be temporary updated whenever first transfer of preminted voucher.
* Burn preminted vouchers
  * Preminted vouchers can be burned when offer expires or is voided.
  * Burning allows owner to removed unusable vouchers from their wallet.

* Implemented [here](https://github.com/bosonprotocol/boson-protocol-contracts/pull/483).


### Protocol
Offer creation remains the same as it was in previous versions. To enable preminted vouchers for an offer, seller just needs to call reserveRange method in the protocol, which effectively converts offer into a (partially) preminted offer.
Offer can be
  - fully preminted: reserved range matches initial quantity available. It makes it impossible to commit to offer directly.
  - partially preminted: reserved range is less than quantity available. It is possible to commit to offer directly on the protocol or through primary transfer of a preminted voucher.
  - not preminted: there is no reserved range for the offer. The only way to commit is directly on the protocol (same as in current version of the protocol).

Reserve range can be called once per offer, so seller must in advance decide how may vouchers can be preminted. Reserve range accepts only offer id and range length, while range start is determined based on current exchange id. Calling reserve range decreases quantity available and increases exchange id counter in the protocol. This means than it is now possible that exchange with higher id is created before the exchange with lower id.

Decision if offer will support preminted vouchers or not does not need to be decided at offer creation time. Even is some user already commit to an offer, seller can reserve a range, as long as quantity available is greater than 0.
If a seller wants to create an offer and reserve range in single transaction, they can use orchestration methods that enable it.

* Implemented [here](https://github.com/bosonprotocol/boson-protocol-contracts/pull/490).
  
## Copyright waiver & license
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
