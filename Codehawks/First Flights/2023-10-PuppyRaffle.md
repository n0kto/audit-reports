# First Flight #2: Puppy Raffle - Findings Report

# Table of contents

- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
  - ### [H-01. Manipulation of Random Number Generation in selectWinner Function](#H-01)
  - ### [H-02. Loss of Funds Due to Invalid Winner's Address in PuppyRaffle Contract](#H-02)
  - ### [H-03. Reentrancy Vulnerability in `refund` function of PuppyRaffle Contract](#H-03)
  - ### [H-04. Manipulation of NFT Rarity in Raffle Winner Selection](#H-04)
- ## Medium Risk Findings
  - ### [M-01. Loss of Funds Due to Invalid Balance Check in withdrawFees Function](#M-01)
- ## Low Risk Findings
  - ### [L-01. entrance fees manipulation with an overflow (induce loss of fund)](#L-01)
  - ### [L-02. Incorrect Return Value in getActivePlayerIndex Function Leads to Potential Revert and Unintended Behavior](#L-02)

# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #2

### Dates: Oct 25th, 2023 - Nov 1st, 2023

[See more contest details here](https://www.codehawks.com/contests/clo383y5c000jjx087qrkbrj8)

# <a id='results-summary'></a>Results Summary

### Number of validated findings:

- High: 4
- Medium: 1
- Low: 2

# High Risk Findings

## <a id='H-01'></a>H-01. Manipulation of Random Number Generation in selectWinner Function

### Relevant GitHub Links

https://github.com/Cyfrin/2023-10-Puppy-Raffle/blob/main/src/PuppyRaffle.sol#L155C1-L155C1

## Summary

The `selectWinner` function in the provided code is vulnerable to manipulation by validators, allowing them to win the raffle by influencing the random number generation (RNG) mechanism.

## Vulnerability Details

The vulnerability arises from the use of a simple RNG mechanism in the `selectWinner` function. The winner is selected based on the result of `uint256(keccak256(abi.encodePacked(msg.sender, block.timestamp, block.difficulty))) % players.length;`. However, this method of RNG is easily predictable and can be manipulated by validators.

## Impact

The impact of this vulnerability is that validators or miners who can manipulate the RNG can win the raffle by influencing the outcome of the random number generation. This undermines the fairness and integrity of the raffle, as it allows for potential manipulation and abuse of the selection process.

## Tools Used

Manual review.

## Recommendations

To improve the security and fairness of the raffle, consider implementing the following recommendations:

1. Utilize a more secure and unpredictable RNG mechanism, such as using a trusted external random number oracle.
2. Avoid relying solely on block-related information, such as `block.timestamp` and `block.difficulty`, for RNG purposes, as they can be manipulated by validators.

## <a id='H-02'></a>H-02. Loss of Funds Due to Invalid Winner's Address in PuppyRaffle Contract

### Relevant GitHub Links

https://github.com/Cyfrin/2023-10-Puppy-Raffle/blob/main/src/PuppyRaffle.sol

## Summary

The `selectWinner` function in the provided code is vulnerable to a loss of funds if the randomly selected winner's address has been replaced with `address(0)` due to a refund.

## Vulnerability Details

The vulnerability arises from the selection of a winner using a randomly generated index from the `players` array. If one or several players have refunded their entrance fees, their addresses are replaced with `address(0)` in the array. However, there is no verification if the randomly selected winner's address is valid, which can result in the loss of funds.

## Impact

If the randomly selected winner's address is `address(0)` due to a refund, the prize and the associated non-fungible token (NFT) will be sent to `address(0)`, resulting in a loss of funds. This can occur when one or more players have refunded their entrance fees before the winner is selected.

## Tools Used

Manual review.

## Recommendations

To mitigate this vulnerability, consider implementing the following measures:

1. Use the `_isActivePlayer` function to ensure that the randomly selected winner's address is not `address(0)` before proceeding with the prize distribution.
2. Implement a mechanism to handle situations where the randomly selected winner's address is invalid, such as selecting an alternative winner or redistributing the prize pool.

## <a id='H-03'></a>H-03. Reentrancy Vulnerability in `refund` function of PuppyRaffle Contract

### Relevant GitHub Links

https://github.com/Cyfrin/2023-10-Puppy-Raffle/blob/main/src/PuppyRaffle.sol

## Summary

The `refund` function in the provided code is vulnerable to reentrancy attacks.

## Vulnerability Details

The vulnerability arises from the order of operations in the `refund` function. The function first sends the `entranceFee` to `msg.sender` using the `.sendValue()` function. However, this function can trigger a fallback function in the recipient contract, allowing for reentrant calls. After the value is sent, the `playerIndex` is set to `address(0)` to mark the player as refunded. This creates a window of opportunity for an attacker to reenter the `receive` function before the `playerIndex` is set to `address(0)`.

## Proof of Concept

The following is a simplified example of how an attacker can exploit the reentrancy vulnerability:

```solidity
contract MaliciousContract {
    PuppyRaffle private vulnerableContract;
    uint256 private index;
    uint256 private feeAmount;

    constructor(address _vulnerableContract, uint256 _feeAmount) {
        vulnerableContract = PuppyRaffle(_vulnerableContract);
        feeAmount = _feeAmount;
    }

    function AttackRaffle() external payable {
        // Call the vulnerable contract's refund function
        address[] player;
        player.push(address(this));
        feeAmount = msg.value;
        vulnerableContract.enterRaffle{value: feeAmount}(player);

        index = vulnerableContract.getActivePlayerIndex(address(this));
        vulnerableContract.refund(index);
    }

    receive() external payable {
        // Reenter the vulnerable contract's receive function
        if (address(vulnerableContract).balance >= feeAmount){
            vulnerableContract.refund(index);
        }
    }
}
```

In this proof of concept, the `MaliciousContract` is deployed and passed the address of the vulnerable contract. When the `AttackRaffle` function is called, it triggers a reentrant call to the `refund` function of the vulnerable contract. The receive function in `MaliciousContract` then reenters the vulnerable contract's `refund` function, allowing the attacker to drain the contract's balance.

## Impact

Allows the attacker to repeatedly call the `refund` function and drain the contract's balance : Steal of funds.

## Tools Used

Manual review.

## Recommendations

To mitigate this vulnerability, consider implementing the following measures:

1. Use the "Checks-Effects-Interactions" pattern to ensure that state changes occur before interacting with external contracts.
2. Alternatively, consider using a reentrancy guard.

## <a id='H-04'></a>H-04. Manipulation of NFT Rarity in Raffle Winner Selection

### Relevant GitHub Links

https://github.com/Cyfrin/2023-10-Puppy-Raffle/blob/main/src/PuppyRaffle.sol#L155C1-L155C1

## Summary

The function `selectWinner()` is vulnerable to RNG manipulation. The vulnerability identified in this function is related to the manipulation of the rarity of the Non-Fungible Tokens (NFTs) that are minted as prizes for the winners of the raffle, thanks to `block.difficulty`.

## Vulnerability Details

The vulnerability lies in the following line of code:

```solidity
uint256 rarity = uint256(keccak256(abi.encodePacked(msg.sender, block.difficulty))) % 100;
```

The `rarity` variable is calculated using a random number generation (RNG) mechanism based on the hash of the `msg.sender` address and the `block.difficulty`. However, this RNG mechanism is predictable and can be manipulated by a malicious validator.

## Impact

By manipulating the RNG mechanism, a malicious validator can control the rarity assigned to the NFTs that are minted as prizes for the raffle winners. This can lead to unfair distribution of NFTs, potentially devaluing the rarity and undermining the integrity of the raffle system.

## Tools Used

Manual review

## Recommendations

To address this vulnerability, it is recommended to use a more secure and unpredictable source of randomness for determining the rarity of the NFTs. Consider integrating with a trusted external randomness oracle (like Chainlink).

# Medium Risk Findings

## <a id='M-01'></a>M-01. Loss of Funds Due to Invalid Balance Check in withdrawFees Function

### Relevant GitHub Links

https://github.com/Cyfrin/2023-10-Puppy-Raffle/blob/main/src/PuppyRaffle.sol

## Summary

The `withdrawFees` function in the provided code is vulnerable to a loss of all fees due to a condition check on the contract's balance.

## Vulnerability Details

The vulnerability arises from the condition check in the `withdrawFees` function:

```
require(address(this).balance == uint256(totalFees), "PuppyRaffle: There are currently players active!");
```

If an attacker performs a self-destruct operation on a self-made contract with some ether in it, it will change the `address(this).balance` value (because contract won’t be able to refuse money), making the equality check never true. As a result, the fees collected in the contract will be locked forever.

## Impact

The impact of this vulnerability is the loss of all fees collected in the contract. Since the distribution of the prize is calculated based solely on the product of the number of players and the entrance fee, 20% of any additional fees sent to the contract will remain locked, as the condition check in the `withdrawFees` function will never pass.

It does not impact the `selectWinner()` function, so 80% of funds will always be distributed to each winner, same for the NFT.

## Tools Used

Manual review.

## Recommendations

To mitigate this vulnerability, consider implementing the following measures:

1. Purpose of this check is to know if there are currently player active. So check `players.length == 0` instead of using `address(this).balance`.
2. Alternatively, implement a lock to know when the raffle is running or not.

# Low Risk Findings

## <a id='L-01'></a>L-01. entrance fees manipulation with an overflow (induce loss of fund)

### Relevant GitHub Links

https://github.com/Cyfrin/2023-10-Puppy-Raffle/blob/main/src/PuppyRaffle.sol

## Summary

The code provided for the `enterRaffle` function has a vulnerability related to overflow.

## Vulnerability Details

The vulnerability arises from the line:

```solidity
require(msg.value == entranceFee * newPlayers.length, "PuppyRaffle: Must send enough to enter raffle");
```

If the multiplication of `entranceFee` and `newPlayers.length` exceeds the maximum value that can be represented by a `uint256` variable, an overflow will occur. Since all these parameters are manipulable by the sender, it is possible for the sender to enter a large number of addresses and a pre-calculated `msg.value` to bypass the check.

This vulnerability can be tested by setting up a large `entranceFee` amount (close to `type(uint256).max`) and sending an array with only two players, triggering an overflow.

## Impact

If an attacker exploits this vulnerability by providing a large number of `newPlayers` addresses and a pre-calculated `msg.value`, they can enter the raffle without sending the required amount of funds and even reduce the fees.
Due to strict equality in `withdrawFees`, it will break this function and all actual and futures fees will be lost forever.

## Tools Used

Manual review.

## Recommendations

To mitigate this vulnerability, consider implementing the following measures:

1. Update to Solidity 0.8.0 or later versions, as these versions include a safe multiplication function, which prevents overflow issues.
2. Alternatively, you can use a safe multiplication function like `safeMul` from OpenZeppelin's SafeMath library to ensure that the multiplication does not result in an overflow.
3. Implement a limit on the number of addresses that can be passed in the `newPlayers` array to prevent potential abuse and reduce the risk of overflow.

## <a id='L-02'></a>L-02. Incorrect Return Value in getActivePlayerIndex Function Leads to Potential Revert and Unintended Behavior

### Relevant GitHub Links

https://github.com/Cyfrin/2023-10-Puppy-Raffle/blob/main/src/PuppyRaffle.sol

## Summary

There is a vulnerability in the function `getActivePlayerIndex(address player)`. This vulnerability stems from an incorrect return value when a player is not found in the `players` array.

## Vulnerability Details

The vulnerability can be found at the last line in the following code snippet:

```
function getActivePlayerIndex(address player) external view returns (uint256) {
    for (uint256 i = 0; i < players.length; i++) {
        if (players[i] == player) {
            return i;
        }
    }
    return 0;
}
```

When the specified `player` is not found in the `players` array, the function returns a value of 0. This implies that the player is located at the first position of the array, potentially leading to undesired behavior and unintended consequences.

## Impact

The impact of this vulnerability is significant, as any contract or people utilizing this function may encounter issues. Contracts or people relying on this function to determine the index of a player and perform actions based on that index, such as refunds, will experience transaction reverting.

## Tools Used

Manual review.

## Recommendations

To address this vulnerability, I recommend updating the code to revert when the specified `player` is not found in the `players` array.
