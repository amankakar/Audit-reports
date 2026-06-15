
# Flare

## Protocol Summary

Flare is the blockchain for data. It is a layer-1, EVM smart contract platform designed to expand the utility of blockchain by delivering data certainty for dApp builders.

FAssets is a trustless, over-collateralized bridge built on Flare that connects non smart contract networks to Flare/Songbird. It enables the creation of wrapped tokens (FAssets) for assets like BTC, DOGE and XRP.

At the core of FAssets v1.1 is a new architecture component called the Core Vault, designed to improve system liquidity, scalability, and capital efficiency.


[Ethereum][Solidity][Crosschain][Staking]

---

## [M-01] Incorrect Minting Cap Check in Minting Process

### Target
https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/library/Minting.sol

### Impact(s)
Contract fails to deliver promised returns, but doesn't lose value

## Brief/Intro
The `checkMintingCap` function in `Minting.sol` only checks the valueAMG against the minting cap, but the actual minting process adds both valueAMG and `poolFeeAMG` to the agent's minted amount. This discrepancy allows the total minted amount to exceed the intended minting cap.

## Vulnerability Details
The vulnerable code is in Minting.sol:

```solidity
/fassets/contracts/assetManager/library/Minting.sol:75
 75:     function selfMint(
 76:         IPayment.Proof calldata _payment,
 77:         address _agentVault,
 78:         uint64 _lots
 79:     )
 80:         internal
 81:     {
.....
 90:         require(collateralData.freeCollateralLots(agent) >= _lots, "not enough free collateral");
 91:         uint64 valueAMG = _lots * Globals.getSettings().lotSizeAMG;
 92:         checkMintingCap(valueAMG); <----@
107:         if (_lots > 0) {
108:             _performMinting(agent, MintingType.SELF_MINT, 0, msg.sender, valueAMG, receivedAmount, poolFeeUBA);
109:         } else {

```

`_performMinting` function :

```solidity
/fassets/contracts/assetManager/library/Minting.sol:189
189:     function _performMinting(
190:         Agent.State storage _agent,
191:         MintingType _mintingType,
192:         uint64 _crtId,
193:         address _minter,
194:         uint64 _mintValueAMG,
195:         uint256 _receivedAmountUBA,
196:         uint256 _poolFeeUBA
197:     )
198:         private
199:     {
200:         uint64 poolFeeAMG = Conversion.convertUBAToAmg(_poolFeeUBA);
201:         Agents.createNewMinting(_agent, _mintValueAMG + poolFeeAMG); <----@ we add both value
```

Here we only check the `valueAMG` not the Fee which will also be added to `agent.mintedAMG`

```solidity
/fassets/contracts/assetManager/library/Minting.sol:139
139:     function checkMintingCap(
140:         uint64 _increaseAMG
141:     )
142:         internal view
143:     {
144:         AssetManagerState.State storage state = AssetManagerState.get();
145:         AssetManagerSettings.Data storage settings = Globals.getSettings();
146:         uint256 mintingCapAMG = settings.mintingCapAMG;
147:         if (mintingCapAMG == 0) return;     // minting cap disabled
148:         uint256 totalMintedUBA = IERC20(settings.fAsset).totalSupply();
149:         uint256 totalAMG = state.totalReservedCollateralAMG + Conversion.convertUBAToAmg(totalMintedUBA);
150:         require(totalAMG + _increaseAMG <= mintingCapAMG, "minting cap exceeded");
151:     }
```

## Impact Details
- Severity: Low
- Impact: Allows total minted amount to exceed the minting cap by the amount of pool fees
- Scope: Affects all minting operations in the system either via reserveCollateral, mintFromFreeUnderlying and selfMint

## References
File: contracts/assetManager/library/Minting.sol
Function: `checkMintingCap`
Function: `_performMinting`
Fix the minting cap check to include pool fees:
```solidity
function checkMintingCap(
    uint64 _valueAMG,
    uint64 _poolFeeAMG
)
    internal view
{
    AssetManagerState.State storage state = AssetManagerState.get();
    AssetManagerSettings.Data storage settings = Globals.getSettings();
    uint256 mintingCapAMG = settings.mintingCapAMG;
    if (mintingCapAMG == 0) return;     // minting cap disabled
    uint256 totalMintedUBA = IERC20(settings.fAsset).totalSupply();
    uint256 totalAMG = state.totalReservedCollateralAMG + Conversion.convertUBAToAmg(totalMintedUBA);
    require(totalAMG + _valueAMG + _poolFeeAMG <= mintingCapAMG, "minting cap exceeded");
}
```
Update all calls to `checkMintingCap` to include pool fees

