# First Deposit will receive less shares due to wrong calculation

## Summary
when first user deposit into stakingPool and the user will receive 10**3 less share due to wrong calculation in `_mintShares` function.

## Vulnerability Details
To deposits in stakingPool the user will calls `deposit` function of `PriorityPool` . The `PriorityPool` will calls the `deposit` function of `stakingPool`. The stakingPool than mint the share and deposit the assets in startegy vault to stake the assets on chainlink staking contract. 

However the issue here lies in `_mintShares` function where the user will receive the share which represent his stake in the vault. 

```solidity
function _mintShares(address _recipient, uint256 _amount) internal {
        require(_recipient != address(0), "Mint to the zero address");

        if (totalShares == 0) {
            shares[address(0)] = DEAD_SHARES;
            totalShares = DEAD_SHARES;
            _amount -= DEAD_SHARES; // @audit : share sub
        }

        totalShares += _amount;
        shares[_recipient] += _amount;
    }
```
## POC 
Add following code inside new test file : `mintingBug.test.ts`
```javascript
import { assert, expect } from 'chai'
import {
  toEther,
  deploy,
  fromEther,
  deployUpgradeable,
  getAccounts,
  setupToken,
} from '../../utils/helpers'
import {
  ERC677,
  SDLPoolMock,
  StakingPool,
  PriorityPool,
  StrategyMock,
  WithdrawalPool,
} from '../../../typechain-types'
import { ethers } from 'hardhat'
import { StandardMerkleTree } from '@openzeppelin/merkle-tree'
import { loadFixture } from '@nomicfoundation/hardhat-network-helpers'

describe('PriorityPool', () => {
  async function deployFixture() {
    const { accounts, signers } = await getAccounts()
    const adrs: any = {}

    const token = (await deploy('contracts/core/tokens/base/ERC677.sol:ERC677', [
      'Chainlink',
      'LINK',
      1000000000,
    ])) as ERC677
    adrs.token = await token.getAddress()
    await setupToken(token, accounts, true)

    const stakingPool = (await deployUpgradeable('StakingPool', [
      adrs.token,
      'Staked LINK',
      'stLINK',
      [],
      toEther(10000),
    ])) as StakingPool
    adrs.stakingPool = await stakingPool.getAddress()

    const strategy = (await deployUpgradeable('StrategyMock', [
      adrs.token,
      adrs.stakingPool,
      toEther(1000),
      toEther(100),
    ])) as StrategyMock
    adrs.strategy = await strategy.getAddress()

    const sdlPool = (await deploy('SDLPoolMock')) as SDLPoolMock
    adrs.sdlPool = await sdlPool.getAddress()

    const pp = (await deployUpgradeable('PriorityPool', [
      adrs.token,
      adrs.stakingPool,
      adrs.sdlPool,
      toEther(100),
      toEther(1000),
    ])) as PriorityPool
    adrs.pp = await pp.getAddress()

    const withdrawalPool = (await deployUpgradeable('WithdrawalPool', [
      adrs.token,
      adrs.stakingPool,
      adrs.pp,
      toEther(10),
      0,
    ])) as WithdrawalPool
    adrs.withdrawalPool = await withdrawalPool.getAddress()

    await stakingPool.addStrategy(adrs.strategy)
    await stakingPool.setPriorityPool(adrs.pp)
    await stakingPool.setRebaseController(accounts[0])
    await pp.setDistributionOracle(accounts[0])
    await pp.setWithdrawalPool(adrs.withdrawalPool)

    for (let i = 0; i < signers.length; i++) {
      await token.connect(signers[i]).approve(adrs.pp, ethers.MaxUint256)
    }


    return { signers, accounts, adrs, token, stakingPool, strategy, sdlPool, pp, withdrawalPool }
  }

  it('deposit should work correctly but shares are not correctly minted', async () => {
    const { signers, accounts, adrs, pp, token, strategy, stakingPool } = await loadFixture(
      deployFixture
    )
    // if user deposit less then 100o it will revert , 
    await expect(pp.deposit(100, false, ['0x'])).to.be.reverted;

    await pp.deposit(1000, false, ['0x'])
    assert.equal(fromEther(await stakingPool.balanceOf(accounts[0])), 0) // user will recevice 0 share

  })


})
```
Run with :
`npx hardhat test test/core/priorityPool/mintingBug.test.ts` 
Logs :
```javascript
shares 0n
    ✔ deposit should work correctly but shares are not correctly minted (2475ms)
```


