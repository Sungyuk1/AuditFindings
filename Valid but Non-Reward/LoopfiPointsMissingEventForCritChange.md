### Missing Event for critical parameters change
Events help non-contract tools to track changes, and events prevent users from being surprised by changes.

This is another instance of NC-10 from the 4naly3er report.

https://urldefense.com/v3/__https://github.com/code-423n4/2024-05-loop/blob/40167e469edde09969643b6808c57e25d1b9c203/src/PrelaunchPoints.sol*L364__;Iw!!HXCxUKc!2HvilXOnu_pB9NdEVPWv4GKz4lZQZ8cQPk2dyWWT5NJ7zPQFYKOcpt3lahi4lJLsmCtaZNZruCrKU4PsAazGYgG_diI$

```solidity
File: src/PrelaunchPoints.sol

364:     function allowToken(address _token) external onlyAuthorized {
      isTokenAllowed[_token] = true;
   }

```