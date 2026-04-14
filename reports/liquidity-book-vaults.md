# Metropolis / liquidity-book-vaults


## Protocol Summary

Maker Vault Code Introduction: Maker Vaults are fully on-chain, non-custodial smart contracts that deploy and manage liquidity into DLMM pools. Managed by wallets, these vaults simplify liquidity provisioning while ensuring transparency.



[Ethereum][Solidity][DeFi][DLMM]

---

### [M-01] AUM Fee Limit Discrepancy Between Factory and Strategy Contracts

#### Summary
A discrepancy  between the maximum AUM fee limits in the `VaultFactory` and `Strategy` contracts. While the Factory contract allows setting fees up to 30%, the Strategy contract enforces a hardcoded maximum of 25%, causing potential reverts when operators attempt to set fees within the Factory's allowed range.

#### Finding Description
The issue stems from an inconsistency in fee limits between two  contracts:

1. In `Strategy.sol`, the maximum AUM fee is hardcoded to 25%:
```solidity
/liquidity-book-vaults/src/Strategy.sol:56
56:     uint256 private constant _MAX_AUM_ANNUAL_FEE = 0.25e4; // 25%
```

2. In `VaultFactory.sol` the Maximum AUM Fee is hardcoded to 30%:
```solidity
/liquidity-book-vaults/src/VaultFactory.sol:40
40:     uint256 private constant MAX_AUM_FEE = 0.3e4; // 30% fee
```
When the Operator deploy new market maker vault via Factory and provide 30% has AUM fee , The `createMarketMakerOracleVault` will calls 
`setPendingAumAnnualFee` to set the Pending Fee But this function will revert Because the `_MAX_AUM_ANNUAL_FEE=25%` and the Operator has Provided `30%`
```solidity
/liquidity-book-vaults/src/Strategy.sol:374
374:     function setPendingAumAnnualFee(uint16 pendingAumAnnualFee) external override onlyFactory {
375:         if (pendingAumAnnualFee > _MAX_AUM_ANNUAL_FEE) revert Strategy__InvalidFee();
376: 
377:         _pendingAumAnnualFeeSet = true;
378:         _pendingAumAnnualFee = pendingAumAnnualFee;
379: 
380:         emit PendingAumAnnualFeeSet(pendingAumAnnualFee);
381:     }
```

This discrepancy can lead to unexpected reverts and expose inconsistency in the protocol's fee-setting mechanism.

#### Impact  Explanation
The impact is Medium/Low As there is no lose of assets involved expect the gas cost. But will create confusion about the actual maximum allowed fee and Operators may experience unexpected reverts when setting fees they believe are valid.
#### Likelihood Explanation
Likelihood is High/Medium depending on Operator provided Fee value, However it will always revert if operator set the Fee to `30%`.
#### Proof of Concept
Add following function to `01_POC.t.sol` and run with command : `forge test --mt test_AUM_Fee_Inconsistency -vvv`.

```solidity
    function test_AUM_Fee_Inconsistency() public {
        hoax(max, 50 ether);
        vm.expectRevert(IStrategy.Strategy__InvalidFee.selector);
        (address vault, address strategy) =
            _factory.createMarketMakerOracleVault{value: 50 ether}(ILBPair(S_USDC_E_PAIR), 0.3e4);

        (, uint256 oracleLength, , , ) = ILBPair(S_USDC_E_PAIR).getOracleParameters();
        if (oracleLength < 1) {
            ILBPair(S_USDC_E_PAIR).increaseOracleLength(1);
        }

    }
```
 
#### Recommendations
1. Align the maximum AUM fee limits between Factory and Strategy contracts
2. Either:
   - Increase Strategy's `_MAX_AUM_ANNUAL_FEE` to 30% to match Factory
   - Decrease Factory's maximum allowed fee to 25% to match Strategy


#### Related Files

- `src/Strategy.sol` (lines 56-56)
- `src/VaultFactory.sol` (lines 40-40)

---

### [M-02] Loss of Rewards in Final Round When All Users Have Queued Withdrawals


