# [MEDIUM-4] Rent can be created with invalid hooks, preventing rentals from being stopped

 # Lines of code
 https://github.com/re-nft/smart-contracts/blob/3ddd32455a849c3c6dc3c3aad7a33a6c9b44c291/src/policies/Stop.sol#L210 https://github.com/re-nft/smart-contracts/blob/3ddd32455a849c3c6dc3c3aad7a33a6c9b44c291/src/policies/Create.sol#L464
 
 ## Impact
 Each order can contain an array of hooks, which should be whitelisted by the admin. After the function for finalizing order creation gets called (`Create.validateOrder`) a check is made if the provided hooks are whitelisted to run `onStart()` (at the start of the rent). However, no check is made if they are allowed to run `hookOnStop` (when rent is ended). This check is made only when the rent is being stopped (`Stop.stopRent`). This means rentals with invalid `hookOnStop` hooks can be created. As a consequence stopping will not be possible, because the check will revert
 
 ## Proof of Concept
 Quoting the explanation on hooks from the docs:
 
 ```
 When signing a rental order, the lender can decide to include an array of Hook structs along with it. These are bespoke restrictions or added functionality that can be applied to the rented token within the wallet.
 ```
 
 The rental is finalized in the protocol when `validateOrder` in `Create.sol` get's called. Inside of it a validation is made on the provided hooks:
 
 ```
  function _addHooks(
         Hook[] memory hooks,
         SpentItem[] memory offerItems,
         address rentalWallet
     ) internal {
       .......
 
         // Loop through each hook in the payload.
         for (uint256 i = 0; i < hooks.length; ++i) {
             // Get the hook's target address.
             target = hooks[i].target;
 
             // @audit no check for `hookOnStop` is made
             // Check that the hook is reNFT-approved to execute on rental start.
             if (!STORE.hookOnStart(target)) {
                 revert Errors.Shared_DisabledHook(target);
             }
        ........
       }
     }
 ```
 
 No validation is made if the hooks provided will also run when the rent is concluded.
 
 This could be a problem because the `stopRent` function inside `Stop.sol` does a validation on the hooks and reverts if even one of them is not whitelisted:
 
 ```
 function _removeHooks(
         Hook[] calldata hooks,
         Item[] calldata rentalItems,
         address rentalWallet
     ) internal {
         ....
 
         // Loop through each hook in the payload.
         for (uint256 i = 0; i < hooks.length; ++i) {
             // Get the hook address.
             target = hooks[i].target;
 
             // Check that the hook is reNFT-approved to execute on rental stop.
             if (!STORE.hookOnStop(target)) {        <------------- This will revert
                 revert Errors.Shared_DisabledHook(target);
             }
 
      ........
     }
 ```
 
 The result is that invalid rental orders are allowed to be created, which obviously could be a problem.
 
 Imagine a scenario with a `PAY` order (where the lender gives his assets and also token payment to the renter). And he wants to end the rent prematurely, because he needs the assets or probably the renter does not behave as expected.
 
 Even in normal circumstances when the appropriate conditions are met, rentals should not be blocked from being stopped, since this affects the normal functioning of the protocol.
 
 To further reinforce my arguments. I'm quoting the provided docs on hooks: `Only hooks which have been enabled by the admin will be valid when passed to the address target field`
 
 https://github.com/code-423n4/2024-01-renft/blob/main/docs/hooks.md
 
 ## Tools Used
 Manual review
 
 ## Recommended Mitigation Steps
 Add the following check inside `_addHooks` in `Create.sol`:
 
 ```
 if (!STORE.hookOnStop(target)) {
                 revert Errors.Shared_DisabledHook(target);
             }
 ```
 
 ## Assessed type
 Invalid Validation


