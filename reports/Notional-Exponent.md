# Notional Exponent

## Protocol Summary

Notional Exponent is a leveraged yield protocol. Notional Exponent enables users to borrow from Morpho to establish leveraged staking, leveraged PT, and leveraged liquidity strategies.


[Ethereum][Solidity][DeFi][Lending]

---

### [H-01] The Dinero integration is not correct in some edge cases

#### Summary

To withdraw from Dinero, we call initiateWithdraw with the target assetsToWithdraw. Dinero splits the assets into 32 ETH batches and assigns them sequential batchIds (token IDs). The initialBatchId and finalBatchId are recorded and packed with a random s_batchNonce.

In edge cases, the finalBatchId may not be owned by the user. Despite this, the system still includes it in the validation check during finalizeWithdraw.

If the validator tied to this extra batchId (not owned by the WithdrawRequestManager) is slashed or dissolved, the withdrawal finalization will fail even if:

The user-owned batchId is still valid and unstaked.
The actual claimable assets are unaffected.

#### Root Cause

The Root Cause for this issue lies at 2 places.

we store the initialBatchId and finalBatchId , before and after initiateRedemption call.

```solidity
        uint256 initialBatchId = PirexETH.batchId();  
        pxETH.approve(address(PirexETH), amountToWithdraw);  
        // TODO: what do we put for should trigger validator exit?  
        PirexETH.initiateRedemption(amountToWithdraw, address(this), false);  
        uint256 finalBatchId = PirexETH.batchId();  
```

