# Super DCA Liquidity Network

## Protocol Summary

This contest reviews the contracts that make up the Super DCA Liquidity Network, powered by Uniswap V4 Hooks. It focuses on the security and economic soundness of the contract suite coordinating dynamic fees, emitting DCA tokens using a Curve-style gauge, and locking liquidity when new tokens are listed.




[Ethereum][Solidity][DeFi][Uniswap V4 Hook]





### [H-01] The rewards Will be Lost if an attacker front run the accrueReward transaction with stake/unStake 

#### Summary
...

#### Root Cause

We update the lastRewardsIndex of token with out any minting whenever user stake or unStake from SuperDCAStaking contract :

/SuperDCAStaking.sol:204  

```solidity
204:     function stake(address token, uint256 amount) external override {  
...  
212:         // Update global reward index to current time  
213:         _updateRewardIndex();  
...  
218:         // Update token bucket accounting and user stakes  
219:         TokenRewardInfo storage info = tokenRewardInfoOf[token];  
220:         info.stakedAmount += amount;  
221:         info.lastRewardIndex = rewardIndex; // @audit Set to last reward index of token without minting the tokens for rewards  
222:   
223:         totalStakedAmount += amount;  
224:         userStakes[msg.sender][token] += amount;  
```

---

/SuperDCAStaking.sol:237  

``` solidity
237:     function unstake(address token, uint256 amount) external override {  
...  
241:         // Check both token bucket and user balances are sufficient  
242:         TokenRewardInfo storage info = tokenRewardInfoOf[token];  
...  
246:         // Update global reward index to current time  
247:         _updateRewardIndex();  
248:   
249:         // Update token bucket accounting and user stakes  
250:         info.stakedAmount -= amount;  
251:         info.lastRewardIndex = rewardIndex; // @audit Set to last reward index of token without minting the tokens for rewards  
```

As shown in the code, when a user stakes or unstakes, the global rewardIndex is updated, and the token’s lastRewardIndex is also set to the current rewardIndex.
However, updating lastRewardIndex without minting rewards causes the loss of unminted tokens.

This happens because the accrueReward function in the gauge contract uses lastRewardIndex to calculate the pending rewards, assuming tokens were already minted at that point. As a result, the skipped rewards are never distributed.

/SuperDCAStaking.sol:276  

```
276:     function accrueReward(address token) external override onlyGauge returns (uint256 rewardAmount) {  
277:         // Always update the global reward index to current time  
278:         _updateRewardIndex();  
279:   
280:         TokenRewardInfo storage info = tokenRewardInfoOf[token];  
281:         if (info.stakedAmount == 0) return 0;  
282:   
283:         // Calculate reward delta for the specific token bucket  
284:         uint256 delta = rewardIndex - info.lastRewardIndex;  
285:         if (delta == 0) return 0;  
286:   
287:         // Compute and return reward amount for distribution  
288:         rewardAmount = Math.mulDiv(info.stakedAmount, delta, 1e18);  
289:   
290:         // Update the token's last reward index to current index  
291:         info.lastRewardIndex = rewardIndex;  
292:         return rewardAmount;  
293:     }  
```

At line 284, the contract calculates the delta between rewardIndex and lastRewardIndex, determines the rewardAmount, and then updates the token’s lastRewardIndex. This update is correct within accrueReward, but performing it inside stake or unStake is incorrect.