# [MEDIUM-8] The Guard does NOT prevent burning and pausing of assets while rent is active

 # Lines of code
 https://github.com/re-nft/smart-contracts/blob/3ddd32455a849c3c6dc3c3aad7a33a6c9b44c291/src/policies/Guard.sol#L195
 
 ## Impact
 While a rental is active, the `Guard` contract should prevent any token operations such as (transfer, approve, etc.), so that lender assets could not be stolen. Problem is that `Guard.sol` does not account for burnable & pausable tokens (which are quite common as well). This means lenders could lose such assets if the renter acts maliciously.
 
 ## Proof of Concept
 According to the docs:
 
 `The Guard Prevents transfers of ERC721 and ERC1155 tokens while a rental is active, as well as preventing token approvals and enabling of non-whitelisted gnosis safe modules.`
 
 The validation of the functions being called happens inside `_checkTransaction` of `Guard.sol`:
 
 https://github.com/re-nft/smart-contracts/blob/3ddd32455a849c3c6dc3c3aad7a33a6c9b44c291/src/policies/Guard.sol#L195
 
 ```solidity
  function _checkTransaction(address from, address to, bytes memory data) private view {
         bytes4 selector;
 
         // Load in the function selector.
         assembly {
             selector := mload(add(data, 0x20))
         }
 
        
         //@audit burn && pause NOT checked
         if (selector == e721_safe_transfer_from_1_selector) {
             _revertSelectorOnActiveRental(...)
         } else if (selector == e721_safe_transfer_from_2_selector) {
            _revertSelectorOnActiveRental(...)
         } else if (selector == e721_transfer_from_selector) {
            _revertSelectorOnActiveRental(...)
         }
     
       ...... 
 }
 ```
 
 Problem is that no validation is made for the following methods:
 
 * `burn` (ERC721)
 * `pause` (ERC721)
 * `burnBatch` (ERC1155)
 
 This means that a renter could call any of those methods on the assets in his safe and destroy the assets(if calling `burn` | `burnBatch`) or pause them and prevent the transfer back to the lender once rental period is over.
 
 I would like to refer to the publicly known issues section on `Dishonest ERC721/ERC1155 Implementations`, which states:
 
 `The Guard contract can only protect against the transfer of tokens that faithfully implement the ERC721/ERC1155 spec. A dishonest implementation that adds an additional function to transfer the token to another wallet cannot be prevented`
 
 The mentioned methods are NOT dishonest or some exotic implementation of ERC721/ERC1155. They are part of the official OpenZeppelin contracts for the two standards and are thus wildly used all the time.
 
 * https://docs.openzeppelin.com/contracts/3.x/api/token/erc721#ERC721Pausable
 * https://docs.openzeppelin.com/contracts/3.x/api/token/erc721#ERC721Burnable
 * https://docs.openzeppelin.com/contracts/3.x/api/token/erc1155#ERC1155Burnable
 
 Myself as a web3 dev have implemented such tokens 3 times already!
 
 ## Tools Used
 Manual review
 
 ## Recommended Mitigation Steps
 One option is to add the selectors for those methods to the `_checkTransaction` function.
 
 Even better approach would be to invert the mechanism. Instead of checking for each method separately (the list can get quite long), a whitelist of allowed methods (primarily get methods) could be made. This way any other call would not be allowed, thus making the safe secure from dishonest standart implementations
 
 ## Assessed type
 Invalid Validation


 # [MEDIUM-11] Protocol does not implement EIP712 correctly on multiple occasions

 # Lines of code
https://github.com/re-nft/smart-contracts/blob/3ddd32455a849c3c6dc3c3aad7a33a6c9b44c291/src/packages/Signer.sol#L151-L151 https://github.com/re-nft/smart-contracts/blob/3ddd32455a849c3c6dc3c3aad7a33a6c9b44c291/src/packages/Signer.sol#L373-L375 https://github.com/re-nft/smart-contracts/blob/3ddd32455a849c3c6dc3c3aad7a33a6c9b44c291/src/packages/Signer.sol#L384-L386 https://github.com/re-nft/smart-contracts/blob/3ddd32455a849c3c6dc3c3aad7a33a6c9b44c291/src/packages/Signer.sol#L232-L238 https://github.com/re-nft/smart-contracts/blob/3ddd32455a849c3c6dc3c3aad7a33a6c9b44c291/src/policies/Create.sol#L636

## Impact
Being not EIP712 compliant can lead to issues with integrators and possibly DOS.

