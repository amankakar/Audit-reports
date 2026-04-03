# Elfi


## Protocol Summary

All assets are tradable. Ultra Portfolio Mode includes multi-assets margin, position & assets risk offset.

[Ethereum][Solidity][DeFi]

---

# [H-01] User will loss the Rewards for stacking  if he redeem without claiming the reward

## Summary
Rewards for staked tokens accumulate over time based on the duration of the staking period. However, Due to an Issue in code The Rewards can not be claimed more details in next section.

## Vulnerability Detail
The Protocol Provide Rewards token as incentive for users to stake at thre platform. To stake token user will call `createMintStakeTokenRequest` and `ROLE_KEEPER` will call `executeMintStakeToken` Here we also update the rewards for staking via `updateAccountFeeRewards`. 
```solidity
function updateAccountFeeRewards(address account, address stakeToken) public {
        StakingAccount.Props storage stakingAccount = StakingAccount.load(account);
        StakingAccount.FeeRewards storage accountFeeRewards = stakingAccount.getFeeRewards(stakeToken);
        FeeRewards.MarketRewards storage feeProps = FeeRewards.loadPoolRewards(stakeToken);
        if (accountFeeRewards.openRewardsPerStakeToken == feeProps.getCumulativeRewardsPerStakeToken()) {
            return;
        }
        uint256 stakeTokens = IERC20(stakeToken).balanceOf(account);
        if (
            stakeTokens > 0 &&
            feeProps.getCumulativeRewardsPerStakeToken() - accountFeeRewards.openRewardsPerStakeToken >
            feeProps.getPoolRewardsPerStakeTokenDeltaLimit()
        ) {
            accountFeeRewards.realisedRewardsTokenAmount += (
                stakeToken == CommonData.getStakeUsdToken()
                    ? CalUtils.mul(
                        feeProps.getCumulativeRewardsPerStakeToken() - accountFeeRewards.openRewardsPerStakeToken,
                        stakeTokens
                    )
                    : CalUtils.mulSmallRate(
                        feeProps.getCumulativeRewardsPerStakeToken() - accountFeeRewards.openRewardsPerStakeToken,
                        stakeTokens
                    )
            );
        }
        accountFeeRewards.openRewardsPerStakeToken = feeProps.getCumulativeRewardsPerStakeToken();
        stakingAccount.emitFeeRewardsUpdateEvent(stakeToken);
    }
```
It can be observed from above function that the `realisedRewardsTokenAmount` will only be update if `stakeTokens>0`. So in our case it will not be updated. after some time user creates a redeem request to redeem stack tokens. No in case of Redeem we update the rewards after redeeming.
```solidity
function executeRedeemStakeToken(uint256 requestId, Redeem.Request memory redeemRequest) external {
        uint256 redeemAmount;
        if (CommonData.getStakeUsdToken() == redeemRequest.stakeToken) {
            redeemAmount = _redeemStakeUsd(redeemRequest);
        } else if (CommonData.isStakeTokenSupport(redeemRequest.stakeToken)) {
            redeemAmount = _redeemStakeToken(redeemRequest);
        } else {
            revert Errors.StakeTokenInvalid(redeemRequest.stakeToken);
        }

@>      FeeRewardsProcess.updateAccountFeeRewards(redeemRequest.account, redeemRequest.stakeToken);

        Redeem.remove(requestId);

        emit RedeemSuccessEvent(requestId, redeemAmount, redeemRequest);
    }
```
Here We can see that When we redeem the Tokens the User stack tokens can be zero , So the `realisedRewardsTokenAmount` can not be updated in this case.
Now lets check what happens if user Tries to Claim Rewards Tokens he Will call `ClaimRewards`
```solidity
function claimRewards(uint256 requestId, ClaimRewards.Request memory request) external {
        address account = request.account;
        address[] memory stakeTokens = CommonData.getAllStakeTokens();
        StakingAccount.Props storage stakingAccount = StakingAccount.load(account);
        StakingAccount.FeeRewards storage accountFeeRewards;
        address[] memory allStakeTokens = new address[](stakeTokens.length + 1);
        uint256[] memory claimStakeTokenRewards = new uint256[](stakeTokens.length + 1);
        for (uint256 i; i < stakeTokens.length; i++) {
            address stakeToken = stakeTokens[i];
            allStakeTokens[i] = stakeToken;
            FeeRewards.MarketRewards storage feeProps = FeeRewards.loadPoolRewards(stakeToken);
            FeeRewardsProcess.updateAccountFeeRewards(account, stakeToken);
            accountFeeRewards = stakingAccount.getFeeRewards(stakeToken);
@>            if (accountFeeRewards.realisedRewardsTokenAmount == 0) {
                continue;
            }
        ....
    }
```
In above code when `realisedRewardsTokenAmount==0` we can not Process it. 

