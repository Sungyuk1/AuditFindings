## Inaccurate comments/function names in ChefIncentivesController, EligibilityDataProvider, and MultiFeeDistribution

LoopFi implements its reward contracts heavily based on the Radiant protocol. However, it is mentioned in the documentation that it does not use the Radiant token for rewards but instead uses its own dLP token. However, it looks like the comments in many of these contracts were just directly copied over from the Radiant protocol. This leads to inaccurate comments regarding the functionality of many of the functions in the listed contracts. The comments found in these contracts should be updated to reflect the expected behavior in the Loopfi protocol instead of directly copying over the comments from the Radiant protocol. This will drastically help improve the readability of these contracts.

A quick example is listed below :
```
    /// @notice Duration of vesting RDNT
    uint256 public vestDuration;
```
The RDNT token is not the actual token that is vested in this contract. Instead it is the dLP token. Therefore, the comment should be updated to "Duration of vesting dLP". Directly copying the comments over from the Radiant protocol hurts the readability of the code, which goes against the core purpose of adding comments to our code which is to improve readability.

The same applies for many of the function names provided in these contracts. Even if the functionality stays the same, the variable/function names should be updated to reflect the behavior of the LoopFi protocol to improve the readability of the code.

Instances :
https://urldefense.com/v3/__https://github.com/code-423n4/2024-07-loopfi/blob/57871f64bdea450c1f04c9a53dc1a78223719164/src/reward/MultiFeeDistribution.sol*L75__;Iw!!HXCxUKc!ymVxikFwkZ_GCBFfY5q0IU0e6Nl5g5YMoGD-MMD_k1XqLL3C3ARW8863SeHfu5_WoZ_9Jy4Oc6IQU4JPv5NQQ4T6vX0$
https://urldefense.com/v3/__https://github.com/code-423n4/2024-07-loopfi/blob/57871f64bdea450c1f04c9a53dc1a78223719164/src/reward/MultiFeeDistribution.sol*L64__;Iw!!HXCxUKc!ymVxikFwkZ_GCBFfY5q0IU0e6Nl5g5YMoGD-MMD_k1XqLL3C3ARW8863SeHfu5_WoZ_9Jy4Oc6IQU4JPv5NQwVI_Xwo$
https://urldefense.com/v3/__https://github.com/code-423n4/2024-07-loopfi/blob/57871f64bdea450c1f04c9a53dc1a78223719164/src/reward/MultiFeeDistribution.sol*L209__;Iw!!HXCxUKc!ymVxikFwkZ_GCBFfY5q0IU0e6Nl5g5YMoGD-MMD_k1XqLL3C3ARW8863SeHfu5_WoZ_9Jy4Oc6IQU4JPv5NQ5CwSlP8$
https://urldefense.com/v3/__https://github.com/code-423n4/2024-07-loopfi/blob/57871f64bdea450c1f04c9a53dc1a78223719164/src/reward/MultiFeeDistribution.sol*L1218C8-L1221C28__;Iw!!HXCxUKc!ymVxikFwkZ_GCBFfY5q0IU0e6Nl5g5YMoGD-MMD_k1XqLL3C3ARW8863SeHfu5_WoZ_9Jy4Oc6IQU4JPv5NQBalForc$
https://urldefense.com/v3/__https://github.com/code-423n4/2024-07-loopfi/blob/57871f64bdea450c1f04c9a53dc1a78223719164/src/reward/ChefIncentivesController.sol*L213__;Iw!!HXCxUKc!ymVxikFwkZ_GCBFfY5q0IU0e6Nl5g5YMoGD-MMD_k1XqLL3C3ARW8863SeHfu5_WoZ_9Jy4Oc6IQU4JPv5NQy2VTKY8$
https://urldefense.com/v3/__https://github.com/code-423n4/2024-07-loopfi/blob/57871f64bdea450c1f04c9a53dc1a78223719164/src/reward/ChefIncentivesController.sol*L289__;Iw!!HXCxUKc!ymVxikFwkZ_GCBFfY5q0IU0e6Nl5g5YMoGD-MMD_k1XqLL3C3ARW8863SeHfu5_WoZ_9Jy4Oc6IQU4JPv5NQCDKV0Kk$
https://urldefense.com/v3/__https://github.com/code-423n4/2024-07-loopfi/blob/57871f64bdea450c1f04c9a53dc1a78223719164/src/reward/ChefIncentivesController.sol*L389__;Iw!!HXCxUKc!ymVxikFwkZ_GCBFfY5q0IU0e6Nl5g5YMoGD-MMD_k1XqLL3C3ARW8863SeHfu5_WoZ_9Jy4Oc6IQU4JPv5NQdw4TNXM$
https://urldefense.com/v3/__https://github.com/code-423n4/2024-07-loopfi/blob/57871f64bdea450c1f04c9a53dc1a78223719164/src/reward/ChefIncentivesController.sol*L787__;Iw!!HXCxUKc!ymVxikFwkZ_GCBFfY5q0IU0e6Nl5g5YMoGD-MMD_k1XqLL3C3ARW8863SeHfu5_WoZ_9Jy4Oc6IQU4JPv5NQXcdG_VM$
https://urldefense.com/v3/__https://github.com/code-423n4/2024-07-loopfi/blob/57871f64bdea450c1f04c9a53dc1a78223719164/src/reward/ChefIncentivesController.sol*L851__;Iw!!HXCxUKc!ymVxikFwkZ_GCBFfY5q0IU0e6Nl5g5YMoGD-MMD_k1XqLL3C3ARW8863SeHfu5_WoZ_9Jy4Oc6IQU4JPv5NQNxKm3o8$
https://urldefense.com/v3/__https://github.com/code-423n4/2024-07-loopfi/blob/57871f64bdea450c1f04c9a53dc1a78223719164/src/reward/ChefIncentivesController.sol*L947__;Iw!!HXCxUKc!ymVxikFwkZ_GCBFfY5q0IU0e6Nl5g5YMoGD-MMD_k1XqLL3C3ARW8863SeHfu5_WoZ_9Jy4Oc6IQU4JPv5NQzOWRIqk$
https://urldefense.com/v3/__https://github.com/code-423n4/2024-07-loopfi/blob/57871f64bdea450c1f04c9a53dc1a78223719164/src/reward/EligibilityDataProvider.sol*L39__;Iw!!HXCxUKc!ymVxikFwkZ_GCBFfY5q0IU0e6Nl5g5YMoGD-MMD_k1XqLL3C3ARW8863SeHfu5_WoZ_9Jy4Oc6IQU4JPv5NQALjgp7E$
https://urldefense.com/v3/__https://github.com/code-423n4/2024-07-loopfi/blob/57871f64bdea450c1f04c9a53dc1a78223719164/src/reward/EligibilityDataProvider.sol*L48__;Iw!!HXCxUKc!ymVxikFwkZ_GCBFfY5q0IU0e6Nl5g5YMoGD-MMD_k1XqLL3C3ARW8863SeHfu5_WoZ_9Jy4Oc6IQU4JPv5NQ0Zspt7I$
https://urldefense.com/v3/__https://github.com/code-423n4/2024-07-loopfi/blob/57871f64bdea450c1f04c9a53dc1a78223719164/src/reward/EligibilityDataProvider.sol*L174__;Iw!!HXCxUKc!ymVxikFwkZ_GCBFfY5q0IU0e6Nl5g5YMoGD-MMD_k1XqLL3C3ARW8863SeHfu5_WoZ_9Jy4Oc6IQU4JPv5NQkhBNNLc$
