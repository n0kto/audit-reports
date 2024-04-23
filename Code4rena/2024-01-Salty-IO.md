Anyone can delay or prevent the liquidation of their collaterals
bug
3 (High Risk)
satisfactory
duplicate-312
Lines of code
https://github.com/code-423n4/2024-01-salty/blob/main/src/stable/CollateralAndLiquidity.sol#L154

Vulnerability details
Description
In the CollateralAndLiquidity::liquidateUser function, the _decreaseUserShare function is employed with the cooldown boolean set to true. The problem arises from this cooldown being contingent on the user undergoing liquidation. If the user injects additional collaterals before the liquidation, they effectively reset the cooldown, thereby staving off the liquidation of their collaterals for a specific duration. Furthermore, post-liquidation, the user facing liquidation is barred from adding new liquidity for one hour, as the cooldown resets each time this boolean is set to true.

Exploiting this vulnerability enables an attacker to frontrun liquidations until the asset's price rebounds or the cost becomes prohibitively high. The cost for combating liquidation is theoretically 5% of the remaining collateral's current price, as this is the portion allocated to liquidators. However, there is a ceiling of $500 for liquidators, meaning that beyond $10k, the cost diminishes. For instance, if the collateral price stands at $100k, the attacker only needs to pay 0.5% ($500) to safeguard their collaterals. Given that liquidators, who do not earn money, have no incentive to participate in liquidations exceeding this percentage or amount of reward.

This susceptibility could result in a loss of funds for the protocol because one hour is a substantial timeframe in the crypto market, and even a modest 5% decrease in price can lead to a loss of funds:

When a liquidator successfully executes a liquidation, they are entitled to 5% of the total collaterals liquidated (which are at most 110% of the borrowed USDS). If the remaining collaterals only cover 105% of the borrowed USDS value, it implies that the liquidator receives 5.25% of the current collateral’s price (with a maximum limit of $500). Consequently, only the remaining 99.75% of the collateral's actual price goes to the protocol.

    function liquidateUser( address wallet ) external nonReentrant
    {
        ...
@>        _decreaseUserShare( wallet, collateralPoolID, userCollateralAmount, true );
        ...
    }
Impact
Likelihood: High

Any user adding collateral to the pool before liquidation, attempting to avoid liquidation.
Any attacker aware of the flaws and adding a small amount of money before every liquidation.
Impact: High

Denial of service for a liquidation for a minimum of one hour, and more if the attacker wants to keep their collaterals. It will cost at most $500 if the collateral's current price is higher than $10k, else 5% of the collateral's current price, every hour.
Denial of service for one hour for a user wanting to add liquidities after a liquidation.
Loss of funds if the price goes down during this one hour (or more). A minimal decrease of 5% of the collaterals' price is enough to result in a loss of funds.
Proof of Concept
Foundry PoC added in CollateralAndLiquidity.t.sol

function testUnlimitedCooldownToLiquidatePosition() public {
    // Bob deposits collateral so Alice can be liquidated
    vm.startPrank(bob);
    collateralAndLiquidity.depositCollateralAndIncreaseShare(
        wbtc.balanceOf(bob),
        weth.balanceOf(bob),
        0,
        block.timestamp,
        false
    );
    vm.stopPrank();
    // Alice will deposit all her collaterals and borrow the maximum value of USDS
    _depositHalfCollateralAndBorrowMax(alice);

    vm.warp(block.timestamp + 1 hours);

    _crashCollateralPrice();

    assertTrue(collateralAndLiquidity.canUserBeLiquidated(alice));

    // Alice adds before being liquidated
    vm.prank(alice);
    collateralAndLiquidity.depositCollateralAndIncreaseShare(
        1 * 10 ** 8,
        1 ether,
        0,
        block.timestamp,
        false
    );

    // Even if she is undercollateralized after adding, she is protected from liquidation
    assertTrue(collateralAndLiquidity.canUserBeLiquidated(alice));

    // Trying to liquidate Alice's position
    vm.prank(bob);

    // Will fail for 1 hour.
    // This is obviously not an expected revert; it is here for the second one to fail
    vm.expectRevert();
    collateralAndLiquidity.liquidateUser(alice);

    vm.warp(block.timestamp + 1 hours);

    // After one hour, Alice can add new liquidity, frontrunning any liquidation
    // to prevent their collateral from being lost
    vm.prank(alice);
    collateralAndLiquidity.depositCollateralAndIncreaseShare(
        1 * 10 ** 8,
        1 ether,
        0,
        block.timestamp,
        false
    );

    vm.prank(bob);
    // Will fail for 1 hour more.
    collateralAndLiquidity.liquidateUser(alice);
}
Recommended Mitigation
Since there is no reason for an external liquidator to interfere, or be interrupted by the cooldown of another user, the better solution seems to set this cooldown boolean to false for the liquidateUser function:

