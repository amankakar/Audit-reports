# USG - Tangent

## Protocol Summary

USG is a debt collateralized stablecoin backed by productive LPs and yield-bearing assets from blue chip DeFi protocols.
The focus of the audit is our lending mechanism, dynamic interest rate model, LP oracle, integrations with Curve, Convex, and Pendle, as well as reward streaming and zapping functionalities.

[Ethereum][Solidity][DeFi][Lending]

---

### [M-01] User will not be able to migrate Debt if deposit is paused

#### Summary

The migration process lets users move their collateral or debt across markets. However, if a user wants to migrate only debt while deposits on marketTo are paused even though borrowing is still allowed. the migration will fail and the user won’t be able to move their debt.

#### Root Cause

When migrating between markets, users can transfer either collateral or debt. However, the code enforces that deposits must not be paused even for debt-only migration, despite no collateral being moved.


```solidity
/tangent-contracts/src/USG/Market/abstract/MarketCore.sol:664  
664:     function _migrateTo(IControlTower _controlTower, address account, uint256 collatToAdd, uint256 debtToAdd) internal returns (uint256) {  
665:         _verifySenderMigrator(_controlTower);  
666:         _verifyIsDepositNotPaused();// @audit : check here if collatToAdd > 0 than the migrator will also deposit collateral  
```

