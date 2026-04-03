# Burve

## Protocol Summary

Burve is a 16-token multi-swap for pegged assets launching on Berachain with rehypothecation yields, moving peg handling, an analytic stableswap solution, depeg-protection, and subset-LPing so users can limit themselves to tokens they feel safest in.
[Ethereum][Solidity][DeFi]

---

### [H-01] No Fee will be charges in case of `removeValueSingle`

#### Summary
When a user withdraws assets via `removeValueSingle`, the protocol uses `removedBalance` (which is `0`) to compute `realTax`.  
As a result, `realTax` is always `0`, allowing the user to bypass withdrawal fees entirely.

#### Root Cause
In [ValueFacet.sol#L235-L239](https://github.com/sherlock-audit/2025-04-burve-amankakar/blob/main/Burve/src/multi/facets/ValueFacet.sol#L235-L239) we multiple `removedBalance` with other values , But the `removedBalance = 0` here.

#### Internal Pre-conditions
The protocol Charge Fee on deposit and withdrawal of values.

#### External Pre-conditions
Nil

#### Attack Path
Nil
#### Impact
Since no fee is charged during `removeValueSingle` withdrawals, the protocol incurs a loss of value, undermining its fee model.

#### PoC

#### Mitigation
The Best Fix would be following:
```diff
        uint256 realTax = FullMath.mulDiv(
-           removedBalance,
+           realRemoved,
            nominalTax,
            removedNominal
        );

```
---

### [H-02] Missing `acceptOwnership` Selector from `BaseAdminFacet` for diamond cuts

#### Summary
The `BaseAdminFacet` includes key admin functions:
1. `transferOwnership`
2. `owner`
3. `adminRights`
4. `acceptOwnership`

However, the `acceptOwnership` function is not included in the diamond cut, making it inaccessible via the diamond proxy and preventing ownership transfer completion.

#### Root Cause
From the following code it can be seen that the `acceptOwnership` is not added to `adminSelectors` list.     
```solidity
{
            bytes4[] memory adminSelectors = new bytes4[](3);
            adminSelectors[0] = BaseAdminFacet.transferOwnership.selector;
            adminSelectors[1] = BaseAdminFacet.owner.selector;
            adminSelectors[2] = BaseAdminFacet.adminRights.selector; // @audit : no acceptOwnership function in list
            cuts[2] = FacetCut({
                facetAddress: address(new BaseAdminFacet()),
                action: FacetCutAction.Add,
                functionSelectors: adminSelectors
            });
        }
```
[Diamond.sol#L68-L76](https://github.com/sherlock-audit/2025-04-burve-amankakar/blob/main/Burve/src/multi/Diamond.sol#L68-L76)
#### Internal Pre-conditions
Nil
#### External Pre-conditions
Nil
#### Attack Path
The Admin calls the `transferOwnership` function. to transfer the owner right but the new owner can not be able to accept the ownership request.
#### Impact
The ownership of protocol can not transfer to new owner.
#### PoC
I have added following changed to `test/facets/MultiSetup.u.sol`:
```diff
diff --git a/Burve/test/facets/MultiSetup.u.sol b/Burve/test/facets/MultiSetup.u.sol
index 9eb0189..b75be6b 100644
--- a/Burve/test/facets/MultiSetup.u.sol
+++ b/Burve/test/facets/MultiSetup.u.sol
@@ -11,6 +11,7 @@ import {InitLib, BurveFacets} from "../../src/multi/InitLib.sol";
 import {SimplexDiamond} from "../../src/multi/Diamond.sol";
 import {SimplexFacet} from "../../src/multi/facets/SimplexFacet.sol";
 import {LockFacet} from "../../src/multi/facets/LockFacet.sol";
+import {BaseAdminFacet} from "Commons/Util/Admin.sol";
 import {MockERC20} from "../mocks/MockERC20.sol";
 import {MockERC4626} from "../mocks/MockERC4626.sol";
 import {StoreManipulatorFacet} from "./StoreManipulatorFacet.u.sol";
@@ -34,6 +35,7 @@ contract MultiSetupTest is Test {
     SimplexFacet public simplexFacet;
     SwapFacet public swapFacet;
     LockFacet public lockFacet;
+    BaseAdminFacet public adminFacet;
     StoreManipulatorFacet public storeManipulatorFacet; // testing only
 
     uint16 public closureId;
@@ -65,7 +67,7 @@ contract MultiSetupTest is Test {
         simplexFacet = SimplexFacet(diamond);
         swapFacet = SwapFacet(diamond);
         lockFacet = LockFacet(diamond);
-
+        adminFacet = BaseAdminFacet(diamond);
         _cutStoreManipulatorFacet();
         storeManipulatorFacet = StoreManipulatorFacet(diamond);
     }
```
The Test case added to `ValueFacet.t.sol`  test File:

```solidity
function testTransferOwner() public {
        vm.startPrank(owner);
        adminFacet.transferOwnership(alice);
        vm.stopPrank();
        vm.expectRevert();
        vm.startPrank(alice);
        adminFacet.acceptOwnership();
        vm.stopPrank();
    }
```
Run with command : `forge test --mt testTransferOwner -vvv`.
#### Mitigation
Apply The Following changes to `Diamond.sol` file:

```diff
diff --git a/Burve/src/multi/Diamond.sol b/Burve/src/multi/Diamond.sol
index d3de78f..0aa9433 100644
--- a/Burve/src/multi/Diamond.sol
+++ b/Burve/src/multi/Diamond.sol
@@ -65,10 +65,11 @@ contract SimplexDiamond is IDiamond {
         }
 
         {
-            bytes4[] memory adminSelectors = new bytes4[](3);
+            bytes4[] memory adminSelectors = new bytes4[](4);
             adminSelectors[0] = BaseAdminFacet.transferOwnership.selector;
             adminSelectors[1] = BaseAdminFacet.owner.selector;
-            adminSelectors[2] = BaseAdminFacet.adminRights.selector;
+            adminSelectors[2] = BaseAdminFacet.adminRights.selector; // @audit : no acceptOwner function in list
+            adminSelectors[3] = BaseAdminFacet.acceptOwnership.selector;
             cuts[2] = FacetCut({
                 facetAddress: address(new BaseAdminFacet()),
                 action: FacetCutAction.Add,
```
---

### [H-03] The `commit` function will corrupt the `withdraw/deposit` values into vault.
#### Summary
The `commit` function handles batched `withdraw/deposit` operations. If `assetsToWithdraw > assetsToDeposit`, it intends to cancel the deposit and subtract its value from `assetsToWithdraw`. However, the code mistakenly sets `assetsToDeposit = 0` before the subtraction, effectively subtracting zero. This results in the deposit value being ignored and may lead to lost funds.

#### Root Cause
The `assetsToDeposit` first sets to 0 than subtract from `assetsToWithdraw`.
[E4626.sol#L71-L74](https://github.com/sherlock-audit/2025-04-burve-amankakar/blob/main/Burve/src/multi/vertex/E4626.sol#L71-L74)

#### Internal Pre-conditions
The temporary variables hold both `assetsToWithdraw` and `assetsToDeposit`. When `assetsToWithdraw > assetsToDeposit` han this issue will occur.

#### External Pre-conditions
Nil
#### Attack Path
1. User Deposit assets
2. The protocol have accumalted earning in Fee.
3. User tries to claim the earning via `collectEarnings`
4. The `assetsToWithdraw > assetsToDeposit`.
#### Impact
The assets To Deposit will be lost , as the totalShare have already increased this will disturbed the internal shares calculation. 
#### PoC
Add the Following POC to : 
```solidity
    function testDespotAndWithdrawLoseValue() public {

        VertexId vid = VertexLib.newId(0);
        Vertex storage v = Store.load().vertices[vid];
        ClosureId cid = ClosureId.wrap(0x1);
        v.init(vid, address(token), address(e4626), VaultType.E4626);
        // But if we add some tokens to the vault, then it's fine.
        VaultProxy memory vProxy = VaultLib.getProxy(vid);
        vProxy.deposit(cid, 4e6);
        vProxy.commit();
        vProxy = VaultLib.getProxy(vid);
        vProxy.deposit(cid, 3e6);
        vProxy.withdraw(cid , 5e6);
        vProxy.commit();
        VaultTemp memory temp;
        vault.fetch(temp);
        assertEq(vault.totalVaultShares, 0);
        console.log("totalVaultShares" , vault.totalVaultShares);
        uint256 shares = vault.shares[cid];
        assertEq(shares, 0);
        console.log("shares" , shares);
        assertEq(vault.totalShares, shares);
   
    }
```
Run with command : `forge test --mt testDespotAndWithdrawLoseValue  -vvv`:
#### Note : Note this POC call vault directly but in normal case this issue will happen when user calls `collectEarnings` function. So in that case the `totalShares` will increase but `totalVaultShares` will not increase due no deposit occur.
#### Mitigation
One of the potential Fix could be Following :
```diff
diff --git a/Burve/src/multi/vertex/E4626.sol b/Burve/src/multi/vertex/E4626.sol
index 576f6f5..945ca8f 100644
--- a/Burve/src/multi/vertex/E4626.sol
+++ b/Burve/src/multi/vertex/E4626.sol
@@ -62,15 +62,14 @@ library VaultE4626Impl {
     function commit(VaultE4626 storage self, VaultTemp memory temp) internal {
         uint256 assetsToDeposit = temp.vars[1];
         uint256 assetsToWithdraw = temp.vars[2];
-
         if (assetsToDeposit > 0 && assetsToWithdraw > 0) {
             // We can net out and save ourselves some fees.
             if (assetsToDeposit > assetsToWithdraw) {
                 assetsToDeposit -= assetsToWithdraw;
                 assetsToWithdraw = 0;
             } else if (assetsToWithdraw > assetsToDeposit) {
+                assetsToWithdraw -= assetsToDeposit; // @audit-issue 4 : wrong flow here , it will still withdraw all the value bit deposit is set to 0 here
                 assetsToDeposit = 0;
-                assetsToWithdraw -= assetsToDeposit;
```

---

### [M-01] The Protocol does not align with the ERC4626 vault with Fees

#### Summary

Burve allows users to deposit assets into ERC4626 vaults (including vault with fee which are in-scope). However, it stores the asset amount directly in its own storage **without accounting for the vault's deposit/withdrawal fee**. As a result, users can withdraw the full amount without incurring the intended ERC4626 vault fees, effectively bypassing them.

#### Root Cause
when user add Value using `valueFacet.sol` The code calls the deposit function on vertex which deposit the assets into the linked vault, after this it add the user provided `value` to user assets storage.
[facets/ValueFacet.sol#L95C44-L95C49](https://github.com/sherlock-audit/2025-04-burve-amankakar/blob/main/Burve/src/multi/facets/ValueFacet.sol#L95C44-L95C49)
[facets/ValueFacet.sol#L134](https://github.com/sherlock-audit/2025-04-burve-amankakar/blob/main/Burve/src/multi/facets/ValueFacet.sol#L134)
#### Internal Pre-conditions
The link valut with the clouser must charge the deposit/withdraw fee.

#### External Pre-conditions
Nil
#### Attack Path
Nil
#### Impact
Since the protocol pays the fee to the ERC4626 vault but doesn't charge it to the user, the user can withdraw the full deposited amount if the vault holds sufficient liquidity. This allows the user to bypass fees entirely, leaving the protocol to absorb the cost.

#### PoC
For the POC we need to replace the `MockERC4626.sol` with the following code:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

import "openzeppelin-contracts/token/ERC20/extensions/ERC4626.sol";
import "openzeppelin-contracts/token/ERC20/ERC20.sol";
import  "openzeppelin-contracts/utils/math/Math.sol";

contract MockERC4626 is ERC4626 {
    using Math for uint256;
    constructor(
        ERC20 asset,
        string memory name,
        string memory symbol
    ) ERC20(name, symbol) ERC4626(asset) {}

    uint256 private constant _BASIS_POINT_SCALE = 1e4;

    // === Overrides ===

    /// @dev Preview taking an entry fee on deposit. See {IERC4626-previewDeposit}.
    function previewDeposit(uint256 assets) public view virtual override returns (uint256) {
        uint256 fee = _feeOnTotal(assets, _entryFeeBasisPoints());
        return super.previewDeposit(assets - fee);
    }

    /// @dev Preview adding an entry fee on mint. See {IERC4626-previewMint}.
    function previewMint(uint256 shares) public view virtual override returns (uint256) {
        uint256 assets = super.previewMint(shares);
        return assets + _feeOnRaw(assets, _entryFeeBasisPoints());
    }

    /// @dev Preview adding an exit fee on withdraw. See {IERC4626-previewWithdraw}.
    function previewWithdraw(uint256 assets) public view virtual override returns (uint256) {
        uint256 fee = _feeOnRaw(assets, _exitFeeBasisPoints());
        return super.previewWithdraw(assets + fee);
    }

    /// @dev Preview taking an exit fee on redeem. See {IERC4626-previewRedeem}.
    function previewRedeem(uint256 shares) public view virtual override returns (uint256) {
        uint256 assets = super.previewRedeem(shares);
        return assets - _feeOnTotal(assets, _exitFeeBasisPoints());
    }

    /// @dev Send entry fee to {_entryFeeRecipient}. See {IERC4626-_deposit}.
    function _deposit(address caller, address receiver, uint256 assets, uint256 shares) internal virtual override {
        uint256 fee = _feeOnTotal(assets, _entryFeeBasisPoints());
        address recipient = _entryFeeRecipient();

        super._deposit(caller, receiver, assets, shares);

        if (fee > 0 && recipient != address(this)) {
            SafeERC20.safeTransfer(IERC20(asset()), recipient, fee);
        }
    }

    /// @dev Send exit fee to {_exitFeeRecipient}. See {IERC4626-_deposit}.
    function _withdraw(
        address caller,
        address receiver,
        address owner,
        uint256 assets,
        uint256 shares
    ) internal virtual override {
        uint256 fee = _feeOnRaw(assets, _exitFeeBasisPoints());
        address recipient = _exitFeeRecipient();

        super._withdraw(caller, receiver, owner, assets, shares);

        if (fee > 0 && recipient != address(this)) {
            SafeERC20.safeTransfer(IERC20(asset()), recipient, fee);
        }
    }

    // === Fee configuration ===

    function _entryFeeBasisPoints() internal view virtual returns (uint256) {
        return 200; // replace with e.g. 100 for 1%
    }

    function _exitFeeBasisPoints() internal view virtual returns (uint256) {
        return 200; // replace with e.g. 100 for 1%
    }

    function _entryFeeRecipient() internal view virtual returns (address) {
        return address(0x1); // replace with e.g. a treasury address
    }

    function _exitFeeRecipient() internal view virtual returns (address) {
        return address(0x1); // replace with e.g. a treasury address
    }

    // === Fee operations ===

    /// @dev Calculates the fees that should be added to an amount `assets` that does not already include fees.
    /// Used in {IERC4626-mint} and {IERC4626-withdraw} operations.
    function _feeOnRaw(uint256 assets, uint256 feeBasisPoints) private pure returns (uint256) {
        return assets.mulDiv(feeBasisPoints, _BASIS_POINT_SCALE, Math.Rounding.Ceil);
    }

    /// @dev Calculates the fee part of an amount `assets` that already includes fees.
    /// Used in {IERC4626-deposit} and {IERC4626-redeem} operations.
    function _feeOnTotal(uint256 assets, uint256 feeBasisPoints) private pure returns (uint256) {
        return assets.mulDiv(feeBasisPoints, feeBasisPoints + _BASIS_POINT_SCALE, Math.Rounding.Ceil);
    }
}
```
As Now we have Fee Supported ERC4626 vault So lets add the following test case in `ValueFacet.t.sol`:
```solidity
function testAliceAddRemoveSingleForValue() public {
        vm.startPrank(alice);
        // Simply add and remove.
        uint256 beforeAssets = vaults[0].totalAssets();
        uint256 beforeShare = vaults[0].balanceOf(address(valueFacet));
        uint256 aliceBeforeBal = ERC20(tokens[0]).balanceOf(alice);
        valueFacet.addSingleForValue(alice, 0xF, tokens[0], 100e18, 0, 0);
        
        valueFacet.removeSingleForValue(alice, 0xF, tokens[0], 100e18-3, 0, 0);
        vm.stopPrank();
        uint256 afterShare = vaults[0].balanceOf(address(valueFacet));

        uint256 aliceAfterBal = ERC20(tokens[0]).balanceOf(alice);
        assertApproxEqRel(afterShare , beforeShare , 3960784313725490196);// lose is 3.9e18
        assertApproxEqRel(beforeAssets , vaults[0].totalAssets() , 3960784313725490196);// lose is 3.9e18
        assertApproxEqRel(aliceBeforeBal , aliceAfterBal, 3);
    }
```
Run with command : `forge test --mt testAliceAddRemoveSingleForValue  -vvv`
#### Mitigation
I think we need to redesign this part of the protocol and also count for the fee which the protocol will pay to the vault.
