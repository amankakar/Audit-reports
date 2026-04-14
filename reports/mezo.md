# Mezo

Mezo is a Bitcoin-centric platform designed to enhance Bitcoin’s utility through seamless borrowing, spending, and earning. Bitcoin has changed how people think about money, control, security, and transparency. Bitcoin excels as a store of value, but it currently lacks the tools to make it easily usable in everyday financial activities. Mezo bridges this gap by creating a Bitcoin-native ecosystem that transforms BTC from a static asset into a dynamic financial tool.

MUSD is a permissionless stablecoin 100% backed by Bitcoin reserves and designed to maintain a 1:1 peg with the U.S. dollar. It is the native stablecoin on Mezo, accessible via Mezo’s ‘Borrow’ feature or decentralized exchanges on Mezo Network, a chain with Bitcoin as the native asset.

Anyone can mint MUSD by depositing BTC into Mezo borrow, thus creating a loan position. Bitcoin collateral for MUSD positions is publicly verifiable onchain, and proof-of-reserves are viewable 24-7. Users can close their MUSD positions by returning the borrowed MUSD and accumulated interest to receive their initial Bitcoin collateral.



[Ethereum][Solidity][DeFi][Lending]

---

### [M-01] `adjustTroveWithSignature` will always revert if borrower does not have `mUSD`


#### Summary
The protocol allows borrowers to delegate Trove operations to other users via `BorrowerOperationsSignatures`, which verifies the borrower's signature and forwards the call to `BorrowerOperations.sol`.

However, in the case of `adjustTroveWithSignature`, where the caller intends to decrease the debt, the transaction reverts. This happens because the protocol checks the borrower's MUSD balance but attempts to burn MUSD from the caller. As a result, it unintentionally requires the borrower to hold the MUSD balance, even though the action is being performed by a delegate.

**Note:** This issue is similar to my other report regarding `closeTroveWithSignature`, which also reverts under similar conditions. Both were reported separately because fixing one does not  resolve the other. Despite sharing the same root cause, they occur in different functions.

#### Finding Description
To understand this issue lets start with adjustment of trove via `BorrowerOperationsSignatures`:
```solidity
/home/aman/Desktop/audits/musd/solidity/contracts/BorrowerOperationsSignatures.sol:239
239:     function adjustTroveWithSignature(
240:         uint256 _collWithdrawal,
241:         uint256 _debtChange,
242:         bool _isDebtIncrease,
243:         address _upperHint,
244:         address _lowerHint,
245:         address _borrower,
246:         address _recipient,
247:         bytes memory _signature,
248:         uint256 _deadline
249:     ) external payable {
250:         AdjustTrove memory adjustTroveData = AdjustTrove({
251:             collWithdrawal: _collWithdrawal,
252:             debtChange: _debtChange,
253:             isDebtIncrease: _isDebtIncrease,
254:             upperHint: _upperHint,
255:             lowerHint: _lowerHint,
256:             borrower: _borrower,
257:             recipient: _recipient,
258:             deadline: _deadline
259:         });
...
276:         borrowerOperations.restrictedAdjustTrove{value: msg.value}(
277:             adjustTroveData.borrower,
278:             adjustTroveData.recipient,
279:             msg.sender,
280:             adjustTroveData.collWithdrawal,
281:             adjustTroveData.debtChange,
282:             adjustTroveData.isDebtIncrease,
283:             adjustTroveData.upperHint,
284:             adjustTroveData.lowerHint
285:         );
286:     }
```

