

# AlchemixV3

## Protocol Summary

Alchemix is your unified platform for saving, earning, borrowing, and fixed-term fixed-yield opportunities—all in one place. Built on years of iteration since launching the original self-repaying loan in 2021, Alchemix v3 brings all three pillars together with a smarter, more flexible design.


[Ethereum][Solidity][Lending]

---


## [H-01] No slippage provided in Auto strategy implementation will open room for MEV attacks

### Target
https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/strategies/mainnet/TokeAutoEth.sol

### Impact(s)
Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Brief/Intro
While depositing into the Auto Pool, we use the router implementation from Auto Finance, which calls the `depositMax` function.
This function accepts `minSharesOut` as a parameter; however, in our implementation, we pass 0 as the `minSharesOut` value.

This can lead to the vault receiving fewer shares than expected if the transaction gets delayed for a few blocks, as demonstrated in the PoC.
Additionally, the `rebalancing` and `updateDebtReporting` calls can further affect the outcome depending on the current state of the pool.

## Vulnerability Details
This issue can occur in TokeAutoEth and TokeAutoUSDStrategy. But I will focus on TokeAutoEth in this reports. The vault will calls _allocate function with the given amount. The router::depositMax will deposit the assets in auto pool and return the shares to strategy contract.

```javascript
/v3-poc/src/strategies/mainnet/TokeAutoEth.sol:57
57:     function _allocate(uint256 amount) internal override returns (uint256) {
58:         require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than amount");
59:         TokenUtils.safeApprove(address(weth), address(router), amount);
60:         uint256 shares = router.depositMax(autoEth, address(this), 0); // @audit : no slippage protection here
61:         TokenUtils.safeApprove(address(autoEth), address(rewarder), shares);
62:         rewarder.stake(address(this), shares);
63:         return amount;
64:     }
```

The `router::depositMaxfunction` takes 3 parameter the pool address , the receiver address and the `minSharesOut`.

