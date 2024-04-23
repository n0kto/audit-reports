# First Flight #7: Horse Store - Findings Report

# Table of contents

- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
  - ### [H-01. `HuffStore.huff::IS_HAPPY_HORSE` returns the opposite of the expected result](#H-01)
  - ### [H-02. `HorseStore.huff::FEED_HORSE` reverts when the timestamp is modulo 17](#H-02)
  - ### [H-03. `HorseStore.huff::TOTAL_SUPPLY` never increases, breaking the protocol.](#H-03)
  - ### [H-04. `HuffingStore.huff::MINT_HORSE` doesn't load storage which breaks the function](#H-04)
  - ### [H-05. Fallback in `HorseStore.huff` calls `GET_TOTAL_SUPPLY`](#H-05)
  - ### [H-06. Unused line in `HorseStore.huff::MAIN` can lead to future errors](#H-06)
- ## Medium Risk Findings
  - ### [M-01. All functions in `HorseStore.huff` are payable, which is not similar to the Solidity version](#M-01)
  - ### [M-02. No `SafeTransferFrom` in `HorseStore.huff`](#M-02)
- ## Low Risk Findings
  - ### [L-01. Due to Arbitrum Sequencer, horses can be happy 0 or 2 days](#L-01)
  - ### [L-02. Horses can be fed without existing](#L-02)
  - ### [L-03. Arbitrum doesn't support PUSH0](#L-03)

# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #7

### Dates: Jan 11th, 2024 - Jan 18th, 2024

