# DittoETH

## Protocol Summary
The system mints pegged assets (stablecoins) using an orderbook, using over-collateralized staked ETH.

[Ethereum][Solidity][DeFi][OrderBook]

### [H-01] The ownership of the NFT remains claimable even after the short record has been deleted.            

#### Summary
The protocol does support NFT for shot record holders. However, a vulnerability has been identified in the exitShortx function.

#### Vulnerability Details
Upon reviewing the ExitShortx functions, it has been noted that the code does not verify whether an NFT has been minted. Furthermore, if an NFT is indeed minted, the code does not proceed to revoke or eliminate the user's ownership of that NFT.

```git
  function test_NFTOwnerShipAfterExistShortRecord() public {
        vm.prank(address(diamond));
            token.mint(sender, DEFAULT_AMOUNT);
        createShortAndMintNFT();
        console.log(diamond.ownerOf(1));
        exitShortWallet(Constants.SHORT_STARTING_ID, DEFAULT_AMOUNT, sender);
        assertEq(getShortRecordCount(sender), 0);
        // the owner must not be sender after exit shor record.
        assertEq(diamond.ownerOf(1), address(sender));

 }
```
#### Impact
The current impact is relatively minimal. However, it could potentially result in undefined or unpredictable ownership of NFTs in the future. however we are aware of the fact that the user still can't transfer this NFT.
#### Tools Used
Manual review
#### Recommendations
The system incorporates a 'burnNFT' function, which is designed to eliminate the ownership record of a NFT and its associated user. It is advisable to invoke this function only if a complete record exists and the user possesses the NFT. The following code snippet is provided for your reference:
```js
+            // @audit : check if have NFT minted them call LibShortRecord.burnNFT()
+            if(s.nftMapping[short.tokenId].owner != address(0)){
+            LibShortRecord.burnNFT(short.tokenId);
+            }
             LibShortRecord.deleteShortRecord(asset, msg.sender, id);
```
    
---

### [L-01] mintNFT Function Lacks ERC721Receiver Check            

#### Summary
The user is able to mint NFt for there short  record. however whi.e obervating the code the folloing vulberablity has been found.

#### Vulnerability Details
The audit of the shortRecord's NFT minting process revealed that the mintNFT(address asset, uint256 shortRecordId) function within the ERC721Facet contract lacks a necessary ERC721 receivable check. This omission raises concerns about potential security vulnerabilities, including the risk of NFT loss

#### POC
```js    
// this function would create short and mint NFT
    function createShortAndMintNFTToNonERC721Receipient(address nonNft) public {
        assertEq(diamond.balanceOf(address(nonNft)), 0);
        assertEq(diamond.getTokenId(), 1);
        fundLimitBidOpt(DEFAULT_PRICE, DEFAULT_AMOUNT, receiver);
        fundLimitShortOpt(DEFAULT_PRICE, DEFAULT_AMOUNT, address(nonNft));
        assertEq(diamond.balanceOf(address(nonNft)), 0);
        vm.prank(address(nonNft));
        diamond.mintNFT(asset, Constants.SHORT_STARTING_ID);
        assertEq(diamond.getTokenId(), 2);
        assertEq(diamond.balanceOf(address(nonNft)), 1);
        assertEq(diamond.ownerOf(1), address(nonNft));

        //@dev give extra an initial short to test that shortRecordId changes appropriately when
        fundLimitBidOpt(DEFAULT_PRICE, DEFAULT_AMOUNT, receiver);
        fundLimitShortOpt(DEFAULT_PRICE, DEFAULT_AMOUNT, extra);
    }

function test_MintToNonERC721Recipient() public {
        address nonRecipient = address(new NonERC721Recipient());

        createShortAndMintNFTToNonERC721Receipient(nonRecipient);
        // address nonRecipient = address(new NonERC721Recipient());
        assertEq(diamond.balanceOf(nonRecipient), 1);
    } 
```


#### Impact
The NFT could be lose, No Safety and Compatibility with ERC721 standard.

#### Tools Used
Manual review
#### Recommendations
Either add SafeMint Function or add following check in side mint function:
```js
if (!_checkOnERC721Received(address(this), msg.sender, s.tokenIdCounter, "")) {
             revert Errors.ERC721InvalidReceiver(msg.sender);
         }
```


