
# [M-1] User can mint dollars even if the price is beyond the mintPriceThreshold

## Summary
`LibUbiquityPool.mintDollar()` includes the following check:

```solidity
  function mintDollar(
        uint256 collateralIndex,
        uint256 dollarAmount,// based on logic below should be 1e6
        uint256 dollarOutMin,
        uint256 maxCollateralIn
    )internal
        collateralEnabled(collateralIndex)
        returns (uint256 totalDollarMint, uint256 collateralNeeded)
    {

...... 

  // update Dollar price from Curve's Dollar Metapool
        LibTWAPOracle.update();
        // prevent unnecessary mints
        require(
            getDollarPriceUsd() >= poolStorage.mintPriceThreshold,
            "Dollar price too low"
        );

......
}
```

The idea of `mintPriceThreshold` is to prevent mints at prices that wouldn't be acceptable to the protocol. However any mints that occur in the same block that a pool gets set will bypass that check.

## Vulnerability Detail
Before the check for `poolStorage.mintPriceThreshold` is made `LibTWAPOracle.update()` is called so that the latest price get's fetched from `TWAPOracle`. This is the function for updating the price:

```solidity
function update() internal {
        TWAPOracleStorage storage ts = twapOracleStorage();
        (
            uint256[2] memory priceCumulative,
            uint256 blockTimestamp
        ) = currentCumulativePrices();
   
        if (blockTimestamp - ts.pricesBlockTimestampLast > 0) {
            // get the balances between now and the last price cumulative snapshot
            uint256[2] memory twapBalances = IMetaPool(ts.pool)
                .get_twap_balances(
                    ts.priceCumulativeLast,
                    priceCumulative,
                    blockTimestamp - ts.pricesBlockTimestampLast
                );

            // price to exchange amountIn Ubiquity Dollar to 3CRV based on TWAP
            ts.price0Average = IMetaPool(ts.pool).get_dy(
                0,
                1,
                //@audit-ok - since dollar is 1e6 decimals is 1e18 accurate as amount in -!!!!
                // this is price for 1e18/1e6 tokens, not a single ubiquity dollar
                1 ether,
                twapBalances
            );

            // price to exchange amountIn 3CRV to Ubiquity Dollar based on TWAP
            ts.price1Average = IMetaPool(ts.pool).get_dy(
                1,
                0,
                1 ether,
                twapBalances
            );
            // we update the priceCumulative
            ts.priceCumulativeLast = priceCumulative;
            ts.pricesBlockTimestampLast = blockTimestamp;
        }
    }
```

It's important to notice here that the price is updated only if an update was not made in the same block already -`if (blockTimestamp - ts.pricesBlockTimestampLast 0)`

When a pool is being set in the oracle, the `ts.pricesBlockTimestampLast` is set to the latest recorded twap timestamp and prices are hardcoded to 1 ether:

```solidity
function setPool(address _pool, address _curve3CRVToken1) internal {
       .......
        ts.priceCumulativeLast = IMetaPool(_pool).get_price_cumulative_last();
        ts.pricesBlockTimestampLast = IMetaPool(_pool).block_timestamp_last();
        ts.pool = _pool;
        // dollar token is inside the diamond
        ts.token1 = _curve3CRVToken1;
        ts.price0Average = 1 ether;
        ts.price1Average = 1 ether;
    }
```

This means that for the duration of that block no more price updates will happen and dollar price will be 1$ regardless of the real price(which is actually around 1.34$ according to the current Curve pool).

This means that the update before the check for `poolStorage.mintPriceThreshold` in `mintDollar` will not do anything and the price returned will be 1$ (instead the real one 1.34).

So an advantageous user/bot might monitor the pool for the `setPool()` transaction and mint quite a big amount of dollar tokens at a discounted rate. After that he can swap those tokens in the Curve pool and profit from the difference 0.34 "cents" per Dollar.

Considering the pool itself has low liquidity of 40K, this might distort the prices heavily. I'm quoting a comment from the sponsors provided in the `Audit List` document under `Things to double-check`:

"Check that [LibTWAPOracle](https://github.com/ubiquity/ubiquity-dollar/blob/9e41ca5626d7376451f286d04106acead1f9e71d/packages/contracts/src/dollar/libraries/LibTWAPOracle.sol) updates average prices correctly. The old Curve's [metapool](https://etherscan.io/address/0x20955CB69Ae1515962177D164dfC9522feef567E) (which we plan to redeploy) has 40k Dollars in liquidity so we should make sure that it's hard to manipulate Curve's TWAP with 40k of liquidity which is pretty low."

## Impact
Advantageous parties can mint dollars at discounted prices and possibly affect the prices the Curve pool if enough tokens are minted.

## Code Snippet
https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibUbiquityPool.sol#L344-L348 https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibTWAPOracle.sol#L68 https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibTWAPOracle.sol#L52-L58

## Tool used
Manual Review

## Recommendation
Consider restricting minting for the first couple of block after the pool is set so that there is enough time for the prices to update.


