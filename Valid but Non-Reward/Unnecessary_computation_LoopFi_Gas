## Unnecessary computation performed in CDPVault.sol on calls to deposit() and repay() leading to higher gas costs

The CDPVault::modifyCollateralAndDebt() function is the function responsible for modifying a Position's collateral and debt balances. This function is the core logic behind the external repay(), withdraw(), deposit(), and borrow() functions.

```
function modifyCollateralAndDebt(
        address owner,
        address collateralizer,
        address creditor,
        int256 deltaCollateral,
        int256 deltaDebt
    ) public {

         ..........

        VaultConfig memory config = vaultConfig;
        uint256 spotPrice_ = spotPrice();
        uint256 collateralValue = wmul(position.collateral, spotPrice_);

        if (
            (deltaDebt > 0 || deltaCollateral < 0) &&
            !_isCollateralized(calcTotalDebt(_calcDebt(position)), collateralValue, config.liquidationRatio)
        ) revert CDPVault__modifyCollateralAndDebt_notSafe();

        //quotaRevenueChange will always be equal to default uint value 0 for call originating from deposit.
        if (quotaRevenueChange != 0) {
            IPoolV3(pool).updateQuotaRevenue(quotaRevenueChange); // U:[PQK-15]
        }
        emit ModifyCollateralAndDebt(owner, collateralizer, creditor, deltaCollateral, deltaDebt);
    }
```

Notice the following variable declaration :
```
uint256 collateralValue = wmul(position.collateral, spotPrice_);
```

Note that the variables collateralValue, spotPrice_ and config are declared outside the scope of the following if statement, but these variables are not utilized in the code anywhere else except for inside the scope of the following if statement.

The if statement is only executed when the provided deltaDebt parameter is greater than 0 or if the provided deltaCollateral parameter is less than 0. Otherwise, it will short circuit the && and therefore also short circuit the following !_isCollateralized(calcTotalDebt(_calcDebt(position)), collateralValue, config.liquidationRatio), which happens to be the only place in the code that the mentioned variables are used. Therefore, in situations where the if statement short circuits, the computation done to initialize the config, spotPrice_, and collateralValue memory variable were a waste since the variables are never used in the code.

The function calls to CDPVault::deposit() and CDPVault::repay() will always call modifyCollateralAndDebt() with a deltaDebt value <=0 and a deltaCollateral parameter >= 0. Therefore, calls to CDPVault::deposit() and CDPVault::repay() will always short circuit the logic in the if statement and therefore there is no need to compute the values of config, spotPrice_, and collateralValue for these functions. In doing so we can save gas on the opcodes that would have been used to load the vaultConfig value from storage, get the price from the oracle, and compute the value of collateralValue as well as avoid potential memory expansion costs.

A more gas efficient implementation of the modifyCollateralAndDebt() is provided below.

```
function modifyCollateralAndDebt(
        address owner,
        address collateralizer,
        address creditor,
        int256 deltaCollateral,
        int256 deltaDebt
    ) public {
       
    .......

        if ((deltaDebt > 0 || deltaCollateral < 0)){
            VaultConfig memory config = vaultConfig;
            uint256 spotPrice_ = spotPrice();
            uint256 collateralValue = wmul(position.collateral, spotPrice_);
            if(!_isCollateralized(calcTotalDebt(_calcDebt(position)), collateralValue, config.liquidationRatio)){
                revert CDPVault__modifyCollateralAndDebt_notSafe();
            }
        }

        //quotaRevenueChange will always be equal to default uint value 0 for call originating from deposit.
        if (quotaRevenueChange != 0) {
            IPoolV3(pool).updateQuotaRevenue(quotaRevenueChange); // U:[PQK-15]
        }
        emit ModifyCollateralAndDebt(owner, collateralizer, creditor, deltaCollateral, deltaDebt);
    }
```

Here the computation for initializing the config, spotPrice_, and collateralValue variables are moved inside an if statement to avoid having to perform the calculation in situations where it is unnecessary.

Running "forge snapshot --diff" on the testing suite with the only difference being that the new more gas optimized implementation is used shows that this approach leads to a small gas cost decrease in a large number of tests which utilize the modifyCollateralAndDebt() function while not showing an increase in gas costs for any test.

For example we can see that we save 6836 gas in the test_deposit() and 2135 gas in the test_repay() functions just by using the new implementation :
test_deposit() (gas: -6836 (-0.136%))
test_deposit_from_proxy_collateralizer() (gas: -6262 (-1.895%))
test_repay() (gas: -2135 (-0.323%))
test_repay_with_interest() (gas: -2135 (-0.264%))
Overall gas change: -99458 (-0.045%)