[Here](https://github.com/sherlock-audit/2025-06-notional-exponent/blob/main/notional-v4/src/withdraws/Dinero.sol#L27-L31)

when we check that the withdraw Can be finalized we check all the batchIds we have decoded from requestId:

```solidity
        (uint256 initialBatchId, uint256 finalBatchId) = _decodeBatchIds(requestId);  
        uint256 totalAssets;  
  
        for (uint256 i = initialBatchId; i <= finalBatchId; i++) {  
            IPirexETH.ValidatorStatus status = PirexETH.status(PirexETH.batchIdToValidator(i));  
  
            if (status != IPirexETH.ValidatorStatus.Dissolved && status != IPirexETH.ValidatorStatus.Slashed) {  
                // Can only finalize if all validators are dissolved or slashed  
                return false;  
            }  
```

[Here](https://github.com/sherlock-audit/2025-06-notional-exponent/blob/main/notional-v4/src/withdraws/Dinero.sol#L72C1-L82C1)

#### Internal Pre-conditions

The the assetsToWithdraw after deducting the Dinero Fee is exactly 32 ETH.

```solidity
        while (pendingWithdrawal / DEPOSIT_SIZE != 0) {  
            uint256 _allocationPossible = DEPOSIT_SIZE +  
                _pxEthAmount -  
                pendingWithdrawal;  
  
            upxEth.mint(_receiver, batchId, _allocationPossible, "");  
  
            (bytes memory _pubKey, , , , ) = _stakingValidators.getNext(  
                withdrawalCredentials  
            );  
  
            pendingWithdrawal -= DEPOSIT_SIZE;  
            _pxEthAmount -= _allocationPossible;  
  
            oracleAdapter.requestVoluntaryExit(_pubKey);  
  
            batchIdToValidator[batchId++] = _pubKey;  
            status[_pubKey] = DataTypes.ValidatorStatus.Withdrawable;  
        }  
```

[PirexEthValidators.sol::_initiateRedemption](https://vscode.blockscan.com/ethereum/0xD664b74274DfEB538d9baC494F3a4760828B02b0)

#### External Pre-conditions

The Next Validator got slashed or dissolved to which the finalBatchId got mapped in PirexEthValidators.sol::batchIdToValidator.

```solidity
    batchIdToValidator[batchId++] = _pubKey;  
```

#### Attack Path

1. Alice stakes 32 ETH on PxETH and the batchId = 1 at PirexEthValidators.sol.
2. Alice initiates a withdrawal of 32 ETH, which increments the batchId to 2.
3. At this point: `initialBatchId = 1`, `finalBatchId = 2`.
4. The withdrawal request spans both batches, but tokenId = 2 holds 0 assets as it is not minted yet.
5. The assigned pubKeys in batchIdToValidator are as follows:
   - For batchId = 1: `0x8c9f8152b8efa4fb3788070650b9d1bf0e3a984c07d76affb6042d0a991f4d2d3a364d50d3222bb9eda434d31cab3294`
   - For batchId = 2: 0x (unset)
6. Bob initiate a withdrawal under tokenId = 2, and is assigned:
   - pubKey = 0xac8601b780e5a5246279af39e405a976eba31fd1c418da342550164fb01bec93e2d19deab7e66ab7e3e0804a915d218a
7. Bob's validator (tokenId = 2) gets slashed or dissolved. His withdrawal cannot be finalized as expected.
8. Alice now attempts to call finalizeWithdraw, but:
   - canFinalizeWithdrawRequest returns false due to tokenId = 2 being slashed
   - Alice has no assets claimable from tokenId = 2
9. To the best of my knowledge, Her withdrawal is permanently blocked.

Note : Here i assumed the fee at Dinero is 0

#### Impact

Due to the inclusion of an extra batchId in the withdrawal finalization logic, if the pubKey associated with that batchId gets slashed, the user will be unable to finalize their withdrawal even if they do not own that batchId.

#### PoC

Nil

#### Mitigation

In canFinalizeWithdrawRequest also check if balanceOf(address(this) , i) >0 along with the status check than return false.

```diff
-             if (status != IPirexETH.ValidatorStatus.Dissolved && status != IPirexETH.ValidatorStatus.Slashed) {  
+             if (status != IPirexETH.ValidatorStatus.Dissolved && status != IPirexETH.ValidatorStatus.Slashed && upxETH.balanceOf(address(this), i)>0) {  
```

---

### [M-01] User will not be able to Migrate Collateral in Some cases.

#### Summary

The migratePosition function, which is designed to move a user's entire position (collateral and debt) to a new lending router, fails for users who have only supplied collateral and have no outstanding debt. The underlying call to MORPHO.repay reverts when both the asset and share amounts for repayment are zero, which is the case for these collateral only positions. This effectively blocks a segment of users from using the migration feature.

#### Root Cause

in case of migratePosition we pass the assetToRepay= type(uint256).max to exit position function.

```solidity
  ) internal returns (uint256 sharesReceived) {  
        if (migrateFrom != address(0)) {  
            // Allow the previous lending router to repay the debt from assets held here.  
            ERC20(asset).checkApprove(migrateFrom, assetAmount);  
            sharesReceived = ILendingRouter(migrateFrom).balanceOfCollateral(onBehalf, vault);  
  
            // Must migrate the entire position  
            ILendingRouter(migrateFrom).exitPosition(  
                onBehalf, vault, address(this), sharesReceived, type(uint256).max, bytes("")  
            );  
```

[Here](https://github.com/sherlock-audit/2025-06-notional-exponent/blob/main/notional-v4/src/routers/AbstractLendingRouter.sol#L229-L238)

Inside exitPosition function we check if the assetToRepay > 0 than we call the _exitWithRepay function.

```solidity
        if (assetToRepay == type(uint256).max) {  
            // If assetToRepay is uint256.max then get the morpho borrow shares amount to  
            // get a full exit.  
            sharesToRepay = MORPHO.position(morphoId(m), onBehalf).borrowShares;  
            assetToRepay = 0;  
        }  
  
    ...  
    MORPHO.repay(m, assetToRepay, sharesToRepay, onBehalf, repayData);  
```

As the assetToRepay==type(uint256).max, So we get the borrowShares of user and update assetToRepay =0 and pass these arguments to Morpho repay function to clear the debt on current lending router.

[Here](https://github.com/sherlock-audit/2025-06-notional-exponent/blob/main/notional-v4/src/routers/MorphoLendingRouter.sol#L188-L201)

#### Internal Pre-conditions

The user has 0 borrowShares on old lending router.

#### External Pre-conditions

The user wants to migrate collateral to other lending router which offer best borrow rate so to take loan on that lending router.

#### Attack Path

1. A user supplies 100 USDC as collateral into LendingRouter1 but does not take out a loan, resulting in a collateral only position.
2. The user identifies that LendingRouter2 offers more favorable borrowing rates and decides to move their position.
3. The user calls migratePosition on LendingRouter2, specifying LendingRouter1 as the source.
4. The transaction reverts. This is because the underlying exitPosition call in LendingRouter1 attempts to repay a debt of zero. The subsequent call to MORPHO.repay with both assetToRepay and sharesToRepay as 0 fails, causing the entire migration to be unsuccessful.

#### Impact

Users who have supplied collateral but have no debt are unable to use the migratePosition feature. They are forced to either take on unwanted debt to enable migration or perform a manual, full exit and re-entry into the new lending router.

#### PoC

NIL

#### Mitigation

In MorphoLendingRouter._exitWithRepay, add a check to bypass the call to MORPHO.repay if there is no debt to repay. This ensures that collateral only migrations can proceed without a failing external call.