function liquidateUser(address wallet) external nonReentrant
{
    ...
-   _decreaseUserShare(wallet, collateralPoolID, userCollateralAmount, true);
+   _decreaseUserShare(wallet, collateralPoolID, userCollateralAmount, false);
    ...
}




Denial of Service in `ManagedWallet` Prevents Future Proposals
bug
2 (Med Risk)
downgraded by judge
satisfactory
duplicate-838
Lines of code
https://github.com/code-423n4/2024-01-salty/blob/main/src/ManagedWallet.sol#L59-L69

Vulnerability details
Description
The ManagedWallet contract is utilized within the protocol to designate the main wallet of the protocol team. This contract implements a function to change the wallet in case of problems.

During a proposal, when calling the proposalWallets function, a require statement checks if a new wallet has already been proposed:

        // Ensure we are not overwriting a previous proposal (as only the confirmationWallet can reject proposals)
        require(
@>            proposedMainWallet == address(0),
            "Cannot overwrite a non-zero proposed mainWallet."
        );
However, in the case of the rejection of a proposal, the proposedMainWallet variable is never set to 0; only the activeTimelock is correctly set:

    receive() external payable {
        require(msg.sender == confirmationWallet, "Invalid sender");

        if (msg.value >= .05 ether)
            activeTimelock = block.timestamp + TIMELOCK_DURATION; // establish the timelock
        else activeTimelock = type(uint256).max; // effectively never
@>
    }
Impact
Likelihood: High

Any rejection of a proposal triggers the bug.
Impact: High

Denial of service for the proposeWallets function, preventing anyone from changing the wallet.
The ManagedWallet becomes immutable in the ExchangeConfig contract, hindering any new deployment since ExchangeConfig is widely used in other contracts.
In case the main wallet is compromised (which is the primary reason for the implementation of a change mechanism), an attacker can prevent any wallet change.
Proof of Concept
function testRefuseProposalAndBlockFutureChanges() public {
    // Set up the initial state with main and confirmation wallets
    address initialMainWallet = alice;
    address initialConfirmationWallet = address(0x2222);
    ManagedWallet managedWallet = new ManagedWallet(
        initialMainWallet,
        initialConfirmationWallet
    );

    // Set up the proposed main and confirmation wallets
    address newMainWallet = address(0x3333);
    address newConfirmationWallet = address(0x4444);

    // Prank as the initial main wallet to propose the new wallets
    vm.startPrank(initialMainWallet);
    managedWallet.proposeWallets(newMainWallet, newConfirmationWallet);
    vm.stopPrank();

    // Prank as the current confirmation wallet and send ether to refuse the proposal
    uint256 sentValue = 0.001 ether;
    vm.prank(initialConfirmationWallet);
    vm.deal(initialConfirmationWallet, sentValue);
    (bool success, ) = address(managedWallet).call{value: sentValue}("");
    assertTrue(success, "Cancellation of wallet proposal failed");

    // Set up new proposed wallets
    newMainWallet = address(0x5555);
    newConfirmationWallet = address(0x6666);

    // will revert : Cannot overwrite non-zero proposed mainWallet.
    vm.prank(initialMainWallet);
    managedWallet.proposeWallets(newMainWallet, newConfirmationWallet);
}
Recommended Mitigation
Set the proposedMainWallet to address(0) in the receive function.

    receive() external payable {
        require(msg.sender == confirmationWallet, "Invalid sender");

        if (msg.value >= .05 ether)
            activeTimelock = block.timestamp + TIMELOCK_DURATION; // establish the timelock
-        else activeTimelock = type(uint256).max; // effectively never
+        else {
+            activeTimelock = type(uint256).max; // effectively never
+            proposedMainWallet = address(0);
+        }
    }






# `PoolConfig::whitelistPool` can return PoolID to save gas.

## Description

