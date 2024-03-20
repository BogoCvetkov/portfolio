
# [M-1] Permit functions inside JalaRouter02 will not work
 

## Summary
`JalaRouter02` is a fork of the UniSwap V2 router and as such it implements removal of liquidity through permits. However the LP token `JalaPair` has been stripped from using permits and can only use regular approvals. As a result all attempts to remove liquidity using permits will revert.

## Vulnerability Detail
[JalaRouter02](https://github.com/sherlock-audit/2024-02-jala-swap/blob/main/jalaswap-dex-contract/contracts/JalaRouter02.sol) has the following permit routes defined:

* `removeLiquidityWithPermit`
* `removeLiquidityETHWithPermit`
* `removeLiquidityETHWithPermitSupportingFeeOnTransferTokens`

However the LP token `JalaPair` that gets burned when removing liquidity does not implement the `permit()` function: https://github.com/sherlock-audit/2024-02-jala-swap/blob/main/jalaswap-dex-contract/contracts/JalaPair.sol#L20

Because of this all calls to the permit methods will revert

## Impact
Permit methods defined inside `JalaRouter02` will not work

## Code Snippet
## Tool used
Manual Review

## Recommendation
Either add the permit functionality to `JalaPair` or remove the methods from `JalaRouter02`

The sponsors mentioned that they intentionally skipped the permit functionality inside the `JalaPair`, so maybe removing the permit methods is the option here.
