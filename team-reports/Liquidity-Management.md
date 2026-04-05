# Liquidity Management

## Protocol Summary

The Perpetual Vault Protocol is a decentralized finance (DeFi) system that enables users to engage in leveraged trading on the GMX decentralized exchange. This protocol aims to simplify the process of managing leveraged positions while providing additional features such as automated position management and risk mitigation.
[Ethereum][Solidity][DeFi]

---

### [M-01] The `cancelFlow` will send the executionFee to wrong address in case of WithDraw

#### Summary

The cancelFlow function, triggered by the keeper during a DoS, refunds the execution fee to the last counter owner in case of a withdraw action. However, since withdrawals are not sequential and can be triggered by anyone with a valid deposit ID, the fee will be incorrectly sent to the last deposit ID instead of the intended recipient.

#### Vulnerability Details

Both cancelFlow and setVault functions act as glass-breaking mechanisms, meaning they are only triggered in case of an emergency.
The flow is set to FLOW.WITHDRAW when a user initiates the withdrawal process to withdraw their assets from the Gamma Vault.

`/contracts/PerpetualVault.sol:253`

```solidity
253:   function withdraw(address recipient, uint256 depositId) public payable nonReentrant {
254:     _noneFlow();
255:     flow = FLOW.WITHDRAW;
256:     flowData = depositId;
257: 
258:     if (recipient == address(0)) {
259:       revert Error.ZeroValue();
260:     }
261:     if (depositInfo[depositId].timestamp + lockTime >= block.timestamp) {
262:       revert Error.Locked();
263:     }
264:     if (EnumerableSet.contains(userDeposits[msg.sender], depositId) == false) {
265:       revert Error.InvalidUser();
266:     }
267:     if (depositInfo[depositId].shares == 0) {
268:       revert Error.ZeroValue();
269:     }
270: 
271:     depositInfo[depositId].recipient = recipient;
272:     _payExecutionFee(depositId, false);
273:     if (curPositionKey != bytes32(0)) {
274:       nextAction.selector = NextActionSelector.WITHDRAW_ACTION;
275:       _settle();  // Settles any outstanding fees and updates state before processing withdrawal
276:     } else {
277:       MarketPrices memory prices;
278:       _withdraw(depositId, hex'', prices);
279:     }
280:   }
```

A user sends the executionFee to process their order on GMX at 272. The code sets `flowData = depositId` at 256. If, for any reason, the transaction fails on GMX, the admin/owner may reset the state and call the cancelFlow function.

`/contracts/PerpetualVault.sol:1220`

```solidity
1220:   function _cancelFlow() internal {
...
1232:     } else if (flow == FLOW.WITHDRAW) {
1233:       try IGmxProxy(gmxProxy).refundExecutionFee(
1234:         depositInfo[counter].owner,
1235:         depositInfo[counter].executionFee
1236:       ) {} catch {}
1237:     }
```

The fee is paid to the counter owner. In most cases, this may not correspond to the correct withdraw request ID. As a result, the fee is sent to the wrong address, leading to a loss for the intended user.


#### Impact

In this edge case, the user loses the execution fee while the counter owner incorrectly receives it. Since asset loss is involved, the impact is high, but the likelihood is low. Therefore, the appropriate severity classification is Medium.

#### PoC

Add the following test case to PerpetualVault.t.sol and run with command `forge test --mt test_CancelFlow_Withdraw_Case -vvv --rpc-url arbitrum`:

`/home/aman/Desktop/audits/2025-02-gamma/test/PerpetualVault.t.sol:737`

