# Fjord Token Staking - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)

- ## Medium Risk Findings
    - ### [M-01. Users can bid after auction has ended in FjordAuction.sol. This breaks internal accounting for how many auctionTokens should be sent to a user. ](#M-01)



# <a id='contest-summary'></a>Contest Summary

### Sponsor: Fjord

### Dates: Aug 20th, 2024 - Aug 27th, 2024

[See more contest details here](https://codehawks.cyfrin.io/c/2024-08-fjord)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 0
- Medium: 1
- Low: 0



    
# Medium Risk Findings

## <a id='M-01'></a>M-01. Users can bid after auction has ended in FjordAuction.sol. This breaks internal accounting for how many auctionTokens should be sent to a user.             



## Summary

The FjordAuction.sol contract has external functions for bid(), unbid(), and auctionEnd(). Both the unbid() and bid() functions will revert if the current block.timestamp is larger than auctionEndTime as shown below. However, these functions will not revert if the current block.timestamp is equal to the auctionEndTime. In such a case, a user can place a bid during the same block that the auctionEnd() function is called. This will result in user transferring fjordPoints to the contract which increases their balance in the bids mapping, while not diluting the multiplier calculated in auctionEnd().

## Vulnerability Details

Below in the bid function, notice how the bid function will only revert if block.timestamp is larger than auctionEndTime. If block.timestamp is equal to auctionEndTime the transaction does not revert.

```Solidity
    /**
     * @notice Places a bid in the auction.
     * @param amount The amount of FjordPoints to bid.
     */
    function bid(uint256 amount) external {
        if (block.timestamp > auctionEndTime) {
            revert AuctionAlreadyEnded();
        }

        bids[msg.sender] = bids[msg.sender].add(amount);
        totalBids = totalBids.add(amount);

        fjordPoints.transferFrom(msg.sender, address(this), amount);
        emit BidAdded(msg.sender, amount);
    }
```

Notice how the auctionEnd() function reverts only if the current block.timestamp is less than auctionEndTime.

```Solidity
/**
     * @notice Ends the auction and calculates claimable tokens for each bidder based on their bid proportion.
     */
    function auctionEnd() external {
        if (block.timestamp < auctionEndTime) {
            revert AuctionNotYetEnded();
        }
        if (ended) {
            revert AuctionEndAlreadyCalled();
        }

        ended = true;
        emit AuctionEnded(totalBids, totalTokens);

        if (totalBids == 0) {
            auctionToken.transfer(owner, totalTokens);
            return;
        }

        multiplier = totalTokens.mul(PRECISION_18).div(totalBids);

        // Burn the FjordPoints held by the contract
        uint256 pointsToBurn = fjordPoints.balanceOf(address(this));
        fjordPoints.burn(pointsToBurn);
    }
```

This means that both the bid() and auctionEnd() functions can be called during the same block, when block.timestamp is equal to auctionEndTime. In such scenarios, if a bid() function call is processed after the auctionEnd() function call, the user will increase their balance in the bids mapping while not decreasing the multiplier calculated in auctionEnd().

## Impact

The multiplier state variable is used to calculate how many tokens a user can claim considering the amount they deposited.

```Solidity
function claimTokens() external {
        if (!ended) {
            revert AuctionNotYetEnded();
        }

        uint256 userBids = bids[msg.sender];
        if (userBids == 0) {
            revert NoTokensToClaim();
        }

        uint256 claimable = userBids.mul(multiplier).div(PRECISION_18);
        bids[msg.sender] = 0;

        auctionToken.transfer(msg.sender, claimable);
        emit TokensClaimed(msg.sender, claimable);
    }
```

Since this vulnerability will cause the multiplier to be inflated, that means that each call to claimTokens() will give out more tokens than it should proportionally to how much users have deposited. This will lead to the contract running out of auctionTokens and the users who call claimTokens() late will not be be able to get anything.

## Tools Used

Manual Review

## Recommendations

Super simple fix, just revert on bid/unbid function calls when block.timestamp is greater than or equal to auctionEndTime, rather than just greater

```Solidity
function bid(uint256 amount) external {
        if (block.timestamp >= auctionEndTime) {
            revert AuctionAlreadyEnded();
        }
        .....
}
```

```Solidity
function unbid(uint256 amount) external {
        if (block.timestamp > auctionEndTime) {
            revert AuctionAlreadyEnded();
        }
        ......
}
```