calls `restrictedAdjustTrove` function:
```solidity
/musd/solidity/contracts/BorrowerOperations.sol:408
408:     function restrictedAdjustTrove(
409:         address _borrower,
410:         address _recipient,
411:         address _caller,
412:         uint256 _collWithdrawal,
413:         uint256 _mUSDChange,
414:         bool _isDebtIncrease,
415:         address _upperHint,
416:         address _lowerHint
417:     ) external payable {
418:         _requireCallerIsBorrowerOperationsSignatures();
419:         _adjustTrove(
420:             _borrower,
421:             _recipient,
422:             _caller,
423:             _collWithdrawal,
424:             _mUSDChange,
425:             _isDebtIncrease,
426:             _upperHint,
427:             _lowerHint
428:         );
429:     }
```
Here we pass all the argument to `_adjustTrove` function and it is exactly where the issue resides:
```solidity
/home/aman/Desktop/audits/musd/solidity/contracts/BorrowerOperations.sol:677
677:     function _adjustTrove(
678:         address _borrower,
679:         address _recipient,
680:         address _caller,
681:         uint256 _collWithdrawal,
682:         uint256 _mUSDChange,
683:         bool _isDebtIncrease,
684:         address _upperHint,
685:         address _lowerHint
686:     ) internal {
...
780: 
781:         // When the adjustment is a debt repayment, check it's a valid amount and that the caller has enough mUSD
782:         if (!_isDebtIncrease && _mUSDChange > 0) {
783:             _requireAtLeastMinNetDebt(
784:                 _getNetDebt(vars.debt) - vars.netDebtChange
785:             );
786:             _requireValidMUSDRepayment(vars.debt, vars.netDebtChange);
787:             _requireSufficientMUSDBalance(_borrower, vars.netDebtChange);
788:         }
...
847:         // Use the unmodified _mUSDChange here, as we don't send the fee to the user
848:         _moveTokensAndCollateralfromAdjustment(
849:             contractsCache.activePool,
850:             contractsCache.musd,
851:             _caller,
852:             _recipient,
853:             vars.collChange,
854:             vars.isCollIncrease,
855:             _isDebtIncrease ? _mUSDChange : vars.principalAdjustment,
856:             vars.interestAdjustment,
857:             _isDebtIncrease,
858:             vars.netDebtChange
859:         );
860:     }
```
`L787` we check the mUSD balance of borrower which is not correct as the mUSD will be payed by caller as in `_moveTokensAndCollateralfromAdjustment`:
```solidity
/home/aman/Desktop/audits/musd/solidity/contracts/BorrowerOperations.sol:443
443:     function _moveTokensAndCollateralfromAdjustment(
444:         IActivePool _activePool,
445:         IMUSD _musd,
446:         address _caller,
447:         address _recipient,
448:         uint256 _collChange,
449:         bool _isCollIncrease,
450:         uint256 _principalChange,
451:         uint256 _interestChange,
452:         bool _isDebtIncrease,
453:         uint256 _netDebtChange
454:     ) internal {
455:         if (_isDebtIncrease) {
456:             _withdrawMUSD(
457:                 _activePool,
458:                 _musd,
459:                 _recipient,
460:                 _principalChange,
461:                 _netDebtChange
462:             );
463:         } else {
464:             _repayMUSD(
465:                 _activePool,
466:                 _musd,
467:                 _caller,
468:                 _principalChange,
469:                 _interestChange
470:             );
471:         }
...
478:     }
```
In this case it will hit the `_repayMUSD` function , where we will burn the  `_principalChange + _interestChange` from caller.
In a nutshell we check the mUSD balance of borrower and than burn the mUSD from caller. 

#### Impact Explanation
The impact of this issue is high. A borrower may not have the required MUSD and ETH to call `adjustTroveWithSignature`, which is precisely why they delegate the action to another user and calls `adjustTroveWithSignature`.

If the delegate is unable to repay the debt of the Trove due to this bug.

In other words, the borrower is effectively forced to hold a MUSD balance, even when delegating the `adjustTroveWithSignature` action to another user.

#### Likelihood Explanation
Likelihood is high, As the borrower could not have MUSD.
#### Proof of Concept
Apply the following patch to `BorrowerOperations.test.ts`

