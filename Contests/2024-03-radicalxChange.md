## Summary

|ID|Title|
|:-:|-----|
|[H-01](#h-01-the-highest-bidder-can-steal-the-collateral-and-win-the-auction-without-paying)|The Highest Bidder can steal the collateral and win the auction without paying|
|||
|[M-01](#m-01-no-fees-state-makes-the-auction-process-insolvable)|No Fees state makes the Auction process insolvable|


---

## [H-01] The Highest Bidder can steal the collateral and win the auction without paying

### Summary
Because of not checking the bidder of the bid being canceled canceling in `EnglishPeriodicAuctionInternal::_cancelAllBids`, the Highest bidder can cancel his Bid keeping himself as the Highest Bidder.

### Vulnerability Detail

When a Bidder wants to cancel his Bid using `EnglishPeriodicAuctionInternal::_cancelBid`, the function provides a check to prevent the Highest Bidder from canceling his Bid. And this is a must to prevent him from taking his `collateralAmount` and winning the Auction without paying.

[auction/EnglishPeriodicAuctionInternal.sol#L393-L396](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L393-L396)
```solidity
    function _cancelBid( ... ) internal {
        ...

        // @audit The Highest can not cancel his Bid
        require(
 <@         bidder != l.highestBids[tokenId][round].bidder,
            'EnglishPeriodicAuction: Cannot cancel bid if highest bidder'
        );

        ...
    }

```

But in `EnglishPeriodicAuctionInternal::_cancelAllBids`, this check is missing. besides, it takes the `currentAuctionRound` into consideration. 

```solidity
    function _cancelAllBids(uint256 tokenId, address bidder) internal {
        ...

        // @audit this loop even take currentAuctionRound in consideration
        for (uint256 i = 0; i <= currentAuctionRound; i++) {
            Bid storage bid = l.bids[tokenId][i][bidder];
           
            if (bid.collateralAmount > 0) {
                // @audit No checking if this Bid is the Highest Bid or not
❌️              l.availableCollateral[bidder] += bid.collateralAmount;
                ...
            }
        }
```

This means anyone even the Highest Bidder of the `currentAuctionRound` can cancel his Bid using this function, and this behavior should not occur to prevent stealing collateral as we discussed earlier.

This will make the Highest Bidder (Attacker) withdraw his collateral (`ETH`) before the end of the auction, and when closing the Auction, the following will occur.

1. The Highest Bidder (Attacker) will receive the token without paying anything.
2. The `oldBidder` (Owner of the token), will gain nothing, and lose his token. 
3. The `Beneficiary` will not gain his fees for that Auction Period.


### Impact
1. The Highest Bidder (Attacker) will receive the token without paying anything.
2. The `oldBidder` (Owner of the token), will gain nothing, and lose his token. 
3. The `Beneficiary` will not gain his fees for that Auction Period.


### Code Snippet
https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L422-L433

### Tool used
Manual Review

### Recommendation
Prevent the execution of the function if the caller (Bidder) is the Highest bidder of the current round.

> EnglishPeriodicAuctionInternal::_cancelAllBids
```diff
    function _cancelAllBids(uint256 tokenId, address bidder) internal {
        EnglishPeriodicAuctionStorage.Layout
            storage l = EnglishPeriodicAuctionStorage.layout();

        uint256 currentAuctionRound = l.currentAuctionRound[tokenId];

+       require(
+           bidder != l.highestBids[tokenId][currentAuctionRound].bidder,
+           'EnglishPeriodicAuction: Cannot cancel bid if highest bidder'
+       );

        for (uint256 i = 0; i <= currentAuctionRound; i++) { ... }
    }
```

---

## [M-01] No Fees state makes the Auction process insolvable

### Summary
If the Collection Admin sets no fees, the auction process will break

### Vulnerability Detail

The Collection Admin (Artist) can set fees to get collected after the finalization of the auction. The fees are set by using [`feeDenominator`](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/pco/facets/PeriodicPCOParamsFacet.sol#L124-L128) and [`feeNumerator`](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/pco/facets/PeriodicPCOParamsFacet.sol#L108-L112).

If the Artist wants to set fees to zero, he can set the `feeNumerator` to zero, to make the fees `0%`.

The problem is that the Auction Contract does not take `zero fees state` in consideration. Where it forces a check that the `collateralAmount` should be greater than the `bidAmount`.

[auction/EnglishPeriodicAuctionInternal.sol#L329-L332](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L329-L332)
```solidity
    function _placeBid( ... ) internal {
        ...

        if (bidder == currentBidder) {
            // If current bidder, collateral is entire fee amount
            feeAmount = totalCollateralAmount;
        } else {
            // @audit totalCollateralAmount equals bidAmount in zero fees state, the tx will revert in this case
            require(
❌️              totalCollateralAmount > bidAmount,
                'EnglishPeriodicAuction: Collateral must be greater than current bid'
            );
            // If new bidder, collateral is bidAmount + fee
            feeAmount = totalCollateralAmount - bidAmount;
        }

        // @audit check that the bidAmound to fees is correct 
        require(
            _checkBidAmount(bidAmount, feeAmount),
            'EnglishPeriodicAuction: Incorrect bid amount'
        );
        
        ...
    }

```

This is the case if there is fees taken from the bid, but in case of zero fees this is not the case, as the `collateralAmount` will equal the `bidAmount`.

The bidder can not increase the bid by `1 wei` to just pass the check, as if he did that, the feeAmount will equal 1, and the transaction will revert in `_checkBidAmount()`, as the `feeAmount` should be zero.

This will make All Bidding processes in a DoS, if the Collection Admin (Artist) made fees equal to zero.

The issue could be more serious, where if the Collection Admin (Artist) chooses to make his NFT Collection Fully decentralized By initializing the Auction without `Auction Pitch Admin` role (this is normal behavior and allowed), he will not be able to change the fees again.

### Impact
The Artist who wants to make their NFT collections with zero fees will make their auction process break and insolvable.

### Code Snippet
https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L329-L332

### Tool used
Manual Review

### Recommendation
Change the Condition by making it `>=` rather than `>` to accept the zero fees state.

> auction/EnglishPeriodicAuctionInternal::_placeBid() L:330
```diff
    function _placeBid( ... ) internal {
        ...

        if (bidder == currentBidder) {
            ...
        } else {
            require(
-               totalCollateralAmount > bidAmount,
+               totalCollateralAmount >= bidAmount,
                'EnglishPeriodicAuction: Collateral must be greater than current bid'
            );
            ...
        }

        ...
    }
``` 