[Here](https://github.com/sherlock-audit/2025-08-usg-tangent/blob/main/tangent-contracts/src/USG/Market/abstract/MarketCore.sol#L657)

#### Internal Pre-conditions

The deposits are paused on marketTo and Borrow is not paused.

#### External Pre-conditions

...

#### Attack Path

...

#### Impact

Users migrating only debt to a market with paused deposits face a DoS, leaving their existing position vulnerable to liquidation.

#### PoC

Add following test case to MigratePendlePTToPT.t.sol and run with command `forge test --mt test_migrate_revert_on_zero_deposit -vvv`:

```solidity
    function setUp() public {  
        marketFrom = deployBasicERC20Market(collatTokenFrom);  
        marketTo = deployBasicERC20Market(collatTokenFrom);  
  
        vm.startPrank(usr1);  
        deal(address(collatTokenFrom), usr1, collatIn);  
        collatTokenFrom.approve(address(marketFrom), MAX_UINT);  
        marketFrom.depositAndBorrow(collatIn, 50_000 ether);  
  
        swapParams.push(Array.memoryUint256([uint256(0), uint256(1), uint256(9), uint256(0), uint256(0)]));  
    }  
  
...  
  
function test_migrate_revert_on_zero_deposit() external {  
        deal(address(collatTokenFrom), usr1, collatIn);  
        collatTokenFrom.approve(address(marketTo), MAX_UINT);  
        marketTo.deposit(usr1, 50_000 ether);  
        vm.startPrank(address(pauser));  
        marketTo.setIsDepositPaused(true);  
        vm.stopPrank();  
        MigrateStruct memory migrateStruct = MigrateStruct({  
            markets: Array.memoryAddress([address(marketFrom), address(marketTo)]),  
            collatToWithdraw: 0, // @audit : collatToWithdraw set to 0 , which means we did not mve collateral so there will be no deposit on marketTo  
            debtToRemove: debtToRemove,  
            debtToRepay: debtToRepay  
        });  
  
        console.log("Debt before",marketTo.userDebt(usr1));  
  
        vm.startPrank(usr1);  
        migratoor.migrate(  
            migrateStruct,  
            ZapMigrateStruct({zap: ZapStruct({router: address(0), routerCall: ""}), minCollatToOut: 0})  
        );  
        console.log("debt After",marketTo.userDebt(usr1));  
        vm.stopPrank();  
    }  
```

output :

```
[FAIL: DepositPaused()] test_migrate_revert_on_zero_deposit() (gas: 620753)  
```

#### Mitigation

If collatToAdd > 0, enforce the deposit pause check; otherwise, skip it.

---

### [M-02] WStable:redeem Always Reverts When sUSDe is SavingAccount

#### Summary

...

#### Root Cause

sUSDe enforces a cooldown period before withdrawals:

```solidity
// sUSDe ensureCooldownOff modifier:  
  modifier ensureCooldownOff() {  
    if (cooldownDuration != 0) revert OperationNotAllowed();  
    _;  
  }  
```

Since WStable.redeem calls burn(..., false) → savingAccount.withdraw(...), the withdrawal from sUSDe will always revert due to the cooldown restriction.

```solidity
// sUSDe withdraw function  
  function withdraw(uint256 assets, address receiver, address _owner)  
    public  
    virtual  
    override  
    ensureCooldownOff  <<----@   
    returns (uint256)  
  {  
    return super.withdraw(assets, receiver, _owner);  
  }  
```

According to current configuration of sUSDe the cooldown duration is set to 1 WEEK , and if the withdraw amount is not cooldown the assets can not be withdraw so the redeem call will always revert because we set isSaving=false.


```solidity
/tangent-contracts/src/USG/Tokens/WStable.sol:105
 91:     function burn(uint256 amount, address receiver, bool isSaving) public {  
 92:         require(amount != 0, ZeroAmount());  
 93:   
 94:         if (isSaving) {  
 95:             IERC4626 _savingAccount = savingAccount;  
 96:             _savingAccount.transfer(receiver, _savingAccount.previewWithdraw(amount));  
 97:         } else {  
 98:             savingAccount.withdraw(amount, receiver, address(this));  
 99:         }  
100:   
101:         // Burn wStable from the sender  
102:         _burn(msg.sender, amount);  
103:     }  
104:   
  
105:     /**  
106:      *  @notice          Burns wStable for Stable. Uses the burn function  
107:      *  @dev             This function has been created to match the ERC4626 and to be used in the Curve Router  
108:      *  @param amount    Amount of WStable to burn in exchange of stable. Always at 1:1 ratio.  
109:      *  @param receiver  Receiver of the stable  
110:      *  @param owner     Param not used, keeped only to match the ERC4626 signature.  
111:      *  @return          Amount of WStable to burn and of Stable to receive.  
112:      */  
113:     function redeem(uint256 amount, address receiver, address owner) external returns (uint256) {  
114:         burn(amount, receiver, false);  
115:         return amount;  
116:     }  
```

[Here](https://github.com/sherlock-audit/2025-08-usg-tangent/blob/main/tangent-contracts/src/USG/Tokens/WStable.sol#L103)
and in side burn function if isSaving is false we will call withdraw function on underlying savingAccount.

So sUSDe is in scope for saving assets anf accroding to netspac the redeem function will called in case of This function has been created to match the ERC4626 and to be used in the Curve Router.

#### Internal Pre-conditions

sUSDe is configured as the saving account and cooldownDuration != 0 (currently set to 1 week).

#### External Pre-conditions

...

#### Attack Path

...

#### Impact

redeem always reverts when sUSDe is set as the savingAccount, causing a DoS: higher level integrations like Curve Router fail, and users are unable to redeem WStable into underlying assets.

#### PoC

For this test case you have to uncomment wUSDE.t.sol and add following test case:

```solidity
    function test_redeem() external {  
        vm.startPrank(usr1);  
        uint256 amountIn = 10_000 ether;  
        deal(address(stable), usr1, amountIn);  
        stable.approve(address(wUSDE), MAX_UINT);  
        vm.startSnapshotGas("WStable", "Mint wUSDE");  
        wUSDE.mint(amountIn,usr1, false);  
        vm.stopSnapshotGas();  
        vm.stopPrank();  
        assertEq(stable.balanceOf(usr1), 0);  
        assertEq(wUSDE.balanceOf(usr1), amountIn);  
        vm.startPrank(usr2);  
        deal(address(stable), usr2, amountIn);  
        stable.approve(address(wUSDE), MAX_UINT);  
        wUSDE.mint(amountIn,usr2, false);  
        assertEq(stable.balanceOf(usr2), 0);  
        assertEq(wUSDE.balanceOf(usr2), amountIn);  
        wUSDE.redeem(amountIn,usr2,usr2); // @audit : sUSDe redeem function is not correct so this line will always revert  
        vm.stopPrank();  
    }  
```

run with command : `forge test --mc wUSDE --mt test_mint -vvv`

output:

```
    │   ├─ [656] sUSDe::withdraw(10000000000000000000000 [1e22], User2: [0xf30B6147971ec7F782F0704aF06881B0790b2529], wUSDE: [0xF3574595B613bCb893e17B86B78Eb755c11C7B39])  
    │   │   └─ ← [Revert] custom error 0xf50a3b52  
    │   └─ ← [Revert] custom error 0xf50a3b52  
    └─ ← [Revert] custom error 0xf50a3b52  
```

#### Mitigation

Nil