```solidity
    function depositMax(
        IAutopool vault,
        address to,
        uint256 minSharesOut
    ) public payable override returns (uint256 sharesOut) {
        IERC20 asset = IERC20(vault.asset());
        uint256 assetBalance = asset.balanceOf(msg.sender);
        uint256 maxDeposit = vault.maxDeposit(to);
        uint256 amount = maxDeposit < assetBalance ? maxDeposit : assetBalance;
        pullToken(asset, amount, address(this));

        approve(IERC20(vault.asset()), address(vault), amount);
        return deposit(vault, to, amount, minSharesOut);
    }
```
[AutopilotRouter::depositMax](https://vscode.blockscan.com/ethereum/0x37dD409f5e98aB4f151F4259Ea0CC13e97e8aE2).

From `AutoPool` Docs : **Depending on the conditions of the `Autopool`, the overall market, and the timing of the debt reporting process slippage may be encountered on both entering and exiting the `Autopool`. It is very important to always check the shares received on entering, and the assets received on exiting, are greater than an expected amount. [4626-compliance#slippage](https://docs.auto.finance/developer-docs/integrating/4626-compliance#slippage).**

## Impact Details
Due to no slippage the vault will receive less shares than expected.

## References
[TokeAutoEth](https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/strategies/mainnet/TokeAutoEth.sol) [TokeAutoUSDStrategy](https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/strategies/mainnet/TokeAutoUSDStrategy.sol)

## Proof of Concept
Add the following file to `test/strategies` dir with the name `POC.t.sol` :

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;
// Adjust these imports to your layout

import {TokeAutoEthStrategy} from "src/strategies/mainnet/TokeAutoEth.sol";
import {BaseStrategyTest} from "../libraries/BaseStrategyTest.sol";
import {IMYTStrategy} from "../../interfaces/IMYTStrategy.sol";
import {MYTStrategy} from "../../MYTStrategy.sol";
import {AlchemistAllocator} from "../../AlchemistAllocator.sol";
import {MockAlchemistAllocator} from "../mocks/MockAlchemistAllocator.sol";
import {IERC4626Like} from "src/strategies/mainnet/TokeAutoEth.sol";
import {IVaultV2} from "../../../lib/vault-v2/src/interfaces/IVaultV2.sol";
import {console} from "forge-std/console.sol";

interface IERC20 {
    function balanceOf(address account) external view returns (uint256);
}



contract MockTokeAutoEthStrategy is TokeAutoEthStrategy {
    constructor(
        address _myt,
        StrategyParams memory _params,
        address _autoEth,
        address _router,
        address _rewarder,
        address _weth,
        address _oracle,
        address _permit2Address
    ) TokeAutoEthStrategy(_myt, _params, _autoEth, _router, _rewarder, _weth, _oracle, _permit2Address) {}
}



contract POCTest is BaseStrategyTest {
    address public constant TOKE_AUTO_ETH_VAULT = 0x0A2b94F6871c1D7A32Fe58E1ab5e6deA2f114E56;
    address public constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
    address public constant MAINNET_PERMIT2 = 0x000000000022d473030f1dF7Fa9381e04776c7c5;
    address public constant AUTOPILOT_ROUTER = 0x37dD409f5e98aB4f151F4259Ea0CC13e97e8aE21;
    address public constant REWARDER = 0x60882D6f70857606Cdd37729ccCe882015d1755E;
    address public constant ORACLE = 0x61F8BE7FD721e80C0249829eaE6f0DAf21bc2CaC;

    function getStrategyConfig() internal pure override returns (IMYTStrategy.StrategyParams memory) {
        return IMYTStrategy.StrategyParams({
            owner: address(1),
            name: "TokeAutoEth",
            protocol: "TokeAutoEth",
            riskClass: IMYTStrategy.RiskClass.MEDIUM,
            cap: 10_000e18,
            globalCap: 1e18,
            estimatedYield: 100e18,
            additionalIncentives: false,
            slippageBPS: 1
        });
    }

    function getTestConfig() internal pure override returns (TestConfig memory) {
        return TestConfig({vaultAsset: WETH, vaultInitialDeposit: 1000e18, absoluteCap: 10_000e18, relativeCap: 1e18, decimals: 18});
    }

    function createStrategy(address vault, IMYTStrategy.StrategyParams memory params) internal override returns (address) {
        return address(new MockTokeAutoEthStrategy(vault, params, TOKE_AUTO_ETH_VAULT, AUTOPILOT_ROUTER, REWARDER, WETH, ORACLE, MAINNET_PERMIT2));
    }

    function getForkBlockNumber() internal pure override returns (uint256) {
        return 22024000;
    }

    function getRpcUrl() internal view override returns (string memory) {
        return vm.envString("MAINNET_RPC_URL");
    }
  

    function test_strategy_auto_slippage() public {
        uint256 amountToAllocate = 40e18;
        uint256 expectedShares = IERC4626Like(TOKE_AUTO_ETH_VAULT).previewDeposit(amountToAllocate); // 1. the vault initiate the transaction to deposit assets in AutoPool
        console.log("Expected share the vaultShares would received " , expectedShares);
        vm.rollFork(block.number + (2)); // Assume the transaction has been in mempool for these 2 blocks
        uint256 receivedShares = IERC4626Like(TOKE_AUTO_ETH_VAULT).previewDeposit(amountToAllocate); // 2 . At this step suppose the transaction got mined 
        console.log("The share the vaultShares received " , receivedShares);
        assertLt(receivedShares, expectedShares , "No slippage happened");
        vm.startPrank(vault);
        deal(testConfig.vaultAsset, strategy, amountToAllocate);
        bytes memory prevAllocationAmount = abi.encode(0);
        IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocate, "", address(vault));
        uint256 initialRealAssets = IMYTStrategy(strategy).realAssets();
        require(initialRealAssets > 0, "Initial real assets is 0");
        uint256 vaultShares = IERC20(REWARDER).balanceOf(strategy);
        console.log("The share the vaultShares received " , vaultShares);
        assertEq(vaultShares , receivedShares , "shares not equal");
    }
    

}
```
And Run With command : `forge test --match-test test_strategy_auto_slippage  -vvv` 

---

## [M-01] In Stargate Incorrect Allocation Cap Accounting Leading to Unnecessary DoS

### Target
https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/strategies/optimism/StargateEthPoolStrategy.sol

### Impact(s)
Contract fails to deliver promised returns, but doesn't lose value

## Brief/Intro
While allocating deposits into the Stargate strategy, the implementation removes the dust amount from the provided deposit value and only deposits the remaining amount into the strategy.However, the vault still records the full provided amount (including the removed dust) in the allocation caps mapping.
As a result, the stored allocation value becomes inaccurate, leading to a situation where the system incorrectly reports that the allocation cap has been reached even though the actual deposited amount is lower.

## Vulnerability Details
To understand this behavior, let’s first examine the Vault’s allocation flow: allocate -> allocateInternal

```solidity
/v3-poc/lib/vault-v2/src/VaultV2.sol:570
570:     function allocateInternal(address adapter, bytes memory data, uint256 assets) internal {
571:         (bytes32[] memory ids, int256 change) = IAdapter(adapter).allocate(data, assets, msg.sig, msg.sender);
572: 
573:         for (uint256 i; i < ids.length; i++) {
574:             Caps storage _caps = caps[ids[i]];
575:             _caps.allocation = (int256(_caps.allocation) + change).toUint256();
576: 
577:             require(_caps.absoluteCap > 0, ErrorsLib.ZeroAbsoluteCap());
578:             require(_caps.allocation <= _caps.absoluteCap, ErrorsLib.AbsoluteCapExceeded());
579:             require(
580:                 _caps.relativeCap == WAD || _caps.allocation <= firstTotalAssets.mulDivDown(_caps.relativeCap, WAD),
581:                 ErrorsLib.RelativeCapExceeded()
582:             );
583:         }
584:         emit EventsLib.Allocate(msg.sender, adapter, assets, ids, change);
585:     }
586: 
```

At `line 571`, the value stored in change is added to `caps.allocation`.
Now, let’s take a closer look at the Stargate strategy:

```solidity
/v3-poc/src/strategies/optimism/StargateEthPoolStrategy.sol:42
42:     function _allocate(uint256 amount) internal override returns (uint256) {
43:         require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "not enough WETH");
44:         // unwrap to native ETH for Pool Native
45:         weth.withdraw(amount);
46:         uint256 amountToDeposit = (amount / 1e12) * 1e12;
47:         uint256 dust = amount - amountToDeposit;
48:         if (dust > 0) {
49:             emit StrategyAllocationLoss("Strategy allocation loss due to rounding.", amount, amountToDeposit);
50:         }
51:         pool.deposit{value: amountToDeposit}(address(this), amountToDeposit);
52:         return amount;
53:     }
54: 
```

Here we can observe that at `line 47`, the dust is subtracted from the provided amount before depositing into the strategy, while at `line 52`, the function returns the original amount (including the dust).

By combining both code snippets, we can summarize the behavior as follows:

```bash
1. Admin allocates: 3999999999999999996
2. After removing dust: 3999999000000000000
3. The _allocate function still returns 3999999999999999996, and the vault records caps.allocation = 3999999999999999996.
Even after a full deallocation, the vault still shows caps.allocation = 999999999996, which corresponds exactly to the dust amount stored in step 3.
```
## Impact Details
Due to incorrect `caps.allocation` calculation, the vault reports inaccurate cap values, which can cause unexpected reverts and potential DoS during allocation.

## References
[StargateEthPoolStrategy.sol#L46-L52](https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/strategies/optimism/StargateEthPoolStrategy.sol#L46-L52)

## Proof of Concept
Add Following file to `test/strategies` Dir with the name `POC.t.sol` :

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;
// Adjust these imports to your layout

import {BaseStrategyTest} from "../libraries/BaseStrategyTest.sol";
import {IMYTStrategy} from "../../interfaces/IMYTStrategy.sol";
import {AlchemistAllocator} from "../../AlchemistAllocator.sol";

import {StargateEthPoolStrategy} from "../../strategies/optimism/StargateEthPoolStrategy.sol";
import {IVaultV2} from "../../../lib/vault-v2/src/interfaces/IVaultV2.sol";
import {console} from "forge-std/console.sol";


contract MockStargateEthPoolStrategy is StargateEthPoolStrategy {
    constructor(address _myt, StrategyParams memory _params, address _weth, address _pool, address _permit2Address)
        StargateEthPoolStrategy(_myt, _params, _weth, _pool, _permit2Address)
    {}
}

contract MockAlchemistAllocator is AlchemistAllocator {
    constructor(address _myt, address _admin, address _operator) AlchemistAllocator(_myt, _admin, _operator) {}
}


contract POCTest is BaseStrategyTest {
    address public constant STARGATE_ETH_POOL = 0xe8CDF27AcD73a434D661C84887215F7598e7d0d3;
    address public constant WETH = 0x4200000000000000000000000000000000000006;
    address public constant OPTIMISM_PERMIT2 = 0x000000000022d473030f1dF7Fa9381e04776c7c5;

    function getStrategyConfig() internal pure override returns (IMYTStrategy.StrategyParams memory) {
        return IMYTStrategy.StrategyParams({
            owner: address(1),
            name: "StargateEthPool",
            protocol: "StargateEthPool",
            riskClass: IMYTStrategy.RiskClass.HIGH,
            cap: 10_000e18,
            globalCap: 1e18,
            estimatedYield: 100e18,
            additionalIncentives: false,
            slippageBPS: 1
        });
    }

    function getTestConfig() internal pure override returns (TestConfig memory) {
        return TestConfig({vaultAsset: WETH, vaultInitialDeposit: 10e18, absoluteCap: 10_000e18, relativeCap: 1e18, decimals: 18});
    }

    function createStrategy(address vault, IMYTStrategy.StrategyParams memory params) internal override returns (address) {
        return address(new MockStargateEthPoolStrategy(vault, params, WETH, STARGATE_ETH_POOL, OPTIMISM_PERMIT2));
    }

    function getForkBlockNumber() internal pure override returns (uint256) {
        return 141_751_698;
    }

    function getRpcUrl() internal view override returns (string memory) {
        return vm.envString("OPTIMISM_RPC_URL");
    }

    function test_allocation_caps() public {
        // set 1 : Allocator and strategy setup
        vm.startPrank(admin);
        allocator = address(new MockAlchemistAllocator(address(vault), admin, operator));
        vm.stopPrank();
        vm.startPrank(curator);
        _vaultSubmitAndFastForward(abi.encodeCall(IVaultV2.setIsAllocator, (address(allocator), true)));
        IVaultV2(vault).setIsAllocator(address(allocator), true);
        _vaultSubmitAndFastForward(abi.encodeCall(IVaultV2.addAdapter, address(strategy)));
        IVaultV2(vault).addAdapter(address(strategy));
        vm.stopPrank();
        // allocate and deallocate with dust amount to check allocation caps mapping in vault
        uint256 amountToAllocate=3999999999999999996;
         uint256 amountToDeallocate = 3999999000000000000; //this amount will got deposited in Stragate Strategy
        vm.startPrank(vault);
        deal(WETH, strategy, amountToAllocate);
        // allocation before
        bytes32 strategyId = IMYTStrategy(strategy).adapterId();
        uint256 allocationBefore = IVaultV2(vault).allocation(strategyId);
        console.log("Allocation before: %s", allocationBefore);
        vm.startPrank(admin);
        MockAlchemistAllocator(address(allocator)).allocate(address(strategy), amountToAllocate); // it will only allocate the amount after removing dust
        vm.stopPrank();
        // allocation 
        uint256 allocationAfter= IVaultV2(vault).allocation(strategyId); // the allocation caps stores the complete amount passed in allocation
        console.log("Allocation After allocate: %s", allocationAfter);
        vm.startPrank(admin);
        MockAlchemistAllocator(address(allocator)).deallocate(address(strategy), amountToDeallocate); // even after deallocation we have dust amount in allocation caps mapping
        vm.stopPrank();
         allocationAfter= IVaultV2(vault).allocation(strategyId);
        console.log("Allocation After deallocate: %s", allocationAfter);
    }

}
```