#### Summary
When all active users queued withdrawal and the current round is final which means after this round there will be no deposit in `LbPair` and the strategy will be shutdown the rewards accumulated in this round will be lost. This occurs because the rewards are harvested but cannot be distributed to users who have already queued their withdrawals.

#### Finding Description
The vulnerability arises from a sequence of events where users queue their withdrawals prior to the final rebalance operation. During this final rebalance call, rewards are harvested as per normal protocol operation. However, since all users have already queued their withdrawals, their shares got _transfer to strategy and will be burned on `redeemWithdrawal` call. 
```solidity
/liquidity-book-vaults/src/BaseVault.sol:477
477:     function queueWithdrawal(uint256 shares, address recipient)
...
485:     {
486:         // Check that the strategy is set.
487:         address strategy = address(_strategy);
488:         if (strategy == address(0)) revert BaseVault__InvalidStrategy();
489: 
490:         _updatePool();
491: 
492:         // Transfer the shares to the strategy, will revert if the user does not have enough shares.
493:         _transfer(msg.sender, strategy, shares);
494: 
495:         _modifyUser(msg.sender, -int256(shares));
496: 
```
This creates a  situation where the harvested rewards from the final round remain trapped in the vault, as there are no active shares available for reward distribution. Consequently, when users proceed to redeem their queued withdrawals, they are unable to claim these final round rewards, resulting in a complete loss of rewards earned during the last operational period of the strategy.

#### Impact  Explanation
The impact is a complete loss of rewards earned during the final round. The impact is high due to lose of rewards.
#### Likelihood Explanation
The Likelihood is some where between medium/low ,if the operator calls the `harvestRewards` before each `queueWithdrawal` in the last round then the lose will be minimal , but it is no guaranteed that the `harvestRewards` will be called in this way. So in final round if all users queued withdrawal it is very highly likely that we lose some rewards. But the Likelihood is limited to final/last round.

#### Proof of Concept
Add following function to `OracleRewardVault.t.sol` and run with command :`forge test --mt testRewardLossOnLastRoundIfAllUsersHaveQueuedWithdrawals -vvv`.

```solidity
function testRewardLossOnLastRoundIfAllUsersHaveQueuedWithdrawals() public {
    // Setup vault and strategy
    hoax(max, 75 ether);
    (address vault, address strategy) = _factory.createMarketMakerOracleVault{value: 75 ether}(
        ILBPair(IOTA_USDC_E_PAIR), 
        0.1e4
    );

    // Multiple users deposit
    depositToVault(address(rewardVault), alice, tokenX);
    depositToVault(address(rewardVault), bob, tokenX);
    
    // First rebalance
    vm.warp(block.timestamp + 1 days);
    _rebalanceSingleSide(max, IOTA_USDC_E_PAIR, strategy);
    
    // Second rebalance
    vm.warp(block.timestamp + 1 days);
    _rebalanceSingleSide(max, IOTA_USDC_E_PAIR, strategy);

    // Users queue withdrawals
    vm.prank(alice);
    IBaseVault(vault).queueWithdrawal(aliceShares, alice);
    vm.prank(bob);
    IBaseVault(vault).queueWithdrawal(bobShares, bob);

    // Final rebalance
    vm.warp(block.timestamp + 1 days);
    _rebalanceReset(max, strategy);

    // Process withdrawals
    vm.startPrank(alice);
    IOracleVault(vault).redeemQueuedWithdrawal(0, alice);
    rewardVault.claim();
    vm.stopPrank();
    
    vm.startPrank(bob);
    IOracleVault(vault).redeemQueuedWithdrawal(0, bob);
    rewardVault.claim();
    vm.stopPrank();

    // Verify rewards are stuck in vault
    uint256 totalRewardsStuckInVault = afterLastRoundVaultBalance - beforeLastRoundVaultBalance;
    assertGt(totalRewardsStuckInVault, 0);
}
```

#### Recommendations
The Best fix would be to recheck the reward distribution mechanism as it will always result in lose of rewards between the round , and after the final round if all user have queued withdrawal.Add some function which will distribute the rewards among the user who were active before the last round.


