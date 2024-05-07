# First Flight #14: AirDropper - Findings Report

# Table of contents

- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings

  - ### [H-01. Users can claim multiple times and steal all funds.](#H-01)
  - ### [H-02. Incorrect Merkle root due to incorrect decimals at the tree's creation](#H-02)
  - ### [H-03. Incorrect USDC address in `Deploy.s.sol`](#H-03)

- ## Low Risk Findings
  - ### [L-01. Fees are too low, transaction to retrieve them costs more than the earnings](#L-01)

# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #14

### Dates: Apr 25th, 2024 - May 2nd, 2024

[See more contest details here](https://www.codehawks.com/contests/clvb821kr0001jzdbi6ggixb0)

# <a id='results-summary'></a>Results Summary

### Number of findings:

- High: 3
- Medium: 0
- Low: 1

# High Risk Findings

## <a id='H-01'></a>H-01. Users can claim multiple times and steal all funds.

### Relevant GitHub Links

https://github.com/Cyfrin/2024-04-airdropper/blob/main/src/MerkleAirdrop.sol#L30C5-L40C6

## Description

The `claim` function in `MerkleAirdrop.sol` allows a user to collect their airdrop. However, it doesn't account for whether a user has already claimed their airdrop. This allows any user with a valid proof to claim multiple times and steal all the remaining funds in the contract.

Since the `account` is specified by the sender, anyone can drain the contract by sending all funds to any valid user in the Merkle root tree.

```javascript
    function claim(
        address account,
        uint256 amount,
        bytes32[] calldata merkleProof
    ) external payable {
        if (msg.value != FEE) {
            revert MerkleAirdrop__InvalidFeeAmount();
        }
        bytes32 leaf = keccak256(
            bytes.concat(keccak256(abi.encode(account, amount)))
        );
        if (!MerkleProof.verify(merkleProof, i_merkleRoot, leaf)) {
            revert MerkleAirdrop__InvalidProof();
        }
        emit Claimed(account, amount);
        i_airdropToken.safeTransfer(account, amount);
    }
```

## Risk

**Likelyhood**: High

- Any user (sending to an airdrop user), at any time.

**Impact**: High

- All funds can be stolen.

## Proof of Concept

<details>

<summary>Foundry PoC to add in `MerkleAirdropTest.t.sol` </summary>

```javascript
    function testUsersCanStealAllAirdrop() public {
        uint256 startingBalance = token.balanceOf(collectorOne);
        vm.deal(collectorOne, airdrop.getFee() * 4);

        vm.startPrank(collectorOne);
        for (uint i = 0; i < 4; i++) {
            airdrop.claim{value: airdrop.getFee()}(
                collectorOne,
                amountToCollect,
                proof
            );
        }
        vm.stopPrank();

        uint256 endingBalance = token.balanceOf(collectorOne);
        assertEq(endingBalance - startingBalance, amountToCollect * 4);
    }
```

</details>

## Recommended Mitigation

Create a `mapping` to keep track of users' claims and prevent any user to claim more than once.

## <a id='H-02'></a>H-02. Incorrect Merkle root due to incorrect decimals at the tree's creation

### Relevant GitHub Links

https://github.com/Cyfrin/2024-04-airdropper/blob/main/script/Deploy.s.sol#L11

## Description

In `Deploy.s.sol`, the Airdrop is created using the Merkle root from `makeMerkle.js`. The issue arises because the tree is created with 18 decimals in the JavaScript file, while all Solidity contracts assume amounts have 6 decimals. Given that only 100(e6) USDC are sent to the Airdrop contract, no user can retrieve a number with 18 decimals due to insufficient funds. This prevents users from claiming rewards because the Merkle proof won't pass with a 6-decimal number.

Moreover, the correct root is used in the test file.

<details>

<summary>`Deploy.s.sol` </summary>

```javascript
contract Deploy is Script {
    address public s_zkSyncUSDC = 0x1D17CbCf0D6d143135be902365d2e5E2a16538d4;
    bytes32 public s_merkleRoot =

0

xf69aaa25bd4dd10deb2ccd8235266f7cc815f6e9d539e9f4d47cae16e0c36a05;
    // 4 users, 25 USDC each
    uint256 public s_amountToAirdrop = 4 * (25 * 1e6);

    // Deploy the airdropper
    function run() public {
        vm.startBroadcast();
        MerkleAirdrop airdrop = deployMerkleDropper(s_merkleRoot, IERC20(s_zkSyncUSDC));
        // Send USDC -> Merkle Air Dropper
        IERC20(0x1d17CBcF0D6D143135aE902365D2E5e2A16538D4).transfer(address(airdrop), s_amountToAirdrop);
        vm.stopBroadcast();
    }

    function deployMerkleDropper(bytes32 merkleRoot, IERC20 zkSyncUSDC) public returns (MerkleAirdrop) {
        return (new MerkleAirdrop(merkleRoot, zkSyncUSDC));
    }
}
```

</details>

<details>

<summary>`makeMerkle.js` </summary>

```javascript
const amount = (25 * 1e18).toString();
const userToGetProofOf = "0x20F41376c713072937eb02Be70ee1eD0D639966C";

// (1)
const values = [
  [userToGetProofOf, amount],
  ["0x277D26a45Add5775F21256159F089769892CEa5B", amount],
  ["0x0c8Ca207e27a1a8224D1b602bf856479b03319e7", amount],
  ["0xf6dBa02C01AF48Cf926579F77C9f874Ca640D91D", amount],
];

/*//////////////////////////////////////////////////////////////
                            PROCESS
//////////////////////////////////////////////////////////////*/
// (2)
const tree = StandardMerkleTree.of(values, ["address", "uint256"]);
```

</details>

The following file shows the generated root, which is the same as the one used in the deployment script.

<details>

<summary>`tree.json` </summary>

`{"format":"standard-v1","tree":["0xf69aaa25bd4dd10deb2ccd8235266f7cc815f6e9d539e9f4d47cae16e0c36a05",...`

</details>

## Risk

**Likelihood**: High

- The contract will never send USDC to users unless the user deposits 100e12 USDC into it.

**Impact**:

- Loss of funds, which become stuck in the Airdrop contract.

## Recommended Mitigation

Recreate a valid root with 6 decimals (like USDC) and replace the one in the deployment script.
For reference, the calculated Merkle root on my end is `0x3b2e22da63ae414086bec9c9da6b685f790c6fab200c7918f2879f08793d77bd`, which matches the one in the test file.

## <a id='H-03'></a>H-03. Incorrect USDC address in `Deploy.s.sol`

### Relevant GitHub Links

https://github.com/Cyfrin/2024-04-airdropper/blob/main/script/Deploy.s.sol

## Description

Two different addresses are used for USDC in the deployment script. The first one, used in the MerkleAirdrop contract creation, is incorrect, leading to an inability to claim the airdrop. The second one is correct and sends the amount of the airdrop. Since the owner can only retrieve the fee (ether), the USDC airdrop will be stuck in the contract.

1st address on ZkSync explorer (incorrect one): https://explorer.zksync.io/address/0x1D17CbCf0D6d143135be902365d2e5E2a16538d4
2nd address on ZkSync explorer (correct one): https://explorer.zksync.io/address/0x1d17CBcF0D6D143135aE902365D2E5e2A16538D4

```javascript
contract Deploy is Script {
@>    address public s_zkSyncUSDC = 0x1D17CbCf0D6d143135be902365d2e5E2a16538d4;
    bytes32 public s_merkleRoot = 0xf69aaa25bd4dd10deb2ccd8235266f7cc815f6e9d539e9f4d47cae16e0c36a05;
    // 4 users, 25 USDC each
    uint256 public s_amountToAirdrop = 4 * (25 * 1e6);

    // Deploy the airdropper
    function run() public {
        vm.startBroadcast();
@>        MerkleAirdrop airdrop = deployMerkleDropper(s_merkleRoot, IERC20(s_zkSyncUSDC));
        // Send USDC -> Merkle Air Dropper
@>        IERC20(0x1d17CBcF0D6D143135aE902365D2E5e2A16538D4).transfer(address(airdrop), s_amountToAirdrop);
        vm.stopBroadcast();
    }

    function deployMerkleDropper(bytes32 merkleRoot, IERC20 zkSyncUSDC) public returns (MerkleAirdrop) {
        return (new MerkleAirdrop(merkleRoot, zkSyncUSDC));
    }
}
```

## Risk

**Likelyhood**: High

- The contract will never send USDC to users.

**Impact**: High

- Loss of funds, stuck in the `MerkleAirdrop` contract.

## Recommended Mitigation

- Replace the `s_zkSyncUSDC` contract address with the correct one and use this variable instead of using the hexadecimal address.
- Add fork tests to verify with external contracts (USDC).

```diff
contract Deploy is Script {
-    address public s_zkSyncUSDC = 0x1D17CbCf0D6d143135be902365d2e5E2a16538d4;
+    address public s_zkSyncUSDC = 0x1d17CBcF0D6D143135aE902365D2E5e2A16538D4;
    bytes32 public s_merkleRoot = 0xf69aaa25bd4dd10deb2ccd8235266f7cc815f6e9d539e9f4d47cae16e0c36a05;
    // 4 users, 25 USDC each
    uint256 public s_amountToAirdrop = 4 * (25 * 1e6);

    // Deploy the airdropper
    function run() public {
        vm.startBroadcast();
        MerkleAirdrop airdrop = deployMerkleDropper(s_merkleRoot, IERC20(s_zkSyncUSDC));
        // Send USDC -> Merkle Air Dropper
-        IERC20(0x1d17CBcF0D6D143135aE902365D2E5e2A16538D4).transfer(address(airdrop), s_amountToAirdrop);
+        IERC20(s_zkSyncUSDC).transfer(address(airdrop), s_amountToAirdrop);
        vm.stopBroadcast();
    }

    function deployMerkleDropper(bytes32 merkleRoot, IERC20 zkSyncUSDC) public returns (MerkleAirdrop) {
        return (new MerkleAirdrop(merkleRoot, zkSyncUSDC));
    }
}
```

# Low Risk Findings

## <a id='L-01'></a>L-01. Fees are too low, transaction to retrieve them costs more than the earnings

### Relevant GitHub Links

https://github.com/Cyfrin/2024-04-airdropper/blob/main/src/MerkleAirdrop.sol#L15

## Description

The protocol asks for 1e9 fees per claim, which lead at most at 4 Gwei if all users claim their airdrop before the `claimFees` function is called. However price of the transaction on ZkSync will be higher. The price per used gas is 0.025 gwei on ZkSync (recently this has been a pretty constant number) and the function will consume about 8700 gas according to Foundry. So, the transaction will cost about 217 Gwei, which is 57 times larger than the collected fees.

## Risk

**Likelyhood**: High

- The owner will always lose money collecting the fees.

**Impact**: Low/Medium

- No earning for the protocol
- Loss of funds to collect fees.

## Recommended Mitigation

Increase the fees or remove this functionnality.
