## Additional instances of [NC-24] Incomplete NatSpec: @param is missing on actually documented functions.

This is another instance of the Incomplete NatSpec non-critical QA issue from the automated findings report.

The calcDecrease() function in CDPVault.sol is missing a @param NatSpec for the uint128 cumulativeQuotaInterest parameter. However, it is not included as an instance in the provided Automated Report :
https://urldefense.com/v3/__https://github.com/code-423n4/2024-07-loopfi/blob/57871f64bdea450c1f04c9a53dc1a78223719164/src/CDPVault.sol*L657__;Iw!!HXCxUKc!ymVxikFwkZ_GCBFfY5q0IU0e6Nl5g5YMoGD-MMD_k1XqLL3C3ARW8863SeHfu5_WoZ_9Jy4Oc6IQU4JPv5NQT3vqBkg$

```
 /// @dev Computes new debt principal and interest index (and other values) after decreasing debt
    ///      - Debt comprises of multiple components which are repaid in the following order:
    ///        quota update fees => quota interest => base interest => debt principal.
    ///        New values for all these components depend on what portion of each was repaid.
    ///      - Debt principal, for example, only decreases if all previous components were fully repaid
    ///      - The new credit account's interest index stays the same if base interest was not repaid at all,
    ///        is set to the current interest index if base interest was repaid fully, and is a solution to
    ///        the equation `debt * (indexNow / indexLastUpdate - 1) - delta = debt * (indexNow / indexNew - 1)`
    ///        when only `delta` of accrued interest was repaid
    /// @param amount Amount of debt to repay
    /// @param debt Debt principal before repayment
    /// @param cumulativeIndexNow The current interest index
    /// @param cumulativeIndexLastUpdate Credit account's interest index as of last update
    /// @return newDebt Debt principal after repayment
    /// @return newCumulativeIndex Credit account's quota interest after repayment
    /// @return profit Amount of underlying tokens received as fees by the DAO
    /// @return newCumulativeQuotaInterest Credit account's accrued quota interest after repayment
    // @return newQuotaFees Amount of unpaid quota fees left after repayment
    function calcDecrease(
        uint256 amount,
        uint256 debt,
        uint256 cumulativeIndexNow,
        uint256 cumulativeIndexLastUpdate,
        uint128 cumulativeQuotaInterest
    )
        internal
        pure
        returns (uint256 newDebt, uint256 newCumulativeIndex, uint256 profit, uint128 newCumulativeQuotaInterest)
    {
        ....
    }
```

The manualStopEmissionsFor() function in ChefIncentivesController.sol is also missing an @param for the _tokens parameter

```
  /**
     * @notice function to stop user emissions
     * @param _user address of user to stop emissions for
     */
    function manualStopEmissionsFor(address _user, address[] memory _tokens) public isWhitelisted {

    ..........

}
```

https://urldefense.com/v3/__https://github.com/code-423n4/2024-07-loopfi/blob/57871f64bdea450c1f04c9a53dc1a78223719164/src/reward/ChefIncentivesController.sol*L813__;Iw!!HXCxUKc!ymVxikFwkZ_GCBFfY5q0IU0e6Nl5g5YMoGD-MMD_k1XqLL3C3ARW8863SeHfu5_WoZ_9Jy4Oc6IQU4JPv5NQJLeHbBo$