#### Related Files

- `src/BaseVault.sol` (lines 493-493)
- `src/BaseVault.sol` (lines 495-495)
- `src/OracleRewardVault.sol` (lines 232-233)
- `src/Strategy.sol` (lines 329-329)

---

### [M-03] Incorrect Reward Distribution Due to Missing Harvest from `LbPair` in `BaseVault::Deposit` function


#### Summary
The `deposit` function fails to harvest rewards before updating `userDebt` record, So it will read the outdated `_accRewardsPerShare` and `_extraRewardsPerShare` values. This results in later depositors receiving disproportionately higher/same rewards as previous/current users.

#### Finding Description
The issue occurs because:
1. The `deposit` function doesn't call `_harvestRewards` before updating `userDebt`.
2. This causes `_accRewardsPerShare` and `_extraRewardsPerShare` to be outdated.
3. When rewards are later harvested, they are distributed based on current `userDebt` counts without accounting for the time when assets were actually deposited
4. This creates a situation where later depositors receive rewards for periods before their deposit.

#### Impact Explanation
The impact is loss of rewards for early depositors So it is High. In the normal case:
1. Rewards are harvested after rebalance
2. They can also be called by Operator or Strategy via `harvestRewards()`
3. The protocol applies new `_accRewardsPerShare` and `_extraRewardsPerShare` when `harvestRewards` is called
4. Between these calls, `_accRewardsPerShare` and `_extraRewardsPerShare` remain stale
5. If there is significant time between i.e the first and second depositor, the second user will receive more rewards than intended

#### Likelihood Explanation:
Likelihood is High as the `harvestRewards` in case of rebalance will be called after 72 hours as recommended in Dapp.


#### Proof of Concept
Add following function to `OracleRewardVault.t.sol` and run with command : `forge test --mt test_stealing_of_rewards -vvv`.

```solidity
function test_stealing_of_rewards() public {
    // Setup vault and oracle
    hoax(max, 75 ether);
    (address vault, address strategy) = _factory.createMarketMakerOracleVault{value: 75 ether}(
        ILBPair(IOTA_USDC_E_PAIR), 
        0.1e4
    );

    // Bob deposits first
    deal(bob, amountX);
    vm.startPrank(bob);
    IWNative(WNATIVE).deposit{value: amountX}();
    tokenX.approve(vault, amountX);
    rewardVault.deposit(amountX, amountY, 0); // No harvest called
    vm.stopPrank();

    skip(1 days);
    _rebalanceSingleSide(max, IOTA_USDC_E_PAIR, strategy);

    // Alice deposits after first rebalance
    skip(1 days);
    deal(alice, amountX);
    vm.startPrank(alice);
    IWNative(WNATIVE).deposit{value: amountX}();
    tokenX.approve(vault, amountX);
    rewardVault.deposit(amountX, amountY, 0); // No harvest called
    vm.stopPrank();

    // Second rebalance
    _rebalanceSingleSide(max, IOTA_USDC_E_PAIR, strategy);

    // Claim rewards
    vm.prank(bob);
    rewardVault.claim();
    uint256 bobRewards = Strategy(strategy).getRewardToken().balanceOf(bob);

    vm.prank(alice);
    rewardVault.claim();
    uint256 aliceRewards = Strategy(strategy).getRewardToken().balanceOf(alice);

    // Alice receives more rewards despite depositing later
    assertGt(aliceRewards, bobRewards);
}
```

#### Recommendations
Can be fixed via following suggestion:
 Update `_accRewardsPerShare` and `_extraRewardsPerShare` at the time of deposit via calling `harvestRewards()`.


#### Related Files

- `src/BaseVault.sol` (lines 390-390)
- `src/BaseVault.sol` (lines 398-398)
- `src/OracleRewardVault.sol` (lines 221-221)

---

### [M-04] Queued withdrawal will DoS the `setStrategy` Function


