# LEND


## Protocol Summary

LEND is a cross chain lending protocol with real yield value extraction, from protocol, to holder. The audit will focus on the cross-chain functionalities within the LEND infrastructure.


[Ethereum][Solidity][DeFi][Lending][Cross Chain]

---

### [H-0] CrossChain liquidation will always reverts

#### Summary
While in case of cross chain liquidation to fetch the borrow position we passed the `destID=0` and use this in if condition to find the borrow position of distention chain. However this will always return with `found=0` Because the borrow position  is not stored with `destId=0`.

#### Root Cause
```solidity
        (bool found, uint256 index) = lendStorage.findCrossChainCollateral(
            payload.sender,
            underlying,
            currentEid, // srcEid is current chain
 @----->    0, // We don't know destEid yet, but we can match on other fields
            payload.destlToken,
            payload.srcToken
        );
```
Here we passed the `destEid=0`  , but use this value inside if to find the `crossChainCollateral` entry.
```solidity
/2025-05-lend-audit-contest/Lend-V2/src/LayerZero/LendStorage.sol:694
694:         for (uint256 i = 0; i < userCollaterals.length;) {
695:             if (
696:                 userCollaterals[i].srcEid == srcEid && userCollaterals[i].destEid == destEid
697:                     && userCollaterals[i].borrowedlToken == borrowedlToken && userCollaterals[i].srcToken == srcToken
698:             ) {
699:                 return (true, i);
700:             }
701:             unchecked {
702:                 ++i;
703:             }
704:         }
705:         return (false, 0);
```
[LendStorage.sol#L695](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/LendStorage.sol#L695)
[CrossChainRouter.sol#L448-L455](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CrossChainRouter.sol#L448-L455)

#### Internal Pre-conditions
User has borrowed some assets.


#### External Pre-conditions
Liquidator want to liquidate user via crossChain liquidation process.


#### Attack Path



#### Impact

Cross Chain Liquidation will always revert and the borrower will not be liquidated via cross chain liquidation.

#### PoC
To run the Test I have added these changes inside setUp function:
```diff
diff --git a/Lend-V2/test/TestLiquidations.t.sol b/Lend-V2/test/TestLiquidations.t.sol
index 2000df6..73679a3 100644
--- a/Lend-V2/test/TestLiquidations.t.sol
+++ b/Lend-V2/test/TestLiquidations.t.sol
@@ -4,7 +4,7 @@ pragma solidity 0.8.23;
 import {Test, console2} from "forge-std/Test.sol";
 import {Deploy} from "../script/Deploy.s.sol";
 import {HelperConfig} from "../script/HelperConfig.s.sol";
-import {CrossChainRouterMock} from "./mocks/CrossChainRouterMock.sol";
+import {CrossChainRouter} from "../src/LayerZero/CrossChainRouter.sol";
 import {CoreRouter} from "../src/LayerZero/CoreRouter.sol";
 import {LendStorage} from "../src/LayerZero/LendStorage.sol";
 import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
@@ -28,7 +28,7 @@ contract TestLiquidations is LayerZeroTest {
     address public liquidator;
 
     // Chain A (Source)
-    CrossChainRouterMock public routerA;
+    CrossChainRouter public routerA;
     LendStorage public lendStorageA;
     CoreRouter public coreRouterA;
     Lendtroller public lendtrollerA;
@@ -37,7 +37,7 @@ contract TestLiquidations is LayerZeroTest {
     address[] public lTokensA;
 
     // Chain B (Destination)
-    CrossChainRouterMock public routerB;
+    CrossChainRouter public routerB;
     LendStorage public lendStorageB;
     CoreRouter public coreRouterB;
     Lendtroller public lendtrollerB;
@@ -79,7 +79,7 @@ contract TestLiquidations is LayerZeroTest {
         ) = deployA.run(address(endpointA)); // Pass the endpoint address to Deploy.run
 
         // Store Chain A values
-        routerA = CrossChainRouterMock(payable(routerAddressA));
+        routerA = CrossChainRouter(payable(routerAddressA));
         vm.label(address(routerA), "Router A");
         lendStorageA = LendStorage(lendStorageAddressA);
         vm.label(address(lendStorageA), "LendStorage A");
@@ -109,7 +109,7 @@ contract TestLiquidations is LayerZeroTest {
         ) = deployB.run(address(endpointB));
 
         // Store Chain B values
-        routerB = CrossChainRouterMock(payable(routerAddressB));
+        routerB = CrossChainRouter(payable(routerAddressB));
         vm.label(address(routerB), "Router B");
         lendStorageB = LendStorage(lendStorageAddressB);
         vm.label(address(lendStorageB), "LendStorage B");
@@ -130,6 +130,7 @@ contract TestLiquidations is LayerZeroTest {
             lendStorageA.addUnderlyingToDestUnderlying(supportedTokensA[i], supportedTokensB[i], CHAIN_B_ID);
             lendStorageA.addUnderlyingToDestlToken(supportedTokensA[i], lTokensB[i], CHAIN_B_ID);
             lendStorageA.setChainAssetMap(supportedTokensA[i], block.chainid, supportedTokensB[i]);
+            lendStorageA.setChainAssetMap(supportedTokensB[i], block.chainid, supportedTokensA[i]);
             lendStorageA.setChainLTokenMap(lTokensA[i], block.chainid, lTokensB[i]);
             lendStorageA.setChainLTokenMap(lTokensB[i], block.chainid, lTokensA[i]);
             console2.log("Mapped l token: ", lTokensA[i], "to", lTokensB[i]);
@@ -142,10 +143,10 @@ contract TestLiquidations is LayerZeroTest {
             lendStorageB.addUnderlyingToDestUnderlying(supportedTokensB[i], supportedTokensA[i], CHAIN_A_ID);
             lendStorageB.addUnderlyingToDestlToken(supportedTokensB[i], lTokensA[i], CHAIN_A_ID);
             lendStorageB.setChainAssetMap(supportedTokensB[i], block.chainid, supportedTokensA[i]);
+            lendStorageB.setChainAssetMap(supportedTokensA[i], block.chainid, supportedTokensB[i]);
             lendStorageB.setChainLTokenMap(lTokensB[i], block.chainid, lTokensA[i]);
             lendStorageB.setChainLTokenMap(lTokensA[i], block.chainid, lTokensB[i]);
             console2.log("Mapped l token: ", lTokensB[i], "to", lTokensA[i]);
```

Add the following test Case to `TestLiquidation.t.sol` file and run with test `forge test --mt test_cross_chain_liquidation_borrow_not_found -vvv` :
```solidity
    function test_cross_chain_liquidation_borrow_not_found()
        public
    {
        uint256 supplyAmount = 100e18;
        uint256 borrowAmount=55e18;
        uint256 newPrice = 1e14;
        // Bound inputs to reasonable values
        supplyAmount =  1000e18;
        borrowAmount = supplyAmount * 75 / 100;//bound(borrowAmount, 50e18, supplyAmount * 100 / 100); // Max 60% LTV
        newPrice =  1e12; // 0.1-2% of original price

        // Supply collateral on Chain A
        (address tokenA, address lTokenA) = _supplyA(deployer, supplyAmount, 0);

         _supplyA(liquidator, supplyAmount, 0);
        // Supply liquidity on Chain B for borrowing
        (address tokenB, address lTokenB) = _supplyB(liquidator, supplyAmount * 2, 0);

        vm.deal(address(routerA), 1 ether); // For LayerZero fees

        // Borrow cross-chain from Chain A to Chain B
        vm.startPrank(deployer);
        routerA.borrowCrossChain(borrowAmount, tokenA, CHAIN_B_ID);
        vm.stopPrank();

        // Simulate price drop of collateral (tokenA) only on first chain
        priceOracleA.setDirectPrice(tokenA, newPrice);
        // Simulate price drop of collateral (tokenA) only on second chain
        priceOracleB.setDirectPrice(
            lendStorageB.lTokenToUnderlying(lendStorageB.crossChainLTokenMap(lTokenA, block.chainid)), newPrice
        );

        // Set up liquidator with borrowed asset on Chain A
        vm.deal(liquidator, 1 ether);
        vm.startPrank(liquidator);

        // We need the Chain A version of the tokens for liquidation on Router A
        ERC20Mock(tokenA).mint(liquidator, borrowAmount);
        IERC20(tokenA).approve(address(routerA), borrowAmount);

        // Repay 50% of the borrow
        uint256 repayAmount = borrowAmount * 5 / 10;

        ERC20Mock(tokenB).mint(liquidator, 1e30);

        // approve router b to spend repay amount
        IERC20(tokenB).approve(address(routerB), type(uint256).max);
        IERC20(tokenB).approve(address(coreRouterB), type(uint256).max);


        // Call liquidateBorrow on Router B (where the borrow exists)
        routerB.liquidateCrossChain(
            deployer, // borrower
            repayAmount, // amount to repay
            31337, // chain where the collateral exists
            lTokenB, // collateral lToken (on Chain B)
            tokenB // borrowed asset (Chain B version)
        );
        vm.stopPrank();
    }
```


#### Mitigation
pass the exact destEid to `findCrossChainCollateral` function.

---

### [H-02]Cross Chain Liquidation will always revert Because of wrong `destlToken`

#### Summary
While performing Cross Chain liquidation the code uses the passed or payload  token addresses for `destlToken`  , The `destlToken` is used get underlying token  to find the user collateral token for the borrower. As the `destlToken` address is valid for destination chain A not valid for current chain B.It will always revert.


#### Root Cause
The root cause is related with `destlToken` value being used
This is used to fetch the underlying token and also used in `findCrossChainCollateral` :
```solidity
/Lend-V2/src/LayerZero/CrossChainRouter.sol:450
450:         address underlying = lendStorage.lTokenToUnderlying(payload.destlToken);
451: 
452:         // Find the specific collateral record
453:         (bool found, uint256 index) = lendStorage.findCrossChainCollateral(
454:             payload.sender,
455:             underlying,
456:             currentEid, // srcEid is current chain
457:             0, // We don't know destEid yet, but we can match on other fields
458:             payload.destlToken,
459:             payload.srcToken
460:         );
```
Here at `L450` we fetch the `underlying` address using `destlToken` and also passed to `findCrossChainCollateral` to find the borrowed assets entry.
[CrossChainRouter.sol#L445](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CrossChainRouter.sol#L445)
[CrossChainRouter.sol#L453](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CrossChainRouter.sol#L453)
#### Internal Pre-conditions
Nil


#### External Pre-conditions

Nil

#### Attack Path
1. The borrower borrows assets on Chain B.
2. The borrower can be liquidated.
3. The liquidator can liquidate borrower position on chain B, using cross chain liquidation.
4. Liquidator Initiate the Liquidation on chain B.
5. on Chain A `_lzReceive` function call with the `cType=CrossChainLiquidationExecute`  and after ome validation calls back the Chain B with the given data and `cType=LiquidationSuccess`.
6. On Chain B , here we uses the `destlToken` which are valid on chain A to find the Borrow Position But on Chain B it is not correct. Because here we need to fetch it from mapping of  `crossChainLTokenMap` and also form underlying we need first fetch address of ltoken from `crossChainLTokenMap` and than fetch underlying asset address. 


#### Impact
The borrower can not be liquidated via Cross Chain Liquidation Due to wrong `destlToken` token address.

#### PoC


#### Mitigation
One of the fix could be following :
```diff
diff --git a/Lend-V2/src/LayerZero/CrossChainRouter.sol b/Lend-V2/src/LayerZero/CrossChainRouter.sol
index 6328953..dac9bdc 100644
--- a/Lend-V2/src/LayerZero/CrossChainRouter.sol
+++ b/Lend-V2/src/LayerZero/CrossChainRouter.sol
@@ -442,7 +447,7 @@ contract CrossChainRouter is OApp, ExponentialNoError {
      */
     function _handleLiquidationSuccess(LZPayload memory payload) private {
         // Find the borrow position on Chain B to get the correct srcEid
-        address underlying = lendStorage.lTokenToUnderlying(payload.destlToken);
+       address underlying = lendStorage.lTokenToUnderlying(lendStorage.crossChainLTokenMap(payload.destlToken, currentEid));
 
         // Find the specific collateral record
         (bool found, uint256 index) = lendStorage.findCrossChainCollateral(
@@ -450,7 +455,7 @@ contract CrossChainRouter is OApp, ExponentialNoError {
             underlying,
             currentEid, // srcEid is current chain
             0, // We don't know destEid yet, but we can match on other fields
-            payload.destlToken,
+            lendStorage.crossChainLTokenMap(payload.destlToken, currentEid),
             payload.srcToken
         );
```

---

### [H-03] Cross Chain Liquidation will always revert Because of wrong `srcToken`

#### Summary
While performing Cross Chain liquidation the code uses the passed or payload  token addresses for `srcToken`, The `srcToken` is used  to find the user cross chain collateral entry for the borrower. As the `srcToken` address is valid for destination chain A not valid for current chain B.It will always revert.


#### Root Cause
This issue root cause is  related with `srcToken` 
```solidity
/Lend-V2/src/LayerZero/CrossChainRouter.sol:450
450:         address underlying = lendStorage.lTokenToUnderlying(payload.destlToken);
451: 
452:         // Find the specific collateral record
453:         (bool found, uint256 index) = lendStorage.findCrossChainCollateral(
454:             payload.sender,
455:             underlying,
456:             currentEid, // srcEid is current chain
457:             0, // We don't know destEid yet, but we can match on other fields
458:             payload.destlToken,
459:             payload.srcToken
460:         );
```
Here At `L459` we passed `srcToken` address to `findCrossChainCollateral`.
Inside `findCrossChainCollateral` we uses these addresses to fetch user collateral record:
```solidity
/Lend-V2/src/LayerZero/LendStorage.sol:694
694:         for (uint256 i = 0; i < userCollaterals.length;) {
695:             if (
696:                 userCollaterals[i].srcEid == srcEid && userCollaterals[i].destEid == destEid // @audit : also note i have reported userCollaterals[i].destEid == destEid in separated issue
697:                     && userCollaterals[i].borrowedlToken == borrowedlToken && userCollaterals[i].srcToken == srcToken
698:             ) {
699:                 return (true, i);
700:             }
701:             unchecked {
702:                 ++i;
703:             }
704:         }
705:         return (false, 0);
```
[CrossChainRouter.sol#L454](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CrossChainRouter.sol#L454)
[LendStorage.sol#L696](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/LendStorage.sol#L696)
#### Internal Pre-conditions

Nil

#### External Pre-conditions
Nil

#### Attack Path
1. The borrower borrows assets on Chain B.
2. The borrower can be liquidated.
3. The liquidator can liquidate borrower position on chain B, with the Borrower Collateral on chain A.
4. Liquidator Initiate the Liquidation on chain B.
5. on Chain A `_lzReceive` function call with the `cType=CrossChainLiquidationExecute`  and call back the Chain B with the given data and `cType=LiquidationSuccess`.
6. On Chain B , here we uses the `srcToken` which are valid on chain A to find the Borrow Position But on Chain B it is not correct. Because here we need to fetch it from `crossChainLTokenMap` mapping. 

Note : The `srcEid` and `destlToken` values are also wrong here which i reported separately.

#### Impact
The borrower can not be liquidated via Cross Chain Liquidation Due to wrong `srcToken` token address.

#### PoC
Nil

#### Mitigation
One of the Fix could be Following:
```diff
diff --git a/Lend-V2/src/LayerZero/CrossChainRouter.sol b/Lend-V2/src/LayerZero/CrossChainRouter.sol
index 6328953..9c4f564 100644
--- a/Lend-V2/src/LayerZero/CrossChainRouter.sol
+++ b/Lend-V2/src/LayerZero/CrossChainRouter.sol
@@ -451,7 +456,7 @@ contract CrossChainRouter is OApp, ExponentialNoError {
             currentEid, // srcEid is current chain
             0, // We don't know destEid yet, but we can match on other fields
             payload.destlToken,
-            payload.srcToken
+            lendStorage.crossChainAssetMap(payload.srcToken, currentEid)
         );
```

---
### [H-04] The Attacker can Drain the Liquidity 

#### Summary
While first time borrowing the `CoreRouter:borrow` function as vulnerability which will allow an attacker to drain all the available liquidity.

#### Root Cause
```solidity
        (uint256 borrowed, uint256 collateral) =
            lendStorage.getHypotheticalAccountLiquidityCollateral(msg.sender, LToken(payable(_lToken)), 0, _amount);

        LendStorage.BorrowMarketState memory currentBorrow = lendStorage.getBorrowBalance(msg.sender, _lToken);

@-----> uint256 borrowAmount = currentBorrow.borrowIndex != 0
            ? ((borrowed * LTokenInterface(_lToken).borrowIndex()) / currentBorrow.borrowIndex)
            : 0;

        require(collateral >= borrowAmount, "Insufficient collateral"); // @audit : here borrowAmount will be 0 for first borrow
```
In case of first borrow the `borrowAmount=0` , that why the next require will always pass and the attacker will be able to borrow all available liquidity.
[CoreRouter.sol#L157-L159](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CoreRouter.sol#L157-L159)

#### Internal Pre-conditions
User has no active borrow 


#### External Pre-conditions
First time borrowing

#### Attack Path



#### Impact
All the available Liquidity can be Drain.
#### PoC
First apply this git diff to test file `TestBorrowing.t.sol`
```diff
diff --git a/Lend-V2/test/TestBorrowing.t.sol b/Lend-V2/test/TestBorrowing.t.sol
index 31e767b..235080a 100644
--- a/Lend-V2/test/TestBorrowing.t.sol
+++ b/Lend-V2/test/TestBorrowing.t.sol
@@ -83,6 +84,15 @@ contract TestBorrowing is Test {
         coreRouter.supply(amount, token);
     }
 
+    function _supply(uint256 amount , address user) internal returns (address token, address lToken) {
+        token = supportedTokens[0];
+        lToken = lendStorage.underlyingTolToken(token);
+
+        ERC20Mock(token).mint(user, amount);
+        IERC20(token).approve(address(coreRouter), amount);
+        coreRouter.supply(amount, token);
+    }
+
```
Add following test case to `TestBorrowing.t.sol` file and run with command `forge test --mt test_borrow_and_drain_liquidity -vvv`:
```solidity
    function test_borrow_and_drain_liquidity() public {
        // Bound amount between 1e18 and 1e30 to ensure reasonable test values
        uint256 amount = 100e18;
        amount = bound(amount, 1e18, 1e30);
        address liquidityProvider = makeAddr("liquidityProvider");

        vm.startPrank(liquidityProvider);
        _supply(amount*2,liquidityProvider);
        vm.stopPrank();
        vm.startPrank(deployer);

        // First supply tokens as collateral
        (address token, address lToken) = _supply(amount);

        // Calculate maximum allowed borrow (70% of collateral to leave some safety margin)
        uint256 maxBorrow = (amount * 100) / 100;

        // Get initial balances
        uint256 initialTokenBalance = IERC20(token).balanceOf(deployer);

        // Expect BorrowSuccess event
        vm.expectEmit(true, true, true, true);
        emit BorrowSuccess(deployer, lToken, maxBorrow*2);

        // Borrow tokens
        coreRouter.borrow(maxBorrow*2, token);

        // Verify balances after borrowing
        assertEq(
            IERC20(token).balanceOf(deployer) - initialTokenBalance,
            maxBorrow*2,
            "Should receive correct amount of borrowed tokens"
        );

        // Verify borrow balance is tracked correctly
        assertEq(
            lendStorage.borrowWithInterestSame(deployer, lToken),
            maxBorrow*2,
            "Borrow balance should be tracked correctly"
        );

        vm.stopPrank();
    }
```

#### Mitigation
Make sure in case of first time borrowing don't allow borrower to borrow more than collateral.

---

### [H-05] An Attacker Can Exploit Cross Chain borrow to take more borrower than collateral

#### Summary
The current implementation allow the borrower to take borrow on same chain and/or cross chain, However this current design can be exploited by borrowing on different chains to take more loan than collateral. As we only check the borrowed amount on target chain and collateral on src chain. There is no way to validate that the borrower has no active borrow on same chain or other chain 


#### Root Cause
while Performing cross chain borrow on src chain we fetch the collateral and than validate it on target chain with the borrowed amount :
```solidity
/2025-05-lend-audit-contest/Lend-V2/src/LayerZero/CrossChainRouter.sol:113
113:     function borrowCrossChain(uint256 _amount, address _borrowToken, uint32 _destEid) external payable {
....
136:         // Get current collateral amount for the LayerZero message
137:         // This will be used on dest chain to check if sufficient
138:         (, uint256 collateral) =
139:             lendStorage.getHypotheticalAccountLiquidityCollateral(msg.sender, LToken(_lToken), 0, 0);
140: 
141:         // Send message to destination chain with verified sender
142:         // borrowIndex of 0 initially - will be set correctly on dest chain
143:         _send(
144:             _destEid,
145:             _amount,
146:             0, // Initial borrowIndex, will be set on dest chain
147:             collateral,
148:             msg.sender,
149:             destLToken,
150:             address(0), // liquidator
151:             _borrowToken,
152:             ContractType.BorrowCrossChain
153:         );
154:     }
```
`147` collateral than got validated on targetChain :
On the Target chain : 
```solidity
/2025-05-lend-audit-contest/Lend-V2/src/LayerZero/CrossChainRouter.sol:581
581:     function _handleBorrowCrossChainRequest(LZPayload memory payload, uint32 srcEid) private {
582:         // Accrue interest on borrowed token on destination chain
.... 
616:         // Get existing borrow amount
617:         (uint256 totalBorrowed,) = lendStorage.getHypotheticalAccountLiquidityCollateral(
618:             payload.sender, LToken(payable(payload.destlToken)), 0, payload.amount
619:         );
620: 
621:         // Verify the collateral from source chain is sufficient for total borrowed amount
622:         require(payload.collateral >= totalBorrowed, "Insufficient collateral");
```
Here we check on target chain where borrower will received borrow amount that the collateral on other chain is sufficient to cover the borrow amount.
[CrossChainRouter.sol#L138-L139](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CrossChainRouter.sol#L138-L139)
[CrossChainRouter.sol#L622](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CrossChainRouter.sol#L622)

In case of same Chain borrow we only check the use has any borrow on same chain:
```solidity
/home/aman/Desktop/audits/2025-05-lend-audit-contest/Lend-V2/src/LayerZero/CoreRouter.sol:147
147:     function borrow(uint256 _amount, address _token) external {
....
154:         (uint256 borrowed, uint256 collateral) =
155:             lendStorage.getHypotheticalAccountLiquidityCollateral(msg.sender, LToken(payable(_lToken)), 0, _amount);
156: 
157:         LendStorage.BorrowMarketState memory currentBorrow = lendStorage.getBorrowBalance(msg.sender, _lToken);
158: 
159:         uint256 borrowAmount = currentBorrow.borrowIndex != 0
160:             ? ((borrowed * LTokenInterface(_lToken).borrowIndex()) / currentBorrow.borrowIndex)
161:             : 0;
162: 
163:         require(collateral >= borrowAmount, "Insufficient collateral");
```
here the `currentBorrow::borrowIndex` will be zero as the user has cross chain borrow.
[CoreRouter.sol#L155-L161](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CoreRouter.sol#L155-L161)

#### Internal Pre-conditions
User has Cross chain Borrow


#### External Pre-conditions

Nil

#### Attack Path
1. Attacker supplies token A as collateral on Chain A
2. Attacker initiates cross-chain borrow from Chain B using maximum allowed amount
3. Protocol records the borrow position on Chain B
4. Attacker cannot borrow more on Chain B due to insufficient collateral coverage
5. Attacker initiates another borrow on Chain A where their collateral is located
6. Since in borrow function we do not check cross chain borrow.
7. Attacker successfully borrows more than their collateral should allow



#### Impact
By combining same chain and cross chain borrow the attacker can take more loan than its collateral. The attack can also work with different cross chain borrow


#### PoC
Add the following Test case to `TestLiquidations.t.sol` and run with command `forge --mt test_cross_chain_borrow_with_two -vvv`:
```solidity
    function test_cross_chain_borrow_with_two()
        public
    {
        uint256 supplyAmount = 100e18;
        uint256 borrowAmount=55e18;
        supplyAmount =  1000e18;
        borrowAmount = supplyAmount * 65 / 100;

        // Supply collateral on Chain A
        (address tokenA, address lTokenA) = _supplyA(deployer, supplyAmount, 0);

         _supplyA(liquidator, supplyAmount, 0);
        // Supply liquidity on Chain B for borrowing
        (address tokenB, address lTokenB) = _supplyB(liquidator, supplyAmount * 2, 0);

        vm.deal(address(routerA), 1 ether); // For LayerZero fees

        // Borrow cross-chain from Chain A to Chain B
        vm.startPrank(deployer);
        routerA.borrowCrossChain(borrowAmount/2, tokenA, CHAIN_B_ID);
        // vm.roll(100);
        routerA.borrowCrossChain(borrowAmount/2, tokenA, CHAIN_B_ID);
        console2.log("Cross chain borrow will revert as we have alrready borrowed the allowed amount");
        vm.expectRevert();
        routerA.borrowCrossChain(borrowAmount/2, tokenA, CHAIN_B_ID);
        console2.log("But the borrower can still perform same chain borrow and in this way borrower can borrow more than allowed amount");
        coreRouterA.borrow(borrowAmount*2, tokenA);
        vm.stopPrank();
    }
```

#### Mitigation
Inside borrow function also check if user has any crossChain borrow

---

### [H-06] Incorrect Exchange Rate Usage in Supply and Redeem Operations

#### Summary
The `CoreRouter` contract uses outdated exchange rates when calculating minted and redeemed tokens, leading to potential loss of assets for suppliers. The issue occurs because the contract uses the exchange rate before minting/redeeming operations, while the actual operations operate on updated the exchange rate.

#### Root Cause
The issue stems from using exchange rates before they are updated by the mint/redeem operations:

In supply operation:
```solidity
uint256 exchangeRateBefore = LTokenInterface(_lToken).exchangeRateStored();

// Mint lTokens
require(LErc20Interface(_lToken).mint(_amount) == 0, "Mint failed");

// Calculate actual minted tokens using exchangeRate from before mint
uint256 mintTokens = (_amount * 1e18) / exchangeRateBefore;
```

In redeem operation:
```solidity
uint256 exchangeRateBefore = LTokenInterface(_lToken).exchangeRateStored();

// Calculate expected underlying tokens
uint256 expectedUnderlying = (_amount * exchangeRateBefore) / 1e18;

// Perform redeem
require(LErc20Interface(_lToken).redeem(_amount) == 0, "Redeem failed");

// Transfer underlying tokens to the user
IERC20(_token).transfer(msg.sender, expectedUnderlying);
```

#### Internal Pre-conditions
- User has sufficient collateral to supply
- User has sufficient lTokens to redeem
- Exchange rate changes between operations

#### External Pre-conditions
- Multiple users interacting with the protocol
- Interest accrual affecting exchange rates
- Users performing supply and redeem operations

#### Attack Path
1. User A supplies collateral using outdated exchange rate and borrow assets
2. Exchange rate updates due to interest accrual
3. User B supplies and redeems their tokens
4. User A  repay borrow and attempts to redeem their tokens
5. User A receives fewer tokens than expected due to incorrect exchange rate calculation

#### Impact

Loss of assets for suppliers due to incorrect exchange rate calculations.Systematic underpayment to suppliers when redeeming their collateral.Chains with faster block times (e.g., 2 seconds) experience more frequent exchange rate updates so the impact on such chains will be very high.

#### PoC
```solidity
function test_outdated_usage_of_exchangeRate_issue() public {
    // Bound array length to a reasonable size
    uint256 collateralAmount = 10e18;
    address borrower = makeAddr("borrower");
    address borrower1 = makeAddr("borrower1");
    address token = supportedTokens[0];

    address lToken = lendStorage.underlyingTolToken(token);

    uint256 exchangeRate = LTokenInterface(lToken).exchangeRateStored();
    uint256 expectedTokens = (collateralAmount * 1e18) / exchangeRate;

    vm.startPrank(borrower);
    // Supply collateral
    _supply(collateralAmount, borrower);
    vm.stopPrank();
    assertEq(
        LTokenInterface(lToken).balanceOf(address(coreRouter)),
        expectedTokens,
        "Router should have received correct amount of lTokens First deposit"
    );
    uint256 borrowAmount = 7e18;
    vm.startPrank(borrower);
    // Get initial balance
    uint256 initialTokenBalance = IERC20(token).balanceOf(borrower);
    vm.expectEmit(true, true, true, true);
    emit BorrowSuccess(borrower, lToken, borrowAmount);
    coreRouter.borrow(borrowAmount, token);
    vm.stopPrank();
    // Verify balances after borrowing
    assertEq(
        IERC20(token).balanceOf(borrower) - initialTokenBalance,
        borrowAmount,
        "Should receive correct amount of borrowed tokens"
    );

    // Verify final borrow balance
    assertEq(
        lendStorage.borrowWithInterestSame(borrower, lToken), borrowAmount, "Total borrowed amount should match"
    );        
    vm.roll(100);
    uint256 exchangeRate1 = LTokenInterface(lToken).exchangeRateStored();
    uint256 expectedTokens1 = (collateralAmount * 1e18) / exchangeRate1;

    vm.startPrank(borrower1);
    // Supply collateral
    (token, ) = _supply(collateralAmount, borrower1);
    coreRouter.redeem(expectedTokens1, payable(lToken));
    vm.stopPrank();

    vm.startPrank(borrower);
    borrowAmount = lendStorage.borrowWithInterestSame(borrower, lToken);
    ERC20Mock(token).mint(borrower, borrowAmount); // mint extra token, that borrower can pay all his borrower amount + interest 
    IERC20(token).approve(address(coreRouter), borrowAmount);
    coreRouter.repayBorrow(borrowAmount, lToken); // @audit: repay complete borrow
    vm.expectRevert(); // will revert on Error.INSUFFICIENT_LIQUIDITY
    coreRouter.redeem(expectedTokens, payable(lToken)); // @audit: here the borrower will not be able to redeem all of his collateral, due to bug in the code
    vm.stopPrank();
    // @audit: the router must have lToken balance of borrower, but it does not have, So the next check will fail
    assertEq(
        LTokenInterface(lToken).balanceOf(address(coreRouter)),
        expectedTokens,
        "Router should have received correct amount of lTokens"
    );
}
```
Add the above test case to `test/TestBorrowing.t.sol` and run with command `forge test --mt test_outdated_usage_of_exchangeRate_issue -vvvv`
#### Mitigation
1. Use exchange rates after mint/redeem operations:
```solidity
// For supply
require(LErc20Interface(_lToken).mint(_amount) == 0, "Mint failed");
uint256 exchangeRateAfter = LTokenInterface(_lToken).exchangeRateStored();
uint256 mintTokens = (_amount * 1e18) / exchangeRateAfter;

// For redeem
require(LErc20Interface(_lToken).redeem(_amount) == 0, "Redeem failed");
uint256 exchangeRateAfter = LTokenInterface(_lToken).exchangeRateStored();
uint256 expectedUnderlying = (_amount * exchangeRateAfter) / 1e18;
IERC20(_token).transfer(msg.sender, expectedUnderlying);
```

---

### [M-01] The liquidator will not be able to redeem received collateral

#### Summary
The liquidator receives the borrower’s collateral at a discounted rate upon liquidation. However, if the liquidation occurs when no active collateral is deposited by liquidator, the liquidator cannot redeem any assets because their address isn't added to the `suppliedAssets` mapping during the liquidation process.

#### Root Cause
The collateral liquidator received has not been added to `userSuppliedAssets`:
```solidity
function liquidateSeizeUpdate(
        address sender,
        address borrower,
        address lTokenCollateral,
        address borrowedlToken,
        uint256 repayAmount
    ) internal {
 ...
        lendStorage.updateTotalInvestment(
            sender,
            lTokenCollateral,
            lendStorage.totalInvestment(sender, lTokenCollateral) + (seizeTokens - currentReward)
        );
    }
```
Here we do add the received collateral to `totalInvestment` of liquidator. But we do not add it to `userSuppliedAssets`.
[CoreRouter.sol#L278-L318](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CoreRouter.sol#L278-L318)

#### Internal Pre-conditions
The liquidator does not have active deposited of a received collateral.


#### External Pre-conditions
The borrower is Liquidatable.


#### Attack Path



#### Impact
The liquidator will not be able to redeem the collateral received through liquidation.  
To redeem these assets, the liquidator is forced to deposit the same collateral using the `supply` function.


#### PoC
Add Following POC to test file `TestLiquidations.t.sol` and run with command `forge test --mt test_redeem_revert_after_liquidation  -vvv`:
```solidity
    function test_redeem_revert_after_liquidation() public {
        // Bound inputs to reasonable values
        uint256 supplyAmount=100e18; 
        uint256 borrowAmount=50e18; 
        uint256 newPrice=1e16;
        supplyAmount = bound(supplyAmount, 100e18, 1000e18);
        borrowAmount = bound(borrowAmount, 50e18, supplyAmount * 60 / 100); // Max 60% LTV
        newPrice = bound(newPrice, 1e16, 5e16); // 1-5% of original price

        // Supply token0 as collateral on Chain A
        (address tokenA, address lTokenA) = _supplyA(deployer, supplyAmount, 0);

        // Supply token1 as liquidity on Chain A from random wallet
        (address tokenB, address lTokenB) = _supplyA(address(1), supplyAmount, 1);

        vm.prank(deployer);
        coreRouterA.borrow(borrowAmount, tokenB);

        // Simulate price drop of collateral (tokenA) only on first chain
        priceOracleA.setDirectPrice(tokenA, newPrice);
        // Simulate price drop of collateral (tokenA) only on second chain
        priceOracleB.setDirectPrice(
            lendStorageB.lTokenToUnderlying(lendStorageB.crossChainLTokenMap(lTokenA, block.chainid)), newPrice
        );

       address liquidator1 = makeAddr("liquidator1");

        // Attempt liquidation
        vm.startPrank(liquidator1);
        ERC20Mock(tokenB).mint(liquidator1, borrowAmount);
        IERC20(tokenB).approve(address(coreRouterA), borrowAmount);

        // Expect LiquidateBorrow event
        vm.expectEmit(true, true, true, true);
        emit LiquidateBorrow(liquidator1, lTokenB, deployer, lTokenA);

        // Repay 0.5% of the borrow
        uint256 repayAmount = borrowAmount / 200;

        coreRouterA.liquidateBorrow(deployer, repayAmount, lTokenA, tokenB);
        coreRouterA.redeem(131220000000,payable(lTokenA));// totalInvestment = 131220000000
        vm.stopPrank();

        // Verify liquidation was successful
        assertLt(
            lendStorageA.borrowWithInterest(deployer, lTokenB),
            borrowAmount,
            "Borrow should be reduced after liquidation"
        );
    }

```
output :
```shell
  ├─ [22928] CoreRouter A::redeem(131220000000 [1.312e11], LErc20Immutable: [0x4f559F30f5eB88D635FDe1548C4267DB8FaB0351])
    │   ├─ [2661] LendStorage A::lTokenToUnderlying(LErc20Immutable: [0x4f559F30f5eB88D635FDe1548C4267DB8FaB0351]) [staticcall]
    │   │   └─ ← [Return] ERC20Mock: [0x34A1D3fff3958843C43aD80F30b94c510645C316]
    │   ├─ [794] LendStorage A::totalInvestment(liquidator1: [0x30b6Bf39Ea8A033e5e4EB21A9015d84891DC6719], LErc20Immutable: [0x4f559F30f5eB88D635FDe1548C4267DB8FaB0351]) [staticcall]
    │   │   └─ ← [Return] 131220000000 [1.312e11]
    │   ├─ [15974] LendStorage A::getHypotheticalAccountLiquidityCollateral(liquidator1: [0x30b6Bf39Ea8A033e5e4EB21A9015d84891DC6719], LErc20Immutable: [0x4f559F30f5eB88D635FDe1548C4267DB8FaB0351], 131220000000 [1.312e11], 0) [staticcall]
    │   │   ├─ [4335] PriceOracle A::getUnderlyingPrice(LErc20Immutable: [0x4f559F30f5eB88D635FDe1548C4267DB8FaB0351]) [staticcall]
    │   │   │   ├─ [1255] LErc20Immutable::symbol() [staticcall]
    │   │   │   │   └─ ← [Return] "lE20M"
    │   │   │   ├─ [448] LErc20Immutable::underlying() [staticcall]
    │   │   │   │   └─ ← [Return] ERC20Mock: [0x34A1D3fff3958843C43aD80F30b94c510645C316]
    │   │   │   └─ ← [Return] 10000000000000000 [1e16]
    │   │   ├─ [668] Lendtroller A::getCollateralFactorMantissa(LErc20Immutable: [0x4f559F30f5eB88D635FDe1548C4267DB8FaB0351]) [staticcall]
    │   │   │   └─ ← [Return] 750000000000000000 [7.5e17]
    │   │   ├─ [2065] LErc20Immutable::exchangeRateStored() [staticcall]
    │   │   │   ├─ [582] ERC20Mock::balanceOf(LErc20Immutable: [0x4f559F30f5eB88D635FDe1548C4267DB8FaB0351]) [staticcall]
    │   │   │   │   └─ ← [Return] 100000000000000000000 [1e20]
    │   │   │   └─ ← [Return] 200000000000000000000000000 [2e26]
    │   │   └─ ← [Return] 196830000000000000 [1.968e17], 0
    │   └─ ← [Revert] revert: Insufficient liquidity
    └─ ← [Revert] revert: Insufficient liquidity
```

#### Mitigation
Inside `liquidateSeizeUpdate` function add the collateral to liquidator storage:
```diff
diff --git a/Lend-V2/src/LayerZero/CoreRouter.sol b/Lend-V2/src/LayerZero/CoreRouter.sol
index 7212162..5e291f3 100644
--- a/Lend-V2/src/LayerZero/CoreRouter.sol
+++ b/Lend-V2/src/LayerZero/CoreRouter.sol
@@ -307,6 +310,7 @@ contract CoreRouter is Ownable, ExponentialNoError {
         lendStorage.updateTotalInvestment(
             borrower, lTokenCollateral, lendStorage.totalInvestment(borrower, lTokenCollateral) - seizeTokens
         );
+        lendStorage.addUserSuppliedAsset(msg.sender, lTokenCollateral);
         lendStorage.updateTotalInvestment(
             sender,
```