### Problem 1
The implementation of the hook hash ([here](https://github.com/re-nft/smart-contracts/blob/3ddd32455a849c3c6dc3c3aad7a33a6c9b44c291/src/packages/Signer.sol#L151C56-L151C56)) is done incorrectly. `hook.extraData` is of type `bytes` which according to EIP712 it is referred to as a `dynamic type`. Dynamic types must be first hashed with `keccak256` to become one 32-byte word before being encoded and hashed together with the typeHash and the other values.

#### Mitigation to Problem 1:
```diff
function _deriveHookHash(Hook memory hook) internal view returns (bytes32) {
  // Derive and return the hook as specified by EIP-712.
    return
        keccak256(
-           abi.encode(_HOOK_TYPEHASH, hook.target, hook.itemIndex, hook.extraData)
+           abi.encode(_HOOK_TYPEHASH, hook.target, hook.itemIndex, keccak256(hook.extraData))
        );
}
```

### Problem 2
Some TypeHashes are computed by using `abi.encode` instead of `abi.encodePacked` ([here](https://github.com/re-nft/smart-contracts/blob/3ddd32455a849c3c6dc3c3aad7a33a6c9b44c291/src/packages/Signer.sol#L373C1-L375)) which makes the typeHash value and the whole final hash different from the hash that correctly implementing EIP712 entities would get.

```solidity
// Construct the Item type string.
bytes memory itemTypeString = abi.encodePacked(
    "Item(uint8 itemType,uint8 settleTo,address token,uint256 amount,uint256 identifier)"
);

// Construct the Hook type string.
bytes memory hookTypeString = abi.encodePacked(
    "Hook(address target,uint256 itemIndex,bytes extraData)"
);

// Construct the RentalOrder type string.
bytes memory rentalOrderTypeString = abi.encodePacked(
    "RentalOrder(bytes32 seaportOrderHash,Item[] items,Hook[] hooks,uint8 orderType,address lender,address renter,address rentalWallet,uint256 startTimestamp,uint256 endTimestamp)"
);

...

rentalOrderTypeHash = keccak256(
    abi.encode(rentalOrderTypeString, hookTypeString, itemTypeString)
);
```

The problem with this is that abi.encode-ing strings results in bytes with arbitrary length. In such cases (like the one here) there is a high chance that the bytes will not represent an exact N words in length (X * 32 bytes length) and the data is padded to conform uniformly to 32-byte words. This padding results in an incorrect hash of the typeHash and it will make the digest hash invalid when compared with properly implemented hashes from widely used libraries such as ethers.

#### Proof of Concept
Place the following code in any of the tests and run `forge test -—mt test_EIP712_encoding`

```solidity
function test_EIP712_encoding() public {
		// Copied from the reNFT codebase
    bytes memory itemTypeString = abi.encodePacked(
        "Item(uint8 itemType,uint8 settleTo,address token,uint256 amount,uint256 identifier)"
    );
    bytes memory hookTypeString = abi.encodePacked(
        "Hook(address target,uint256 itemIndex,bytes extraData)"
    );
    bytes memory rentalOrderTypeString = abi.encodePacked(
        "RentalOrder(bytes32 seaportOrderHash,Item[] items,Hook[] hooks,uint8 orderType,address lender,address renter,address rentalWallet,uint256 startTimestamp,uint256 endTimestamp)"
    );
		// protocol implementation
    bytes32 rentalOrderTypeHash = keccak256(
        abi.encode(rentalOrderTypeString, hookTypeString, itemTypeString) // <-----
    );

    // correct implementation
    bytes32 rentalOrderTypeHashCorrect = keccak256(
        abi.encodePacked(rentalOrderTypeString, hookTypeString, itemTypeString) // <-----
    );

    // the correct typehash
    bytes32 correctTypeHash = keccak256(
        "RentalOrder(bytes32 seaportOrderHash,Item[] items,Hook[] hooks,uint8 orderType,address lender,address renter,address rentalWallet,uint256 startTimestamp,uint256 endTimestamp)Hook(address target,uint256 itemIndex,bytes extraData)Item(uint8 itemType,uint8 settleTo,address token,uint256 amount,uint256 identifier)"
    );

    assertNotEq(rentalOrderTypeHash, rentalOrderTypeHashCorrect);
    assertNotEq(rentalOrderTypeHash, correctTypeHash);
    assertEq(rentalOrderTypeHashCorrect, correctTypeHash);
}
```

This test shows that the `rentalOrderTypeHashCorrect` is the correct typeHash.

#### Mitigation to Problem 2
```diff
rentalOrderTypeHash = keccak256(
-    abi.encode(rentalOrderTypeString, hookTypeString, itemTypeString)
+    abi.encodePacked(rentalOrderTypeString, hookTypeString, itemTypeString)
);
```

### Problem 3
`_deriveOrderMetadataHash` constructs the hash incorrectly because _ORDER_METADATA_TYPEHASH includes `uint8 orderType` and `bytes emittedExtraData` (it can be seen [here](https://github.com/re-nft/smart-contracts/blob/3ddd32455a849c3c6dc3c3aad7a33a6c9b44c291/src/packages/Signer.sol#L384-L386C15)) but these values are not provided below the typeHash (it can be seen [here](https://github.com/re-nft/smart-contracts/blob/3ddd32455a849c3c6dc3c3aad7a33a6c9b44c291/src/packages/Signer.sol#L232-L238)).

```solidity
function _deriveOrderMetadataHash(
    OrderMetadata memory metadata
) internal view returns (bytes32) {
    bytes32[] memory hookHashes = new bytes32[](metadata.hooks.length);

    for (uint256 i = 0; i < metadata.hooks.length; ++i) {
        hookHashes[i] = _deriveHookHash(metadata.hooks[i]);
    }

    return
        keccak256(
            abi.encode(
                _ORDER_METADATA_TYPEHASH,// OrderMetadata(uint8 orderType,uint256 rentDuration,Hook[] hooks,bytes emittedExtraData)
								// <---- misses uint8 orderType
                metadata.rentDuration,
                keccak256(abi.encodePacked(hookHashes))
								// <---- misses bytes emittedExtraData
            )
        );
}
```

This hash is important because it is compared to the zoneHash inside `Create.sol:validateOrder → _rentFromZone → _isValidOrderMetadata` ([link](https://github.com/re-nft/smart-contracts/blob/3ddd32455a849c3c6dc3c3aad7a33a6c9b44c291/src/policies/Create.sol#L636)): So if the provided zoneHash from Seaport was generated correctly and it is not the same as the one generated by reNFT, the protocol will not be able to create any rentalOrders resulting in **DOS**.

In any case, this implementation is not according to EIP712 and either the fields must be included or the `_ORDER_METADATA_TYPEHASH` must remove `uint8 orderType` and `bytes emittedExtraData`

## Tools Used
Manual review and Foundry for testing

## Recommendations
Apply the described fixes for each example

## Assessed type
Othe