```diff
diff --git a/solidity/test/normal/BorrowerOperations.test.ts b/solidity/test/normal/BorrowerOperations.test.ts
index 6c43669..df64448 100644
--- a/solidity/test/normal/BorrowerOperations.test.ts
+++ b/solidity/test/normal/BorrowerOperations.test.ts
 
@@ -4716,6 +4738,41 @@ describe("BorrowerOperations in Normal Mode", () => {
       // Open an additional trove to keep us from going into recovery mode
       await setupCarolsTrove()
     })
+    it.only("the caller pays the mUSD , but it will revert because the borrower does not have mUSD", async () => {
+      const { borrower, recipient, domain, deadline, nonce } =
+        await setupSignatureTests(bob)
+      // @note : to revert i will transfer bob musd to other account so that bob does not hold musd as intended
+      await contracts.musd.connect(bob.wallet).transfer(alice.wallet, await contracts.musd.balanceOf(bob.wallet));
+      const value = {
+        collWithdrawal,
+        debtChange,
+        isDebtIncrease: false,
+        assetAmount,
+        borrower,
+        recipient,
+        nonce,
+        deadline,
+      }
+
+      const signature = await bob.wallet.signTypedData(domain, types, value)
+
+      await updateWalletSnapshot(contracts, alice, "before")
+      await updateWalletSnapshot(contracts, bob, "before")
+
+      await expect(contracts.borrowerOperationsSignatures
+        .connect(alice.wallet)
+        .adjustTroveWithSignature(
+          collWithdrawal,
+          debtChange,
+          value.isDebtIncrease,
+          upperHint,
+          lowerHint,
+          borrower,
+          borrower,
+          signature,
+          deadline,
+        )).to.be.revertedWith('BorrowerOps: Caller doesnt have enough mUSD to make repayment');
+      })
```
and run with `npx hardhat test` 
#### Recommendation
The fix of this could be to replace the borrower with the caller as in normal flow the caller and borrower will be same but in case of delegating the caller will be the one who need to pay the `MUSD`.
```diff
diff --git a/solidity/contracts/BorrowerOperations.sol b/solidity/contracts/BorrowerOperations.sol
index 185d30c..69f8eab 100644
--- a/solidity/contracts/BorrowerOperations.sol
+++ b/solidity/contracts/BorrowerOperations.sol
@@ -784,7 +784,7 @@ contract BorrowerOperations is
                 _getNetDebt(vars.debt) - vars.netDebtChange
             );
             _requireValidMUSDRepayment(vars.debt, vars.netDebtChange);
-            _requireSufficientMUSDBalance(_borrower, vars.netDebtChange);
+            _requireSufficientMUSDBalance(_caller, vars.netDebtChange);
         }
```


#### Related Files

- `musd/solidity/contracts/BorrowerOperationsSignatures.sol` (lines 276-276)
- `musd/solidity/contracts/BorrowerOperations.sol` (lines 440-440)
- `musd/solidity/contracts/BorrowerOperations.sol` (lines 464-464)
- `musd/solidity/contracts/BorrowerOperations.sol` (lines 787-787)

---

### [L-01] No Slippage protection in `redeemCollateral`


#### Summary
The `redeemCollateral` function lacks a slippage check, which can lead to redeemer receive less collateral. If the MUSD price is low on the MUSD protocol, a user may initiate redemption expecting an arbitrage opportunity. However, without a minimum amount safeguard, they might receive significantly less collateral than expected, Due to sudden increase in price.

#### Finding Description
The Docs state that the `redeemCollateral` function could be used as a arbitrage if the price of mUSD on exchange is less than $1.
>We maintain the **price floor of $1** through arbitrage, an external USD <-> BTC price oracle, and the ability to redeem mUSD for BTC $1 for $1 (via `TroveManager.redeemCollateral`). Imagine that mUSD was trading for $0.80 on an exchange and that bitcoin is selling for 1 BTC = $100k. A arbitrageur with $800 could:

>1. Trade $800 for 1000 mUSD
>1. Redeem 1000 mUSD for 0.01 BTC ($1000 worth of BTC)
>1. Sell 0.01 BTC for $1000

In this case as we know that, The `redeemCollateral` function lacks slippage protection, preventing the redeemer from controlling the exact amount of BTC received. If BTC price spikes during the transaction, the redeemer may receive less BTC than expected. Note: this issue concerns BTC amount, not its USD value.

#### Impact Explanation
The redeemer will receive less BTC amount than expected. So it will leads to lose of value.
#### Likelihood Explanation
This issue will occur in case of high volatile market condition. As the price of BTC is most of the time remain stable that why the likelihood is medium/low. and also in some edge cases where the trove will get closed after redemption.

#### Proof of Concept
Add the following test case to `TroveManager.test.ts` file also with the mention imports:

```javascript
// imports these packages
import { ethers } from "hardhat"
import {
  SnapshotRestorer,
  takeSnapshot,
} from "@nomicfoundation/hardhat-toolbox/network-helpers"

```
```javascript
// run with command : npx hardhat test
it.only("Redemption slippage issue", async () => { 
      await setupRedemptionTroves()
      const capBefore = await contracts.troveManager.getTroveMaxBorrowingCapacity(alice.wallet);
    
      const completeRedemptionAmount = to1e18("2010") // complete Redemption
      const redemptionAmount =   completeRedemptionAmount
      const price = await contracts.priceFeed.fetchPrice()
      
      const snapshot = await takeSnapshot();
      const price1 = to1e18("60,000") // @note : BTC price pumps,
      await contracts.mockAggregator.connect(deployer.wallet).setPrice(price1)
     let collBeforeRedeemer =  await ethers.provider.getBalance(dennis.address);
     let collAfterAlice =  await contracts.troveManager.getTroveColl(alice.wallet);
     console.log("collBeforeAlice" , collAfterAlice );
      
      await contracts.troveManager
        .connect(dennis.wallet)
        .redeemCollateral(
          redemptionAmount,
          carol.address,
          "0x0000000000000000000000000000000000000000",
          "0x0000000000000000000000000000000000000000",
          0,  
          0,
          NO_GAS,
        )

      
      let collAfterRedeemer =  await ethers.provider.getBalance(dennis.address); //await getUserBalance(dennis);
      let inCasePriceIncrease = collAfterRedeemer - collBeforeRedeemer;
       collAfterAlice =  await contracts.troveManager.getTroveColl(alice.wallet);

      console.log("============restore snapshot and continue with the normal flow====================");
      await snapshot.restore();
       collBeforeRedeemer =  await ethers.provider.getBalance(dennis.address);
       collAfterAlice =  await contracts.troveManager.getTroveColl(alice.wallet);
      console.log("collBeforeAlice" , collAfterAlice );
       
       await contracts.troveManager
         .connect(dennis.wallet)
         .redeemCollateral(
           redemptionAmount,
           carol.address,
           "0x0000000000000000000000000000000000000000",
           "0x0000000000000000000000000000000000000000",
           0,
           0,
           NO_GAS,
         )
        collAfterRedeemer =  await ethers.provider.getBalance(dennis.address);
      let inCasePriceSame = collAfterRedeemer - collBeforeRedeemer;

       console.log("collDennisAlice inCasePriceSame", inCasePriceSame);
      console.log("collDennisAlice inCasePriceIncrease", inCasePriceIncrease);

         expect(inCasePriceSame).to.gt(inCasePriceIncrease);
    })
```
#### Recommendation

The fixed would be to add `minCollateral` to `redeemCollateral` function and than check if user will receive at least provided collateral amount.  

#### Related Files

- `musd/solidity/contracts/TroveManager.sol` (lines 303-303)
- `musd/solidity/contracts/TroveManager.sol` (lines 451-451)
- `musd/solidity/contracts/TroveManager.sol` (lines 1240-1242)

---

### [L-02] The max capacity does not got update in case of partial redemption


#### Summary
The protocol enforces a `maxBorrowingCapacity` limit based on collateral value, which updates when collateral is added or removed. However, during redemptions, only the collateral balance is updated—not the max borrow capacity. This discrepancy allows borrowers to operate with an outdated limit and potentially borrow more than intended, especially during market fluctuations.

#### Finding Description
The `maxBorrowingCapacity` serves as a second layer of security to ensure that, even during high market fluctuations, borrowers can only borrow up to a safe, predefined limit. This limit is updated during collateral changes, such as when removing collateral in `BorrowerOperations::_adjustTrove` or when opening a new Trove.
This behavior can be observed in the following code:

```solidity
/musd/solidity/contracts/BorrowerOperations.sol:807
807:         if (!vars.isCollIncrease && vars.collChange > 0) {
808:             uint256 newMaxBorrowingCapacity = _calculateMaxBorrowingCapacity(
809:                 vars.newColl,
810:                 vars.price
811:             );
812: 
813:             uint256 currentMaxBorrowingCapacity = contractsCache
814:                 .troveManager
815:                 .getTroveMaxBorrowingCapacity(_borrower);
816: 
817:             uint256 finalMaxBorrowingCapacity = LiquityMath._min(
818:                 currentMaxBorrowingCapacity,
819:                 newMaxBorrowingCapacity
820:             );
821: 
822:             contractsCache.troveManager.setTroveMaxBorrowingCapacity(
823:                 _borrower,
824:                 finalMaxBorrowingCapacity
825:             );
826:         }
827: 
```
As shown in the code above, when a borrower withdraws or removes collateral, the `maxBorrowingCapacity` is recalculated and updated in the Trove state.
However, during redemptions, collateral is also effectively withdrawn from the Trove—meaning the `newColl` is reduced. Logically, this should trigger an update to the `maxBorrowingCapacity` as well. Unfortunately, this update is missing.