```solidity
737: function test_CancelFlow_Withdraw_Case() external { // @audit-issue : withdraw case
738:     IERC20 collateralToken = PerpetualVault(vault).collateralToken();
739: 
740:     address bob = makeAddr("bob");
741: 
742:     address alice = makeAddr("alice");
743:     payable(alice).transfer(10 ether);
744:     payable(bob).transfer(10 ether);
745: 
746:     depositFixture(alice, 1e10);
747: 
748:     address keeper = PerpetualVault(vault).keeper();
749:     MarketPrices memory prices = mockData.getMarketPrices();
750: 
751:     bytes[] memory data = new bytes[](1);
752:     data[0] = abi.encode(3380000000000000);
753:     vm.prank(keeper);
754:     PerpetualVault(vault).runNextAction(prices, data);
755:     GmxOrderExecuted(true);
756:     vm.prank(keeper);
757:     PerpetualVault(vault).runNextAction(prices, new bytes[](0));
758: 
759:     depositFixture(bob, 1e10);
760:     vm.prank(keeper);
761:     PerpetualVault(vault).runNextAction(prices, data);
762:     GmxOrderExecuted(true);
763:     vm.prank(keeper);
764:     PerpetualVault(vault).runNextAction(prices, new bytes[](0));
765:     // Now alice wants to withdraw her tokens
766:     uint256 lockTime = PerpetualVault(vault).lockTime();
767:     vm.warp(block.timestamp + lockTime + 1);
768:     uint256 executionFee = PerpetualVault(vault).getExecutionGasLimit(false);
769:     uint256[] memory depositIds = PerpetualVault(vault).getUserDeposits(alice);
770:     vm.prank(alice); // alice requested for withdraw
771:     PerpetualVault(vault).withdraw{value: executionFee * tx.gasprice}(alice, depositIds[0]);
772:     uint256 aliceBalance = address(alice).balance;
773:     uint256 bobBalance = address(bob).balance;
774:     console.log("aliceBalance" , aliceBalance);
775:     console.log("bobBalance" , bobBalance);
776: 
777:     vm.expectRevert();
778:     vm.prank(keeper);
779:     PerpetualVault(vault).cancelFlow();
780:     address owner = PerpetualVault(vault).owner();
781:         (PerpetualVault.NextActionSelector selector, bytes memory data1) = PerpetualVault(vault).nextAction();
782:    PerpetualVault.Action memory action = PerpetualVault.Action(
783:     selector,
784:     data1
785:    );
786:     vm.prank(owner);
787:     PerpetualVault(vault).setVaultState(
788:       PerpetualVault(vault).flow(),
789:       depositIds[0], // set flowData to deposit ID as we set it inside withdraw function
790:       PerpetualVault(vault).beenLong(),
791:       PerpetualVault(vault).curPositionKey(),
792:       PerpetualVault(vault).positionIsClosed(),
793:       false, // set gmx lock to false
794:       action
795:     );
796:     vm.prank(keeper);
797:     PerpetualVault(vault).cancelFlow();
798:     uint256 bobBalanceAfter = address(bob).balance;
799:     uint256 aliceBalanceAfter = address(alice).balance;
800:     console.log("aliceBalanceAfter" , aliceBalanceAfter);
801:     console.log("bobBalanceAfter" , bobBalanceAfter);
802:     assertGt(bobBalanceAfter , bobBalance);
803:     }
```

For this test case we need that the `positionIsClosed = false`, so inside `PerpetualVault::initialize()` function.

```
Logs:
  aliceBalance 9993923067766576000
  bobBalance 9997974355922192000
  aliceBalanceAfter 9993923067766576000
  bobBalanceAfter 10000000000000000000
```

Tools used: Manual review.

#### Mitigation

One possible fix is to either pass the depositId as an argument in the cancelFlow function or utilize the flowData variable, which is already set in the case of a withdrawal.

```diff
diff --git a/contracts/PerpetualVault.sol b/contracts/PerpetualVault.sol
index ed4fba2..ed537f1 100644
--- a/contracts/PerpetualVault.sol
+++ b/contracts/PerpetualVault.sol
@@ -1230,11 +1231,12 @@ contract PerpetualVault is IPerpetualVault, Initializable, Ownable2StepUpgradeab
       delete depositInfo[depositId];
     } else if (flow == FLOW.WITHDRAW) {
       try IGmxProxy(gmxProxy).refundExecutionFee(
-        depositInfo[counter].owner,
-        depositInfo[counter].executionFee
+        depositInfo[flowData].owner,
+        depositInfo[flowData].executionFee
       ) {} catch {}
```

---

### [L-01] CancelFlow does not update the totalShare in case of Deposit Flow

