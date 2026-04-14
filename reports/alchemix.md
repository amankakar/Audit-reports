# Alchemix / alchemix-v3

Alchemix lets you instantly access loans representing your collateral's future yield. Over time, the interest your deposit earns is used to repay your debt automatically. Alchemix loans are self-repaying, interest-free, and non-liquidating.


[Ethereum][Solidity][DeFi][Lending]

---
### [H-01] DoS Vulnerability in Earmark System Through poke calling on each block for certain time.

#### Summary
A malicious actor can cause the Dos if attacker calls the `poke` after each block as shown in POC. This Dos occur in `_earmark` function.  
#### Finding Description
The vulnerability exists in the `WeightIncrement` function which is called during the earmarking process:

```solidity
function WeightIncrement(uint256 increment, uint256 total) internal pure returns (uint256) {
    unchecked {
        require(increment <= total); // @audit : revert would occur here          //support ratios of 1.0 or less
        require(total <= type(uint128).max);   //Overflow check for (total - increment)<<128

        if (increment == 0) {
            return 0;
        }

        uint256 ratio = ((total - increment) << 128) / total;
        if (ratio == 0) {
            return LOG2NEGFRAC_1+1;
        }
        return Log2NegFrac(ratio);
    }
}
```

The issue occurs because:
1. The function requires `increment <= total`
2. A malicious actor can call `poke` repeatedly to manipulate these values
3. When `increment > total`, the function reverts
4. This prevents the earmarking process from completing
5. Critical functions like `repay` and `mint` become unusable

#### Impact Explanation
**Severity: High**

The impact is high because:
1. Users cannot repay their debt
2. New users cannot mint tokens
3. The protocol's core functionality is disrupted
4. The attack is cheap to execute
5. The attack affects all users of the protocol

#### Likelihood Explanation
**Likelihood: Medium**

The attack is easy to execute.The attack can be performed by any user. However it will occur after nearly at the end of `5_256_000` blocks.

#### Proof of Concept
```solidity
// add this test case to AlchemistV3.t.sol

function testDosInQueryGraphCall() external {
    // Setup initial position
    uint256 amount = 100e18;
    vm.startPrank(address(0xbeef));
    SafeERC20.safeApprove(address(fakeYieldToken), address(alchemist), amount + 100e18);
    alchemist.deposit(amount, address(0xbeef), 0);
    uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
    alchemist.mint(tokenId, (amount / 2), address(0xbeef));
    vm.stopPrank();

    // Create redemption
    vm.startPrank(address(0xdad));
    SafeERC20.safeApprove(address(alToken), address(transmuterLogic), 50e18);
    transmuterLogic.createRedemption(50e18);
    vm.stopPrank();

    // Perform DoS attack
    for(int i=0 ; i < 100 ; i++){
        vm.roll(block.number + 1);
        alchemist.poke(tokenId);
    }

    // Verify DoS effect
    vm.prank(address(0xbeef));
    vm.expectRevert();
    alchemist.repay(25e18, tokenId);

    // Verify new users affected
    vm.startPrank(address(externalUser));
    SafeERC20.safeApprove(address(fakeYieldToken), address(alchemist), amount + 100e18);
    alchemist.deposit(amount, address(externalUser), 0);
    uint256 tokenId1 = AlchemistNFTHelper.getFirstTokenId(address(externalUser), address(alchemistNFT));
    vm.expectRevert();
    alchemist.mint(tokenId1, (amount / 2), address(externalUser));
}
forge test --mt testDosInQueryGraphCall -vvv
```

#### Recommendation
The Best approach would be to return `0` in such case instead of reverting. 

#### Related Files

- `src/AlchemistV3.sol` (lines 602-605)
- `src/libraries/PositionDecay.sol` (lines 41-41)

---
### [H-02] Incorrect Fee Transfer in Repay Function Leading to Double Payment


#### Summary
The `repay` function incorrectly transfers the same amount (`creditToYield`) to both the transmuter and protocol fee receiver, causing users to pay double the intended amount.

#### Finding Description
In the `repay` function, there's a critical issue where the same amount of yield tokens (`creditToYield`) is being transferred to both the transmuter and protocol fee receiver. This is incorrect because:

1. The protocol fee is already being deducted from the user's collateral balance:
```solidity
account.collateralBalance -= creditToYield * protocolFee / BPS;
```

2. Then the same full amount is being transferred twice:
```solidity
TokenUtils.safeTransferFrom(yieldToken, msg.sender, transmuter, creditToYield);
TokenUtils.safeTransfer(yieldToken, protocolFeeReceiver, creditToYield);
```

This means users are effectively paying:
- The full amount to the transmuter
- The full amount again to the protocol fee receiver
- Plus the fee deduction from their collateral balance

#### Impact Explanation

