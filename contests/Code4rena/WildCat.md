 # Lines of code
 https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/WildcatMarketController.sol#L468-L488 
 
 https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/market/WildcatMarketConfig.sol#L149-L159
 
 # Vulnerability details

 When a market is initially deployed by the `WildcatMarketController` through the `deployMarket` method an internal method is called (`enforceParameterConstraints`) to check if the provided values for `annualInterestBips`, `reserveRatioBips` and other key parameters are between the `MIN/MAX` constraints set by the `WildcatMarketControllerFactory`.
 
 After a market has been deployed, its state variable for `annualInterestBips` can be changed through the `setAnnualInterestBips` method of the `WildcatMarketController` contract. In this method no check is made if the provided value for the new `annualInterestBips` violates the `MIN/MAX` constraints. Also no check is made in the underlying function being called(with the same name - `setAnnualInterestBips`) of the `WildcatMarketConfig` contract which updates the actual state variable. This way the boundaries set by the `MinimumAnnualInterestBips` & `MaximumAnnualInterestBips` state variables (set in the controller by the factory contract) can be violated, which leads to the following discrepancy for a market: `annualInterestBips`  `MaximumAnnualInterestBips` OR `annualInterestBips` < `MinimumAnnualInterestBips`
 
 ## Proof of Concept
 ```js
 /// 1. External function called by the borrower inside `WildcatMarketController`
 function setAnnualInterestBips(
         address market,
         uint16 annualInterestBips
     ) external virtual onlyBorrower onlyControlledMarket(market) {
            ...
         //No check for MIN/MAX violations of annualInterestBips is made
 
         WildcatMarket(market).setAnnualInterestBips(annualInterestBips);
     }
 
 /// 2. External function inside `WildcatMarketConfig` called by the above controller 
 function setAnnualInterestBips(
         uint16 _annualInterestBips
     ) public onlyController nonReentrant {
         MarketState memory state = _getUpdatedState();
        
         // This only checks for maximum possible value of BIPS 10000 - 100%
         if (_annualInterestBips  BIP) {
             revert InterestRateTooHigh();
         }
 
         //No check for MIN/MAX violations of annualInterestBips is made
 
         state.annualInterestBips = _annualInterestBips;
         _writeState(state);
         emit AnnualInterestBipsUpdated(_annualInterestBips);
     }
 ```
 
 ## Recommended Mitigation Steps
 At the beginning of the `setAnnualInterestBips` method in `WildcatMarketController` the following check could be added:
 
 ```
   assertValueInRange(
             annualInterestBips,
             MinimumAnnualInterestBips,
             MaximumAnnualInterestBips,
             AnnualInterestBipsOutOfBounds.selector
         );
 ```
 
 This is the same method used for validation during market deployment
 
 ## Assessed type
 Invalid Validation