The `DAO::_finalizeTokenWhitelisting` function whitelists pools using `poolsConfig.whitelistPool`. This function already retrieves the poolID, but it doesn't return it. Modifying `PoolConfig::whitelistPool` to return the poolID can save approximately 1838 gas units per call, resulting in cost savings.

These savings can be significant, especially considering the current gas prices. With an average gas price of 20 gwei per unit and the price of Ether at \$2250, this modification can save more than \$0.08 for each call. It's important to note that `PoolConfig::whitelistPool` is only called in `DAO::_finalizeTokenWhitelisting`, which doesn’t imply a big refactor.

```javascript
    function _finalizeTokenWhitelisting(uint256 ballotID) internal {
        if (proposals.ballotIsApproved(ballotID)) {
            ...
@>            poolsConfig.whitelistPool(
@>                pools,
@>                IERC20(ballot.address1),
@>                exchangeConfig.wbtc()
@>            );
@>            poolsConfig.whitelistPool(
@>                pools,
@>                IERC20(ballot.address1),
@>                exchangeConfig.weth()
@>            );

@>            bytes32 pool1 = PoolUtils._poolID(
@>                IERC20(ballot.address1),
@>                exchangeConfig.wbtc()
@>            );
@>            bytes32 pool2 = PoolUtils._poolID(
@>                IERC20(ballot.address1),
@>               exchangeConfig.weth()
@>            );
            ...
        }
        ...
    }
```

## Impact

Optimizes gas consumption.

## Proof of Concept

### Foundry PoC added to `DAO.t.sol`

```javascript
function testWhitelistTokenApprovedGasEfficiency() public {
    vm.startPrank(alice);
    staking.stakeSALT(1000000 ether);

    IERC20 token = new TestERC20("TEST", 18);

    proposals.proposeTokenWhitelisting(token, "", "");

    uint256 ballotID = 1;
    proposals.castVote(ballotID, Vote.YES);

    // Increase block time to finalize the ballot
    vm.warp(block.timestamp + 11 days);

    // Test Parameter Ballot finalization
    salt.transfer(address(dao), 399999 ether);
    salt.transfer(address(dao), 5 ether);

    uint gas_before = gasleft();
    dao.finalizeBallot(ballotID);
    uint gas_after = gasleft();
    console.log(gas_before - gas_after);
}
```

## Recommended Mitigation

**IPoolsConfig.sol**

```diff
interface IPoolsConfig {
    function whitelistPool(
        IPools pools,
        IERC20 tokenA,
        IERC20 tokenB
-    ) external; // onlyOwner
+    ) external returns (bytes32); // onlyOwner

    ...
}
```

**PoolsConfig.sol**

```diff
function whitelistPool(
    IPools pools,
    IERC20 tokenA,
    IERC20 tokenB
-) external onlyOwner {
+) external onlyOwner returns (bytes32 poolID) {
    require(
        _whitelist.length() < maximumWhitelistedPools,
        "Maximum number of whitelisted pools already reached"
    );
    require(tokenA != tokenB, "tokenA and tokenB cannot be the same token");
-   bytes32 poolID = PoolUtils._poolID(tokenA, tokenB);
+    poolID = PoolUtils._poolID(tokenA, tokenB);

    // Add to the whitelist and remember the underlying tokens for the pool
    _whitelist.add(poolID);
    underlyingPoolTokens[poolID] = TokenPair(tokenA, tokenB);

    // Make sure that the cached arbitrage indicies in PoolStats are updated
    pools.updateArbitrageIndicies();

    emit PoolWhitelisted(address(tokenA), address(tokenB));
}
```

**DAO.sol**

```diff
    function _finalizeTokenWhitelisting(uint256 ballotID) internal {
        if (proposals.ballotIsApproved(ballotID)) {
            ...
-            poolsConfig.whitelistPool(
+            bytes32 pool1 = poolsConfig.whitelistPool(
                pools,
                IERC20(ballot.address1),
                exchangeConfig.wbtc()
            );
-            poolsConfig.whitelistPool(
+            bytes32 pool2 = poolsConfig.whitelistPool(
                pools,
                IERC20(ballot.address1),
                exchangeConfig.weth()
            );

-            bytes32 pool1 = PoolUtils._poolID(
-                IERC20(ballot.address1),
-                exchangeConfig.wbtc()
-            );
-            bytes32 pool2 = PoolUtils._poolID(
-                IERC20(ballot.address1),
-               exchangeConfig.weth()
-            );
            ...
        }
        ...
    }
```