#### Summary

CancelFlow is added as a glass breaking function which will help to revert the current flow in case of any accident from Gamma side or GMX side, however in case of deposit flow it sent the collateral back and delete the user record but does not update the totalShare.

#### Vulnerability Details

The vault can enter a state where it cannot proceed to the next flow and may become stuck due to an accident. To handle such scenarios, the Gamma team has implemented the cancelFlow function as a glass-breaking mechanism. This function allows the team to reset the flow and restore the vault to a normal state. If the flow is in a deposit state, cancelFlow will return the collateral tokens to the user and reset the vault's state.

`/contracts/PerpetualVault.sol:1220`

```solidity
1220:   function _cancelFlow() internal {
1221:     if (flow == FLOW.DEPOSIT) {
1222:       uint256 depositId = counter;
1223:       collateralToken.safeTransfer(depositInfo[depositId].owner, depositInfo[depositId].amount);
1224:       totalDepositAmount = totalDepositAmount - depositInfo[depositId].amount;
1225:       EnumerableSet.remove(userDeposits[depositInfo[depositId].owner], depositId);
1226:       try IGmxProxy(gmxProxy).refundExecutionFee(
1227:         depositInfo[counter].owner,
1228:         depositInfo[counter].executionFee
1229:       ) {} catch {}
1230:       delete depositInfo[depositId];
```

From the above code, it can be observed that cancelFlow sends back the collateralToken, updates totalDepositAmount, refunds the fee, and deletes the depositInfo. However, there is an edge case where virtual shares are minted to the user, but totalShare is not updated accordingly. This issue can lead to incorrect share calculations for other users, ultimately impacting withdrawals.

#### Impact

Failing to update totalShare affects how users withdraw their funds. Since withdrawals rely on dividing shares by totalShare, users will consistently receive less than intended.

#### PoC

Add following test case to PerpetualVault.t.sol and run with command `forge test --mt test_CancelFlow_Deposit_Case -vvv --rpc-url arbitrum`:

```solidity
  function test_CancelFlow_Deposit_Case() external {
    IERC20 collateralToken = PerpetualVault(vault).collateralToken();
    address alice = makeAddr("alice");
    payable(alice).transfer(10 ether);
    vm.startPrank(address(alice));
    deal(address(collateralToken), address(vault), 1e10);
    uint256 beforeAnyDeposit = PerpetualVault(vault).totalShares();
    assertEq(beforeAnyDeposit , 0);
    depositFixture(alice, 1e10);
    address keeper = PerpetualVault(vault).keeper();
    MarketPrices memory prices = mockData.getMarketPrices();

    bytes[] memory data = new bytes[](1);
    data[0] = abi.encode(3380000000000000);
    vm.prank(keeper);
    PerpetualVault(vault).runNextAction(prices, data);
    GmxOrderExecuted(true);
    // @audit : assume  some thing by accident happen here and we need to cancel the flow
    vm.prank(keeper);
    PerpetualVault(vault).cancelFlow();

     uint afterCancelFlowTotalShares = PerpetualVault(vault).totalShares();
    assertGt(afterCancelFlowTotalShares , 0);
  }
```

For this test case we need that the `positionIsClosed = false` so inside `PerpetualVault::initialize()` function.

Tools used: Manual Review.

#### Mitigation

In case of deposit flow also check subtract the user shares from totalShare.

```diff
diff --git a/contracts/PerpetualVault.sol b/contracts/PerpetualVault.sol
index ed4fba2..441a03d 100644
--- a/contracts/PerpetualVault.sol
+++ b/contracts/PerpetualVault.sol
@@ -1222,6 +1222,7 @@ contract PerpetualVault is IPerpetualVault, Initializable, Ownable2StepUpgradeab
       uint256 depositId = counter;
       collateralToken.safeTransfer(depositInfo[depositId].owner, depositInfo[depositId].amount);
       totalDepositAmount = totalDepositAmount - depositInfo[depositId].amount;
+      totalShares = totalShares - depositInfo[depositId].shares;
       EnumerableSet.remove(userDeposits[depositInfo[depositId].owner], depositId);
```
