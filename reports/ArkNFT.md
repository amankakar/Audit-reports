# ArkProject: NFT Bridge 

## Protocol Summary

ArkProject offers a liquidity layer for digital assets: uniting markets, empowering creators, bridging the gap to mass adoption, built on top of Starknet.


[Ethereum][Starknet][Solidity][Cario][NFT]


### [H-01] `_white_list_collection` and `_whiteListCollection` does not check that whitelisting is enabled could lead to DoS            


#### Summary

The Layer 2 bridge contract will whitelist a contract if it is not already whitelisted when `withdraw_auto_from_l1` is called. However, it does not check whether whitelisting is enabled.

This issue affects the StarkNet contract, but it also impacts the Ethereum contract.

#### Vulnerability Details

The `_white_list_collection` function can be triggered by an admin or when the bridge contract on Layer 2 receives calls from `l1_handler`. However, it lacks a check to verify whether whitelisting is enabled.

```javascript
    fn _white_list_collection(ref self: ContractState, collection: ContractAddress, enabled: bool) {
        let no_value = starknet::contract_address_const::<0>();
@>        let (current, _) = self.white_listed_list.read(collection); 
        if current != enabled {
            let mut prev = self.white_listed_head.read();
            if enabled {
                self.white_listed_list.write(collection, (enabled, no_value));
                if prev.is_zero() {
                    self.white_listed_head.write(collection);
                    return;
                }
                // find last element
                loop {
                    let (_, next) = self.white_listed_list.read(prev);
                    if next.is_zero() {
                        break;
                    }
                    let (active, _) = self.white_listed_list.read(next);
                    if !active {
                        break;
                    }
                    prev = next;
                };
                self.white_listed_list.write(prev, (true, collection));
            } else { 
                // change head
                if prev == collection {
                    let (_, next) = self.white_listed_list.read(prev);
                    self.white_listed_list.write(collection, (false, no_value));
                    self.white_listed_head.write(next);
                    return;
                }
                // removed element from linked list
                loop {
                    let (active, next) = self.white_listed_list.read(prev);
                    if next.is_zero() {
                        // end of list
                        break;
                    }
                    if !active {
                        break;
                    }
                    if next == collection {
                        let (_, target) = self.white_listed_list.read(collection);
                        self.white_listed_list.write(prev, (active, target));
                        break;
                    }
                };
                self.white_listed_list.write(collection, (false, no_value));
            }
        }
    }
```

In the above code, we only check if the collection is whitelisted. If it is not, we whitelist it. However, the intended behavior is that if whitelisting is not enabled, any token should be allowed to be bridged.

```javascript
fn _is_white_listed(self: @ContractState, collection: ContractAddress) -> bool {
            let enabled = self.white_list_enabled.read();
            if (enabled) {
                let (ret, _) = self.white_listed_list.read(collection);
                return ret;
            }
            true
    }
```

From the above function, it can be seen that if whitelisting is not enabled, we return `true`.

#### Impact

1. The collection is added to the whitelist when whitelisting is not enabled, which is not the intended behavior according to the given code.
2. Whitelisting can be disabled, which allows any user to bridge their tokens. If many token collections are added while whitelisting is not enabled, it could create a  DoS for new collections due to the loop used in `_white_list_collection`. This issue affects both Ethereum and StarkNet contracts.

#### Tools Used

Manual Review

#### Recommendations

In the `_white_list_collection` function, check if whitelisting is not enabled. If so, do not add the collection to the whitelist and simply return.

---




### [L-01] `_callBaseUri` will return true even in case of `_baseUri` is empty String ""            



#### Summary

The `_callBaseUri` function performs a `staticcall` to the ERC721 contract to check if it implements either the `_baseUri()` or `baseUri` function and returns a valid string(URI). It will return `true` if the function call is successful, but it will also return `true` if the function returns an empty string `""`.

#### Vulnerability Details

Inside `_callBaseUri`, the Ark Project code will use `staticcall` to check if the ERC721 contract implements either `_baseUri` or `baseUri`. The `staticcall` will not revert if the ERC721 contract does not implement the called functions.

