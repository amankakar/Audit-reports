# First Deposit will receive less shares due to wrong calculation

## Summary
when first user deposit into stakingPool, the user will receive 10**3 less share due to wrong calculation in `_mintShares` function.

## Vulnerability Details
To deposit in stakingPool the user will call `deposit` function of `PriorityPool` . The `PriorityPool` will call the `deposit` function of `stakingPool`. The stakingPool than mint the share and deposit the assets in startegy vault to stake the assets on chainlink staking contract. 

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
Add the following code inside the new test file : `mintingBug.test.ts`
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

  it('deposit works correctly but shares are not  minted correctly', async () => {
    const { signers, accounts, adrs, pp, token, strategy, stakingPool } = await loadFixture(
      deployFixture
    )
    // if user deposit less then 100 it will revert , 
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
    ✔ deposit work correctly but shares are not minted correctly (2475ms)
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
The perform upkeep tries to distubute the rewards to all the fee receivers that return true from the `checkUpKeep` call, but the `performUpKeep` can also be called directly on the specific splitter, which will lead to reverts, and the `performUpKeep` cannot be succeeded.
## Vulnerability Details
To split the rewards, first we need to confirm that the rewards can be split; for this we call `checkUpkeep`.
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
if it returns true then we will call the `performUpKeep` function, which will call `LSTRewardsSplitter:performUpKeep` and split the rewards in fee receiver.

The PerformUpkeep function of `LSTRewardsSplitter` checks that the total lst balance and pricnipal deposit is more than the rewards threshold, then calls the `_splitRewards` function, which distributes the fee to the fee receiver and updates the `principalDeposits=lst.balanceOf(address(this))`.
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
1. The keeper calls the `checkupKeep` and receives true for the overall split and with the 4 splitter.
2. Keeper submits the transaction for `performUpKeep` for these 4 splitters.
3. Bob monitors the transaction and frontruns the Keeper transaction and `performUpkeep` on one of the splitters.
4. When the execution node picks the Keeper transaction, it will be reverted with this message: `InsufficientRewards`.      

## Impact
The keeper transaction will be frontrun by malicious users and will result in DoS for keeper.

## Tools Used
Manual Review

## Recommendations
Remove the revert condition and simply return it, no rewards to split.

# The user cannot withdraw assets if the remaining amount, after partial filling the queue deposit, is below the minimum in the `queueWithdrawal` function.


## Summary

The protocol allows immediate withdrawals and deposits in Stakelink staking, for efficient asset flow. This is achieved by transferring pending deposits to users withdrawing assets from Stakelink, and vice versa.

However, the issue arises when queuing a withdrawal. First, if the deposit queue is non-zero, the withdrawal amount is filled with the queue deposit. If some amount remains, the `queueWithdrawal` function is called. This function enforces a `minWithdrawalAmount` check, which prevents users from withdrawing their assets if the remaining amount is below the minimum threshold.


## Vulnerability Details

To withdraw assets from pool, user will call `PriorityPool:withdraw` function. which will call internal `_withdraw` function 
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

===================================================================================================================
# The `removeSplitter` function still transfer the same amount before reward transfer

## Summary
To remove splitter from controller, we first check that if the balance of splitter is not zero and also balance is not equal to 'principalDeposits', then we call `the'splitRewards` function to distribute the rewards among the fee receivers. However, the issue here is that we still have to `withdraw` the exact same balance amount to the splitter owner's address. 

## Vulnerability Details

Let us have a splitter contract with the balance of 100 LST and the total deposit of 90; the 10 LST is for rewards distribution. 
So here the `balance = 100 LST` and `principalDeposits = 90 LST` are not equal, we need to distribute the rewards.

```solidity
2024-09-stakelink/contracts/core/lstRewardsSplitter/LSTRewardsSplitterController.sol:134
134:         uint256 balance = IERC20(lst).balanceOf(address(splitter));
135:         uint256 principalDeposits = splitter.principalDeposits();
136:         if (balance != 0) {
137:             if (balance != principalDeposits) splitter.splitRewards();
138:             splitter.withdraw(balance, _account); // @audit : the balance will revert as we will distribute the rewards and the balance will not be same as already cached??
139:         }
140: 
``` 
Here at Line `138` we still try to withdraw 100 LST; in fact, we do not have 100 LST available in balance.
Therefor it will revert and not allow the owner to remove the splitter.

POC : add the following test case inside `test/core/lst-rewards-splitter.test.ts` :
```javascript

  it.only('should be able to remove splitter', async () => {
    const { accounts, controller, token, splitter1 , splitter0} = await loadFixture(deployFixture)
    await token.transfer(splitter0 , toEther(100));
    await controller.removeSplitter(accounts[0])

    await expect(controller.removeSplitter(accounts[0])).to.be.revertedWithCustomError(
      controller,
      'SplitterNotFound()'
    )

    assert.deepEqual(await controller.getAccounts(), [accounts[1]])

    assert.equal(await controller.splitters(accounts[0]), ethers.ZeroAddress)

    assert.equal(await splitter1.controller(), controller.target)
    assert.equal(await splitter1.lst(), token.target)
    assert.deepEqual(await splitter1.getFees(), [
      [accounts[7], 2000n],
      [accounts[8], 4000n],
    ])
  })
```
and run with command : `npx hardhat test test/core/lst-rewards-splitter.test.ts `.

## Impact
The owner will not be able to withdraw the reward splitter due to wrong amount provided in withdraw functions.

## Tools Used
Manual review
## Recommendations
before withdrawal fetch the latest amount available for withdraw.


=========================================================================================
New 
# 5.  The `FundFlowController` presumes that the `unbondingPeriod` will not alter at linkStaking 

## Summary
The `FundFlowController` constructor recieves the current `unbonding period` value used in the  chain link `Stakie.sol` contract, but the problem here is that the unbonding period can also be updated; this will affect all the operations of 'FundFlowController.` More details are given in the subsequent section.ngPoolBas

## Vulnerability Details
Upon reviewing the `FundFlowController` code, we found that the unbonding period is set within the initialization process. However, there is no setter function available to update the unbonding period if it changes in the chain link staking contract.

```solidity
2024-09-stakelink/contracts/linkStaking/FundFlowController.sol:61
61:         unbondingPeriod = _unbondingPeriod; // @audit : add setter function for unbouningPeroid
```
Now, let's take a look at the chain link staking pool base, where they have implemented a function to update the unbonding period

```solidity
 function setUnbondingPeriod(uint256 newUnbondingPeriod)
    external
    onlyRole(DEFAULT_ADMIN_ROLE)
    whenBeforeClosing
  {
    _setUnbondingPeriod(newUnbondingPeriod);
  }

```
A limit check is in place to control the extent to which the claim period can be updated.
```solidity
  function _setUnbondingPeriod(uint256 unbondingPeriod) internal {
    if (unbondingPeriod == 0 || unbondingPeriod > i_maxUnbondingPeriod) {
      revert InvalidUnbondingPeriod();
    }
```
current claim Period : `728 days` , but it can be changed to any value between `>0 to 60 days.`

The following functions of `FundFlowController` and `VaultDepositController` contract functions will be affected:
1. claimPeriodActive
2. updateVaultGroups
3. VaultControllerStrategy:withdraw
4. VaultControllerStrategy::getMinDeposits()



## Impact
1. Since the unbonding period can be increased or decreased, it may result in a denial of service (DoS) for unbonding operations.
2. The `claimPeriodActive` function may return incorrect responses.
3. The `withdraw` function could face a DoS issue, as the protocol might assume it can withdraw funds when, in reality, it cannot.
## Tools Used

Manual Review

## Recommendations
Instead of storing the `claimPeroid` within the `FundFlowController` contract, use the `StakingPoolBase::getUnbondingParams` function, which will always return the current `claimPeriod.

```solidity
  function getUnbondingParams() external view returns (uint256, uint256) {
    return (s_pool.configs.unbondingPeriod, s_pool.configs.claimPeriod);
  }

```


===================================================================================================================
# 6. The `FundFlowController` assumes that the `claimPeriod` will remain constant.

## Summary
The `FundFlowController` constructor retrieves the current `claimPeriod` value utilized in the linked `StakingPoolBase.sol` contract. However, it is important to note that the `claimPeriod` can be updated, which will impact all operations of the `FundFlowController`. Further details will be provided in the subsequent section.


## Vulnerability Details
Upon reviewing the `FundFlowController` code, it is observed that the `claimPeriod` is set within the `initialize` function, and there is no setter function available to update the `claimPeriod` if it changes in the Chainlink staking contract.

```solidity
2024-09-stakelink/contracts/linkStaking/FundFlowController.sol:62
62:         claimPeriod = _claimPeriod; // @audit : add setter function for claimPeroid 
63:         numVaultGroups = _numVaultGroups; 
```
Now, let us examine the Chainlink Staking Pool Base, which includes a function designed to update the `claimPeriod`.

```solidity
  function setClaimPeriod(uint256 claimPeriod)
    external
    onlyRole(DEFAULT_ADMIN_ROLE)
    whenBeforeClosing
  {
    _setClaimPeriod(claimPeriod);
  }
```
Limit check to which the claim period can be updated
```solidity
  function _setClaimPeriod(uint256 claimPeriod) private {
    if (claimPeriod < i_minClaimPeriod || claimPeriod > i_maxClaimPeriod) {
      revert InvalidClaimPeriod();
    }

```
The current `claimPeriod` is set to `7 days`, but it can be adjusted to any value ranging from `1 day to 30 days`.

The following functions in the `FundFlowController` and `VaultDepositController` contracts will be affected:
1. `claimPeriodActive`
2. `updateVaultGroups`
3. `VaultControllerStrategy:withdraw`
4. `VaultControllerStrategy::getMinDeposits()`



## Impact

1. If the `claimPeriod` is adjusted, the protocol erroneously assumes it can unbind the funds from Chainlink staking, while in reality, it cannot.
2. The `claimPeriodActive` function will yield an incorrect response.
3. The `withdraw` function is vulnerable to a Denial of Service (DoS) attack, as the protocol assumes it can withdraw funds when it actually cannot. 

## Tools Used

Manual Review

## Recommendations
Instead of storing the `claimPeriod` within the `FundFlowController` contract, it is recommended to utilize the `StakingPoolBase::getUnbondingParams`, which will consistently return the current `claimPeriod`.

```solidity
  function getUnbondingParams() external view returns (uint256, uint256) {
    return (s_pool.configs.unbondingPeriod, s_pool.configs.claimPeriod);
  }

```
Or set a setter function for `claimPeriod` which will update the `claimPeriod`.


# 7. Some contracts may not receive the initializer due to an incorrect `reinitializer` modifier or the use of an incorrect version.

## Summary
Some contracts are already deployed, and their current version will be upgraded following the deployment of contracts in this audit. However, the issue arises because the newly deployed contract will not be initialized due to the incorrect modifier version used.

## Vulnerability Details
The following contracts will not be upgraded, or their `initialize` function will revert:

1. `OperatorVCS`: This contract uses an incorrect `initialize` version.
2. `OperatorVault`: This contract uses an incorrect `initialize` version.
3. `CommunityVCS`: This contract employs the `initialize` function.
4. `CommunityVault`: This contract employs the `initialize` function.


A simple code snippet to demonstrate this issue:
Setup a simple foundry project and install openzeppelin contracts than add following code :
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts-upgradeable/proxy/utils/Initializable.sol";
import "@openzeppelin/contracts-upgradeable/proxy/utils/UUPSUpgradeable.sol";



contract MyContractV1 is Initializable, UUPSUpgradeable {
    uint256 public value;

    // Initialize function for UUPS
    function initialize() public initializer {
        value = 42; // Set an initial value
    }

    // Upgrade authorization (required for UUPS proxy)
    function _authorizeUpgrade(address newImplementation) internal override {}

    function setValue(uint256 newValue) public {
        value = newValue;
    }
}


contract MyContractV2 is Initializable, UUPSUpgradeable {
    uint256 public value;
    uint256 public newValue;

    // Added function in V2
    function increment() public {
        value += 1;
    }

    // New reinitializer for V2
    function reinitializeV2() public reinitializer(1) {
        newValue = 100; // Some new logic for V2
    }

    function _authorizeUpgrade(address newImplementation) internal override  {}
}
``` 
Add following test code inside test dir :
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/Test.sol";
import {MyContractV1 , MyContractV2}  from "../src/MyContract.sol";
import "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";
import "@openzeppelin/contracts-upgradeable/proxy/utils/UUPSUpgradeable.sol";

contract UUPSTest is Test {
    MyContractV1 public myContractV1;
    MyContractV2 public myContractV2;
    ERC1967Proxy public proxy;
    MyContractV1 public proxyAsV1;
    MyContractV2 public proxyAsV2;

    event UpdateStrategyRewards(
     int rewardsAmount
    );

    function setUp() public {
        // Deploy the V1 contract and initialize
        myContractV1 = new MyContractV1();
        bytes memory data = abi.encodeWithSignature("initialize()");
        
        // Deploy proxy pointing to V1
        proxy = new ERC1967Proxy(address(myContractV1), data);

        // Cast the proxy as MyContractV1
        proxyAsV1 = MyContractV1(address(proxy));

        // Ensure the initial state is correct
        assertEq(proxyAsV1.value(), 42);
    }

    function testUpgradeToV2WithReinitializer() public {
        // Deploy V2 contract
        myContractV2 = new MyContractV2();

        vm.prank(address(this)); // Assuming msg.sender is needed for authorization
        proxyAsV1.upgradeTo(address(myContractV2));

        // Cast the proxy as MyContractV2
        proxyAsV2 = MyContractV2(address(proxy));

        // Ensure that the state is retained after the upgrade
        assertEq(proxyAsV2.value(), 42);

        // Call the reinitializer (simulating post-upgrade initialization)
        proxyAsV2.reinitializeV2();

        // Ensure that the reinitializer logic worked
        assertEq(proxyAsV2.newValue(), 100);

        // Test new functionality in V2
        proxyAsV2.increment();
        assertEq(proxyAsV2.value(), 43);
    }

    function testEvent() external {
        int256 a = -1234;
        emit  UpdateStrategyRewards(a);
    }
}

```
Run with command : `forge test --mt testUpgradeToV2WithReinitializer`.

## Impact
These contracts can not be deployed and will result in a revert.

## Tools Used
Manual review

## Recommendations
Check the versions of all already deployed contracts and ensure that the correct version is used for new deployments
For example, for `OperatorVCS`, the current version is 3, and we need to use version 4 here.