The impact is high because:
Users lose more funds than intended. The protocol receives more fees than designed

#### Likelihood Explanation

This is highly likely to occur because:
It affects every repayment transaction

#### Proof of Concept
```solidity
// add this test case to AlchemistV3.t.sol
    function test2AMANRepayUnearmarkedDebtOnly() external { // @audit : POc for wrong amount being send to fee receiver lose for user
        uint256 amount = 100e18;

        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(fakeYieldToken), address(alchemist), amount + 100e18);
        alchemist.deposit(amount, address(0xbeef), 0);
        // a single position nft would have been minted to 0xbeef
        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        alchemist.mint(tokenId, amount / 2, address(0xbeef));

        uint256 preRepayBalance = fakeYieldToken.balanceOf(address(0xbeef));
        uint256 preBalance = fakeYieldToken.balanceOf(address(alchemist));


        vm.roll(block.number + 2);

        alchemist.repay(100e18, tokenId);
        // alchemist.withdraw(amount, address(0xbeef), tokenId);

        vm.stopPrank();

        (, uint256 userDebt,) = alchemist.getCDP(tokenId);

        assertEq(userDebt, 0);

        // Test that transmuter received funds
        assertEq(fakeYieldToken.balanceOf(address(transmuterLogic)), alchemist.convertDebtTokensToYield(amount / 2));
        uint256 postBalance = fakeYieldToken.balanceOf(address(alchemist));
        console.log("ffffF" , preBalance - postBalance); // @audit : lose of assets
    }
Run with command forge test --mt test2AMANRepayUnearmarkedDebtOnly 
```

The POC demonstrates:
1. User deposits yield tokens
2. User mints debt tokens
3. User attempts to repay
4. The same amount is transferred twice
5. User loses more tokens than intended

#### Recommendation
The fix should separate the fee amount from the repayment amount:

```diff 
function repay(uint256 amount, uint256 recipientTokenId) public returns (uint256) {
    // ... existing checks ...

    uint256 yieldToDebt = convertYieldTokensToDebt(amount);
    uint256 credit = yieldToDebt > debt ? debt : yieldToDebt;
    uint256 creditToYield = convertDebtTokensToYield(credit);
    
    // Calculate fee amount
    uint256 feeAmount = creditToYield * protocolFee / BPS;
+   uint256 transmuterAmount = creditToYield - feeAmount;

    // Repay debt from earmarked amount of debt first
    uint256 earmarkToRemove = credit > account.earmarked ? account.earmarked : credit;
    account.earmarked -= earmarkToRemove;

    // Update collateral balance
+   account.collateralBalance -= feeAmount;
    
    _subDebt(recipientTokenId, credit);

    // Transfer correct amounts
+   TokenUtils.safeTransferFrom(yieldToken, msg.sender, transmuter, transmuterAmount);
+   TokenUtils.safeTransferFrom(yieldToken, msg.sender, protocolFeeReceiver, feeAmount);
}
```

#### Related Files

- `src/AlchemistV3.sol` (lines 522-522)

---
### [M-01] Liquidation Function Vulnerable to DoS Due to Price Drops


#### Summary
The liquidation function can revert in two scenarios: when the price drops significantly, causing insufficient ERC20 tokens for transfer, and when there are other user deposits, causing issues with `rawLocked` amount calculations but in this case if call succeed than assets of other users will be transferred.

#### Finding Description
The liquidation process is vulnerable to reverts in two critical scenarios:

1. **Price Drop Scenario:**
```solidity
collateralInUnderlying = totalValue(accountId);
collateralizationRatio = collateralInUnderlying * FIXED_POINT_SCALAR / account.debt;

if (collateralizationRatio <= collateralizationLowerBound) {
    // ... liquidation calculations ...
    TokenUtils.safeTransfer(yieldToken, transmuter, adjustedDebtToBurn);
}
```
When the price drops significantly:
- The calculated `adjustedDebtToBurn` might be larger than available tokens
- The `safeTransfer` will revert due to insufficient balance
- This prevents liquidation of undercollateralized positions

2. **RawLocked Amount Issue:**
```solidity
uint256 toFree = convertDebtTokensToYield(amount) * minimumCollateralization / FIXED_POINT_SCALAR;

if (toFree > _totalLocked) {
    toFree = _totalLocked;
}

_totalLocked -= toFree;
account.rawLocked -= toFree;
```
When there are other user deposits:
- The `toFree` calculation might be incorrect
- `account.rawLocked` might be less than `toFree`
- This causes underflow in the subtraction operation

#### Impact Explanation
**Severity: High**

The impact is high because:
 Undercollateralized positions cannot be liquidated. Protocol solvency is at risk.

#### Likelihood Explanation
**Likelihood: Medium**