Run With Command : `forge test --match-test test_allocation_caps -vvv --decode-internal`

---

## [M-02] When the strategy is at a loss, the assets cannot be withdrawn

### Target
https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/strategies/mainnet/TokeAutoEth.sol

### Impact(s)
Temporary freezing of funds for at least 24 hour

## Brief/Intro
The code indicates that when deallocating from a strategy, we should use the `previewAdjustedWithdraw` function to determine the amount to be passed to the deallocate function. However, if the strategy is operating at a loss and returns less than the expected amount, the deallocate function will revert.

## Vulnerability Details
From the allocation contract, where the vault’s deallocate function is invoked, the comments indicate that the `previewAdjustedWithdraw` function should be used.

```solidity
/v3-poc/src/AlchemistAllocator.sol:46
46:     /// @notice When deallocating total amounts,
47:     /// consider using IMYTStrategy(address(strategy)).previewAdjustedWithdraw(amount)
48:     /// as the amount to be passed in for `amount`
49:     /// to estimate the correct amount that can be fully withdrawn,
50:     /// accounting for losses due to slippage, protocol fees, and rounding differences
51:     function deallocate(address adapter, uint256 amount) external {
52:         require(msg.sender == admin || operators[msg.sender], "PD");
53:         bytes32 id = IMYTStrategy(adapter).adapterId();
54:         uint256 absoluteCap = vault.absoluteCap(id);
55:         uint256 relativeCap = vault.relativeCap(id);
56:         // FIXME get this from the StrategyClassificationProxy for the respective risk class
57:         uint256 daoTarget = type(uint256).max;
58:         uint256 adjusted = absoluteCap < relativeCap ? absoluteCap : relativeCap;
59:         if (msg.sender != admin) {
60:             // caller is operator
61:             adjusted = adjusted < daoTarget ? adjusted : daoTarget;
62:         }
63:         // pass the old allocation to the adapter
64:         bytes memory oldAllocation = abi.encode(vault.allocation(id));
65:         vault.deallocate(adapter, oldAllocation, amount);
66:     }
```

The vault’s deallocate function then calls the strategy’s `_deallocate` function. In this analysis, I’ll focus on the `TokeAutoEth` strategy, though the issue can occur in other strategies as well.