- [SuperDCAStaking.sol#L221](https://github.com/sherlock-audit/2025-09-super-dca/blob/main/super-dca-gauge/src/SuperDCAStaking.sol#L221)
- [SuperDCAStaking.sol#L251](https://github.com/sherlock-audit/2025-09-super-dca/blob/main/super-dca-gauge/src/SuperDCAStaking.sol#L251)
- [SuperDCAStaking.sol#L284](https://github.com/sherlock-audit/2025-09-super-dca/blob/main/super-dca-gauge/src/SuperDCAStaking.sol#L284)

#### Internal Pre-conditions

...

#### External Pre-conditions

...

#### Attack Path

Alice stakes 100 WETH in SuperStakingContract.
After 2 hours the contract accrues rewards.
An attacker front-runs a AddLiquidity/RemoveLiquidity call on the DCA token pool by staking 1 wei (and unstaking in the same block), enabling flashloan style attack vectors.
After the attacker’s tx, pendingRewards becomes zero. the rewards accrued over the 2 hour period are lost.

#### Impact

An attacker can grief unclaimed rewards via front running AddLiquidity/RemoveLiquidity with a minimum stake of just 1 wei.

#### PoC

Add the Following test case to OptimismStakingIntegration.t.sol and run with command : forge test --mt testFork_rewards_tokens_hack -vvv

```solidity
    function testFork_rewards_tokens_hack() public {  
        // ---- Arrange ----  
        _setupTokenAndStake(user1, WETH, STAKE_AMOUNT, POSITION_AMOUNT0, POSITION_AMOUNT1);  
        // ---- Act ----  
        _simulateTimePass(3600); // 1 hour  
        uint256 PendingRewards = staking.previewPending(WETH);  
        // Trigger reward index update by staking more  
        assertEq(PendingRewards , 13888888888888886000 , "Pending rewards should be 13888888888888886000");  
  
        _performStake(user2, WETH, 1);  
        _simulateTimePass(360); // 1 hour  
  
        uint256 PendingRewardsAfter2 = staking.previewPending(WETH);  
        assertLt(PendingRewardsAfter2, PendingRewards, "Pending rewards should Decrease");  
        uint256 expectedReward = 360 * staking.mintRate(); // the only reward available to claim should be for the last 360 seconds  
        assertApproxEqAbs(PendingRewardsAfter2 , expectedReward , 1000, "Pending rewards should be zero");  
  
        }  
```

#### Mitigation

Don't need to update the lastRewardIndex inside stake and unStake function . only update while minting rewardAmount inside accrueReward function.

```diff
diff --git a/super-dca-gauge/src/SuperDCAStaking.sol b/super-dca-gauge/src/SuperDCAStaking.sol  
index 7b64a96..b1680b5 100644  
--- a/super-dca-gauge/src/SuperDCAStaking.sol  
+++ b/super-dca-gauge/src/SuperDCAStaking.sol  
@@ -218,7 +218,6 @@ contract SuperDCAStaking is ISuperDCAStaking, Ownable2Step {  
         // Update token bucket accounting and user stakes  
         TokenRewardInfo storage info = tokenRewardInfoOf[token];  
         info.stakedAmount += amount;  
-        info.lastRewardIndex = rewardIndex;  
   
         totalStakedAmount += amount;  
         userStakes[msg.sender][token] += amount;  
@@ -248,7 +247,6 @@ contract SuperDCAStaking is ISuperDCAStaking, Ownable2Step {  
   
         // Update token bucket accounting and user stakes  
         info.stakedAmount -= amount;  
-        info.lastRewardIndex = rewardIndex;  
   
         totalStakedAmount -= amount;  
         userStakes[msg.sender][token] -= amount;  
```
---


### [M-01] The reward will also accrued for the time when there was no stacker

#### Summary

On deployment, lastMinted is initialized to block.timestamp. During staking, it only updates if totalStakeAmount > 0. This causes an issue if the first staker joins after a delay, as rewardIndex stays 0 and lastMinted isn’t updated, leading to rewards being minted when no tokens were staked.

#### Root Cause

The root cause of this issue is in _updateRewardIndex function :

/SuperDCAStaking.sol:181  

```solidity
181:     function _updateRewardIndex() internal {  
182:         // Return early if no stakes exist or no time has passed  
183:         if (totalStakedAmount == 0) return;  
184:         uint256 elapsed = block.timestamp - lastMinted;  
185:         if (elapsed == 0) return;  
186:   
187:         // Calculate mint amount based on elapsed time and mint rate  
```

As it can be seen when totalStakedAmount==0 we just return and does not update lastMinted to current time.

- [SuperDCAStaking.sol#L183](https://github.com/sherlock-audit/2025-09-super-dca/blob/main/super-dca-gauge/src/SuperDCAStaking.sol#L183)

#### Internal Pre-conditions

After deployment, time elapses without any stakers.

#### External Pre-conditions

The user is the first staker.

#### Attack Path

...

#### Impact

The first minted rewards will also account for the period when no tokens were staked.

#### PoC

Add the following test case to OptimismStakingIntegration.t.sol and run with command forge test --mt testFork_rewards_tokens_claim_unstake_time -vvv:

```solidity
    function testFork_rewards_tokens_claim_() public {  
        _simulateTimePass(3600); // add 1 hour delay between first deposit and deployment  
        uint256 PendingRewardsAfter2 = staking.previewPending(WETH);  
        console.log("PendingRewardsAfter2" , PendingRewardsAfter2);  
        assertEq(PendingRewardsAfter2 , 0 , "Pending rewards should be zero");  
        // ---- Arrange ----  
        _setupTokenAndStake(user1, WETH, STAKE_AMOUNT, POSITION_AMOUNT0, POSITION_AMOUNT1);  
        //@note :  As no time has passed pending rewards should be zero but it is not  
        uint256 PendingRewards = staking.previewPending(WETH);  
        console.log("PendingRewards" ,PendingRewards);  
        assertGt(PendingRewards , 0 , "Pending rewards should be more than zero");  
        uint256 expectedReward = 3600 * staking.mintRate(); // the only reward available to claim should be for the last 360 seconds  
        assertApproxEqAbs(PendingRewards , expectedReward , 1000, "Pending rewards should be approx to expected reward");  
        }  
```

output :

```
[PASS] testFork_rewards_tokens_claim_unstake_time() (gas: 966254)  
Logs:  
  PendingRewardsAfter2 0  
  PendingRewards 13888888888888886000  
```

#### Mitigation

We need to update the lastMinted in case if the totalStakeAmount == 0

```diff
diff --git a/super-dca-gauge/src/SuperDCAStaking.sol b/super-dca-gauge/src/SuperDCAStaking.sol  
index 7b64a96..187b40e 100644  
--- a/super-dca-gauge/src/SuperDCAStaking.sol  
+++ b/super-dca-gauge/src/SuperDCAStaking.sol  
@@ -180,10 +180,13 @@ contract SuperDCAStaking is ISuperDCAStaking, Ownable2Step {  
      */  
     function _updateRewardIndex() internal {  
         // Return early if no stakes exist or no time has passed  
-        if (totalStakedAmount == 0) return;  
         uint256 elapsed = block.timestamp - lastMinted;  
         if (elapsed == 0) return;  
   
+        if (totalStakedAmount == 0) {  
+            lastMinted = block.timestamp;  
+            return;  
+        };  
         // Calculate mint amount based on elapsed time and mint rate  
         uint256 mintAmount = elapsed * mintRate;  
```