## Proof of Concept
1. The `MintingCapAMG` is set to `1000`.
2. An agent attempts to mint `valueAMG` of `1000`.
3. The PoolFeeAMG is calculated as 100 (10% of the minting amount).
4. The code only checks the valueAMG against the minting cap, so the check require(totalAMG + _increaseAMG <= mintingCapAMG, "minting cap exceeded") passes because 0 + 1000 <= 1000.
5. After the check, the system increases the agent's mintedAMG by _valueAMG, where _valueAMG = valueAMG + feeAMG.
6. The final value of agent.MintedAMG becomes 1100 (1000 + 100).
7. This demonstrates that despite the minting cap being set to 1000, the agent successfully minted 1100 AMG, exceeding the intended cap by 100 AMG. For simplicity, decimal and dust calculations have been ignored in this example.

---

## [M-02] Wrong check in `redeemFromCoreVault` will result in unnecessary revert

### Target
https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/library/CoreVault.sol

### Impact(s)
- Temporary freezing of funds for at least 1 hour
- Temporary freezing of funds for at least 24 hour
- Contract fails to deliver promised returns, but doesn't lose value

## Brief/Intro
The `CoreVault` redemption process checks available lots before fee deduction, which can cause unnecessary reverts. When a user requests redemption, the system checks if there are enough available lots in `CoreVault` before deducting the redemption fee. This is inefficient because after fee deduction, the actual amount needed would be less, potentially allowing the redemption to proceed. This leads to unnecessary redemption failures.

## Vulnerability Details
The issue occurs in the `redeemFromCoreVault` function where the available lots check happens before fee deduction:

```solidity
function redeemFromCoreVault(
    uint64 _lots,
    string memory _redeemerUnderlyingAddress
) internal {
    // Check available lots before fee deduction
    uint64 availableLots = getCoreVaultAmountLots();
@>    require(_lots <= availableLots, "not enough available on core vault"); // @audit: premature check

    // ... other checks ...

    // Fee deduction happens after the check
    uint256 redeemedUBA = Conversion.convertAmgToUBA(redeemedAMG);
    uint256 redemptionFeeUBA = redeemedUBA.mulBips(state.redemptionFeeBIPS);
    uint128 paymentUBA = (redeemedUBA - redemptionFeeUBA).toUint128();

    // Actual transfer request uses reduced amount
    paymentReference = state.coreVaultManager.requestTransferFromCoreVault(
        _redeemerUnderlyingAddress, paymentReference, paymentUBA, false);
}
```

**The problem is that the system:**

- Checks available lots against full redemption amount
- Deducts fee after the check

## Impact Details
**The impact is significant because:**
- Users cannot redeem when they should be able to
- Unnecessary transaction failures

## References
`CoreVault.sol::redeemFromCoreVault`

## Mitigation :
The Best Fix would be to apply the following changes:

```diff
diff --git a/contracts/assetManager/library/CoreVault.sol b/contracts/assetManager/library/CoreVault.sol
index 89cb6188..b9e440e9 100644
--- a/contracts/assetManager/library/CoreVault.sol
+++ b/contracts/assetManager/library/CoreVault.sol
@@ -218,16 +218,17 @@ library CoreVault {
             "underlying address not allowed by core vault");
         AssetManagerSettings.Data storage settings = Globals.getSettings();
         uint64 availableLots = getCoreVaultAmountLots();
-        require(_lots <= availableLots, "not enough available on core vault");
         uint64 minimumRedeemLots = SafeMath64.min64(state.minimumRedeemLots, availableLots);
         require(_lots >= minimumRedeemLots, "requested amount too small");
         // burn the senders fassets
         uint64 redeemedAMG = _lots * settings.lotSizeAMG;
         uint256 redeemedUBA = Conversion.convertAmgToUBA(redeemedAMG);
-        Redemptions.burnFAssets(msg.sender, redeemedUBA);
-        // subtract the redemption fee
         uint256 redemptionFeeUBA = redeemedUBA.mulBips(state.redemptionFeeBIPS);
+        // subtract the redemption fee
         uint128 paymentUBA = (redeemedUBA - redemptionFeeUBA).toUint128();
+        uint64 _lotsConsumed  = Conversion.convertUBAToAmg(paymentUBA)/settings.lotSizeAMG;
+        require(_lotsConsumed <= availableLots, "not enough available on core vault"); // The correct check would be that after fee deduction check that we have this much fund available
+        Redemptions.burnFAssets(msg.sender, redeemedUBA);
```

## Proof of Concept
Add the Following test case to `CoreVault.ts` and run with command yarn `testHH`:

