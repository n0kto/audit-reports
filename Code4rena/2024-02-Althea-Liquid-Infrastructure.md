Inclusion of address(0) in `holders array on token burn
bug
3 (High Risk)
satisfactory
duplicate-77
Lines of code
https://github.com/code-423n4/2024-02-althea-liquid-infrastructure/blob/main/liquid-infrastructure/contracts/LiquidInfrastructureERC20.sol#L127-L146

Vulnerability details
Description
In the _beforeTokenTransfer override, any address receiving the LiquidInfrastructureERC20.sol will be added to the holders array. The issue is that all burning functions also use this internal function and will consequently add address(0) to the array.

    function _beforeTokenTransfer(
        address from,
        address to,
        uint256 amount
    ) internal virtual override {
        ...
@>        bool exists = (this.balanceOf(to) != 0);
@>        if (!exists) {
@>            holders.push(to);
@>        }
    }
Impact
Likelihood: High

Occurs whenever tokens are burned or transferred to address(0).
Impact: Medium

address(0) won't collect any rewards until it is approved via "approveHolder". However, since disapproved holders are not considered during reward calculation, each distribution will allocate a portion for any disapproved holder (including address(0)), preventing other users from collecting the entirety of rewards. Additionally, this may leave residual rewards in the contract when remaining rewards are less than the totalSupply of LiquidInfrastructureERC20: uint256 entitlement = balance / supply;
Proof of Concept
Foundry PoC:

contract PoC is Test {
    address owner = makeAddr("owner");
    LiquidInfrastructureERC20 liquidERC20;

    address[] public erc20s;

    function setUp() public {
        address[] memory holders;
        address[] memory NFTs;
        TestERC20A erc20a = new TestERC20A();
        erc20s.push(address(erc20a));
        TestERC20B erc20b = new TestERC20B();
        erc20s.push(address(erc20b));
        TestERC20C erc20c = new TestERC20C();
        erc20s.push(address(erc20c));
        vm.prank(owner);
        liquidERC20 = new LiquidInfrastructureERC20(
            "Infra",
            "INFRA",
            NFTs,
            holders,
            500,
            erc20s
        );
    }

    function test_0IsAHolderAfterBurn() public {
        address user = makeAddr("user");
        vm.startPrank(owner);
        liquidERC20.approveHolder(user);

        // user is added as the first holder during _afterTransfer
        liquidERC20.mint(user, 2e18);
        vm.stopPrank();

        console.log(liquidERC20.holders(0));

        vm.startPrank(user);
        // burn tokens will push address(0) in holders during _afterTransfer
        liquidERC20.burnAndDistribute(1e18);

        // it prints address(0) !!
        console.log(liquidERC20.holders(1));
        vm.stopPrank();
    }
}
Recommended Mitigation
Add a holder only if the receiver is not address(0).
A possible solution:

    function _beforeTokenTransfer(
        address from,
        address to,
        uint256 amount
    ) internal virtual override {
        require(!LockedForDistribution, "distribution in progress");
        if (!(to == address(0))) {
            require(
                isApprovedHolder(to),
                "receiver not approved to hold the token"
            );
+            bool exists = (this.balanceOf(to) != 0);
+            if (!exists) {
+                holders.push(to);
+            }
        }
        if (from == address(0) || to == address(0)) {
            _beforeMintOrBurn();
        }
-        bool exists = (this.balanceOf(to) != 0);
-        if (!exists) {
-            holders.push(to);
-        }
    }



Distribution fails to exclude disapproved holders
bug
2 (Med Risk)
satisfactory
duplicate-703
Lines of code
https://github.com/code-423n4/2024-02-althea-liquid-infrastructure/blob/main/liquid-infrastructure/contracts/LiquidInfrastructureERC20.sol#L216-L227
https://github.com/code-423n4/2024-02-althea-liquid-infrastructure/blob/main/liquid-infrastructure/contracts/LiquidInfrastructureERC20.sol#L270-L277

Vulnerability details
Description
The distribution process, specifically the _beginDistribution function, calculates total rewards per collected stablecoin without excluding disapproved holders. The formula is as follows:

function _beginDistribution() internal {
        ...
        uint256 supply = this.totalSupply();
        for (uint i = 0; i < distributableERC20s.length; i++) {
            uint256 balance = IERC20(distributableERC20s[i]).balanceOf(
                address(this)
            );
@>            uint256 entitlement = balance / supply;
            erc20EntitlementPerUnit.push(entitlement);
        }
        ...
    }
However, the issue arises in the distribute function, where disapproved holders are excluded from receiving rewards:

    function distribute(uint256 numDistributions) public nonReentrant {
        ...
        for (i = nextDistributionRecipient; i < limit; i++) {
            address recipient = holders[i];
@>            if (isApprovedHolder(recipient)) {
                ...
            }
        }
        ...
    }
This discrepancy results in disapproved holders not receiving rewards but their portion remaining in the contract, eventually leading to distribution process corruption and potential dust accumulation in the contract.

Impact
Likelihood: Medium/High

Occurs as soon as one holder is disapproved.
Impact: Medium

Corruption in the distribution process until disapproved holders sell or burn their tokens.
Potential accumulation of tokens in the contract due to the absence of a dust collector.
Proof of Concept
Foundry PoC:

contract PoC is Test {
    address owner = makeAddr("owner");
    LiquidInfrastructureERC20 liquidERC20;

    address[] public erc20s;

    function setUp() public {
        address[] memory holders;
        address[] memory NFTs;
        TestERC20A erc20a = new TestERC20A();
        erc20s.push(address(erc20a));
        TestERC20B erc20b = new TestERC20B();
        erc20s.push(address(erc20b));
        TestERC20C erc20c = new TestERC20C();
        erc20s.push(address(erc20c));
        vm.prank(owner);
        liquidERC20 = new LiquidInfrastructureERC20(
            "Infra",
            "INFRA",
            NFTs,
            holders,
            500,
            erc20s
        );
    }

    function test_disapprovedHoldersWasteOthersMoney() public {
        address user = makeAddr("user");
        address user2 = makeAddr("user2");
        vm.startPrank(owner);
        liquidERC20.approveHolder(user);
        liquidERC20.approveHolder(user2);

        // 2 users receive both 50% of the supply.
        liquidERC20.mint(user, 1e18);
        liquidERC20.mint(user2, 1e18);
        vm.stopPrank();

        // LiquidERC20 collect fees/money
        deal(erc20s[0], address(liquidERC20), 1000e18);
        assertTrue(
            TestERC20A(erc20s[0]).balanceOf(address(liquidERC20)) == 1000e18
        );

        vm.roll(600);
        vm.prank(owner);
        liquidERC20.disapproveHolder(user);

        liquidERC20.distributeToAllHolders();

        // It remains all the part of the disapproved holder!
        assertTrue(
            TestERC20A(erc20s[0]).balanceOf(address(liquidERC20)) == 500e18
        );
    }
}
Recommended Mitigation
Exclude disapproved holders from the rewarding formula by not including their supply.
Implement a locker in disapproveHolder during distribution to prevent new formulas from having bugs.
Implement a dust collector to collect remaining tokens when the round-down division leaves tokens in the contract.