# symbioticfi-core

## Protocol Summary
Symbiotic is a shared security protocol enabling decentralized networks to control and customize their own multi-asset restaking implementation.

[Ethereum][Solidity][DeFi][re-staking]


### [M-01] A re-org will disrupt the Veto Slashing process.

#### Summary
The protocol allows slashing to be vetoed through a resolver, with each slash request indexed by its position in an array. However, a re-org attack could disrupt this process, leading to incorrect vetoing or causing the slash execution to revert.


#### Finding Description

##### Slashing Mechanism
Slashing can occur in two ways:

1. **Instant Slashing**: The slashing is executed immediately.
2. **Veto Slashing**: 

   - In this case, the network middleware first submits a slashing request.
   - After submission, the request can either be vetoed by the resolver or, if the veto period expires, executed by the network middleware.


The issue lies in how request storage is implemented. Each request is stored at a specific index in an array, and later retrieved using the index assigned during its creation.

```solidity
slasher/VetoSlasher.sol:108
108:         slashIndex = slashRequests.length; 
109:         slashRequests.push(
110:             SlashRequest({
111:                 subnetwork: subnetwork,
112:                 operator: operator,
113:                 amount: amount,
114:                 captureTimestamp: captureTimestamp,
115:                 vetoDeadline: vetoDeadline,
116:                 completed: false
117:             })
118:         );
119: 
```
The `slashIndex` indexed is then used to perform futher opertion on this slash request. it could be either vetoed or completed.

```solidity
slasher/VetoSlasher.sol:133
133:     function executeSlash(
134:         uint256 slashIndex,
135:         bytes calldata hints
136:     ) external initialized nonReentrant returns (uint256 slashedAmount) {
137:         ExecuteSlashHints memory executeSlashHints;
138:         if (hints.length > 0) {
139:             executeSlashHints = abi.decode(hints, (ExecuteSlashHints));
140:         }
141: 
142:         if (slashIndex >= slashRequests.length) {
143:             revert SlashRequestNotExist();
144:         }
145: 
146:         SlashRequest storage request = slashRequests[slashIndex];
```
In case of execution we provide this `slashIndex` and fetch the request data from `slashRequests` and the same is done In case of veto .

```solidity
slasher/VetoSlasher.sol:198
198:     function vetoSlash(uint256 slashIndex, bytes calldata hints) external initialized nonReentrant {
199:         VetoSlashHints memory vetoSlashHints;
200:         if (hints.length > 0) {
201:             vetoSlashHints = abi.decode(hints, (VetoSlashHints));
202:         }
203: 
204:         if (slashIndex >= slashRequests.length) {
205:             revert SlashRequestNotExist();
206:         }
207: 
208:         SlashRequest storage request = slashRequests[slashIndex];
209: 
```

#### Impact Explanation
- The `executeSlash` function call may either revert or complete the wrong slash request, which the network middleware did not intend to execute.
- The `vetoSlash` function will revert if the current caller is not the resolver for the specified subnetwork in the request data or may end up vetoing a legitimate slash request instead of the incorrect one.

#### Likelihood Explanation
A 1-block re-org is common in Ethereum. If the slasher receives two slash requests from two different networks  and a re-org occurs, one of the above scenarios is highly likely to happen.

#### Proof of Concept (if required)
1. In **Block 1**, **Network A** submits a slash request for **Vault A**, which is stored with `slashIndex = 0`.
2. The resolver vetoes the slash request from **Network A**.
3. In **Block 2**, **Network B** submits a slash request for **Vault A**, which is stored with `slashIndex = 1`.
4. A **block re-org** occurs, causing **Block 1** to be dropped and replaced by **Block 2**. As a result, **Network B's** slash request is now assigned `slashIndex = 0`.
5. Finally, **Network B's** request is executed and slashes **Vault A**, even though it was not supposed to be slashed.

**Note**: Both **Network A** and **Network B** share the same resolver address. The same issue can occur during the request completion process.

#### Recommendation (optional)

Store the hash of request id with the hash of its content and verify it inside the `vetoSlash` and `executeSlash`. 


#### Related Files

- `src/contracts/slasher/VetoSlasher.sol` (lines 108-108)
- `src/contracts/slasher/VetoSlasher.sol` (lines 202-202)

---