```javascript
it.only("Available lots checking before fee deduction will reslt in unnecessary revert", async () => {
        const agent = await Agent.createTest(context, agentOwner1, underlyingAgent1);
        const minter = await Minter.createTest(context, minterAddress1, underlyingMinter1, context.underlyingAmount(10000000));
        const redeemer = await Redeemer.create(context, redeemerAddress1, underlyingRedeemer1);
        let agentInfo1 = await agent.getAgentInfo();

        await prefundCoreVault(minter.underlyingAddress, 1e6);
        // allow CV manager addresses
        await coreVaultManager.addAllowedDestinationAddresses([redeemer.underlyingAddress], { from: governance });
        // make agent available
        await agent.depositCollateralLotsAndMakeAvailable(101);
        // mint
        const [minted] = await minter.performMinting(agent.vaultAddress, 101);
        await minter.transferFAsset(redeemer.address, minted.mintedAmountUBA);
        // agent requests transfer for some backing to core vault
        const transferAmount = context.convertLotsToUBA(100);
        await agent.transferToCoreVault(transferAmount);
        agentInfo1 = await agent.getAgentInfo();
        console.log("minted1 | redeeming1");
        console.log(agentInfo1.mintedUBA, agentInfo1.redeemingUBA);

        const lots = 101;
        // redeemer requests direct redemption from CV
        await expectRevert(context.assetManager.redeemFromCoreVault(lots, redeemer.underlyingAddress, { from: redeemer.address }),
            "not enough available on core vault");
    });
```

The above test case will revert , But after apply the fix than run the 2nd test case it will be executed successfully
```javascript
    it.only("Available lots checking before fee deduction will executed", async () => {
        const agent = await Agent.createTest(context, agentOwner1, underlyingAgent1);
        const minter = await Minter.createTest(context, minterAddress1, underlyingMinter1, context.underlyingAmount(10000000));
        const redeemer = await Redeemer.create(context, redeemerAddress1, underlyingRedeemer1);
        let agentInfo1 = await agent.getAgentInfo();

        await prefundCoreVault(minter.underlyingAddress, 1e6);
        // allow CV manager addresses
        await coreVaultManager.addAllowedDestinationAddresses([redeemer.underlyingAddress], { from: governance });
        // make agent available
        await agent.depositCollateralLotsAndMakeAvailable(101);
        // mint
        const [minted] = await minter.performMinting(agent.vaultAddress, 101);
        await minter.transferFAsset(redeemer.address, minted.mintedAmountUBA);
        // agent requests transfer for some backing to core vault
        const transferAmount = context.convertLotsToUBA(100);
        await agent.transferToCoreVault(transferAmount);
        agentInfo1 = await agent.getAgentInfo();
        console.log("minted1 | redeeming1");
        console.log(agentInfo1.mintedUBA, agentInfo1.redeemingUBA);

        const lots = 101;
        // redeemer requests direct redemption from CV
        const [paymentAmount1, paymentReference1] = await testRedeemFromCV(redeemer, lots);
        const trigRes = await coreVaultManager.triggerInstructions({ from: triggeringAccount });
        const paymentReqs = filterEvents(trigRes, "PaymentInstructions");
        assertWeb3Equal(paymentReqs[0].args.account, coreVaultUnderlyingAddress);
        assertWeb3Equal(paymentReqs[0].args.destination, redeemer.underlyingAddress);
        assertWeb3Equal(paymentReqs[0].args.amount, paymentAmount1);
        agentInfo1 = await agent.getAgentInfo();
        console.log("minted1 | redeeming1");
        console.log(agentInfo1.mintedUBA, agentInfo1.redeemingUBA);
        const { 0: immediatelyAvailable, 1: totalAvailable } = await context.assetManager.coreVaultAvailableAmount();
        console.log("totalAvailable" , totalAvailable.toString())
        let AvailableLots = Number(context.convertUBAToAmg(totalAvailable)) / Number(context.settings.lotSizeAMG);
        console.log("AvailableLots" , AvailableLots)

    });

```
---

## [L-01] Incorrect `msg.Value` check in `CoreVault` Transfer

### Target
https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/library/CoreVault.sol

### Impact(s)
- Permanent freezing of funds

## Brief/Intro
The `transferToCoreVault` function in `CoreVault.sol` incorrectly assumes that the `TRANSFER_GAS_ALLOWANCE` should be retained by the contract, when in fact the gas costs are paid by the agent `(msg.sender)` directly. This leads to unnecessary retention of funds that should be returned to the agent.

## Vulnerability Details
The vulnerable code is in `CoreVault.sol`:

