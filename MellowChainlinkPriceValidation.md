Sungyuk1

Medium

Deposits will use the wrong price if the price of a token is outside the min/max range of a Chainlink Oracle

## Summary

Chainlink aggregators have a built in circuit breaker if the price of an asset goes outside of a predetermined price range. The result is that if an asset experiences a flash crash in value, the price that the oracle returns will be the minPrice instead of the actual price of the asset. An example of such a scenario would be the LUNA crash.

The _validateAndGetPrice() function in ChainlinkOracle.sol does not check if the priced returned is equal to the min or max price. This enables the depositor to make deposits with the crashing token priced at minPrice rather than the actual price of the token.

This was the underlying cause behind the Venus Protocol and Blizz Finance drainings : https://rekt.news/venus-blizz-rekt/

## Vulnerability Details

Let's review how the deposit function calculates how many lpTokens to mint :

We iterate through the provided token amounts and calculate the number of tokens that should be deposited so that we can maintain the ratios of the vault.

We iterate through all the underlying tokens. At each iteration, we use the priceOracle to get the priceX96 of the current token. We calculate the value of the current token held by the contract, and also the value of the deposit of the current token. We also transfer the correct amount of tokens from the sender to the user.

We use the internal function _processLpAmount() to calculate the amount of lp tokens that should be minted for a deposit of the current value. As long as this deposit is not the first deposit, the number of lp tokens minted will be :

(depositValue * total supply of lp tokens) / total value of underlying tokens in the contract

If the price of a token goes outside of Chainlink's predetermined price range, the Chainlink aggregator will return the min/max prices. ChainlinkOracle.sol uses the function _validateAndGetPrice() to get the price of a token as well as to check that it is valid. The function has checks for invalid data and stale prices, but no checks for min/max prices.

```
    function _validateAndGetPrice(
        AggregatorData memory data
    ) private view returns (uint256 answer, uint8 decimals) {
        if (data.aggregatorV3 == address(0)) revert AddressZero();
        (, int256 signedAnswer, , uint256 lastTimestamp, ) = IAggregatorV3(
            data.aggregatorV3
        ).latestRoundData();
        // The roundId and latestRound are not used in the validation process to ensure compatibility
        // with various custom aggregator implementations that may handle these parameters differently
        if (signedAnswer < 0) revert InvalidOracleData();
        answer = uint256(signedAnswer);
        if (block.timestamp - data.maxAge > lastTimestamp) revert StaleOracle();
        decimals = IAggregatorV3(data.aggregatorV3).decimals();
    }
```

## Impact

This allows the user to mint more tokens at a higher value than they should have been

Lets say for simplicity's sake that the min/max prices are 10.
Lets say we have 4 tokens in the vault at [100, 100, 100, 100]
The target ratio for the tokens is also 1/4 for each token.
Let's say there are 500000 lp tokens currently in existance

Let's say that there was a flashcrash and now the oracle returns the minPrice for token b (10 in this case) and a price of 1000 for the rest of the tokens. However, lets say that the actual price of token b is 1. The price of those tokens according to the oracle are [10000, 10, 10000, 10000]

The user goes to deposit with amounts = [10, 10, 10, 10]. This meets the ratio of 1/4 for each token.
The depositValue is calculated at (10 * 10000) + (10 * 10) + (10 * 10000) + (10 * 10000) = 300100.
The totalValue is calculated at (100 * 10000) + (100 * 10) + (100 * 10000) + (100 * 10000) = 3001000
So the amount of lp tokens minted will be (300100 * 500000) / 3001000 = 50000
So we get .1666111296 lp tokens per value in deposit

However, lets say that token b was priced correctly. So prices are [10000, 1, 10000, 10000]
The user goes to deposit with amounts = [10, 10, 10, 10]. This meets the ratio of 1/4 for each token.
The depositValue is calculated at (10 * 10000) + (10 * 1) + (10 * 10000) + (10 * 10000) = 300010.
The totalValue is calculated at (100 * 10000) + (100 * 1) + (100 * 10000) + (100 * 10000) = 3000100
So the amount of lp tokens minted will be (300010 * 500000) / 3000100 = 50000
So we get .1666611113 lp tokens per value of deposit.

In other words, when there is a price crash users can deposit to receive more lpTokens than they should be entitled to.

## Code Snippet

https://github.com/sherlock-audit/2024-06-mellow/blob/main/mellow-lrt/src/Vault.sol#L285

https://github.com/sherlock-audit/2024-06-mellow/blob/main/mellow-lrt/src/oracles/ChainlinkOracle.sol#L80

https://github.com/sherlock-audit/2024-06-mellow/blob/main/mellow-lrt/src/oracles/ChainlinkOracle.sol#L56


## Tools Used

Manual Review

## Recommended Mitigation Steps

ChainlinkOracle.sol should have a check to make sure that the answer is in the valid price range. Something like :

```
    if (answer >= maxPrice or answer <= minPrice) revert();
```