## Impact
The impact is only limited to first users in case of assets lost. 

## Tools Used
ManualReview , hardhat

## Recommendations
The One potential fix will be to make `totalShare` and `totalStake` equal to `DEAD_SHARES` and remove the subtraction of _amount :
```diff
diff --git a/contracts/core/StakingPool.sol b/contracts/core/StakingPool.sol
index bb00ace..b0ee799 100644
--- a/contracts/core/StakingPool.sol
+++ b/contracts/core/StakingPool.sol
@@ -5,7 +5,6 @@ import "@openzeppelin/contracts-upgradeable/token/ERC20/utils/SafeERC20Upgradeab
 
 import "./base/StakingRewardsPool.sol";
 import "./interfaces/IStrategy.sol";
-
 /**
  * @title Staking Pool
  * @notice Allows users to stake asset tokens and receive liquid staking tokens 1:1, then deposits staked
@@ -121,6 +120,9 @@ contract StakingPool is StakingRewardsPool {
             token.safeTransferFrom(msg.sender, address(this), _amount);
             _depositLiquidity(_data);
             _mint(_account, _amount);
+            if(totalStaked ==0) {
+                totalStaked = DEAD_SHARES;
+            }
             totalStaked += _amount;
         } else {
             _depositLiquidity(_data);
--------------------------------------------------
diff --git a/contracts/core/base/StakingRewardsPool.sol b/contracts/core/base/StakingRewardsPool.sol
index a58d308..28acb7b 100644
--- a/contracts/core/base/StakingRewardsPool.sol
+++ b/contracts/core/base/StakingRewardsPool.sol
@@ -13,7 +13,7 @@ import "../tokens/base/ERC677Upgradeable.sol";
  */
 abstract contract StakingRewardsPool is ERC677Upgradeable, UUPSUpgradeable, OwnableUpgradeable {
     // used to prevent vault inflation attack
-    uint256 private constant DEAD_SHARES = 10 ** 3;
+    uint256 internal constant DEAD_SHARES = 10 ** 3;
 
     // address of staking asset token
     IERC20Upgradeable public token;
@@ -191,7 +191,6 @@ abstract contract StakingRewardsPool is ERC677Upgradeable, UUPSUpgradeable, Owna
         if (totalShares == 0) {
             shares[address(0)] = DEAD_SHARES;
             totalShares = DEAD_SHARES;
-            _amount -= DEAD_SHARES;
         }
 
         totalShares += _amount;
```

# `LSTRewardsSplitterController:performUpkeep` will be reverted due to front running

## Summary
The perform upkeep tries to distubute the rewards to all the fee receiver which return true from `checkUpKeep` call , but the `performUpKeep` can aslo be called directly on specfic splitter which will lead to reverts and the `performUpKeep` can not be succeded.

## Vulnerability Details
To Split the rewards first we need to confrim that the rewards can be split for this we calls `checkUpkeep`.

```solidity
    function checkUpkeep(bytes calldata) external view returns (bool, bytes memory) {
        bool[] memory splittersToCall = new bool[](accounts.length);
        bool overallUpkeepNeeded;

        for (uint256 i = 0; i < splittersToCall.length; ++i) {
            (bool upkeepNeeded, ) = splitters[accounts[i]].checkUpkeep("");
            splittersToCall[i] = upkeepNeeded;
            if (upkeepNeeded) overallUpkeepNeeded = true;
        }

        return (overallUpkeepNeeded, abi.encode(splittersToCall));
    }
```
if it return true then we will call the `performUpKeep` function which will call `LSTRewardsSplitter:performUpKeep` and split the rewards in fee receiver.

