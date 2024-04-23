# First Flight #12: Kitty Connect - Findings Report

# Table of contents

- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
  - ### [H-01. `KittyBridge::bridgeNftWithData` doesn't approve fee token, causing CCIP to revert](#H-01)
  - ### [H-02. Cat information and `s_ownerToCatsTokenId` are not updated during transfer/bridge, leading to impossibility to bridge](#H-02)

# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #12: Kitty Connect

### Dates: Mar 28th, 2024 - Apr 4th, 2024

[See more contest details here](https://www.codehawks.com/contests/clu7ddcsa000fcc387vjv6rpt)

# <a id='results-summary'></a>Results Summary

### Number of findings:

- High: 2
- Medium: 0
- Low: 0

# High Risk Findings

## <a id='H-01'></a>H-01. `KittyBridge::bridgeNftWithData` doesn't approve fee token, causing CCIP to revert

### Relevant GitHub Links

https://github.com/Cyfrin/2024-03-kitty-connect/blob/main/src/KittyBridge.sol#L53C5-L77C6

## Description

In the `bridgeNftWithData` function of the `KittyBridge` contract, there is no approval of fee tokens before calling the `ccipSend` function. According to the Chainlink CCIP documentation, the bridge contract needs to own and approve LINK tokens to use the CCIP product. Without this approval, CCIP will revert, making it impossible to bridge any token.

```javascript
    function bridgeNftWithData(
        ...
    )
        ...
    {
        ...
        uint256 fees = router.getFee(_destinationChainSelector, evm2AnyMessage);

        if (fees > s_linkToken.balanceOf(address(this))) {
            revert KittyBridge__NotEnoughBalance(
                s_linkToken.balanceOf(address(this)),
                fees
            );
        }
@>
        messageId = router.ccipSend(_destinationChainSelector, evm2AnyMessage);

        emit MessageSent(
            messageId,
            _destinationChainSelector,
            _receiver,
            _data,
            address(s_linkToken),
            fees
        );

        return messageId;
    }
```

## Risk

**Likelyhood**:

- Occurs with every bridge.

**Impact**:

- Impossible to bridge NFTs.

## Proof of Concept

- Attempt to bridge any NFT via the Chainlink Testnet.

## Recommended Mitigation

Add approval of the fee tokens in the function:

```diff
    function bridgeNftWithData(
        ...
    )
        ...
    {
        ...
        uint256 fees = router.getFee(_destinationChainSelector, evm2AnyMessage);

        if (fees > s_linkToken.balanceOf(address(this))) {
            revert KittyBridge__NotEnoughBalance(
                s_linkToken.balanceOf(address(this)),
                fees
            );
        }
+       s_linkToken.approve(address(router), fees);

        messageId = router.ccipSend(_destinationChainSelector, evm2AnyMessage);

        emit MessageSent(
            messageId,
            _destinationChainSelector,
            _receiver,
            _data,
            address(s_linkToken),
            fees
        );

        return messageId;
    }
```

## <a id='H-02'></a>H-02. Cat information and `s_ownerToCatsTokenId` are not updated during transfer/bridge, leading to impossibility to bridge

### Relevant GitHub Links

https://github.com/Cyfrin/2024-03-kitty-connect/blob/main/src/KittyConnect.sol#L118C1-L127C6

## Description

Cat information is not updated during other transfer function (only `safeTransferFrom` is override), and `s_ownerToCatsTokenId` is never updated during any transfer. Consequently, `getCatsTokenIdOwnedBy` won't return the real list of tokens owned by a user.

```javascript
    function safeTransferFrom(
        address currCatOwner,
        address newOwner,
        uint256 tokenId,
        bytes memory data
    ) public override onlyShopPartner {
        require(
            _ownerOf(tokenId) == currCatOwner,
            "KittyConnect__NotKittyOwner"
        );

        require(
            getApproved(tokenId) == newOwner,
            "KittyConnect__NewOwnerNotApproved"
        );

        _updateOwnershipInfo(currCatOwner, newOwner, tokenId);
@>
        emit CatTransferredToNewOwner(currCatOwner, newOwner, tokenId);

        _safeTransfer(currCatOwner, newOwner, tokenId, data);
    }
```

During a bridge operation (`mintBridgedNFT`) or any other transfer functions, `_updateOwnershipInfo` is not called, leading to:

- Failure to add the transferred NFT to `s_ownerToCatsTokenId` of the new owner.
- Failure to update `s_catInfo[tokenId].idx` of the token.
- Failure to add the previous owner in the array of the cat.

As a result, it will be impossible to bridge this NFT if the new owner only owns this NFT because `bridgeNftToAnotherChain` will try to remove the tokenId from an empty array. In the worst case, if the owner already has an NFT and tries to bridge a transferred NFT, it will remove the first NFT from the array, making it impossible to bridge this one.

```javascript
    function bridgeNftToAnotherChain(
        uint64 destChainSelector,
        address destChainBridge,
        uint256 tokenId
    ) external {
        address catOwner = _ownerOf(tokenId);

        require(msg.sender == catOwner);

        CatInfo memory catInfo = s_catInfo[tokenId];
        uint256 idx = catInfo.idx;
        bytes memory data = abi.encode(
            catOwner,
            catInfo.catName,
            catInfo.breed,
            catInfo.image,
            catInfo.dob,
            catInfo.shopPartner
        );

        _burn(tokenId);
        delete s_catInfo[tokenId];

        uint256[] memory userTokenIds = s_ownerToCatsTokenId[msg.sender];
@>        uint256 lastItem = userTokenIds[userTokenIds.length - 1];

@>        s_ownerToCatsTokenId[msg.sender].pop();
        ...
    }
```

## Risk

**Likelyhood**: High

- Occurs every time a transfer/bridge operation is executed.

**Impact**: High

- `getCatsTokenIdOwnedBy` will return tokens not owned by the user or won’t return token owned by the user.
- A bridge operation or any normal transfer can result in the loss of NFTs or the impossibility to bridge them.

## Proof of Concept

<details>

<summary>Foundry PoC to add in `KittyTest.t.sol` </summary>

```javascript
    function test_safetransferCatToNewOwner() public {
        string
            memory catImageIpfsHash = "ipfs://QmbxwGgBGrNdXPm84kqYskmcMT3jrzBN8LzQjixvkz4c62";
        uint256 tokenId = kittyConnect.getTokenCounter();
        address newOwner = makeAddr("newOwner");

        vm.prank(partnerA);
        kittyConnect.mintCatToNewOwner(
            user,
            catImageIpfsHash,
            "Meowdy",
            "Ragdoll",
            block.timestamp
        );

        vm.prank(user);
        kittyConnect.approve(newOwner, tokenId);

        vm.expectEmit(false, false, false, true, address(kittyConnect));
        emit CatTransferredToNewOwner(user, newOwner, tokenId);
        vm.prank(partnerA);
        kittyConnect.safeTransferFrom(user, newOwner, tokenId);

        assertEq(kittyConnect.getCatsTokenIdOwnedBy(user).length, 0);
    }
```

</details>

## Recommended Mitigation

- Override `__afterTokenTransfer` and add `_updateOwnershipInfo` in it to update ownership information for every transfer.
- Add `_updateOwnershipInfo` to `mintBridgedNFT`.
- In `_updateOwnershipInfo`, update `s_ownerToCatsTokenId` by accessing the array of the previous owner and erase the transferred tokenId.