```solidity
/musd/solidity/contracts/TroveManager.sol:1222
1222:     function _redeemCollateralFromTrove(
1223:         ContractsCache memory _contractsCache,
1224:         address _borrower,
1225:         uint256 _maxMUSDamount,
1226:         uint256 _price,
1227:         address _upperPartialRedemptionHint,
1228:         address _lowerPartialRedemptionHint,
1229:         uint256 _partialRedemptionHintNICR,
1230:         LocalVariables_redeemCollateral memory redeemCollateralVars
1231:     ) internal returns (SingleRedemptionValues memory singleRedemption) {
1244: 
1245:         // Decrease the debt and collateral of the current Trove according to the mUSD lot and corresponding collateral to send
1246:         vars.newDebt = _getTotalDebt(_borrower) - vars.mUSDLot;
1247:         vars.newColl = Troves[_borrower].coll - singleRedemption.collateralLot;
1260: ...
1287:         } else {
1288:             // calculate 10 minutes worth of interest to account for delay between the hint call and now
1289:             // solhint-disable not-rely-on-time
1290:             vars.upperBoundNICR = LiquityMath._computeNominalCR(
1291:                 vars.newColl,
1292:                 vars.newPrincipal -
1293:                     InterestRateMath.calculateInterestOwed(
1294:                         Troves[_borrower].principal,
1295:                         redeemCollateralVars.interestRate,
1296:                         block.timestamp - 600,
1297:                         block.timestamp
1298:                     )
1299:             );
1300:             // solhint-enable not-rely-on-time
1301:             vars.newNICR = LiquityMath._computeNominalCR(
1302:                 vars.newColl,
1303:                 vars.newPrincipal
1304:             );
1305: ...
1321:             // slither-disable-end calls-loop
1322: 
1323:             // slither-disable-next-line calls-loop
1324:             _contractsCache.sortedTroves.reInsert(
1325:                 _borrower,
1326:                 vars.newNICR,
1327:                 _upperPartialRedemptionHint,
1328:                 _lowerPartialRedemptionHint
1329:             );
1330: 
1331:             _updateTroveDebt(_borrower, vars.mUSDLot);
1332:             Troves[_borrower].coll = vars.newColl;
1333:             _updateStakeAndTotalStakes(_borrower);
1334:     }
```
At line `1247` we update the the collateral value and debt of the trove but we did not update the `maxBorrowingCapacity`.

#### Impact Explanation
The protocol relies on `maxBorrowingCapacity` as a second layer of security to prevent borrowers from maintaining unhealthy positions. However, in this case, it allows users to borrow MUSD beyond their actual `maxBorrowingCapacity`.
This issue creates an unfair advantage for users whose Troves were partially redeemed. Two users with different collateral amounts but the same ICR would be able to borrow the same amount of MUSD, as demonstrated in the provided proof of concept (PoC).


#### Likelihood Explanation
This issue occurs whenever a Trove undergoes a partial redemption, making the likelihood of it happening relatively high.
#### Proof of Concept