```solidity
    function _callBaseUri(
        address collection
    )
        internal
        view
        returns (bool, string memory)
    {
        bool success;
        uint256 returnSize;
        uint256 returnValue;
        bytes memory ret;
        bytes[2] memory encodedSignatures = [abi.encodeWithSignature("_baseUri()"), abi.encodeWithSignature("baseUri()")]; // @audit : sig would be baseURI instead of baseUri as Openzeppline implemnts baseUri

        for (uint256 i = 0; i < 2; i++) {
            bytes memory encodedParams = encodedSignatures[i];
            assembly {
                success := staticcall(gas(), collection, add(encodedParams, 0x20), mload(encodedParams), 0x00, 0x20)
                if success {
                    returnSize := returndatasize()
                    returnValue := mload(0x00)
                    ret := mload(0x40)
                    mstore(ret, returnSize)
                    returndatacopy(add(ret, 0x20), 0, returnSize)
                    // update free memory pointer
                    mstore(0x40, add(add(ret, 0x20), add(returnSize, 0x20)))
                }
            }
@1>--          if (success && returnSize >= 0x20 && returnValue > 0) {
@>2--              return (true, abi.decode(ret, (string)));
            }
        }
        return (false, "");
    }
```

One thing to note is that the default implementation of `_baseURI` will return an empty string (`""`). If such an ERC721 collection is added, all its NFTs metadata will be lost on Layer 2 Starknet due to incorrect checks `@1>--` and `@>2--`.

POC:
Add following simple code to `ethereum/test/TestUri.t.sol` folder:

```solidity
// SPDX-License-Identifier: UNLICENSED

pragma solidity 0.8.22;

import "forge-std/Test.sol";

contract ERC721Contract {
    constructor() {}

    function baseUri() external returns (string memory) {
        return ""; // it will return empty string
    }
}

contract TestUri is Test {
    ERC721Contract public contractERC721;
    function setUp() external {
        contractERC721 = new ERC721Contract();
    }
    // code of _callBaseUri add here for testing
    function _callBaseUri() internal returns (bool, string memory) {
        ERC721Contract _erc721TestContract = contractERC721;
        bool success;
        uint256 returnSize;
        uint256 returnValue;
        bytes memory ret;
        bytes[2] memory encodedSignatures = [
            abi.encodeWithSignature("_baseUri()"),
            abi.encodeWithSignature("baseUri()")
        ];

        for (uint256 i = 0; i < 2; i++) {
            bytes memory encodedParams = encodedSignatures[i];
            assembly {
                success := staticcall(
                    gas(),
                    _erc721TestContract,
                    add(encodedParams, 0x20),
                    mload(encodedParams),
                    0x00,
                    0x20
                )
                if success {
                    returnSize := returndatasize()
                    returnValue := mload(0x00)
                    ret := mload(0x40)
                    mstore(ret, returnSize)
                    returndatacopy(add(ret, 0x20), 0, returnSize)
                    // update free memory pointer
                    mstore(0x40, add(add(ret, 0x20), add(returnSize, 0x20)))
                }
            }
            if (success && returnSize >= 0x20 && returnValue > 0) {
                return (true, abi.decode(ret, (string)));
            }
        }
        return (false, "");
    }

    function test_no_uri_true() external {
        (bool success, string memory val) = checkCase();
        assertEq(val, "");
        assertEq(success, true);
    }
}

```

run with command : `forge test --mt test_no_uri_true -vvv`

#### Impact

The `_callBaseUri` function will return `true` even if the `baseUri` function returns an empty string (`""`). This can result in the loss of NFTs metadata on Layer 2 due to the absence of a URI attached to the NFT.

#### Tools Used

Manual Review , Foundry

#### Recommendations

Check that the value returned from the low-level call is a valid string and that its length is greater than 0. One potential fix would be to add the following check when `success` is `true`:

```diff
diff --git a/apps/blockchain/ethereum/src/token/TokenUtil.sol b/apps/blockchain/ethereum/src/token/TokenUtil.sol
index 41cc17d..dbb6638 100644
--- a/apps/blockchain/ethereum/src/token/TokenUtil.sol
+++ b/apps/blockchain/ethereum/src/token/TokenUtil.sol
@@ -164,7 +164,9 @@ library TokenUtil {
                 }
             }
             if (success && returnSize >= 0x20 && returnValue > 0) {
-                return (true, abi.decode(ret, (string)));
+                string memory s = abi.decode(ret, (string));
+                if(bytes(s).length > 0 ) return (true , s);
+                else return (false, "");
             }
         }
```
---