Price drops are common and also the price can got increase , So this makes the recovery possible only if the price got increase.
#### Proof of Concept
```solidity
// add this test case to AlchemistV3.t.sol

function test_Liquidation_Revert() external {
    // Setup initial positions
    uint256 amount = 200_000e18;
    // ... setup code ...

    // Modify yield token price
    uint256 initialVaultSupply = IERC20(address(fakeYieldToken)).totalSupply();
    fakeYieldToken.updateMockTokenSupply(initialVaultSupply);
    uint256 modifiedVaultSupply = (initialVaultSupply * 1200 / 10_000) + initialVaultSupply;
    fakeYieldToken.updateMockTokenSupply(modifiedVaultSupply);

    // Attempt liquidation
    uint256[] memory accountsToLiquidate = new uint256[](1);
    accountsToLiquidate[0] = tokenIdFor0xBeef;

    // Liquidation reverts
    vm.expectRevert();
    (uint256 assets, uint256 feeInYield, uint256 feeInUnderlying) = alchemist.batchLiquidate(accountsToLiquidate);
}
Run test with command : forge test --mt  test_Liquidation_Revert -vvv
```

#### Recommendation
Add a mechanism that handle such cases.

#### Related Files

- `src/AlchemistV3.sol` (lines 808-808)
- `src/AlchemistV3.sol` (lines 816-816)
- `src/AlchemistV3.sol` (lines 827-827)
- `src/AlchemistV3.sol` (lines 883-883)



---



### [I-01] Redeem Function Vulnerable to Overflow/Underflow Due to Incorrect Fee Calculation


#### Summary
The `redeem` function includes protocol fees in the total amount to be deducted from `_totalLocked`, which can cause overflow/underflow when the total amount exceeds the available locked collateral.

#### Finding Description
In the `redeem` function, there's a critical issue where the protocol fee is incorrectly included in the total amount to be deducted from `_totalLocked`:

```solidity
function redeem(uint256 amount) external onlyTransmuter {
    _earmark();

    _redemptionWeight += PositionDecay.WeightIncrement(amount, cumulativeEarmarked);

    // Calculate current fee price
    uint256 collRedeemed = convertDebtTokensToYield(amount);
    uint256 feeCollateral = collRedeemed * protocolFee / BPS;
    uint256 totalOut = collRedeemed + feeCollateral;  // Issue: Including fee in totalOut

    // Update weights and totals
    uint256 old = _totalLocked;
    _totalLocked = old - totalOut;  // Will revert if totalOut > old
```

The issue occurs because:
1. The function calculates `totalOut` as `collRedeemed + feeCollateral`
2. This total is then subtracted from `_totalLocked`
3. If `totalOut` is greater than `_totalLocked`, the subtraction will underflow
4. The protocol fee is being double-counted (once in `totalOut` and once in separate transfer)

#### Impact Explanation
**Severity: High**

The impact is high because:
1. Redemptions can fail completely
2. Users cannot claim their redeemed tokens
3. Protocol fees are incorrectly calculated
4. The protocol's redemption mechanism is broken
5. Can lead to loss of user funds

#### Likelihood Explanation
**Likelihood: High**

This is highly likely to occur because:
1. It affects every redemption transaction
2. The issue is in the core redemption functionality
3. The POC demonstrates it's easily reproducible
4. No special conditions are needed to trigger it

#### Proof of Concept
```solidity
// add this test case to AlchemistV3.t.sol
for this test we need to active the `protocolFee` I have tested it with the `Fee = 2000`.  
function testRevertDueToFeeAdded() external {
    uint256 amount = 100e18;
    
    // Setup initial position
    vm.startPrank(address(0xbeef));
    SafeERC20.safeApprove(address(fakeYieldToken), address(alchemist), amount + 100e18);
    alchemist.deposit(amount, address(0xbeef), 0);
    uint256 tokenIdFor0xBeef = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
    alchemist.mint(tokenIdFor0xBeef, (amount / 2), address(0xbeef));
    vm.stopPrank();

    // Create redemption
    vm.startPrank(address(0xdad));
    SafeERC20.safeApprove(address(alToken), address(transmuterLogic), alchemist.totalSyntheticsIssued());
    transmuterLogic.createRedemption(alchemist.totalSyntheticsIssued());
    vm.stopPrank();

    // Wait for redemption period
    vm.roll(block.number + 5_256_000+2);

    // Attempt to claim redemption
    vm.startPrank(address(0xdad));
    vm.expectRevert();
    transmuterLogic.claimRedemption(1);
    vm.stopPrank();
}
forge test --mt testRevertDueToFeeAdded -vvv
```
#### Recommendation
The fix should separate the fee calculation from the redemption amount.


#### Related Files

- `src/AlchemistV3.sol` (lines 581-581)
- `src/AlchemistV3.sol` (lines 585-585)

---