```solidity
if (msg.value > transferFeeWei + Transfers.TRANSFER_GAS_ALLOWANCE) {
    Transfers.transferNAT(state.nativeAddress, transferFeeWei);
    Transfers.transferNATAllowFailure(payable(msg.sender), msg.value - transferFeeWei);
}
```

The issue is that the code:

1. Checks if `msg.value` is greater than `transferFeeWei + TRANSFER_GAS_ALLOWANCE`
2. But the `TRANSFER_GAS_ALLOWANCE` is unnecessary because:
    - Gas costs are paid by the agent `(msg.sender)` directly through the transaction
    - The contract doesn't need to retain any additional gas allowance
    - The only fee that should be retained is the `transferFeeWei`

## Impact Details
- Severity: Medium
- Impact: Unnecessarily retains funds that should be returned to the agent
- Likelihood: High - Affects every transfer to core vault where msg.value > transferFeeWei and msg.value <= transferFeeWei + TRANSFER_GAS_ALLOWANCE

## References
- File: `contracts/assetManager/library/CoreVault.sol`
- Function: `transferToCoreVault`

## Recommendations
Remove the `TRANSFER_GAS_ALLOWANCE` check since gas costs are paid by the `msg.sender`:

```solidity
if (msg.value > transferFeeWei) {
    Transfers.transferNAT(state.nativeAddress, transferFeeWei);
    Transfers.transferNATAllowFailure(payable(msg.sender), msg.value - transferFeeWei);
}
```

## Proof of Concept
To proof my point I have added few changed to `CoreVault.sol` file :

```diff
+    uint256 balNativeAddressBefore=  payable(address(state.nativeAddress)).balance;
        if (msg.value > transferFeeWei + Transfers.TRANSFER_GAS_ALLOWANCE) {
            Transfers.transferNAT(state.nativeAddress, transferFeeWei);
            Transfers.transferNATAllowFailure(payable(msg.sender), msg.value - transferFeeWei);
        } else {
            Transfers.transferNAT(state.nativeAddress, msg.value);
        }
+        uint256 balNativeAddressAfter=  payable(address(state.nativeAddress)).balance;
+        console.log("state.nativeAddress balance After balance " , balNativeAddressAfter - balNativeAddressBefore - transferFeeWei);
```
And also update the Agent.ts:
```diff
diff --git a/test/integration/utils/Agent.ts b/test/integration/utils/Agent.ts
index 02980151..6ae8697a 100644
--- a/test/integration/utils/Agent.ts
+++ b/test/integration/utils/Agent.ts
@@ -529,7 +537,7 @@ export class Agent extends AssetContextClient {

     async transferToCoreVault(transferAmount: BNish) {
         const cbTransferFee = await this.assetManager.transferToCoreVaultFee(transferAmount);
-        const res = await this.assetManager.transferToCoreVault(this.vaultAddress, transferAmount, { from: this.ownerWorkAddress, value: cbTransferFee });
+        const res = await this.assetManager.transferToCoreVault(this.vaultAddress, transferAmount, { from: this.ownerWorkAddress, value: cbTransferFee.add(toBN(100_000)) });
         const rdreqs = filterEvents(res, "RedemptionRequested").map(evt => evt.args);
         assert.isAtLeast(rdreqs.length, 1);
And than add following test case to CoreVault.ts file:

    it.only("extra amount not return correctly", async () => {
        const agent = await Agent.createTest(context, agentOwner1, underlyingAgent1);
        const agent2 = await Agent.createTest(context, agentOwner2, underlyingAgent2);
        const minter = await Minter.createTest(context, minterAddress1, underlyingMinter1, context.underlyingAmount(10000000));
        const redeemer = await Redeemer.create(context, redeemerAddress1, underlyingRedeemer1);
        let agentInfo1 = await agent.getAgentInfo();

        await prefundCoreVault(minter.underlyingAddress, 1e6);
        // allow CV manager addresses
        await coreVaultManager.addAllowedDestinationAddresses([redeemer.underlyingAddress], { from: governance });
        // make agent available
        await agent.depositCollateralLotsAndMakeAvailable(50);

        // mint
        let [minted] = await minter.performMinting(agent.vaultAddress, 50);


        await minter.transferFAsset(redeemer.address, minted.mintedAmountUBA);
        // agent requests transfer for some backing to core vault
        const transferAmount = context.convertLotsToUBA(49);
        await agent.transferToCoreVault(transferAmount);
    });
```
Run with command `yarn testHH` console.logs of this test :
```bash
state.nativeAddress balance After balance  100000 # extra amount nativeAddress receives
```
---

## [I-01] Incorrect Minimum Lots Validation in CoreVault Redemption

### Target
https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/library/CoreVault.sol