### [L-02] TokenUtil::_getBaseUri will not work as expected because of using wrong signature `_baseUri()` and `baseUri()`            



#### Summary

OpenZeppelin provides the `baseURI()` function to retrieve the base URL of an NFT collection. There are no functions defined as `_baseUri()` or `baseUri()` in any implementation or standard of OpenZeppelin's ERC721.

#### Vulnerability Details

The Ark Project will fetch the `baseURI` of an NFT collection if the collection implements the `MetadataInterface`. First, it will check if the collection implements the `baseUri` function. If so, it will retrieve the URL from that function and pass it to the `depositTokens` function, which will then send this `baseURI` along with the request to Layer 2 Starknet.

```solidity
    function erc721Metadata(
        address collection,
        uint256[] memory tokenIds
    )
        internal
        view
        returns (string memory, string memory, string memory, string[] memory)
    {        
        bool supportsMetadata = ERC165Checker.supportsInterface(
            collection,
            type(IERC721Metadata).interfaceId
        );
        
        if (!supportsMetadata) {
            return ("", "", "", new string[](0));
        }

        IERC721Metadata c = IERC721Metadata(collection);
        // How the URI must be handled.
        // if a base URI is already present, we ignore individual URI
        // else, each token URI must be bridged and then the owner of the collection
        // can decide what to do
        (bool success, string memory _baseUri) = _callBaseUri(collection);
        if (success) {
            return (c.name(), c.symbol(), _baseUri, new string[](0));
        }
```

`_callBaseUri` function :

```solidity
    function _callBaseUri(
        address collection
    )
        internal
        view
        returns (bool, string memory)
    {
        bool success;
        uint256 returnSize;
        uint256 returnValue;
        bytes memory ret;
@>--    bytes[2] memory encodedSignatures = [abi.encodeWithSignature("_baseUri()"), abi.encodeWithSignature("baseUri()")]; 

        for (uint256 i = 0; i < 2; i++) {
            bytes memory encodedParams = encodedSignatures[i];
            assembly {
                success := staticcall(gas(), collection, add(encodedParams, 0x20), mload(encodedParams), 0x00, 0x20)
                if success {
                    returnSize := returndatasize()
                    returnValue := mload(0x00)
                    ret := mload(0x40)
                    mstore(ret, returnSize)
                    returndatacopy(add(ret, 0x20), 0, returnSize)
                    // update free memory pointer
                    mstore(0x40, add(add(ret, 0x20), add(returnSize, 0x20)))
                }
            }
            if (success && returnSize >= 0x20 && returnValue > 0) {
                return (true, abi.decode(ret, (string)));
            }
        }
        return (false, "");
    }
```

The above function uses the signatures of `_baseUri()` and `baseUri()` functions to fetch the base URI of a collection. However, the issue is that these functions are not defined in the ERC721 standard. It's important to note that OpenZeppelin, up until version 3, provided a `baseURI` function as a public or external function, which can be verified in the documentation: [OpenZeppelin ERC721Metadata baseURI](https://docs.openzeppelin.com/contracts/2.x/api/token/erc721#ERC721Metadata-baseURI--).

In the current version, there is no `baseURI` external/public function available. Moreover, the functions that the Ark Project uses are not included in any ERC721 standard implementation. Therefore, it is assumed that these functions will only work if the NFT collection is using OpenZeppelin version 3 or earlier.

#### Impact

NFT collections that implement the `baseURI` function following the ERC721 standard are not compatible with the Ark Project and will return an empty value in this context. Many NFT collections use the `baseURI` function to retrieve the base URL of the collection and then append the `tokenId` to it, without implementing the `tokenURI` function. All of these collections will be incompatible with the Ark Project, potentially resulting in the loss of NFTs on the Starknet Layer 2.

#### Tools Used

Manual Review

#### Recommendations

One fix would be to add `baseURI()` and `_baseURI()` function to `encodedSignatures` array.