The PerformUpkeep function of `LSTRewardsSplitter` check that the total lst balance and pricnipal depost is more than rewards threshold than it call `_splitRewards` function which distrbute the fee to the fee receiver and update the `principalDeposits=lst.balanceOf(address(this))`.
```solidity
function performUpkeep(bytes calldata) external {
        int256 newRewards = int256(lst.balanceOf(address(this))) - int256(principalDeposits);
        if (newRewards < 0) {
            principalDeposits -= uint256(-1 * newRewards);
        } else if (uint256(newRewards) < controller.rewardThreshold()) {
            revert InsufficientRewards();
        } else {
            _splitRewards(uint256(newRewards));
        }
    }
```
POC:
1. keeper calls the `checkupKeep` and receive true for overall split and with the 4 splitter.
2. keeper submit the transaction for `performUpKeep` for these 4 splitter.
3. Bob moniter the transaction and frontrun Keeper transaction and `performUpkeep` on on2 of the splitter.
4. When execution node pick Keeper transaction it will be reverted with this message `InsufficientRewards`.      

## Impact
The keeper transaction will be frontrun by malicious users and will result in DoS for keeper.

## Tools Used
Manual Review

## Recommendations
remove the revert condition and simply return it no rewards to split.

# The user cannot withdraw assets if the remaining amount after partial filling the queue deposit is below the minimum in the `queueWithdrawal` function.


## Summary

For efficient asset flow, the protocol allows immediate withdrawals and deposits in Stakelink staking. This is achieved by transferring pending deposits to users withdrawing assets from Stakelink, and vice versa.

However, the issue arises when queuing a withdrawal. First, if the deposit queue is non-zero, the withdrawal amount is filled with the queue deposit. If some amount remains, the `queueWithdrawal` function is called. This function enforces a `minWithdrawalAmount` check, which prevents users from withdrawing their assets if the remaining amount is below the minimum threshold.


## Vulnerability Details

To withdraw assets from pool user will call `PriorityPool:withdraw` function. which will call internal `_withdraw` function 
```solidity
2024-09-stakelink/contracts/core/priorityPool/PriorityPool.sol:663
663:     function _withdraw(
664:         address _account,
665:         uint256 _amount,
666:         bool _shouldQueueWithdrawal
667:     ) internal returns (uint256) {
668:         if (poolStatus == PoolStatus.CLOSED) revert WithdrawalsDisabled();
669: 
670:         uint256 toWithdraw = _amount;
671: 
672:         if (totalQueued != 0) {
673:             uint256 toWithdrawFromQueue = toWithdraw <= totalQueued ? toWithdraw : totalQueued;
674: 
675:             totalQueued -= toWithdrawFromQueue;
676:             depositsSinceLastUpdate += toWithdrawFromQueue;
677:             sharesSinceLastUpdate += stakingPool.getSharesByStake(toWithdrawFromQueue);
678:             toWithdraw -= toWithdrawFromQueue;
679:         }
680: 
681:         if (toWithdraw != 0) {
682:             if (!_shouldQueueWithdrawal) revert InsufficientLiquidity();
683:             withdrawalPool.queueWithdrawal(_account, toWithdraw); // @audit : due to min chek it will revert here and user will not be able to withdraw
684:         }
685: 
```
It can be observed from the code that the withdrawal amount will either be fully or partially filled. The issue arises when the amount is partially filled, `_shouldQueueWithdrawal=true`, and `toWithdraw < minWithdrawalAmount`.

```solidity
2024-09-stakelink/contracts/core/priorityPool/WithdrawalPool.sol:304
304:     function queueWithdrawal(address _account, uint256 _amount) external onlyPriorityPool {
305:         if (_amount < minWithdrawalAmount) revert AmountTooSmall(); // @audit : it could revert due to filling the order at from assets in pr pool
306: 
```

## Impact
The user will not be able to withdraw assets from the pool, even if the initial withdrawal request meets the minimum required amount.

## Tools Used
Manual Review

## Recommendations
Add a check inside the `_withdraw` function to ensure that the amount queued for withdrawal is not less than the required minimum if the amount is partially filled from queued deposits.