### [L-01] Lack of Enforcement of Epoch Duration in Vault Allows Slashing Beyond Defined Constraints

#### Summary
The defined constraint for slashing is that it should only occur if the capture time is not behind by more than one epoch duration. However, this condition is not enforced inside the `vault::onslash` function.


#### Finding Description
The Vault can choose not to use the Slasher module, as it is an additional component and not part of the core contract. Therefore, the core contract itself must enforce all necessary constraints within its functions if it is intended to receive calls from other contracts outside the core module. The Vault can also integrate its own Slasher contract, which is evident from the deployment of the `vaultConfigurator` contract.


```solidity
    function create(
        InitParams memory params
    ) public returns (address, address, address) {
        if (params.vaultParams.delegator != address(0) || params.vaultParams.slasher != address(0)) {
            revert DirtyInitParams();
        }

        address vault =
            VaultFactory(VAULT_FACTORY).create(params.version, params.owner, false, abi.encode(params.vaultParams));

        params.vaultParams.delegator = DelegatorFactory(DELEGATOR_FACTORY).create(
            params.delegatorIndex, true, abi.encode(vault, params.delegatorParams)
        );

        bytes memory slasherData;
        if (params.withSlasher) {
            slasherData = abi.encode(vault, params.slasherParams);
            params.vaultParams.slasher = SlasherFactory(SLASHER_FACTORY).create(params.slasherIndex, false, slasherData);
        }

        Vault(vault).initialize(params.version, params.owner, abi.encode(params.vaultParams));

        if (params.withSlasher) {
            BaseSlasher(params.vaultParams.slasher).initialize(slasherData);
        }

        return (vault, params.vaultParams.delegator, params.vaultParams.slasher);
    }
```

To avoid deploying the Vault with the symbiotic Slasher module and instead use a different Slasher contract, you only need to provide the address of the new Slasher contract and set `params.withSlasher=false`.

##### The following diagram illustrates the scenario:

 ![Screenshot from 2024-10-01 10-37-17.png](https://imagedelivery.net/wtv4_V7VzVsxpAFaxzmpbw/9b7b3bba-d395-49aa-cb19-dd8a5955c500/public) 

#### Impact Explanation
If this occur the impact will be that the slash will be executed behind the epoch duration.

#### Likelihood Explanation
It is Low.

#### Proof of Concept (if required)
For simplicity, we will use the following small values:
- `epochDuration = 60 sec`
- `initTime = 120 sec`
- `captureTime = 150 sec`
- `slashExecutionTime = 230 sec`

##### Explanation

The difference between `captureTime` and `slashExecutionTime` is 80 seconds, which exceeds 1 epoch duration. However, due to the rounding down in epoch calculations, the slash will still be executed successfully.

##### Epoch Calculation

- **For current epoch:**
  
  ```solidity
  (Time.timestamp() - epochDurationInit) / epochDuration
  => (230 - 120) / 60 = 1.8, which rounds down to 1.
  ```
- **For Capture Time epoch:**

```solidity
(timestamp - epochDurationInit) / epochDuration
=> (150 - 120) / 60 = 0.5, which rounds down to 0.
```
With the above values, the condition inside the contract will pass:

```solidity
vault/Vault.sol:223
if ((currentEpoch_ > 0 && captureEpoch < currentEpoch_ - 1) || captureEpoch > currentEpoch_) {
    revert InvalidCaptureEpoch();
}
```
In this case:

- **currentEpoch_ is** : 1
`**captureEpoch is** : 0
The condition is satisfied, and the slash will be executed successfully, despite the time difference being larger than 1 epoch duration.

#### Recommendation (optional)
Add the following sanity check to the `vault::onSlash` function:
```diff
diff --git a/src/contracts/vault/Vault.sol b/src/contracts/vault/Vault.sol
index 72ece98..7ea9c97 100644
--- a/src/contracts/vault/Vault.sol
+++ b/src/contracts/vault/Vault.sol
@@ -216,7 +217,7 @@ contract Vault is VaultStorage, MigratableEntity, AccessControlUpgradeable, IVau
         if (msg.sender != slasher) {
             revert NotSlasher();
         }
-
+        if(Time.timestamp() - captureTimestamp > epochDuration)  revert WithError();

```


#### Related Files

- `src/contracts/vault/Vault.sol` (lines 222-222)
