# First Flight #13: Baba Marta - Findings Report

# Table of contents

- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
  - ### [H-01. `MartenitsaMarketplace::collectReward` does not keep track of the real `amountRewards` leading to infinite reward.](#H-01)
  - ### [H-02. `MartenitsaToken::updateCountMartenitsaTokensOwner` lacks of access control leading to storage corruption.](#H-02)
  - ### [H-03. `msg.value` not properly handled in `buyMartenitsa` lead to loss of funds](#H-03)
- ## Medium Risk Findings
  - ### [M-01. `stopEvent` does not clean all variable, breaking the protocol](#M-01)
  - ### [M-02. Transfers are not deactivated in `Healthtoken` and `MartenitsaToken`, breaking all the protocol.](#M-02)
  - ### [M-03. No possible draw in `MartenitsaVoting::announceWinner`, first token with the max votes will win](#M-03)

# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #13

### Dates: Apr 11th, 2024 - Apr 18th, 2024

[See more contest details here](https://www.codehawks.com/contests/cluseb1bf0001s4tjl2rzajup)

# <a id='results-summary'></a>Results Summary

### Number of findings:

- High: 3
- Medium: 4
- Low: 1

# High Risk Findings

## <a id='H-01'></a>H-01. `MartenitsaMarketplace::collectReward` does not keep track of the real `amountRewards` leading to infinite reward.

## Description

`MartenitsaMarketplace::collectReward` distribute rewards according the number of `MartenitsaToken` owned by the caller.
Problem is that `_collectedRewards` is not increased but reset by `amountRewards`. For example, a user with 3 tokens will call the function to have a reward and if they buy again 3 token, call `collectReward` again, the `amountRewards` will be 1 (because reward has already been collected) and set the `_collectedRewards` back to 1 instead of 2. A malicious user can now continue calling `collectReward` to earn infinite `HealthToken`.

```javascript
    function collectReward() external {
        require(
            !martenitsaToken.isProducer(msg.sender),
            "You are producer and not eligible for a reward!"
        );
        uint256 count = martenitsaToken.getCountMartenitsaTokensOwner(
            msg.sender
        );

        uint256 amountRewards = (count / requiredMartenitsaTokens) -
            _collectedRewards[msg.sender];
        if (amountRewards > 0) {
@>            _collectedRewards[msg.sender] = amountRewards;
            healthToken.distributeHealthToken(msg.sender, amountRewards);
        }
    }
```

## Risk

**Likelyhood**:

- Anyone, Anytime : cost 6 MartenitsaToken

**Impact**:

- Infinite HeathToken minting

## Proof of Concept

<details>

<summary>Foundry PoC to add in `HealthToken.t.sol` </summary>

```javascript
    function testInfiniteDistribution() public {
        address attacker = makeAddr("attacker");
        vm.startPrank(attacker);
        martenitsaToken.updateCountMartenitsaTokensOwner(attacker, "add");
        martenitsaToken.updateCountMartenitsaTokensOwner(attacker, "add");
        martenitsaToken.updateCountMartenitsaTokensOwner(attacker, "add");
        marketplace.collectReward();

        assertEq(healthToken.balanceOf(attacker), 1e18);

        martenitsaToken.updateCountMartenitsaTokensOwner(attacker, "add");
        martenitsaToken.updateCountMartenitsaTokensOwner(attacker, "add");
        martenitsaToken.updateCountMartenitsaTokensOwner(attacker, "add");
        marketplace.collectReward();

        // collect infinite rewards without having more MartenitsaToken
        marketplace.collectReward();
        marketplace.collectReward();
        marketplace.collectReward();
        marketplace.collectReward();
        marketplace.collectReward();

        assertEq(healthToken.balanceOf(attacker), 7e18);
        vm.stopPrank();
    }
```

</details>

## Recommended Mitigation

Increase `_collectedRewards` instead of replacing it :

```diff
function collectReward() external {
        require(
            !martenitsaToken.isProducer(msg.sender),
            "You are producer and not eligible for a reward!"
        );
        uint256 count = martenitsaToken.getCountMartenitsaTokensOwner(
            msg.sender
        );

        uint256 amountRewards = (count / requiredMartenitsaTokens) -
            _collectedRewards[msg.sender];
        if (amountRewards > 0) {
-            _collectedRewards[msg.sender] = amountRewards;
+            _collectedRewards[msg.sender] += amountRewards;
            healthToken.distributeHealthToken(msg.sender, amountRewards);
        }
    }
```

## <a id='H-02'></a>H-02. `MartenitsaToken::updateCountMartenitsaTokensOwner` lacks of access control leading to storage corruption.

## Description

`MartenitsaToken` uses a function to keep update the NFT counter of every addresses.
Problem is the function doesn't have access control : anyone can call it with arbitrary parameters and changer the counter of anyone else.

```javascript
    function updateCountMartenitsaTokensOwner(
        address owner,
        string memory operation
@>    ) external {
        if (
            keccak256(abi.encodePacked(operation)) ==
            keccak256(abi.encodePacked("add"))
        ) {
            countMartenitsaTokensOwner[owner] += 1;
        } else if (
            keccak256(abi.encodePacked(operation)) ==
            keccak256(abi.encodePacked("sub"))
        ) {
            countMartenitsaTokensOwner[owner] -= 1;
        } else {
            revert("Wrong operation");
        }
    }
```

The counter is reachable via `getCountMartenitsaTokensOwner` and is used to distribute HealthToken. Anyone can set is counter to a number modulo 3 to steal the HealthToken. An attacker can also prevent any user to receive their reward decreasing the counter.

## Risk

**Likelyhood**: High

- Anyone, anytime

**Impact**: High

- Manipulation of the `countMartenitsaTokensOwner` variable.
- Steal HealthToken or prevent its distribution.

## Proof of Concept

<details>

<summary>Foundry PoC to add in `HealthToken.t.sol` </summary>

```javascript
    function testDistributionManipulation() public {
        address attacker = makeAddr("attacker");
        vm.startPrank(attacker);
        martenitsaToken.updateCountMartenitsaTokensOwner(attacker, "add");
        martenitsaToken.updateCountMartenitsaTokensOwner(attacker, "add");
        martenitsaToken.updateCountMartenitsaTokensOwner(attacker, "add");
        marketplace.collectReward();

        assertEq(healthToken.balanceOf(attacker), 1e18);
        vm.stopPrank();
    }
```

</details>

## Recommended Mitigation

Add a modifier to allow only marketplace contract to call `updateCountMartenitsaTokensOwner`.

## <a id='H-03'></a>H-03. `msg.value` not properly handled in `buyMartenitsa` lead to loss of funds

## Description

`MartenitsaMarketplace::buyMartenitsa` allows users to buy tokens paying the related price.
Problem is that the user can send more than the price but only the listed price will be sent to the producer, resulting to stuck funds in the contract.

```javascript
    function buyMartenitsa(uint256 tokenId) external payable {
        Listing memory listing = tokenIdToListing[tokenId];
        require(listing.forSale, "Token is not listed for sale");
@>        require(msg.value >= listing.price, "Insufficient funds");

        address seller = listing.seller;
        address buyer = msg.sender;
        uint256 salePrice = listing.price;

        martenitsaToken.updateCountMartenitsaTokensOwner(buyer, "add");
        martenitsaToken.updateCountMartenitsaTokensOwner(seller, "sub");

        // Clear the listing
        delete tokenIdToListing[tokenId];

        emit MartenitsaSold(tokenId, buyer, salePrice);

        // Transfer funds to seller
@>        (bool sent, ) = seller.call{value: salePrice}("");
        require(sent, "Failed to send Ether");

        // Transfer the token to the buyer
        martenitsaToken.safeTransferFrom(seller, buyer, tokenId);
    }
```

## Risk

**Likelyhood**: Low

- When a user send more than the exact price.

**Impact**: High

- Loss of funds.

## Recommended Mitigation

Transfer the entire amount sent by the seller to the producer (example below).
Alternatively, `require` statement can check for the exact price or the function may send back the extra value to the buyer.

```diff
    function buyMartenitsa(uint256 tokenId) external payable {
        Listing memory listing = tokenIdToListing[tokenId];
        require(listing.forSale, "Token is not listed for sale");
        require(msg.value >= listing.price, "Insufficient funds");

        address seller = listing.seller;
        address buyer = msg.sender;
        uint256 salePrice = listing.price;

        martenitsaToken.updateCountMartenitsaTokensOwner(buyer, "add");
        martenitsaToken.updateCountMartenitsaTokensOwner(seller, "sub");

        // Clear the listing
        delete tokenIdToListing[tokenId];

        emit MartenitsaSold(tokenId, buyer, salePrice);

        // Transfer funds to seller
-        (bool sent, ) = seller.call{value: salePrice}("");
+        (bool sent, ) = seller.call{value: msg.value}("");
        require(sent, "Failed to send Ether");

        // Transfer the token to the buyer
        martenitsaToken.safeTransferFrom(seller, buyer, tokenId);
```

# Medium Risk Findings

## <a id='M-01'></a>M-01. `stopEvent` does not clean all variable, breaking the protocol

## Description

`MartenitsaEvent::stopEvent` ends an event. However, it does not clean/reset this 3 variables:

- `participants` array.
- `_participants` mapping.
- `producers` array.

When a new event will be launched, all these variables will contains the previous event data.
Here is several unexpected behavior:

- Users won't be able to participate anymore due to this line in `joinEvent`: `require(!_participant[msg.sender],"You have already joined the event");`
- After several events, `participants` array will become too big and the loop in `stopEvent` will revert before finishing (out-of-gas / exceeding block gas limit)
- After several event, `producers` array will be huge and the `getAllProducers` function will revert returning it.

```javascript
    function stopEvent() external onlyOwner {
        require(block.timestamp >= eventEndTime, "Event is not ended");
        for (uint256 i = 0; i < participants.length; i++) {
            isProducer[participants[i]] = false;
        }
    }
```

## Risk

**Likelyhood**: High

- From the second event and for all others

**Impact**: High

- Previous participants cannot join new events
- `stopEvent` will revert after some events leading all the users to stay producers

## Recommended Mitigation

- Clean also the `_participants` mapping, delete the `participants` array in `stopEvent` function.
- Don't push any participant in the producers array since the same data is in the `participants` array.

## <a id='M-02'></a>M-02. Transfers are not deactivated in `Healthtoken` and `MartenitsaToken`, breaking all the protocol.

## Description

Transfers are not allowed in the protocol. Only `MartenitsaMarketplace` can provide `MartenitsaTokens`, and only this contract and the `MartenitsaVoting` can provide HealthTokens. However, all transfer functions are available due to inheritance. Without overriding them, any user can transfer their tokens and break the protocol:

- `MartenitsaToken::updateCountMartenitsaTokensOwner` are not called in any transfer function.
- Have HealthToken without winning them.
- A listed token can be sent to anyone and being sell without the consent of the new owner. Money goes to the first owner.

## Risk

**Likelyhood**: High

- Anyone, Anytime

**Impact**: High

- `countMartenitsaTokensOwner` won't be updated, breaking the reward mechanism.
- `MartenitsaToken` can be transfered and sell without the consent of the new owner.
- `HealthToken` can be owned by anyone without winning them.

## Proof of Concept

<details>

<summary>Foundry PoC to add in `HealthToken.t.sol` </summary>

```javascript
    function testTransferToken() public {
        address attacker = makeAddr("attacker");

        deal(address(healthToken), attacker, 1e18);

        vm.startPrank(attacker);
        healthToken.transfer(address(0x01), 1e18);
    }
```

</details>

## Recommended Mitigation

Override all transfer function to revert in both tokens, except:

- `HealthToken::transferFrom` which has to work only if the `MartenitsaEvent` is the `msg.sender`
- `MartenitsaToken::safeTransferFrom` which has to work only if the `MartenitsaMarketplace` is the `msg.sender`

## <a id='M-03'></a>M-03. No possible draw in `MartenitsaVoting::announceWinner`, first token with the max votes will win

## Description

During the selection of the winner, every time a token has more vote than the previously stored one, it will be stored in `winnerTokenId`. Any token with the same amount of votes will be skipped.

```javascript
    function announceWinner() external onlyOwner {
        require(
            block.timestamp >= startVoteTime + duration,
            "The voting is active"
        );

        uint256 winnerTokenId;
        uint256 maxVotes = 0;

        for (uint256 i = 0; i < _tokenIds.length; i++) {
@>            if (voteCounts[_tokenIds[i]] > maxVotes) {
                maxVotes = voteCounts[_tokenIds[i]];
                winnerTokenId = _tokenIds[i];
            }
        }

        list = _martenitsaMarketplace.getListing(winnerTokenId);
        _healthToken.distributeHealthToken(list.seller, 1);

        emit WinnerAnnounced(winnerTokenId, list.seller);
    }
```

## Risk

**Likelyhood**: Low

- When several tokens have the maximum amount of votes

**Impact**: Low

- Producer entering the voting array first will have more chances to win.

## Recommended Mitigation

Keep all winners in an array and implement a random selection between them to be fairer.