1. Bob Stack 10e18 for 10 days.
2. Bob initial `Stake=0` so his `realisedRewardsTokenAmount` will not be updated.
3. After 10 days Bob redeem stack tokens.
4. we first redeem Bob token so `stake=0`.
5. The `realisedRewardsTokenAmount` will not be updated and remain `0`.

## Impact
The User will loos the rewards token for staking.

## Code Snippet
[https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/process/RedeemProcess.sol#L78](https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/process/RedeemProcess.sol#L78)
## Tool used

Manual Review

## Recommendation
First `updateAccountFeeRewards` then redeem tokens.
```diff
diff --git a/elfi-perp-contracts/contracts/process/RedeemProcess.sol b/elfi-perp-contracts/contracts/process/RedeemProcess.sol
index dedfe16e..8c293d07 100644
--- a/elfi-perp-contracts/contracts/process/RedeemProcess.sol
+++ b/elfi-perp-contracts/contracts/process/RedeemProcess.sol
@@ -67,6 +67,8 @@ library RedeemProcess {
 
     function executeRedeemStakeToken(uint256 requestId, Redeem.Request memory redeemRequest) external {
         uint256 redeemAmount;
+        FeeRewardsProcess.updateAccountFeeRewards(redeemRequest.account, redeemRequest.stakeToken);
+
         if (CommonData.getStakeUsdToken() == redeemRequest.stakeToken) {
             redeemAmount = _redeemStakeUsd(redeemRequest);
         } else if (CommonData.isStakeTokenSupport(redeemRequest.stakeToken)) {
@@ -75,7 +77,6 @@ library RedeemProcess {
             revert Errors.StakeTokenInvalid(redeemRequest.stakeToken);
         }
 
-        FeeRewardsProcess.updateAccountFeeRewards(redeemRequest.account, redeemRequest.stakeToken);
```

---


# [M-01] The USer will receive less amount  than user expected

## Summary
While redeeming the stack tokens, The user provides the `minRedeemAmount` to ensure they receive at least that amount. However,  within `executeRedeemStakeToken` function, the  `minRedeemAmount` check is used before deducting the fee . which could result in  the user  receive less amount than the expected amount.


## Vulnerability Detail
The Protocol allows user to specify the `minRedeemAmount` to insure that the user will receive this amount or in other case the transaction will revert. The User will first submit a request for Redemption where he also specify this `minRedeemAmount` which user expect to receive. The Issue is in the execute redemption request flow.
```solidity
function _executeRedeemStakeToken(
        LpPool.Props storage pool,
        Redeem.Request memory params,
        address baseToken
    ) internal returns (uint256) {
        ...
        cache.redeemTokenAmount = CalUtils.usdToToken(
            cache.unStakeUsd,
            cache.tokenDecimals,
            OracleProcess.getLatestUsdUintPrice(baseToken, false)
        );

        if (pool.getPoolAvailableLiquidity() < cache.redeemTokenAmount) {
            revert Errors.RedeemWithAmountNotEnough(params.account, params.redeemToken);
        }

 @>     if (params.minRedeemAmount > 0 && cache.redeemTokenAmount < params.minRedeemAmount) {
            revert Errors.RedeemStakeTokenTooSmall(cache.redeemTokenAmount);
        }
        ...
        FeeProcess.chargeMintOrRedeemFee(
            redeemFee,
            params.stakeToken,
            params.redeemToken,
            params.account,
            FeeProcess.FEE_REDEEM,
            false
        );
        VaultProcess.transferOut(
            params.stakeToken,
            params.redeemToken,
            params.receiver,
@>         cache.redeemTokenAmount - cache.redeemFee
        );
        pool.subPoolAmount(pool.baseToken, cache.redeemTokenAmount);
        StakeToken(params.stakeToken).burn(params.account, params.unStakeAmount);
        stakingAccountProps.subStakeAmount(params.stakeToken, params.unStakeAmount);

        return cache.redeemTokenAmount;
    }
```
As it can be observed from above code that we first convert the `unStkaeUsd` amount and store receive value in `cache.redeemTokenAmount`.Than we check for `minRedeemAmount`
and than we deduct the fee and transfer the remaining `redeemTokenAmount` to user.
Following case would occur due to this:
1. Bob submit a request to redeem 10e18 token and expect to receive 9e18 token.
2. the Protocol convert the amount using latest oracle price and get 9 token as redeemTokenAmount.
3. The `cache.redeemTokenAmount < params.minRedeemAmount` check will pass as 9e18 < 9e18.
4. The `RedeemFeeRate=10` and `RATE_PRECISION=100000` Now Applying these values to calculate the Fee amount is `9e18*10/100000= 9e14`.
5. The amount Bob will receive is `9e18-9e14≈8.9e17`.


This applies on both functions `_executeRedeemStakeUsd` and `_executeRedeemStakeToken`.

## Impact
The user will receive less amount than expected.


## Code Snippet
[https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/process/RedeemProcess.sol#L157](https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/process/RedeemProcess.sol#L157)

## Tool used

Manual Review

## Recommendation
Use slippage check after deducting the Fee.
```diff
diff --git a/elfi-perp-contracts/contracts/process/RedeemProcess.sol b/elfi-perp-contracts/contracts/process/RedeemProcess.sol
index dedfe16e..eb6c84fe 100644
--- a/elfi-perp-contracts/contracts/process/RedeemProcess.sol
+++ b/elfi-perp-contracts/contracts/process/RedeemProcess.sol
@@ -200,9 +200,7 @@ library RedeemProcess {
             tokenDecimals,
             OracleProcess.getLatestUsdUintPrice(params.redeemToken, false)
         );
-        if (params.minRedeemAmount > 0 && redeemTokenAmount < params.minRedeemAmount) {
-            revert Errors.RedeemStakeTokenTooSmall(redeemTokenAmount);
-        }
         if (pool.getMaxWithdraw(params.redeemToken) < redeemTokenAmount) {
             revert Errors.RedeemWithAmountNotEnough(params.account, params.redeemToken);
         }
@@ -219,6 +217,9 @@ library RedeemProcess {
             FeeProcess.FEE_REDEEM,
             false
         );
+        if (params.minRedeemAmount > 0 && redeemTokenAmount-redeemFee < params.minRedeemAmount) {
+            revert Errors.RedeemStakeTokenTooSmall(redeemTokenAmount);
+        }
 
         StakeToken(params.stakeToken).burn(account, params.unStakeAmount);
         StakeToken(params.stakeToken).transferOut(params.redeemToken, params.receiver, redeemTokenAmount - redeemFee);
```

---


# [M-02] Loss Fee does not get added due to wrong calculation

## Summary
When the `ROLE_KEEPER` executes a transaction, we check if the execution_fee is sufficient to cover the transaction cost. If any amount remains, it is refunded to the user, and if the fee is insufficient, the excess amount is added to the loss amount. However, in the event of a loss, the loss amount is not correctly calculated and therefore not added due to an error in the calculation

## Vulnerability Detail
Whenever user submit a request for any operation we charge `executionFee` in advance . The `ROLE_KEPPER` will submit the request operation and will charge the `exectuion_fee`. Here one pf the following 2 cases can occur.
1. The execution Fee was sufficient and the remaining amount sent back to users.
2. The executionFee was insufficient and loss added to Protocol.
In 2nd case there is an Issue due to which the Loss will never be added.
```solidity
    function processExecutionFee(PayExecutionFeeParams memory cache) external {
        uint256 usedGas = cache.startGas - gasleft();
        uint256 executionFee = usedGas * tx.gasprice;
        uint256 refundFee;
        uint256 lossFee;
        if (executionFee > cache.userExecutionFee) {
@>            executionFee = cache.userExecutionFee;  
 @>           lossFee = executionFee - cache.userExecutionFee;

        } else {
            refundFee = cache.userExecutionFee - executionFee;
        }
        VaultProcess.transferOut(
            cache.from,
            AppConfig.getChainConfig().wrapperToken,
            address(this),
            cache.userExecutionFee
        );
        VaultProcess.withdrawEther(cache.keeper, executionFee);
        if (refundFee > 0) {
            VaultProcess.withdrawEther(cache.account, refundFee);
        }
        if (lossFee > 0) {
            CommonData.addLossExecutionFee(lossFee);
        }
    }
```
From above code We can observed that when `executionFee>cache.userExecutionFee` then we first assign `executionFee = cache.userExecutionFee` and then calculate `LossFee` , So `LossFee` will always be zero.
```solidity
executionFee = 10 gwei;
cache.userExecutionFee = 9 gwei;
// we first assign 
executionFee = cache.userExecutionFee; // so Here executionFee = 9 gwei
lossFee = executionFee - cache.userExecutionFee;// 9 gwei - 9 gwei =0
        if (lossFee > 0) { // here lossFee=0 so it will not be added
            CommonData.addLossExecutionFee(lossFee);
        }
```

## Impact
`processExecutionFee` will never added any loss occur.

## Code Snippet
[https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/process/GasProcess.sol#L22-L25](https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/process/GasProcess.sol#L22-L25)
## Tool used

Manual Review

## Recommendation
```diff
diff --git a/elfi-perp-contracts/contracts/process/GasProcess.sol b/elfi-perp-contracts/contracts/process/GasProcess.sol
index 9a112074..b0552f34 100644
--- a/elfi-perp-contracts/contracts/process/GasProcess.sol
+++ b/elfi-perp-contracts/contracts/process/GasProcess.sol
@@ -20,8 +20,8 @@ library GasProcess {
         uint256 refundFee;
         uint256 lossFee;
         if (executionFee > cache.userExecutionFee) {
-            executionFee = cache.userExecutionFee;
             lossFee = executionFee - cache.userExecutionFee;
+            executionFee = cache.userExecutionFee;
         } else {
             refundFee = cache.userExecutionFee - executionFee;
```

---

# [M-03] `isHoldAmountAllowed` and `isSubAmountAllowed` wrong subtraction will result in DoS

## Summary
The `HoldStableToken` function checks if the given amount can be held by adding `(balance.amount + balance.unsettledAmount-balance.holdAmount)` and `isSubAmountAllowed` checks if `(balance.amount - balance.holdAmount) >= amount`. However, it is possible that the `holdAmount` is greater than the amount.

## Vulnerability Detail
In case of adding the `HoldStableToken` we add  `balance.amount` and `balance.unsettledAmount` in `isHoldAmountAllowed`:
```solidity
function isHoldAmountAllowed(
        TokenBalance memory balance,
        uint256 poolLiquidityLimit,
        uint256 amount
    ) internal pure returns (bool) {
        if (poolLiquidityLimit == 0) {
            return balance.amount + balance.unsettledAmount - balance.holdAmount >= amount;
        } else {
            return
                CalUtils.mulRate(balance.amount + balance.unsettledAmount, poolLiquidityLimit) - balance.holdAmount >=
                amount;
        }
    }
```
In case of `subStableToken` we check `isSubAmountAllowed` 
```solidity
function isSubAmountAllowed(Props storage self, address stableToken, uint256 amount) public view returns (bool) {
        TokenBalance storage balance = self.stableTokenBalances[stableToken];
        if (balance.amount < amount) {
            return false;
        }
        uint256 poolLiquidityLimit = getPoolLiquidityLimit();
        if (poolLiquidityLimit == 0) {
            return balance.amount - balance.holdAmount >= amount; // @audit : this could revert due to overflow/undeflow if holdAmount > amount.
        } else {
            return CalUtils.mulRate(balance.amount - amount, poolLiquidityLimit) >= balance.holdAmount;
        }
    }
```
The following case could occur:
```solidity
// assume here poolLiquidityLimit=0;
balance.amount = 10e18;
balance.unsettled = 10e18;
// while adding the hold amount 12e18 , balance.amount + balance.unsettledAmount - balance.holdAmount >= amount
10e18 + 10e18 - 0 >= 12e18 // it will return true so now holdAmount=12e18
//No rebalance occur the state of token balance is same
// now we want to subtract the amount from token balance  isSubAmountAllowed would be called to check that if amount can be deducted
//return balance.amount - balance.holdAmount >= amount;
10e18 - 12e18>= 5e18 // it will revert due to underFlow/OverFlow
```

## Impact
The Will create DoS for `subStableToken` calls , `subStableToken` function is used in different use cases like redeeming token , PnL updates and Rebalance calls. 

## Code Snippet
[https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/storage/UsdPool.sol#L241C14-L252](https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/storage/UsdPool.sol#L241C14-L252)
[https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/storage/UsdPool.sol#L254-L266](https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/storage/UsdPool.sol#L254-L266)
[https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/storage/UsdPool.sol#L87](https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/storage/UsdPool.sol#L87)
## Tool used

Manual Review

## Recommendation
add one more check inside `isSubAmountAllowed` as follows :
```diff
diff --git a/elfi-perp-contracts/contracts/storage/UsdPool.sol b/elfi-perp-contracts/contracts/storage/UsdPool.sol
index 93d8aca1..dba7d141 100644
--- a/elfi-perp-contracts/contracts/storage/UsdPool.sol
+++ b/elfi-perp-contracts/contracts/storage/UsdPool.sol
@@ -240,12 +240,12 @@ library UsdPool {
 
     function isSubAmountAllowed(Props storage self, address stableToken, uint256 amount) public view returns (bool) {
         TokenBalance storage balance = self.stableTokenBalances[stableToken];
-        if (balance.amount < amount) {
+        if (balance.amount < amount|| balance.amount <balance.holdAmount) {
             return false;
         }
         uint256 poolLiquidityLimit = getPoolLiquidityLimit();
```

---