### Impact(s)
- Contract fails to deliver promised returns, but doesn't lose value
- Documentation Improvements

## Brief/Intro
The redeemFromCoreVault function in `CoreVault.sol` incorrectly validates the minimum lots requirement when the available lots in the core vault are less than the `coreVaultMinimumRedeemLots` setting. This allows redeemers to create redemption requests with fewer lots than the minimum requirement.

## Vulnerability Details
The vulnerable code is in `CoreVault.sol`:

```solidity
function redeemFromCoreVault(
    uint64 _lots,
    string memory _redeemerUnderlyingAddress
)
    internal
    onlyEnabled
{
    State storage state = getState();
    require(state.coreVaultManager.isDestinationAddressAllowed(_redeemerUnderlyingAddress),
        "underlying address not allowed by core vault");
    AssetManagerSettings.Data storage settings = Globals.getSettings();
    uint64 availableLots = getCoreVaultAmountLots();
    uint64 minimumRedeemLots = SafeMath64.min64(state.minimumRedeemLots, availableLots);
    require(_lots >= minimumRedeemLots, "requested amount too small");
    // ... rest of the function
}
```
The issue is that the code:

1. Gets the available lots in the core vault
2. Sets `minimumRedeemLots` as the minimum of `state.minimumRedeemLots` and `availableLots`
3. This means if `availableLots` is less than `minimumRedeemLots`, the requirement is lowered

This contradicts the documentation which states that lots must be larger than `coreVaultMinimumRedeemLots`

## Impact Details
- Impact: Allows redemption requests with fewer lots than the minimum requirement
- Likelihood: Low - Only occurs when core vault has insufficient funds

## References
- File: `contracts/assetManager/library/CoreVault.sol`
- Function: `redeemFromCoreVault`

## Recommendations
Fix the minimum lots validation to always enforce the minimum requirement:

```solidity
function redeemFromCoreVault(
    uint64 _lots,
    string memory _redeemerUnderlyingAddress
)
    internal
    onlyEnabled
{
    State storage state = getState();
    require(state.coreVaultManager.isDestinationAddressAllowed(_redeemerUnderlyingAddress),
        "underlying address not allowed by core vault");
    require(_lots >= state.minimumRedeemLots, "requested amount too small");
    uint64 availableLots = getCoreVaultAmountLots();
    require(_lots <= availableLots, "not enough available on core vault");
    // ... rest of the function
}
```
## Proof of Concept
Add the Following test Case to `CoreVault.ts` :

```javascript
it.only("request direct redemption from core vault less than min Redeem lots", async () => { // @audit-issue : redeem less than min lots
        const agent = await Agent.createTest(context, agentOwner1, underlyingAgent1);
        const minter = await Minter.createTest(context, minterAddress1, underlyingMinter1, context.underlyingAmount(10000000));
        const redeemer = await Redeemer.create(context, redeemerAddress1, underlyingRedeemer1);
        await prefundCoreVault(minter.underlyingAddress, 1e6);
        // allow CV manager addresses
        await coreVaultManager.addAllowedDestinationAddresses([redeemer.underlyingAddress], { from: governance });
        // make agent available
        await agent.depositCollateralLotsAndMakeAvailable(100);
        // mint
        const [minted] = await minter.performMinting(agent.vaultAddress, 2);
        await minter.transferFAsset(redeemer.address, minted.mintedAmountUBA);
        // agent requests transfer for some backing to core vault
        const transferAmount = context.convertLotsToUBA(2);
        await agent.transferToCoreVault(transferAmount);
        let res = await context.assetManager.setCoreVaultMinimumRedeemLots(3, { from: context.governance });
        expectEvent(res, "SettingChanged", { name: "coreVaultMinimumRedeemLots", value: "3" })
        const minLots = await context.assetManager.getCoreVaultMinimumRedeemLots();
        assertWeb3Equal(minLots, 3);
        const lots = 2;
        // redeemer requests direct redemption from CV
        const [paymentAmount1, paymentReference1] = await testRedeemFromCV(redeemer, lots);
        const trigRes = await coreVaultManager.triggerInstructions({ from: triggeringAccount });
        const paymentReqs = filterEvents(trigRes, "PaymentInstructions");
        assertWeb3Equal(paymentReqs[0].args.account, coreVaultUnderlyingAddress);
        assertWeb3Equal(paymentReqs[0].args.destination, redeemer.underlyingAddress);
        assertWeb3Equal(paymentReqs[0].args.amount, paymentAmount1);
        expect(toBN(minLots).gt(toBN(lots))).to.be.true;
    });
```
Run with command `yarn testHH`.
