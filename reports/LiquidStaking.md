# Liquid Staking 

## Protocol Summary

stake.link is a first-of-its-kind liquid delegated staking platform delivering DeFi composability for Chainlink Staking. Built by premier Chainlink ecosystem developer LinkPool, powered by Chainlink node operators, and governed by the stake.link DAO, stake.link's extensible architecture is purpose-built to support Chainlink Staking and to extend participation in the Chainlink Network.


[Ethereum][Solidity][DeFi]

### [M-01]  The `removeSplitter` function still transfer the same amount before reward transfer which will revert the transaction            

#### Summary
To remove splitter from controller, we first check that if the balance of splitter is not zero and also balance is not equal to 'principalDeposits', then we call `the'splitRewards` function to distribute the rewards among the fee receivers. However, the issue here is that we still have to `withdraw` the exact same balance amount to the splitter owner's address. 

#### Vulnerability Details

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

#### Impact
The owner will not be able to withdraw the reward splitter due to wrong amount provided in withdraw functions.

#### Tools Used
Manual review
#### Recommendations
before withdrawal fetch the latest amount available for withdraw.


---



### [L-01] Some contracts will not be initialized due to an incorrect `reinitializer` versions used.            

#### Summary

Some contracts are already deployed, and their current version will be upgraded following the deployment of contracts in this audit. However, the issue arises because the newly deployed contract will not be initialized due to the incorrect modifier version used.

#### Vulnerability Details

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

#### Impact

These contracts can not be deployed and will result in a revert.

#### Tools Used

Manual review

#### Recommendations

Check the versions of all already deployed contracts and ensure that the correct version is used for new deployments
For example, for `OperatorVCS`, the current version is 3, and we need to use version 4 here.

---

### [L-02] The `FundFlowController` presumes that the `unbondingPeriod` will not alter at linkStaking             


#### Summary

The `FundFlowController` constructor recieves the current `unbonding period` value used in the  chain link `Stakie.sol` contract, but the problem here is that the unbonding period can also be updated; this will affect all the operations of 'FundFlowController.\` More details are given in the subsequent section.ngPoolBas

#### Vulnerability Details

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

#### Impact

1. Since the unbonding period can be increased or decreased, it may result in a DoS for unbonding operations.
2. The `claimPeriodActive` function may return incorrect responses.
3. The `withdraw` function could face a DoS issue, as the protocol might assume it can withdraw funds when, in reality, it cannot.

#### Tools Used

Manual Review

#### Recommendations

Instead of storing the `unbondingPeriod` within the `FundFlowController` contract, use the `StakingPoolBase::getUnbondingParams` function, which will always return the current `unbondingPeriod`.

```solidity
  function getUnbondingParams() external view returns (uint256, uint256) {
    return (s_pool.configs.unbondingPeriod, s_pool.configs.claimPeriod);
  }

```
---

### [L-03] The user cannot withdraw assets if the remaining amount, after partial filling the queue deposit, is below the minimum in the `queueWithdrawal` function.            



#### Summary

The protocol allows immediate withdrawals and deposits in Stakelink staking, for efficient asset flow. This is achieved by transferring pending deposits to users withdrawing assets from Stakelink, and vice versa.

However, the issue arises when queuing a withdrawal. First, if the deposit queue is non-zero, the withdrawal amount is filled with the queue deposit. If some amount remains, the `queueWithdrawal` function is called. This function enforces a `minWithdrawalAmount` check, which prevents users from withdrawing their assets if the remaining amount is below the minimum threshold.


#### Vulnerability Details

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

#### Impact
The user will not be able to withdraw assets from the pool, even if the initial withdrawal request meets the minimum required amount.

#### Tools Used
Manual Review

#### Recommendations
Add a check inside the `_withdraw` function to ensure that the amount queued for withdrawal is not less than the required minimum if the amount is partially filled from queued deposits.