[See more contest details here](https://www.codehawks.com/contests/clr6s75ut00013qg9z8bpkalo)

# <a id='results-summary'></a>Results Summary

### Number of validated findings:

- High: 6 (1 self-duplicated)
- Medium: 2
- Low: 3

# High Risk Findings

## <a id='H-01'></a>H-01. `HuffStore.huff::IS_HAPPY_HORSE` returns the opposite of the expected result

### Relevant GitHub Links

https://github.com/Cyfrin/2024-01-horse-store/blob/main/src/HorseStore.huff#L93C1-L115C2

## Description

The `HuffStore.huff::IS_HAPPY_HORSE` macro is intended to check if a horse has been fed after yesterday. However, due to the use of the `lt` opcode, it returns true when the horse is starving and the opposite when it is fed. There is an exception with the `eq` opcode, where it returns true when `FEED_HORSE` is called in the same transaction as `IS_HAPPY_HORSE` (in that order), such as in a smart contract.

```javascript
#define macro IS_HAPPY_HORSE() = takes (0) returns (0) {
    0x04 calldataload                   // [horseId]
    LOAD_ELEMENT(0x00)                  // [horseFedTimestamp]
    timestamp                           // [timestamp, horseFedTimestamp]
    dup2 dup2                           // [timestamp, horseFedTimestamp, timestamp, horseFedTimestamp]
    sub                                 // [timestamp - horseFedTimestamp, timestamp, horseFedTimestamp]
    [HORSE_HAPPY_IF_FED_WITHIN_CONST]   // [HORSE_HAPPY_IF_FED_WITHIN, timestamp - horseFedTimestamp, timestamp, horseFedTimestamp]
@>    lt                                  // [HORSE_HAPPY_IF_FED_WITHIN < timestamp - horseFedTimestamp, timestamp, horseFedTimestamp]
    start_return_true jumpi             // [timestamp, horseFedTimestamp]
    eq                                  // [timestamp == horseFedTimestamp]
    start_return
    jump

    start_return_true:
    0x01

    start_return:
    // Store value in memory.
    0x00 mstore

    // Return value
    0x20 0x00 return
}
```

## Impact

This inconsistency breaks the logic and the protocol.

## Proof of Concept

<details>

<summary>Foundry PoC</summary>

```javascript
    function testHappyWhenStarvingAndAngryWhenFed() public {
        vm.warp(2 days);
        uint id = horseStore.totalSupply();

        vm.startPrank(user);
        horseStore.mintHorse();
        horseStore.feedHorse(id);
        skip(10);
        bool isHappy = horseStore.isHappyHorse(id);

        // will fail because the horse is not happy when it is fed
        assertTrue(isHappy);
        vm.warp(4 days);
        bool isNotHappy = horseStore.isHappyHorse(id);

        // will fail because the horse is happy when it is starving
        assertFalse(isNotHappy);
    }
```

</details>

## Recommended Mitigation

To address this issue, change the `lt` opcode to `gt` opcode. Additionally, you can remove the `eq` and `dup2 dup2` part, as there is no case where it is reachable now. To return false, keep the jump and add `0x00` before. Here is a suggested solution:

```diff
#define macro IS_HAPPY_HORSE() = takes (0) returns (0) {
    0x04 calldataload                   // [horseId]
    LOAD_ELEMENT(0x00)                  // [horseFedTimestamp]
    timestamp                           // [timestamp, horseFedTimestamp]
-    dup2 dup2                           // [timestamp, horseFedTimestamp, timestamp, horseFedTimestamp]
-    sub                                 // [timestamp - horseFedTimestamp, timestamp, horseFedTimestamp]
+    sub                                 // [timestamp - horseFedTimestamp]
-    [HORSE_HAPPY_IF_FED_WITHIN_CONST]   // [HORSE_HAPPY_IF_FED_WITHIN, timestamp - horseFedTimestamp, timestamp, horseFedTimestamp]
+    [HORSE_HAPPY_IF_FED_WITHIN_CONST]   // [HORSE_HAPPY_IF_FED_WITHIN, timestamp - horseFedTimestamp]
-    lt                                  // [HORSE_HAPPY_IF_FED_WITHIN < timestamp - horseFedTimestamp, timestamp, horseFedTimestamp]
+    gt                                  // [HORSE_HAPPY_IF_FED_WITHIN > timestamp - horseFedTimestamp]
    start_return_true jumpi             // [timestamp, horseFedTimestamp]
-    eq                                  // [timestamp == horseFedTimestamp]
+    0x00
    start_return
    jump

    start_return_true:
    0x01

    start_return:
    // Store value in memory.
    0x00 mstore

    // Return value
    0x20 0x00 return
}
```

## <a id='H-02'></a>H-02. `HorseStore.huff::FEED_HORSE` reverts when the timestamp is modulo 17

### Relevant GitHub Links

https://github.com/Cyfrin/2024-01-horse-store/blob/main/src/HorseStore.huff#L86C1-L88C12

## Description

In the `FEED_HORSE` macro, the three lines below calculate the modulo 17 of the timestamp. The issue arises when the timestamp is a multiple of 17, resulting in the modulo operation producing 0. Consequently, the jump instruction is skipped, leading to a transaction revert. It's important to note that Solidity code does not inherently include this specific calculation.

```javascript
@>    0x11 timestamp mod      // [timestamp % 17]
@>    endFeed jumpi           // will revert each time timestamp is modulo 17
@>    revert
    endFeed:
    stop
}
```

## Impact

The impact includes a denial of service with a probability of 1/17 (each time the timestamp is a multiple of 17). This behavior is non-conforming to Solidity code, causing confusion for users, developers, and auditors.

## Proof of Concept

<details>

<summary>Foundry PoC</summary>

```javascript
    function testTimestampMod17() public {
        vm.prank(user);
        horseStore.mintHorse();
        vm.warp(172805); // multiple of 17 (2 days + 5s)
        horseStore.feedHorse(0);
    }
```

</details>

## Recommended Mitigation

To address this issue, remove the three lines above and the label for the jump:

```diff
#define macro FEED_HORSE() = takes (0) returns (0) {
    timestamp               // [timestamp]
    0x04 calldataload       // [horseId, timestamp]
    STORE_ELEMENT(0x00)     // []

    // End execution
-    0x11 timestamp mod
-    endFeed jumpi
-    revert
-    endFeed:
    stop
}
```

## <a id='H-03'></a>H-03. `HorseStore.huff::TOTAL_SUPPLY` never increases, breaking the protocol.

### Relevant GitHub Links

https://github.com/Cyfrin/2024-01-horse-store/blob/main/src/HorseStore.huff#L73C1-L78C2

## Description

The `HorseStore.huff::TOTAL_SUPPLY` forgets to increase after the execution of the `_MINT` macro. Since the total supply is managed by the user (without utilizing a library like `ERC721Enumeration`), the contract needs to autonomously increment the value.

```javascript
#define macro MINT_HORSE() = takes (0) returns (0) {
    [TOTAL_SUPPLY] // [TOTAL_SUPPLY]
    sload // added in another finding to actually use the real value for _MINT
    caller         // [msg.sender, totalSupply]
@>    _MINT()        // []
@>
    stop           // []
}
```

## Impact

The consequence is the impossibility to mint more than one horse.

## Proof of Concept

<details>

<summary>Foundry PoC</summary>

```javascript
    function testTotalSupplyDontIncrease() public {
        uint id = horseStore.totalSupply();
        vm.startPrank(user);
        horseStore.mintHorse();
        // will fail because totalSupply() returns 0
        assertEq(horseStore.totalSupply(), id+1);
        vm.stopPrank();
    }
```

</details>

## Recommended Mitigation

To resolve this issue, increment the `totalSupply` after `_MINT`. Here is a suggested example:

```diff
#define macro MINT_HORSE() = takes (0) returns (0) {
    [TOTAL_SUPPLY] // [TOTAL_SUPPLY]
    sload // added in another finding to actually use the real value for _MINT
    caller         // [msg.sender, totalSupply]
    _MINT()        // []
+    [TOTAL_SUPPLY] // [TOTAL_SUPPLY]
+    sload          // [totalSupply]
+    0x01 add       // [totalSupply+1]
+    [TOTAL_SUPPLY] // [TOTAL_SUPPLY, totalSupply]
+    sstore
    stop           // []
}
```

## <a id='H-04'></a>H-04. `HuffingStore.huff::MINT_HORSE` doesn't load storage which breaks the function

### Relevant GitHub Links

https://github.com/Cyfrin/2024-01-horse-store/blob/main/src/HorseStore.huff#L73C1-L78C2

## Description

The `HuffingStore.huff::MINT_HORSE` function forgets to load storage after obtaining the slot number, resulting in passing this number (0) instead of utilizing the real value from `totalSupply()`.

```javascript
#define macro MINT_HORSE() = takes (0) returns (0) {
    [TOTAL_SUPPLY] // [TOTAL_SUPPLY]
@>
    caller         // [msg.sender, TOTAL_SUPPLY]
    _MINT()        // []
    stop           // []
}
```

## Impact

The consequence is the impossibility to mint more than one horse.

## Proof of Concept

<details>

<summary>Foundry PoC</summary>

```javascript
    function testMultiMint() public {
        vm.startPrank(user);
        horseStore.mintHorse();
        // will revert with ALREADY_MINTED error
        horseStore.mintHorse();
        vm.stopPrank();
    }
```

</details>

## Recommended Mitigation

To address this issue, add the `sload` instruction after placing the slot number on the stack.

```diff
#define macro MINT_HORSE() = takes (0) returns (0) {
    [TOTAL_SUPPLY] // [TOTAL_SUPPLY]
+    sload          // [totalSupply]
-    caller         // [msg.sender, TOTAL_SUPPLY]
+    caller         // [msg.sender, totalSupply]
    _MINT()        // []
    stop           // []
}
```

## <a id='H-05'></a>H-05. Fallback in `HorseStore.huff` calls `GET_TOTAL_SUPPLY`

### Relevant GitHub Links

https://github.com/Cyfrin/2024-01-horse-store/blob/main/src/HorseStore.huff#L167C1-L171C27

## Description

In the `MAIN` macro, no revert was placed before labels. Consequently, for any call to a non-existing function, it will execute `GET_TOTAL_SUPPLY` instead of reverting.

```javascript
#define macro MAIN() = takes (0) returns (0) {
    .
    .
    .
    dup1 __FUNC_SIG(ownerOf) eq ownerOf jumpi

@>

    totalSupply:
        GET_TOTAL_SUPPLY()
    .
    .
    .
```

## Impact

Any user or contract can misinterpret the return data and mistakenly believe that the function they called exists. This behavior is not compliant with Solidity contracts.

## Proof of Concept

<details>

<summary>Foundry PoC</summary>

```javascript
    function testFallbackReturnTotalSupply() public {
        (bool sent, bytes memory data) = address(horseStore).call("");
        // will fail because a fallback is found!
        assertFalse(sent);
        // will succeed because getSupply returns 0 when no horse is minted!
        assertEq(uint(bytes32(data)), 0);
    }
```

</details>

## Recommended Mitigation

To address this issue, add a revert before labels in the `MAIN` macro.

```diff
#define macro MAIN() = takes (0) returns (0) {
    .
    .
    .
    dup1 __FUNC_SIG(ownerOf)eq ownerOf jumpi

+    0x00 0x00 revert

    totalSupply:
        GET_TOTAL_SUPPLY()
    .
    .
    .
```

This modification ensures that if an unknown function is called, the contract reverts, preventing the unintended execution of `GET_TOTAL_SUPPLY` and maintaining compliance with Solidity contracts.

## <a id='H-06'></a>H-06. Unused line in `HorseStore.huff::MAIN` can lead to future errors

### Relevant GitHub Links

https://github.com/Cyfrin/2024-01-horse-store/blob/main/src/HorseStore.huff#L205

## Description

At the end of the `MAIN` macro in `HorseStore.huff`, there is an unused line: `MINT_HORSE()`. In future updates, if a function is added right above this line and does not `return`, `revert`, or `stop`, it could inadvertently call `MINT_HORSE`, leading to protocol errors. Additionally, this line results in unnecessary gas consumption during bytecode deployment.

```javascript
#define macro MAIN() = takes (0) returns (0) {
    .
    .
    .
@>    MINT_HORSE()
}
```

## Impact

- Gas consumption.
- Potential future errors if a function added above doesn't `revert`, `return`, or `stop`.

## Recommended Mitigation

Remove the unused line to prevent potential future errors and reduce unnecessary gas consumption.

```diff
#define macro MAIN() = takes (0) returns (0) {
    .
    .
    .
-    MINT_HORSE()
}
```

# Medium Risk Findings

## <a id='M-01'></a>M-01. All functions in `HorseStore.huff` are payable, which is not similar to the Solidity version

### Relevant GitHub Links

https://github.com/Cyfrin/2024-01-horse-store/blob/main/src/HorseStore.huff

## Description

All functions in `HorseStore.huff` are payable by default, even with the declaration `#define function name() nonpayable returns (string)`. This means that all functions can receive money, and there is currently no mechanism to withdraw it.

## Impact

There is a little likelihood of unintentional loss of funds. This inconsistency is non-conforming to the Solidity version, causing confusion for users, developers or auditors using the Huff version.

## Proof of Concept

<details>

<summary>Foundry PoC</summary>

```javascript
    function testFunctionArePayable() public {
        (bool sent, bytes memory data) = address(horseStore).call{value: 1}(
            abi.encodeWithSignature("name()")
        );

        // will revert for the Huff version!
        assertFalse(sent);
        if (sent) { // If sent (= payable, = Huff version)
            // 0-32 first bits give the pointer at the beginning of the string
            // 32-48 the length of the string
            // 48-64 the ascii hex of the string (name of the contract)
            (, , bytes32 _name) = abi.decode(data, (bytes32, bytes32, bytes32));

            // name is well returned!
            assertEq(_name, bytes32("HorseStore"));
            // balance is well increased!
            assertEq(address(horseStore).balance, 1);
        }
    }
```

</details>

## Recommended Mitigation

Include the `NON_PAYABLE` function from the `huffmate/auth` library and use it at the top of all nonpayable/view functions. If all functions are intentionally payable (although not a best practice), add the keyword "payable" to all functions in the Solidity files.

## <a id='M-02'></a>M-02. No `SafeTransferFrom` in `HorseStore.huff`

### Relevant GitHub Links

https://github.com/Cyfrin/2024-01-horse-store/blob/main/src/HorseStore.huff

## Description

`HorseStore.huff` doesn't implement the `safeTransferFrom` macro, which is available in `HorseStore.sol`. This function is useful for other contracts that want to receive this ERC721.

## Impact

Non-compliance with Solidity contract standards. It prevents the use of `HorseStore.huff` with other contracts that expect to interact with a standard ERC721 implementation.

## Recommended Mitigation

Consider using the `huffmate/token/ERC721` library instead of manually implementing ERC721 functions in `HorseStore.huff`. The library is likely to provide a standardized and safer implementation of ERC721, including `safeTransferFrom`, making the contract more interoperable with other contracts and compliant with industry standards.

# Low Risk Findings

## <a id='L-01'></a>L-01. Due to Arbitrum Sequencer, horses can be happy 0 or 2 days

## Description

On Arbitrum, a sequencer is used for `block.timestamp`, and it can deviate by up to 24 hours earlier or 1 hour in the future compared to real-time ([See Arbitrum docs for more information](https://docs.arbitrum.io/for-devs/concepts/differences-between-arbitrum-ethereum/block-numbers-and-time)).

A horse is supposed to be happy for 24 hours after a meal. However, if `block.timestamp` is 24 hours earlier (due to Arbitrum sequencer) during a call to `isHappyHorse`, a horse can be happy for 2 days. Conversely, if `block.timestamp` is 24 hours earlier during a call to `feedHorse`, a horse can be happy for 0 days if the sequencer corrects the timestamp just after the call.

## Impact

Invariant breaking :

- A horse can be happy for 2 days with just one meal.
- A horse can be happy for 0 days after a meal.

## Recommended Mitigation

Arbitrum is not recommended for this type of contract due to the deviation in `block.timestamp` caused by the Arbitrum sequencer. Consider using other Ethereum Layer 2 solutions or the Ethereum mainnet to avoid issues related to time deviation.

## <a id='L-02'></a>L-02. Horses can be fed without existing

### Relevant GitHub Links

https://github.com/Cyfrin/2024-01-horse-store/blob/main/src/HorseStore.huff#L80C1-L91C2

https://github.com/Cyfrin/2024-01-horse-store/blob/main/src/HorseStore.sol#L32C1-L34C6

## Description

There is no check if a horse exists before feeding it.

```javascript
    function feedHorse(uint256 horseId) external {
@>
        horseIdToFedTimeStamp[horseId] = block.timestamp;
    }
```

The same case exists in `HorseStore.huff`.

## Impact

A user feeding a non-existing horse won't trigger a revert and can lead to confusion.
Unexpected logic for the protocol.

## Proof of Concept

<details>

<summary>Foundry PoC</summary>

```javascript
    function testFeedNonCreatedHorseIsNotHappy() public {
        skip(2 days);
        uint nonCreatedHorseID = 100;
        // will work even if nobody minted this horse!
        horseStore.feedHorse(nonCreatedHorseID);
        // will fail because, in addition, the non-existing horse is considered happy
        assertEq(horseStore.isHappyHorse(nonCreatedHorseID), false);
    }
```

</details>

## Recommended Mitigation

Add a check in both contracts to ensure that a horse exists before feeding it. A simple check is to call `ownerOf` with the `tokenId`; this function will revert if the token is not minted.

## <a id='L-03'></a>L-03. Arbitrum doesn't support PUSH0

### Relevant GitHub Links

https://github.com/Cyfrin/2024-01-horse-store/blob/main/src/IHorseStore.sol

https://github.com/Cyfrin/2024-01-horse-store/blob/main/src/HorseStore.sol

## Description

In Solidity contracts, the compiler version is 0.8.20, which is not supported by Arbitrum due to the PUSH0 opcode. For more information, refer to the [Arbitrum documentation](https://docs.arbitrum.io/for-devs/concepts/differences-between-arbitrum-ethereum/solidity-support).

## Impact

Solidity contracts won't be deployable on Arbitrum.

## Recommended Mitigation

Use Solidity version 0.8.19, which is compatible with Arbitrum. Update the compiler version in the contract to ensure it can be deployed on the Arbitrum network.