```solidity
/v3-poc/src/strategies/mainnet/TokeAutoEth.sol:68
68:     function _deallocate(uint256 amount) internal override returns (uint256) {
69:         uint256 sharesNeeded = autoEth.convertToShares(amount);
70:         uint256 actualSharesHeld = rewarder.balanceOf(address(this));
71:         uint256 shareDiff =  actualSharesHeld - sharesNeeded;
72:         if (shareDiff <= 1e18) {
73:             // account for vault rounding up
74:             sharesNeeded = actualSharesHeld;
75:         }
76:         // withdraw shares, claim any rewards
77:         rewarder.withdraw(address(this), sharesNeeded, true);
78:         uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));
79:         autoEth.redeem(sharesNeeded, address(this), address(this));
80:         uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));
81:         uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore;
82:         if (wethRedeemed < amount) {
83:             emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, wethRedeemed);
84:         }
85:         require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount,"Strategy balance is less than the amount needed");
86:         TokenUtils.safeApprove(address(weth), msg.sender, amount);
87:         return amount;
88:     }
``` 
At `line 82`, if the strategy is at a loss, it will return fewer assets than the provided amount.
Then, at `line 85`, there’s a requirement that the strategy’s balance must be greater than or equal to the given amount.
Therefore, in the case of a loss, this `deallocate` call will always revert.

## Impact Details
DoS can occur when the strategy is at a loss, resulting in no assets being withdrawn from the strategy.