Apply following git diff :
```diff
diff --git a/solidity/test/normal/TroveManager.test.ts b/solidity/test/normal/TroveManager.test.ts
index 8b2d3c4..416271d 100644
--- a/solidity/test/normal/TroveManager.test.ts
+++ b/solidity/test/normal/TroveManager.test.ts
@@ -2111,13 +2111,13 @@ describe("TroveManager in Normal Mode", () => {
     async function setupRedemptionTroves() {
       // Open three troves with ascending ICRs
       await openTrove(contracts, {
-        musdAmount: "2000",
+        musdAmount: "4000",
         ICR: "200",
         sender: alice.wallet,
       })
       await openTrove(contracts, {
         musdAmount: "4000",
-        ICR: "300",
+        ICR: "200",
         sender: bob.wallet,
       })
       await openTrove(contracts, {
@@ -2364,6 +2364,71 @@ describe("TroveManager in Normal Mode", () => {
       expect(carol.trove.debt.after - carol.trove.debt.before).to.equal(0n)
     })
 
+    it.only("max Capacity issue", async () => {
+      await setupRedemptionTroves()
+      const capBefore = await contracts.troveManager.getTroveMaxBorrowingCapacity(alice.wallet);
+    
+      const partialRedemptionAmount = to1e18("2000") //to1e18("1999")
+      const redemptionAmount =   partialRedemptionAmount
+      const price = await contracts.priceFeed.fetchPrice()
+      
+      // Calculate Dennis's hints
+      const {
+        firstRedemptionHint,
+        partialRedemptionHintNICR,
+        upperPartialRedemptionHint,
+        lowerPartialRedemptionHint,
+      } = await getRedemptionHints(contracts, dennis, redemptionAmount, price)
+      
+      await contracts.troveManager
+        .connect(dennis.wallet)
+        .redeemCollateral(
+          redemptionAmount,
+          firstRedemptionHint,
+          upperPartialRedemptionHint,
+          lowerPartialRedemptionHint,
+          partialRedemptionHintNICR,
+          0,
+          NO_GAS,
+        )
+
+      // Check that Carol's debt is untouched because no partial redemption was performed
+      await updateTroveSnapshot(contracts, carol, "after")
+      await updateTroveSnapshot(contracts, alice, "after")
+     let collAfterAlice =  await contracts.troveManager.getTroveColl(alice.wallet);
+
+     const afterRedemptionMaxBorrowCapacityExpected = Number(collAfterAlice * price) / (110 * 1e16); // @note : this is not updated due to which the max limit
 could be passed after redemption      
+      const price1 = to1e18("70,000") // @note : updated the price to bypass the MCR check as with the current Price both MCR and Cap is 110% , this could als
o happen due to market fluctuation
+      await contracts.mockAggregator.connect(deployer.wallet).setPrice(price1)
+      let debtChange =  to1e18("5400");
+      const aliceCap = await contracts.troveManager.getTroveMaxBorrowingCapacity(alice.wallet);
+      const bobCap = await contracts.troveManager.getTroveMaxBorrowingCapacity(alice.wallet);
+
+      expect(capBefore).to.equal(aliceCap) // @note : cap no updated remain the same
+      await contracts.borrowerOperations
+        .connect(alice.wallet)
+        .adjustTrove(0, debtChange, true, alice.wallet, alice.wallet)
+      
+        const debtfinal = await contracts.troveManager.getTroveDebt(alice.wallet);
+        expect(Number(afterRedemptionMaxBorrowCapacityExpected)).to.lt(Number(debtfinal)); // @audit : alice is able to borrow more than expect cap.
+        // 2nd user borrow 
+        debtChange = to1e18("3000");
+        await contracts.borrowerOperations
+        .connect(bob.wallet)
+        .adjustTrove(0, debtChange, true, bob.wallet, bob.wallet)
+
+        const debtfinalBob = await contracts.troveManager.getTroveDebt(bob.wallet);
+        expect(bobCap).to.equal(aliceCap); // @audit : alice and bob cap are equals.
+        expect(bobCap).to.equal(aliceCap); // @audit : alice and bob cap are equals.
+        collAfterAlice =  await contracts.troveManager.getTroveColl(alice.wallet);
+        const collAfterBob =  await contracts.troveManager.getTroveColl(bob.wallet);
+
+        expect(collAfterAlice).to.lt(collAfterBob); // @audit : alice coll is less than bob.
+    })

```
#### Recommendation
The best fix could be to recalculate the `maxBorrowingCapacity` after partial redemption and update the trove state as follows.
```diff
diff --git a/solidity/contracts/TroveManager.sol b/solidity/contracts/TroveManager.sol
index 8866d94..a0a9639 100644
--- a/solidity/contracts/TroveManager.sol
+++ b/solidity/contracts/TroveManager.sol
@@ -1331,6 +1329,8 @@ contract TroveManager is
             _updateTroveDebt(_borrower, vars.mUSDLot);
             Troves[_borrower].coll = vars.newColl;
             _updateStakeAndTotalStakes(_borrower);
+            // Calculate new max borrowing cap @note directly assigning due to stake to deep error
+            Troves[_borrower].maxBorrowingCapacity= _calculateMaxBorrowingCapacity(vars.newColl , _price , _borrower);
 
             emit TroveUpdated(
                 _borrower,
@@ -1345,6 +1345,15 @@ contract TroveManager is
         return singleRedemption;
     }
 
+    function _calculateMaxBorrowingCapacity(uint256 _newColl , uint256 _price , address _borrower) internal view returns(uint256 finalMaxBorrowingCapacity) {
+        uint256 newMaxBorrowingCapacity =  (_newColl * _price) / (110 * 1e16);
+        uint256 currentMaxBorrowingCapacity = Troves[_borrower].maxBorrowingCapacity;
+         returnLiquityMath._min(
+                currentMaxBorrowingCapacity,
+                newMaxBorrowingCapacity
+            );
+    }
```

#### Related Files

- `musd/solidity/contracts/BorrowerOperations.sol` (lines 807-807)
- `musd/solidity/contracts/TroveManager.sol` (lines 1332-1332)



