# [HIGH-1] RestakeManager::calculateTVLs incorrectly calculates the withdraw queue balances, leading to heavily distorted TVL results

## Submission Link
https://github.com/code-423n4/2024-04-renzo-findings/issues/275

## Impact
RestakeManager calculates TVLs incorrectly leading to a distorted tvl value, which is used in essential protocol functions. The consequences can be:

* minting fewer/more shares when depositing/withdrawing depending on the `withdrawQueue` balances
* providing inflated/deflated mint rates by `BalancerRateProvider::getRate()`
* DOS on deposits and withdraws

## Vulnerability details
[`RestakeManager::calculateTVLs()` is used to calculate the total value](https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/RestakeManager.sol#L274) currently locked in the protocol. It factors:

* the value of all tokens in all operators
* the latest version of the contract adds the balances of the withdraw queue for each token

The calculated TVL returned from that function is essential when determining the mint/burn ratios for users and thus is vital for the proper functioning of the protocol. Currently it's being used in the following contracts:

* `RestakeManager::deposit` ([link](https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/RestakeManager.sol#L500-L504)) & `RestakeManager::depositETH` ([link](https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/RestakeManager.sol#L594)) - to check max limits and the amount of ezETH(shares) to mint
* `BalancerRateProvider::getRate` ([link](https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/RateProvider/BalancerRateProvider.sol#L31))  - Contract to Provide Mint Rate of LRT(ezETH)
* `WithdrawQueue::withdraw` ([link](https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/Withdraw/WithdrawQueue.sol#L217)) - calculates the ezETH to burn when withdrawing

The problem is in the following [new lines of the `calculateTVLs()` logic](https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/RestakeManager.sol#L316-L321):

```solidity
function calculateTVLs() public view returns (uint256[][] memory, uint256[] memory, uint256) {
        ....

        // Iterate through the ODs
        uint256 odLength = operatorDelegators.length;

        ....

        for (uint256 i = 0; i < odLength; ) {
            // Track the TVL for this OD
            uint256 operatorTVL = 0;

            ....

            uint256 tokenLength = collateralTokens.length;
            for (uint256 j = 0; j < tokenLength; ) {

                ....

                // Set the value in the array for this OD
                operatorValues[j] = renzoOracle.lookupTokenValue(
                    collateralTokens[j],
                    operatorBalance
                );

                ....

                // record token value of withdraw queue
                if (!withdrawQueueTokenBalanceRecorded) {
                    totalWithdrawalQueueValue += renzoOracle.lookupTokenValue(
 ---->                  collateralTokens[i],
 ---->                  collateralTokens[j].balanceOf(withdrawQueue)
                    );
                }

                unchecked {
                    ++j;
                }
            }

    }
```

As you can see above the function loops through every operator using the `i` index and inside every loop there is second nested loop that iterates through all tokens using the `j` index.

The problem is in the calculation of `withdrawQueue` token balances:

```solidity
                // record token value of withdraw queue
                if (!withdrawQueueTokenBalanceRecorded) {
                    totalWithdrawalQueueValue += renzoOracle.lookupTokenValue(
                     
                      //@audit - wrong index  - uses i instead of j
                      collateralTokens[i],
                      collateralTokens[j].balanceOf(withdrawQueue)
                    );
                }
```

Instead of using the index `j` for the `collateralTokens[]` array it uses the `i` index which corresponds to the operators array.

Depending on the difference in count between the operators and tokens, the consequences can be various:

* Complete DOS of the deposit/withdraw functions -  In case the `operatorDelegators` array is bigger than the `collateralTokens` array the function will revert with `array out-of-bounds` error.
* Minting/Burning more/less shares than necessary - In case the `operatorDelegators` array is smaller than `collateralTokens` array it will repeatedly accumulate the first token only -> which can  lead to lower/higher balances for the `withdrawQueue`, which will make the deposit to TVL ratio bigger/smaller.

## Tools Used
Manual review

## Recommended Mitigation Steps
Fix the issue by using the token index instead of the operators one:

```solidity
if (!withdrawQueueTokenBalanceRecorded) {
    totalWithdrawalQueueValue += renzoOracle.lookupTokenValue(
        collateralTokens[j], // <- use j, not i
        collateralTokens[j].balanceOf(withdrawQueue)
    );
}
```

## Assessed type
Other



# [MEDIUM-1] Token deposits and withdraws below the buffer deficit will fail

## Submission Link
https://github.com/code-423n4/2024-04-renzo-findings/issues/182


## Impact
All deposit transactions with amounts that are less than the buffer deficit will revert. When deficits are high it's possible that most of the deposits will be DOSed.

Withdraw completions will also be prevented in case the amount left does not constitute a whole share.

## Vulnerability details
In the latest version of `RestakeManager` the `deposit()` function has been modified so that the buffer deficit inside `withdrawQueue` is checked and in case there is any, the deposit amount is used to fill that buffer:

https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/RestakeManager.sol#L546-L556

```solidity
function deposit(
        IERC20 _collateralToken,
        uint256 _amount,
        uint256 _referralId
    ) public nonReentrant notPaused {
       
        .....

        // Get the value of the collateral token being deposited
        uint256 collateralTokenValue = renzoOracle.lookupTokenValue(_collateralToken, _amount);

        .....

        // Check the withdraw buffer and fill if below buffer target
        uint256 bufferToFill = depositQueue.withdrawQueue().getBufferDeficit(
            address(_collateralToken)
        );

        if (bufferToFill > 0) {
            bufferToFill = (_amount <= bufferToFill) ? _amount : bufferToFill;
            // update amount to send to the operator Delegator
            _amount -= bufferToFill;

            ....

            // fill Withdraw Buffer via depositQueue
            depositQueue.fillERC20withdrawBuffer(address(_collateralToken), bufferToFill);
        }

        // Approve the tokens to the operator delegator
        _collateralToken.safeApprove(address(operatorDelegator), _amount);

        // Call deposit on the operator delegator
        operatorDelegator.deposit(_collateralToken, _amount);

        ....
    }
```

The amount that is left after the buffer has been filled is sent to the operator delegator to be deposited through the `StrategyManager` contract inside `EigenLayer`.

https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/Delegation/OperatorDelegator.sol#L143-L169

```solidity
 function _deposit(IERC20 _token, uint256 _tokenAmount) internal returns (uint256 shares) {
        // Approve the strategy manager to spend the tokens
        _token.safeApprove(address(strategyManager), _tokenAmount);

        // Deposit the tokens via the strategy manager
        return
            strategyManager.depositIntoStrategy(tokenStrategyMapping[_token], _token, _tokenAmount);
    }
```

The `strategyManager.depositIntoStrategy()` function internally calls the `deposit()` function of the respective strategy where the funds actually get stored:

https://github.com/Layr-Labs/eigenlayer-contracts/blob/15b679b08e7a3589ff83a5e84076ca4e0a00ec0b/src/contracts/core/StrategyManager.sol#L333

```solidity
function _depositIntoStrategy(
        address staker,
        IStrategy strategy,
        IERC20 token,
        uint256 amount
    ) internal onlyStrategiesWhitelistedForDeposit(strategy) returns (uint256 shares) {
        ....

        // deposit the assets into the specified strategy and get the equivalent amount of shares in that strategy
        shares = strategy.deposit(token, amount);

        ....

        return shares;
    }
```

`Strategy.deposit()` calculates the shares to mint.

https://github.com/Layr-Labs/eigenlayer-contracts/blob/dbfa12128a41341b936f3e8da5d6da58c6233877/src/contracts/strategies/StrategyBase.sol#L115-L118

```solidity
function deposit(
        IERC20 token,
        uint256 amount
    ) external virtual override onlyWhenNotPaused(PAUSED_DEPOSITS) onlyStrategyManager returns (uint256 newShares) {
        ....
        newShares = (amount * virtualShareAmount) / virtualPriorTokenBalance;

        // extra check for correctness / against edge case where share rate can be massively inflated as a 'griefing' sort of attack
        require(newShares != 0, "StrategyBase.deposit: newShares cannot be zero");

        ....

        return newShares;
    }
```

As you can see `Strategy.deposit()` reverts in case the `newShares` to mint is 0.

And this is exactly the problem that will block user deposits inside `StakeManager`. The issue arises from the fact that `StakeManager.deposit()` never checks before the call to `operatorDelegator.deposit()` if the `_amount` [that is left after the buffer deficit has been deducted is 0](https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/RestakeManager.sol#L548-L549):

```solidity
// update amount to send to the operator Delegator
   _amount -= bufferToFill;
```

If `_amount` is zero this means that the call chain `StakeManager.deposit()` -> `strategyManager.depositIntoStrategy()` -> `Strategy.deposit()` passes 0 as `amoun`t and finally [the shares calculation inside `Strategy.deposit()` equals to 0 which as explained above reverts the execution](https://github.com/Layr-Labs/eigenlayer-contracts/blob/dbfa12128a41341b936f3e8da5d6da58c6233877/src/contracts/strategies/StrategyBase.sol#L115C9-L118C83).

```solidity
newShares = (amount * virtualShareAmount) / virtualPriorTokenBalance;

// extra check for correctness / against edge case where share rate can be massively inflated as a 'griefing' sort of attack
require(newShares != 0, "StrategyBase.deposit: newShares cannot be zero");
```

Considering that the buffer deficits will probably be set quite high in order to cover the rising number of accounts that would want to withdraw, the chances are high that a lot of deposits won't be above that deficit, meaning no value will be left after that to send to EigenLayer, making all those deposits fail and leaving the buffer empty because of everything explained above

EigenPod withdraws through `EigenPod::completeQueuedWithdrawal()` [are also affected by the same problem](https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/Delegation/OperatorDelegator.sol#L307-L308). Although there is a 0 amount check here, still no validation is made if the amount left after the buffer has been filled is enough for minting of a whole share.

## Tools Used
Manual Review

## Recommended Mitigation Steps
Make sure to validate inside `RestakeManager.deposit()` that the amount of shares to be minted from the leftover amount are above 0 before calling `operatorDelegator.deposit()`:

```solidity
function deposit(
        IERC20 _collateralToken,
        uint256 _amount,
        uint256 _referralId
    ) public nonReentrant notPaused {

     ....

        if (bufferToFill > 0) {
            bufferToFill = (_amount <= bufferToFill) ? _amount : bufferToFill;
            // update amount to send to the operator Delegator
            _amount -= bufferToFill;

            ....
        }

+        uint256 sharesToMint = 
operatorDelegator.tokenStrategyMapping[_collateralToken].underlyingToSharesView(_amount );

+        if(sharesToMint > 0){

+            // Approve the tokens to the operator delegator
+            _collateralToken.safeApprove(address(operatorDelegator), _amount);

+            // Call deposit on the operator delegator
+            operatorDelegator.deposit(_collateralToken, _amount);
        }

        // Calculate how much ezETH to mint
        uint256 ezETHToMint = renzoOracle.calculateMintAmount(
            totalTVL,
            collateralTokenValue,
            ezETH.totalSupply()
        );

        .....
}

The same remediation applies for `OperatorDelegator::completeQueuedWithdrawal()`
```

## Assessed type
Dos