## References
[TokeAutoEth.sol#L81-L84](https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/strategies/mainnet/TokeAutoEth.sol#L81-L84)

## Recommendation
A potential fix could be to update the amount to match the actual assets received after withdrawing from the strategy in case of a loss.

```diff
diff --git a/src/strategies/mainnet/TokeAutoEth.sol b/src/strategies/mainnet/TokeAutoEth.sol
index ce58a30..440b2ef 100644
--- a/src/strategies/mainnet/TokeAutoEth.sol
+++ b/src/strategies/mainnet/TokeAutoEth.sol
@@ -79,9 +80,10 @@ contract TokeAutoEthStrategy is MYTStrategy {
         uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));
         uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore;
         if (wethRedeemed < amount) {
+            amount = wethRedeemed;
             emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, wethRedeemed);
         }
```

## Proof of Concept
Add the following file in `test/strategies` Dir with the name `POC.t.sol` :

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;
// Adjust these imports to your layout

import {TokeAutoEthStrategy} from "src/strategies/mainnet/TokeAutoEth.sol";
import {BaseStrategyTest} from "../libraries/BaseStrategyTest.sol";
import {IMYTStrategy} from "../../interfaces/IMYTStrategy.sol";
import {MYTStrategy} from "../../MYTStrategy.sol";
import {AlchemistAllocator} from "../../AlchemistAllocator.sol";
import {MockAlchemistAllocator} from "../mocks/MockAlchemistAllocator.sol";
import {IERC4626Like} from "src/strategies/mainnet/TokeAutoEth.sol";
import {IVaultV2} from "../../../lib/vault-v2/src/interfaces/IVaultV2.sol";
import {console} from "forge-std/console.sol";


contract MockTokeAutoEthStrategy is TokeAutoEthStrategy {
    constructor(
        address _myt,
        StrategyParams memory _params,
        address _autoEth,
        address _router,
        address _rewarder,
        address _weth,
        address _oracle,
        address _permit2Address
    ) TokeAutoEthStrategy(_myt, _params, _autoEth, _router, _rewarder, _weth, _oracle, _permit2Address) {}
}



contract POCTest is BaseStrategyTest {
    address public constant TOKE_AUTO_ETH_VAULT = 0x0A2b94F6871c1D7A32Fe58E1ab5e6deA2f114E56;
    address public constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
    address public constant MAINNET_PERMIT2 = 0x000000000022d473030f1dF7Fa9381e04776c7c5;
    address public constant AUTOPILOT_ROUTER = 0x37dD409f5e98aB4f151F4259Ea0CC13e97e8aE21;
    address public constant REWARDER = 0x60882D6f70857606Cdd37729ccCe882015d1755E;
    address public constant ORACLE = 0x61F8BE7FD721e80C0249829eaE6f0DAf21bc2CaC;

    function getStrategyConfig() internal pure override returns (IMYTStrategy.StrategyParams memory) {
        return IMYTStrategy.StrategyParams({
            owner: address(1),
            name: "TokeAutoEth",
            protocol: "TokeAutoEth",
            riskClass: IMYTStrategy.RiskClass.MEDIUM,
            cap: 10_000e18,
            globalCap: 1e18,
            estimatedYield: 100e18,
            additionalIncentives: false,
            slippageBPS: 1
        });
    }

    function getTestConfig() internal pure override returns (TestConfig memory) {
        return TestConfig({vaultAsset: WETH, vaultInitialDeposit: 1000e18, absoluteCap: 10_000e18, relativeCap: 1e18, decimals: 18});
    }

    function createStrategy(address vault, IMYTStrategy.StrategyParams memory params) internal override returns (address) {
        return address(new MockTokeAutoEthStrategy(vault, params, TOKE_AUTO_ETH_VAULT, AUTOPILOT_ROUTER, REWARDER, WETH, ORACLE, MAINNET_PERMIT2));
    }

    function getForkBlockNumber() internal pure override returns (uint256) {
        return 22024000;
    }

    function getRpcUrl() internal view override returns (string memory) {
        return vm.envString("MAINNET_RPC_URL");
    }
  

    function test_strategy_DoS_when_in_loss() public {
        uint256 amountToAllocate = 40e18;
        uint256 amountToDeallocate = IMYTStrategy(address(strategy)).previewAdjustedWithdraw(amountToAllocate);
        vm.startPrank(vault);
        deal(testConfig.vaultAsset, strategy, amountToAllocate);
        bytes memory prevAllocationAmount = abi.encode(0);
        IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocate, "", address(vault));
        bytes memory prevAllocationAmount2 = abi.encode(amountToAllocate);
        vm.expectRevert("Strategy balance is less than the amount needed");
        IMYTStrategy(strategy).deallocate(prevAllocationAmount2, amountToDeallocate, "", address(vault));
        vm.stopPrank();
    }

}
```
Run With Command : `forge test --match-test test_strategy_DoS_when_in_loss -vvv --decode-internal`

---

## [M-03] A griefer can cause a permanent DoS in TokeAutoETH/TokeAutoUSDCStrategy::allocate.

### Target
https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/strategies/mainnet/TokeAutoEth.sol

### Impact(s)
Temporary freezing of funds for at least 24 hour
Contract fails to deliver promised returns, but doesn't lose value
Temporary freezing of funds for at least 1 hour

## Brief/Intro
In the TokeAuto allocation we use `router::depositMax`, which attempts to pull the entire token balance from the strategy contract but we only approve the exact amount intended for the current allocation. If the strategy’s balance exceeds the approved amount, `depositMax` will revert. An attacker could exploit this to cause a permanent or prolonged DoS by donating to the strategy and repeatedly triggering the revert.

## Vulnerability Details
When allocating into TokeAuto we use the router contract and call its `depositMax` function:

```solidity
/v3-poc/src/strategies/mainnet/TokeAutoEth.sol:56
56:     function _allocate(uint256 amount) internal override returns (uint256) {
57:         require(
58:             TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount,
59:             "Strategy balance is less than amount"
60:         );
61:         TokenUtils.safeApprove(address(weth), address(router), amount);
62:         uint256 shares = router.depositMax(autoEth, address(this), 0);
63:         TokenUtils.safeApprove(address(autoEth), address(rewarder), shares);
64:         rewarder.stake(address(this), shares);
```

We only approve the exact amount intended for the current allocation, so the router can pull only that approved amount from the strategy contract. However, inside `router::depositMax` the implementation may attempt to pull the strategy’s entire balance up to `maxDeposit`:

```solidity
    function depositMax(
        IAutopool vault,
        address to,
        uint256 minSharesOut
    ) public payable override returns (uint256 sharesOut) {
        IERC20 asset = IERC20(vault.asset());
        uint256 assetBalance = asset.balanceOf(msg.sender);
        uint256 maxDeposit = vault.maxDeposit(to);
        uint256 amount = maxDeposit < assetBalance ? maxDeposit : assetBalance;
        pullToken(asset, amount, address(this));
    ...
    }
```

Concretely: if the strategy balance is `100 WETH` but the allocation amount (and thus the approved amount) is `99 WETH`, `depositMax` can revert. An attacker can weaponize this by donating a tiny amount 1 wei to the strategy right before an allocation front-running the allocate transaction to trigger a revert.

## Impact Details
A single 1-wei donation can be exploited to halt allocations to `TokeAutoETH/TokeAutoUSDC`, causing prolonged disruption of yield operations and exposing the protocol to financial risk.

## References
TokeAutoEth : [https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/strategies/mainnet/TokeAutoEth.sol](https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/strategies/mainnet/TokeAutoEth.sol)

TokeAutoUSDStrategy : [https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/strategies/mainnet/TokeAutoUSDStrategy.sol](https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/strategies/mainnet/TokeAutoUSDStrategy.sol)

AutopilotRouter : [https://vscode.blockscan.com/ethereum/0x37dD409f5e98aB4f151F4259Ea0CC13e97e8aE21](https://vscode.blockscan.com/ethereum/0x37dD409f5e98aB4f151F4259Ea0CC13e97e8aE21)

## Proof of Concept
Step-by-step POC explanation:

`. Admin attempts to allocate 40 WETH to the TokeAutoEth strategy.
2. Attacker watches the allocation transaction in the mempool.
3. Attacker front-runs the allocation and donates 1 wei to the strategy.
4. When the admin’s allocation executes, it reverts because only 40 WETH was approved but the strategy balance is now 40 WETH + 1 wei.
5. Inside the call flow, router::depositMax attempts to pull the strategy’s entire balance, causing the revert.

Add following File `POC.t.sol` to `test/strategies` dir:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;
// Adjust these imports to your layout

import {TokeAutoEthStrategy} from "src/strategies/mainnet/TokeAutoEth.sol";
import {BaseStrategyTest} from "../libraries/BaseStrategyTest.sol";
import {IMYTStrategy} from "../../interfaces/IMYTStrategy.sol";
import {MYTStrategy} from "../../MYTStrategy.sol";
import {AlchemistAllocator} from "../../AlchemistAllocator.sol";
import {MockAlchemistAllocator} from "../mocks/MockAlchemistAllocator.sol";
import {IERC4626Like} from "src/strategies/mainnet/TokeAutoEth.sol";
import {IVaultV2} from "../../../lib/vault-v2/src/interfaces/IVaultV2.sol";
import {console} from "forge-std/console.sol";
interface IERC20 {
    function transfer(address to, uint256 value) external returns (bool);
}


contract MockTokeAutoEthStrategy is TokeAutoEthStrategy {
    constructor(
        address _myt,
        StrategyParams memory _params,
        address _autoEth,
        address _router,
        address _rewarder,
        address _weth,
        address _oracle,
        address _permit2Address
    ) TokeAutoEthStrategy(_myt, _params, _autoEth, _router, _rewarder, _weth, _oracle, _permit2Address) {}
}



contract POCTest is BaseStrategyTest {
    address public constant TOKE_AUTO_ETH_VAULT = 0x0A2b94F6871c1D7A32Fe58E1ab5e6deA2f114E56;
    address public constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
    address public constant MAINNET_PERMIT2 = 0x000000000022d473030f1dF7Fa9381e04776c7c5;
    address public constant AUTOPILOT_ROUTER = 0x37dD409f5e98aB4f151F4259Ea0CC13e97e8aE21;
    address public constant REWARDER = 0x60882D6f70857606Cdd37729ccCe882015d1755E;
    address public constant ORACLE = 0x61F8BE7FD721e80C0249829eaE6f0DAf21bc2CaC;

    function getStrategyConfig() internal pure override returns (IMYTStrategy.StrategyParams memory) {
        return IMYTStrategy.StrategyParams({
            owner: address(1),
            name: "TokeAutoEth",
            protocol: "TokeAutoEth",
            riskClass: IMYTStrategy.RiskClass.MEDIUM,
            cap: 10_000e18,
            globalCap: 1e18,
            estimatedYield: 100e18,
            additionalIncentives: false,
            slippageBPS: 1
        });
    }

    function getTestConfig() internal pure override returns (TestConfig memory) {
        return TestConfig({vaultAsset: WETH, vaultInitialDeposit: 1000e18, absoluteCap: 10_000e18, relativeCap: 1e18, decimals: 18});
    }

    function createStrategy(address vault, IMYTStrategy.StrategyParams memory params) internal override returns (address) {
        return address(new MockTokeAutoEthStrategy(vault, params, TOKE_AUTO_ETH_VAULT, AUTOPILOT_ROUTER, REWARDER, WETH, ORACLE, MAINNET_PERMIT2));
    }

    function getForkBlockNumber() internal pure override returns (uint256) {
        return 23688417;
    }

    function getRpcUrl() internal view override returns (string memory) {
        return vm.envString("MAINNET_RPC_URL");
    }
  
    function test_strategy_allocation() public {
        address attacker = makeAddr("Attacker");
        deal(testConfig.vaultAsset, attacker, 1e18);
        vm.startPrank(curator);
        _vaultSubmitAndFastForward(abi.encodeCall(IVaultV2.setIsAllocator, (address(allocator), true)));
        IVaultV2(vault).setIsAllocator(address(allocator), true);
        _vaultSubmitAndFastForward(abi.encodeCall(IVaultV2.addAdapter, address(strategy)));
        IVaultV2(vault).addAdapter(address(strategy));
        vm.stopPrank();
        uint256 amountToAllocate = 40e18; 
        deal(testConfig.vaultAsset, strategy, amountToAllocate);
        vm.startPrank(attacker);
        IERC20(WETH).transfer(address(strategy), 1); // Attacker front runs the allocate call and donate 1 WEI to strategy
        vm.stopPrank();
        vm.startPrank(admin);
        MockAlchemistAllocator(allocator).allocate(address(strategy), amountToAllocate); // the strategy has 40e18 WETH
        vm.stopPrank();

    }
}
```

Run the test case with command : `forge test --mc POCTest  --match-test test_strategy_allocation  -vvv --decode-internal`

---

## [L-01] The `TokeAuto` strategies implementation does not accurately report the actual assets held by the strategy

### Target
https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/strategies/mainnet/TokeAutoEth.sol

### Impact(s)
Contract fails to deliver promised returns, but doesn't lose value

## Brief/Intro
The strategy includes a realAssets function intended to report the current assets held, but due to incorrect implementation in the `TokeAutoEth` and `TokeAutoUSDStrategy` contracts, it consistently reports a higher value than what can actually be redeemed.

## Vulnerability Details
Let have a look how the realAssets() calculate the assets held in strategy:

```solidity
/v3-poc/src/strategies/mainnet/TokeAutoEth.sol:152
152:     function realAssets() external view override returns (uint256) {
153:         uint256 shares = rewarder.balanceOf(address(this));
154:         uint256 assets = autoEth.convertToAssets(shares);
155:         return assets;
156:     }
```
At `line 153`, the code fetches the Toke shares from the rewarder contract and calls `autoEth::convertToAssets`, which returns the potential redeemable assets but not the actual assets.

In my PoC, I used the `previewRedeem` function on the `autoEth` contract to measure the difference, which was approximately `0.00117 WETH` for a `40 WETH` deposit. this discrepancy grows with larger deposits.

## Impact Details
Overstating realAssets makes the strategy unreliable, undermines trust, and can mislead users about the actual funds available for withdrawal.

## References
[https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/strategies/mainnet/TokeAutoEth.sol#L146-L150](https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/strategies/mainnet/TokeAutoEth.sol#L146-L150)

## Proof of Concept
Note: This report assume that report: `58728` recommended fix has been implemented Add Following file to `test/strategies` Dir with the name `POC.t.sol` :

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;
// Adjust these imports to your layout

import {TokeAutoEthStrategy} from "src/strategies/mainnet/TokeAutoEth.sol";
import {BaseStrategyTest} from "../libraries/BaseStrategyTest.sol";
import {IMYTStrategy} from "../../interfaces/IMYTStrategy.sol";
import {MYTStrategy} from "../../MYTStrategy.sol";
import {AlchemistAllocator} from "../../AlchemistAllocator.sol";
import {MockAlchemistAllocator} from "../mocks/MockAlchemistAllocator.sol";
import {IERC4626Like} from "src/strategies/mainnet/TokeAutoEth.sol";
import {IVaultV2} from "../../../lib/vault-v2/src/interfaces/IVaultV2.sol";
import {console} from "forge-std/console.sol";

interface IERC20 {
    function transfer(address to, uint256 value) external returns (bool);
    function approve(address spender, uint256 amount) external returns (bool);
    function balanceOf(address account) external view returns (uint256);
}

interface IAutopool {
    struct AssetBreakdown {
        uint256 totalIdle;
        uint256 totalDebt;
        uint256 totalDebtMin;
        uint256 totalDebtMax;
    }
    function getAssetBreakdown() external view returns (AssetBreakdown memory);
    function getDestinations() external view returns (address[] memory);
    function getDebtReportingQueue() external view returns (address[] memory);
}

// AutoEth's previewDeposit/previewRedeem are non-view in this codebase.
// Calling through a view IERC4626 interface triggers staticcall and fails.
interface IAutoEthNonView {
    function previewDeposit(uint256 assets) external returns (uint256 shares);
    function previewRedeem(uint256 shares) external returns (uint256 assets);
}


contract MockTokeAutoEthStrategy is TokeAutoEthStrategy {
    constructor(
        address _myt,
        StrategyParams memory _params,
        address _autoEth,
        address _router,
        address _rewarder,
        address _weth,
        address _oracle,
        address _permit2Address
    ) TokeAutoEthStrategy(_myt, _params, _autoEth, _router, _rewarder, _weth, _oracle, _permit2Address) {}
}



contract POCTest is BaseStrategyTest {
    address public constant TOKE_AUTO_ETH_VAULT = 0x0A2b94F6871c1D7A32Fe58E1ab5e6deA2f114E56;
    address public constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
    address public constant MAINNET_PERMIT2 = 0x000000000022d473030f1dF7Fa9381e04776c7c5;
    address public constant AUTOPILOT_ROUTER = 0x37dD409f5e98aB4f151F4259Ea0CC13e97e8aE21;
    address public constant REWARDER = 0x60882D6f70857606Cdd37729ccCe882015d1755E;
    address public constant ORACLE = 0x61F8BE7FD721e80C0249829eaE6f0DAf21bc2CaC;

    function getStrategyConfig() internal pure override returns (IMYTStrategy.StrategyParams memory) {
        return IMYTStrategy.StrategyParams({
            owner: address(1),
            name: "TokeAutoEth",
            protocol: "TokeAutoEth",
            riskClass: IMYTStrategy.RiskClass.MEDIUM,
            cap: 10_000e18,
            globalCap: 1e18,
            estimatedYield: 100e18,
            additionalIncentives: false,
            slippageBPS: 1
        });
    }

  
    // ==== Helpers ====

    function getTestConfig() internal pure override returns (TestConfig memory) {
        return TestConfig({vaultAsset: WETH, vaultInitialDeposit: 1000e18, absoluteCap: 10_000e18, relativeCap: 1e18, decimals: 18});
    }

    function createStrategy(address vault, IMYTStrategy.StrategyParams memory params) internal override returns (address) {
        return address(new MockTokeAutoEthStrategy(vault, params, TOKE_AUTO_ETH_VAULT, AUTOPILOT_ROUTER, REWARDER, WETH, ORACLE, MAINNET_PERMIT2));
    }

    function getForkBlockNumber() internal pure override returns (uint256) {
        return 22_089_302;
    }

    function getRpcUrl() internal view override returns (string memory) {
        return vm.envString("MAINNET_RPC_URL");
    }
  


        function test_strategy_auto_slippage() public {
        uint256 amountToAllocate = 400e18;
        uint256 amountToDeallocate = IMYTStrategy(address(strategy)).previewAdjustedWithdraw(amountToAllocate);
        vm.startPrank(vault);
        deal(testConfig.vaultAsset, strategy, amountToAllocate);
        bytes memory prevAllocationAmount = abi.encode(0);
        IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocate, "", address(vault));
        uint256 initialRealAssets = IMYTStrategy(strategy).realAssets();
        uint256 vaultShares = IERC20(REWARDER).balanceOf(strategy); // rewards holds the share of strategy in Auto pool
        uint256 previewWithdrawAmount = IAutoEthNonView(TOKE_AUTO_ETH_VAULT).previewRedeem(vaultShares); // calls the previewRedeem to check how much can we redeem in  assets
        console.log("The share the previewWithdrawAmount " , previewWithdrawAmount);
        console.log("The share the initialRealAssets  " , initialRealAssets);
        assertGt(initialRealAssets ,previewWithdrawAmount , "Initial real assets does not matched" );
         bytes memory prevAllocationAmount2 = abi.encode(amountToAllocate);
        IMYTStrategy(strategy).deallocate(prevAllocationAmount2, amountToDeallocate, "", address(vault));
         vm.stopPrank();
        assertEq(IERC20(WETH).balanceOf(strategy), previewWithdrawAmount , "Strategy Receive WETH Not Equal to previewWithdrawAmount");
        assertEq(IERC20(REWARDER).balanceOf(strategy) , 0 , "Strategy share in rewarder should be zero after full deallocation");
        assertNotEq(IERC20(WETH).balanceOf(strategy), initialRealAssets , "Strategy Receive WETH  Equal to realAssets Reported So No issue");


    }
    
}
```
run with command : `forge test --match-test test_strategy_auto_slippage -vvv --decode-internal`

---

## The Auto ETH and USDC staking rewards will stuck in vault

### Target
https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/strategies/mainnet/TokeAutoEth.sol

### Impact(s)
Permanent freezing of unclaimed royalties

## Brief/Intro
The `autoEth` and `autoUSDC` Share can be staked to earn rewards in the form of TOKE tokens. When withdrawing from the rewarder contract, the reward tokens are automatically claimed and transferred to the vault. However, there is no mechanism for these tokens to be withdrawn or distributed among the strategy’s shareholders.

## Vulnerability Details
I will focus on the `TokeAutoEth` strategy here, but the same logic applies to the `TokeAutoUSDCStrategy`. In the `TokeAutoEth` strategy, we first deposit `WETH` into the `AutoETH` contract and then deposit its shares into the rewarder contract.

```solidity
/v3-poc/src/strategies/mainnet/TokeAutoEth.sol:56
56:     function _allocate(uint256 amount) internal override returns (uint256) {
57:         require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than amount");
58:         TokenUtils.safeApprove(address(weth), address(router), amount);
59:         uint256 shares = router.depositMax(autoEth, address(this), 0);
60:         TokenUtils.safeApprove(address(autoEth), address(rewarder), shares);
61:         rewarder.stake(address(this), shares);
62:         return amount;
63:     }
```

When withdrawing `WETH`, we first need to withdraw the shares from the rewarder contract, and then use those shares to redeem WETH from the AutoETH contract. In the first step, when the shares are withdrawn from the rewarder contract, it also automatically claims the rewards accumulated for the strategy up to that point.

```solidity
/v3-poc/src/strategies/mainnet/TokeAutoEth.sol:67
67:     function _deallocate(uint256 amount) internal override returns (uint256) {
68:         uint256 sharesNeeded = autoEth.convertToShares(amount);
69:         uint256 actualSharesHeld = rewarder.balanceOf(address(this));
70:         uint256 shareDiff = actualSharesHeld - sharesNeeded;
71:         if (shareDiff <= 1e18) {
72:             // account for vault rounding up
73:             sharesNeeded = actualSharesHeld;
74:         }
75:         // withdraw shares, claim any rewards
76:         rewarder.withdraw(address(this), sharesNeeded, true);
77:         uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));
78:         autoEth.redeem(sharesNeeded, address(this), address(this));
79:         uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));
80:         uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore;
81:         if (wethRedeemed < amount) {
82:             emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, wethRedeemed);
83:         }
84:         require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than the amount needed");
85:         TokenUtils.safeApprove(address(weth), msg.sender, amount);
86:         return amount;
87:     }
```
At Link [https://vscode.blockscan.com/ethereum/0x60882D6f70857606Cdd37729ccCe882015d1755E](0x60882D6f70857606Cdd37729ccCe882015d1755E) , you can see that when the withdraw function is called, it also triggers the claim of the TOKE rewards.
```solidity
    function withdraw(address account, uint256 amount, bool claim) public {
        if (msg.sender != account && msg.sender != address(systemRegistry.autoPoolRouter())) {
            revert Errors.AccessDenied();
        }

        _withdraw(account, amount, claim);

        stakingToken.safeTransfer(account, amount);
    }
```

## Impact Details
The reward tokens, such as TOKE in this case, will remain permanently stuck in the Vault contract, as there is no mechanism to withdraw them or convert them into WETH or USDC for distribution as yield among the strategy’s shareholders As intended.

## References
Nil

### Proof of Concept
Apply following git Diff to `TokeAutoETHStrategy` and run test case with command : `forge test --mc TokeAutoETHStrategyTest  --match-test test_strategy_rewards_stuck  -vvv --decode-internal`

```diff
diff --git a/src/test/strategies/TokeAutoETHStrategy.t.sol b/src/test/strategies/TokeAutoETHStrategy.t.sol
index 2185b3c..0895a61 100644
--- a/src/test/strategies/TokeAutoETHStrategy.t.sol
+++ b/src/test/strategies/TokeAutoETHStrategy.t.sol
@@ -5,12 +5,18 @@ pragma solidity 0.8.28;
 import {TokeAutoEthStrategy} from "src/strategies/mainnet/TokeAutoEth.sol";
 import {BaseStrategyTest} from "../libraries/BaseStrategyTest.sol";
 import {IMYTStrategy} from "../../interfaces/IMYTStrategy.sol";
-
+import {MYTStrategy} from "../../MYTStrategy.sol";
+import {console} from "forge-std/console.sol";
 interface IERC20 {
     function approve(address spender, uint256 amount) external returns (bool);
+    function transfer(address to, uint256 amount) external returns (bool);
     function balanceOf(address a) external view returns (uint256);
 }
 
+interface IRewarder {
+    function queueNewRewards(uint256 newRewards) external;
+}
+
 /*contract TokeAutoEthStrategyTest is Test {
     // Addresses sourced from environment so you can swap networks/blocks easily
     address public constant AUTOETH = 0x0A2b94F6871c1D7A32Fe58E1ab5e6deA2f114E56;
@@ -148,6 +154,7 @@ contract TokeAutoETHStrategyTest is BaseStrategyTest {
     address public constant AUTOPILOT_ROUTER = 0x37dD409f5e98aB4f151F4259Ea0CC13e97e8aE21;
     address public constant REWARDER = 0x60882D6f70857606Cdd37729ccCe882015d1755E;
     address public constant ORACLE = 0x61F8BE7FD721e80C0249829eaE6f0DAf21bc2CaC;
+    address public constant TOKE = 0x2e9d63788249371f1DFC918a52f8d799F4a38C94;
 
     function getStrategyConfig() internal pure override returns (IMYTStrategy.StrategyParams memory) {
         return IMYTStrategy.StrategyParams({
@@ -172,7 +179,7 @@ contract TokeAutoETHStrategyTest is BaseStrategyTest {
     }
 
     function getForkBlockNumber() internal pure override returns (uint256) {
-        return 22_089_302;
+        return 23688417;
     }
 
     function getRpcUrl() internal view override returns (string memory) {
@@ -180,18 +187,29 @@ contract TokeAutoETHStrategyTest is BaseStrategyTest {
     }
 
     // Add any strategy-specific tests here
-    function test_strategy_deallocate_reverts_due_to_slippage(uint256 amountToAllocate, uint256 amountToDeallocate) public {
-        amountToAllocate = bound(amountToAllocate, 1e6, testConfig.vaultInitialDeposit);
-        amountToDeallocate = amountToAllocate;
+    function test_strategy_rewards_stuck() public {
+        uint256 amountToAllocate = 20e18; 
         vm.startPrank(vault);
         deal(testConfig.vaultAsset, strategy, amountToAllocate);
         bytes memory prevAllocationAmount = abi.encode(0);
         IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocate, "", address(vault));
         uint256 initialRealAssets = IMYTStrategy(strategy).realAssets();
         require(initialRealAssets > 0, "Initial real assets is 0");
+        // Queue some rewards to be claimed
+        vm.startPrank(address(0x8b4334d4812C530574Bd4F2763FcD22dE94A969B)); //TOKE Treasury
+        IERC20(0x2e9d63788249371f1DFC918a52f8d799F4a38C94).transfer(address(0x123cC4AFA59160C6328C0152cf333343F510e5A3), 100000e18); // transfer Treasury TOKE to an Whitelisted 
EOA
+        vm.stopPrank();
+
+        vm.startPrank(0x123cC4AFA59160C6328C0152cf333343F510e5A3);
+        IERC20(0x2e9d63788249371f1DFC918a52f8d799F4a38C94).approve(address(REWARDER), 50000e18);
+        IRewarder(REWARDER).queueNewRewards(50000e18); // Queue 50,000 TOKE as new rewards
+        vm.stopPrank();
+
+        vm.roll(block.number + 10000); // Fast Forward to accrue rewards
         bytes memory prevAllocationAmount2 = abi.encode(amountToAllocate);
-        vm.expectRevert();
-        IMYTStrategy(strategy).deallocate(prevAllocationAmount2, amountToDeallocate, "", address(vault));
         vm.stopPrank();
+        IMYTStrategy(strategy).claimRewards();
+       uint256 tokeBal =  IERC20(TOKE).balanceOf(address(MYTStrategy(strategy).MYT()));
+       require(tokeBal > 0 , "TOKE balance is not greater than 0");
     }
 }
```