## Summary
The `setStrategy` function will result in DoS if there are queued withdrawals,So it allows an attacker to prevent legitimate strategy changes by exploiting the reentrancy guard through queued withdrawals. This vulnerability is similar to the previously identified issue in `setEmergencyMode` and can be exploited on blockchains with public mempools like Zk-EVM.

## Finding Description
The vulnerability occurs in the following sequence:

1. The `setStrategy` function in `BaseVault.sol` has a `nonReentrant` modifier
2. When changing strategies, it calls `withdrawAll()` on the current strategy
3. `withdrawAll()` processes queued withdrawals through `_transferAndExecuteQueuedAmounts`
4. This creates a  call back to `BaseVault::executeQueuedWithdrawals` in the vault which also has `nonReentrant` modifier.

The attack can be executed by:
1. Depositing a small amount into the vault
2. Waiting for a rebalance to occur
3. Monitoring the mempool for `setStrategy` transactions
4. Frontrunning the `setStrategy` call with a withdrawal queue transaction
5. This causes the strategy change to revert due to the reentrancy guard

## Impact Explanation Explanation
The Impact is high The protocol cannot change strategies when needed. The attack can be repeated indefinitely by sand witching the rebalance call with  `cancelQueueWithdrawal`  and `queueWithdrawal`.it could  Affects all vaults in the protocol which are deployed on blockchains which provide public memPool.

## Likelihood
 The Likelihood is high as any attacker can exploit it easily and also with out any exploit it could be generated when there is a queued withdrawals for the next round.
## Proof of Concept
Add following function to `01_POC.t.sol` and run with command : `forge test --mt testPOc_DOS_SetStrategy -vvv`.

```solidity
function testPOc_DOS_SetStrategy() public {
    // Setup vault and strategy
    hoax(max, 50 ether);
    (address vault, address strategy) = _factory.createMarketMakerOracleVault{value: 50 ether}(
        ILBPair(S_USDC_E_PAIR), 
        0.1e4
    );

    // Setup oracle
    (, uint256 oracleLength, , , ) = ILBPair(S_USDC_E_PAIR).getOracleParameters();
    if (oracleLength < 1) {
        ILBPair(S_USDC_E_PAIR).increaseOracleLength(1);
    }

    // Legitimate users deposit
    depositToVault(vault, alice, 5e18, 60e6);
    uint256 shares = IOracleVault(vault).balanceOf(alice);
    depositToVault(vault, bob, 5e18, 60e6);
    uint256 sharesBob = IOracleVault(vault).balanceOf(bob);

    // Users queue withdrawals
    vm.prank(alice);
    IBaseVault(vault).queueWithdrawal(shares, alice);
    vm.prank(bob);
    IBaseVault(vault).queueWithdrawal(sharesBob, bob);

    // Attacker setup
    address attacker = makeAddr("attacker");
    depositToVault(vault, attacker, 1e3, 1e3);
    uint256 sharesAttacker = IOracleVault(vault).balanceOf(attacker);

    // First rebalance
    _rebalance(max, S_USDC_E_PAIR, strategy, vault);

    // Attacker queues withdrawal
    vm.prank(attacker);
    IBaseVault(vault).queueWithdrawal(sharesAttacker, attacker);

    // Alice redeems
    vm.prank(alice);
    IOracleVault(vault).redeemQueuedWithdrawal(0, alice);

    // Create new strategy
    address newStrategy = _factory.createDefaultStrategy(IBaseVault(vault));

    // Strategy change attempt fails
    vm.expectRevert("ReentrancyGuard: reentrant call");
    _factory.linkVaultToStrategy(IOracleVault(vault), newStrategy);

    // Verify strategy remains unchanged
    assertEq(
        address(IOracleVault(vault).getStrategy()), 
        strategy, 
        "Strategy should not be changed"
    );
}
```
## Recommendations
One Fix could be to Remove the `nonReentrant` modifier from `setStrategy`.


## Related Files

- `src/BaseVault.sol` (lines 651-651)
- `src/BaseVault.sol` (lines 692-692)
- `src/BaseVault.sol` (lines 706-706)
- `src/Strategy.sol` (lines 801-801)

---
