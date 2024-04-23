# Final report of the contest: not written by me! I was the designer of this challenge.

# First Flight #9: Soulmate - Findings Report

# Table of contents

- ### [Contest Summary](#contest-summary)
- ### [Results Summary](#results-summary)
- ## High Risk Findings
  - [H-01. `Staking::claimRewards()` function incorrectly tracks user deposits leading to incorrect distribution of rewards.](#H-01)
  - [H-02. Lack of validations at `Airdrop::claim()` allows attackers to claim lovetokens without a Soulbound NFT.](#H-02)
  - [H-03. `Staking.claimRewards()` does not check that `msg.sender` have a soulmate leads to withdraw all LoveTokens minted for `StakingVault`](#H-03)
  - [H-04. Wrong check for divorce status in `Airdrop.claim()`](#H-04)
- ## Medium Risk Findings
  - [M-01. **Anyone can send a message through the `Soulmate::writeMessageInSharedSpace` so the soulmates with the nft Id0 will have messages sent on his behalf or have override messages**](#M-01)
  - [M-02. User can couple with themself, being able to claim love token. soulmate::mintSoulmateToken](#M-02)
- ## Low Risk Findings
  - [L-01. The event SoulmateAreReunited is triggered with incorrect parameters.](#L-01)
  - [L-02. Staking Rewards Forfeited Upon Deposit Withdrawal](#L-02)
  - [L-03. Misleading Total Souls Count due to Unpaired and Self-Paired Users](#L-03)
  - [L-04. `Soulmate::mintSoulmateToken` fails to validate recipient addresses in its `_mint` function, risking a DOS attack on the token.](#L-04)
  - [L-05. Wrong data emitted in events of `LoveToken`](#L-05)

# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #9

### Dates: Feb 8th, 2024 - Feb 15th, 2024

[See more contest details here](https://www.codehawks.com/contests/clsathvgg0005yhmxmoe455mm)

# <a id='results-summary'></a>Results Summary

### Number of findings:

- High: 4
- Medium: 2
- Low: 5

# High Risk Findings

## <a id='H-01'></a>H-01. `Staking::claimRewards()` function incorrectly tracks user deposits leading to incorrect distribution of rewards.

_Submitted by [kiqo](/profile/clrrtiujv0000ld28mhpbea15), [abhishekthakur](/profile/clkaqh5590000k108p39ktfwl), [Keyword](/profile/clq9wfw3c0000ulgq5mkbqnmf), [jenniferSun](/profile/clsb1ozmi0000unx8zvm2lx40), [0x4non](/profile/clk3udrho0004mb08dm6y7y17), [i3arba](/profile/clpbtp4g6000nyjx66awyzd9w), [robertodf99](/profile/clscdki8s0001sbgc9ukl8ewc), [azanux](/profile/clk45q9ry0000l5080kf923kw), [KiteWeb3](/profile/clk9pzw3j000smh08313lj91l), [0xTheBlackPanther](/profile/clnca1ftl0000lf08bfytq099), [0xlrivo](/profile/clqo26w180006cl27lldffow6), [shikhar229169](/profile/clk3yh639002emf08ywok1hzf), [GlitchicaL](/profile/clru42we4000t8usowjglteia), [mdasifahamed](/profile/clsbm5lfy0000ent4dux9f2bc), [CarlosAlbaWork](/profile/clrw1ekq80007u9z6ke2mx6yr), [Josh4324](/profile/clk75kd79000gjw08o1v6ycq7), [SHJO](/profile/clsj69010000096lzk1vu462s), [Awacs](/profile/clo47qxsq001dm808b0vjbta1), [Coffee](/profile/clln3vyj7000cml0877uhlb7j), [Joshuajee](/profile/clp8tjvhb0000r61pfr2owzuy), [Honour](/profile/clrc98bu4000011oz4po0q5dd), [Louis](/profile/clloixi3x0000la08i46r5hc8), [naman1729](/profile/clk41lnhu005wla08y1k4zaom), [paprikrumplikas](/profile/clrj23lnp006g14oum8qxro25), [Nocturnus](/profile/clk6gsllo0000mn08rjvbjy0x), [0xloscar01](/profile/cllgowxgy0002la08qi9bhab4). Selected submission by: [GlitchicaL](/profile/clru42we4000t8usowjglteia)._  


### Relevant GitHub Links

https://github.com/Cyfrin/2024-02-soulmate/blob/main/src/Staking.sol#L74-L76

## Summary

The `Staking::claimRewards()` function incorrectly tracks timestamp for `msg.sender` and allows anyone to claim token rewards without having to wait 1 week since a deposit.

## Vulnerability Details

The `Staking::claimRewards()` function is meant to give tokens to those who have deposited tokens into the contract for a specific time. For example, if 1 token is deposited, they can claim 1 token after 1 week, however the contract does not follow the intended behavior. The issue lies when setting the `lastClaim` local variable:

```javascript
    function claimRewards() public {
        uint256 soulmateId = soulmateContract.ownerToId(msg.sender);
        // first claim
        if (lastClaim[msg.sender] == 0) {
@>          lastClaim[msg.sender] = soulmateContract.idToCreationTimestamp(
@>            soulmateId
@>          );
        }

        // How many weeks passed since the last claim.
        // Thanks to round-down division, it will be the lower amount possible until a week has completly pass.
        uint256 timeInWeeksSinceLastClaim = ((block.timestamp -
            lastClaim[msg.sender]) / 1 weeks);

        if (timeInWeeksSinceLastClaim < 1)
            revert Staking__StakingPeriodTooShort();

        lastClaim[msg.sender] = block.timestamp;

        // Send the same amount of LoveToken as the week waited times the number of token staked
        uint256 amountToClaim = userStakes[msg.sender] *
            timeInWeeksSinceLastClaim;
        loveToken.transferFrom(
            address(stakingVault),
            msg.sender,
            amountToClaim
        );

        emit RewardsClaimed(msg.sender, amountToClaim);
    }
```

When the contract is first called, the function declares and sets the `soulmateId` of `msg.sender` by calling the `Soulmate::ownerToId()` function. This will set the soulmate NFT ID that belongs to `msg.sender`. It's worth noting, if `msg.sender` has never claimed a soulmate via `Soulmate::mintSoulmateToken()` this will set the `soulmateId` to 0, while if the `msg.sender` is awaiting a soulmate, this will set `soulmateId` to the NFT ID that will be minted.

The function proceeds to see if `lastClaim[msg.sender]` equals 0. If we assume this is the first time a soulmate attempts to claim, it will call `Soulmate::idToCreationTimestamp()` function and set `lastClaim[msg.sender]` to the timestamp their soulmate NFT was created. For a user who has no soulmate, this will set `lastClaim[msg.sender]` to the time the first soulmate NFT was created. For a user who is still awaiting a soulmate, it will set `lastClaim[msg.sender]` to 0 as the timestamp of the next soulmate NFT ID doesn't exist yet.

If we then look at how then the `timeInWeeksSinceLastClaim` local variable is set:

```javascript
        uint256 timeInWeeksSinceLastClaim = ((block.timestamp -
            lastClaim[msg.sender]) / 1 weeks);
```

This takes the current `block.timestamp` and minuses it from `lastClaim[msg.sender]`. This results in the time since the soulmate NFT was created, and not when tokens were deposited into the staking contract.

For a user who has never minted a soulmate, This results in the time since the first soulmate NFT was created, and not when tokens were deposited into the staking contract.

In a case where a user is awaiting a soulmate, `block.timestamp - 0` will result in just `block.timestamp`. Thus the calculation for `timeInWeeksSinceLastClaim` will just be `block.timestamp / 1 weeks` (604800). If we assume `block.timestamp` is equal to `X` amount of weeks, this user will be rewarded `X` amount of tokens regardless of when their tokens were deposited.

It's worth noting if no soulmate NFTs have been minted, there should be no way for tokens to be deposited assuming the only way to claim tokens is initially through the `Airdrop::claim()` function.

## Impact

- A soulmate is rewarded tokens based on when their soulmate NFT was created regardless of the time they deposited.
- A user who has never minted a soulmate will be rewarded tokens based on when the first soulmate NFT was created regardless of the time they deposited.
- A user awaiting a soulmate but has deposited tokens can be rewarded tokens based on current `block.timestamp` divided by 604800 (`1 weeks`) regardless of the time they deposited.

## POC

We can imagine the most-likely scenario and how it plays out:

1. User A calls `Soulmate::mintSoulmateToken()`.
2. User B calls `Soulmate::mintSoulmateToken()` and now NFT #0 has been minted.
3. 7 days passes by since NFT #0 was minted.
4. User A calls `Airdrop::claim()` and now has 7 lovetokens.
5. User A transfers 7 lovetokens to user C.
6. User C deposits 7 lovetokens to Staking contract.
7. User C calls `Soulmate::mintSoulmateToken()` and is awaiting a soulmate.
8. User C calls `Staking::claimRewards()`.

Below you can see a POC of the above scenario including how `block.timestamp` affects rewards:

```javascript
    function test_ClaimRewardsWithoutSoulmate() public {
        vm.warp(block.timestamp + 2 weeks);

        _mintOneTokenForBothSoulmates();

        vm.warp(block.timestamp + 1 weeks + 1 seconds);

        vm.prank(soulmate1);
        airdropContract.claim();

        address attacker = makeAddr("attacker");
        uint256 amountToAttackWith = 7 ether;

        vm.prank(soulmate1);
        loveToken.transfer(attacker, amountToAttackWith);

        vm.startPrank(attacker);
        soulmateContract.mintSoulmateToken();
        loveToken.approve(address(stakingContract), amountToAttackWith);
        stakingContract.deposit(amountToAttackWith);
        stakingContract.claimRewards();

        assertTrue(loveToken.balanceOf(attacker) == 21 ether); // 7 tokens deposited * 3 weeks = 21 tokens!
    }
```

## Tools Used

VS Code, Foundry

## Recommendations

Consider creating a `lastDeposit` mapping and setting the mapping in `Staking::deposit()` for a deposit and reference that timestamp in `Staking::claimRewards()` for the very first claim of an user. You'll also want to include a check to make sure a user has deposited. Note that this could "reset" the ability to claim if the user makes multiple deposits prior to their first claim:

```diff
+   error Staking__NoDepositsMade();

+   mapping(address user => uint256 timestamp) public lastDeposit;

    function deposit(uint256 amount) public {
        if (loveToken.balanceOf(address(stakingVault)) == 0)
            revert Staking__NoMoreRewards();
        // No require needed because of overflow protection
        userStakes[msg.sender] += amount;
+       lastDeposit[msg.sender] = block.timestamp;
        loveToken.transferFrom(msg.sender, address(this), amount);

        emit Deposited(msg.sender, amount);
    }

    function claimRewards() public {
-       uint256 soulmateId = soulmateContract.ownerToId(msg.sender);
+       if (lastDeposit[msg.sender] == 0)
+           revert Staking__NoDepositsMade();

        // first claim
        if (lastClaim[msg.sender] == 0) {
-           lastClaim[msg.sender] = soulmateContract.idToCreationTimestamp(
-               soulmateId
-           );
+           lastClaim[msg.sender] = lastDeposit[msg.sender];
        }

        // How many weeks passed since the last claim.
        // Thanks to round-down division, it will be the lower amount possible until a week has completly pass.
        uint256 timeInWeeksSinceLastClaim = ((block.timestamp -
            lastClaim[msg.sender]) / 1 weeks);

        if (timeInWeeksSinceLastClaim < 1)
            revert Staking__StakingPeriodTooShort();

        lastClaim[msg.sender] = block.timestamp;

        // Send the same amount of LoveToken as the week waited times the number of token staked
        uint256 amountToClaim = userStakes[msg.sender] *
            timeInWeeksSinceLastClaim;
        loveToken.transferFrom(
            address(stakingVault),
            msg.sender,
            amountToClaim
        );

        emit RewardsClaimed(msg.sender, amountToClaim);
    }
```

## <a id='H-02'></a>H-02. Lack of validations at `Airdrop::claim()` allows attackers to claim lovetokens without a Soulbound NFT.

_Submitted by [shaflow01](/profile/clscnul8n00003fvj246ieozj), [Clemmos](/profile/clse90cc70000vk7lm32nbpe4), [kiqo](/profile/clrrtiujv0000ld28mhpbea15), [heavenz52](/profile/clpz0w5io000011olqqwo7b3u), [Aizen](/profile/clrep1f8r0008b3fvl30ma1cw), [64xprt](/profile/clkn2d0rl000gmb08ut2gan42), [jenniferSun](/profile/clsb1ozmi0000unx8zvm2lx40), [0x4non](/profile/clk3udrho0004mb08dm6y7y17), [i3arba](/profile/clpbtp4g6000nyjx66awyzd9w), [Joshuajee](/profile/clp8tjvhb0000r61pfr2owzuy), [azanux](/profile/clk45q9ry0000l5080kf923kw), [dimah7](/profile/clqqo7o2v000copafdu0zb10y), [0xTheBlackPanther](/profile/clnca1ftl0000lf08bfytq099), [Poor4ever](/profile/clqed8yto0000xs7svdgqz0cs), [theirrationalone](/profile/clk46mun70016l5082te0md5t), [shikhar229169](/profile/clk3yh639002emf08ywok1hzf), [dougo](/profile/clnvz1v7u0000mg07ikf4cqcd), [Ritos](/profile/clqc7vjma0000jmo2axfqt88m), [GlitchicaL](/profile/clru42we4000t8usowjglteia), [0xWallSecurity](/profile/clqyonbnu000311viljnrqp2s), [0xRolko](/profile/clpjn5tmo0000g3x52xgalrc5), [0xlrivo](/profile/clqo26w180006cl27lldffow6), [0xjoona](/profile/clsjqlgj500007sax9ajdvn25), [SHJO](/profile/clsj69010000096lzk1vu462s), [Turetos](/profile/clof0okll002ila08y4of251r), [mahivasisth](/profile/clk86z1bk000olh08he15prja), [maziXYZ](/profile/clkek71cx0000mg08jybn4zex), [Miki](/profile/clsmlx0nq000a9dnakiuylfsk), [Coffee](/profile/clln3vyj7000cml0877uhlb7j), [stefanlatinovic](/profile/clpbb43ek0014aevzt4shbvx2), [ceseshi](/profile/cln8zm3hz000gmf08kqdt7i5b), [Bube](/profile/clk3y8e9u000cjq08uw5phym7), [0xe4669da](/profile/clpmlbj6r00037zrxiydwa3k6), [Louis](/profile/clloixi3x0000la08i46r5hc8), [EloiManuel](/profile/clq2hor730000b8vl1unuep88), [paprikrumplikas](/profile/clrj23lnp006g14oum8qxro25), [Nocturnus](/profile/clk6gsllo0000mn08rjvbjy0x). Selected submission by: [Turetos](/profile/clof0okll002ila08y4of251r)._  


### Relevant GitHub Links

https://github.com/Cyfrin/2024-02-soulmate/blob/b3f9227942ffd5c443ce6bccaa980fea0304c38f/src/Airdrop.sol#L51

## Summary

The `Airdrop::claim()` function lacks validations to prevent token claiming for users without a Soulbound NFT.
This enables attackers to call the `claim` function and obtain tokens as if they had the first minted Soulbound NFT.

Additionally, there is a flaw in the use of the `Soulmate::isDivorced` function, where the airdrop contract address is used
instead of the user's address. A divorced user can still claim a Soulbound NFT.

## Vulnerability Details

- The `isDivorced` check will always pass since it sends the `msg.sender` of the contract to the function, which consistently returns false for the
  Airdrop contract address.

- `numberOfDaysInCouple` is computed using the timestamp when the NFT was minted, but for the attacker without a minted NFT, it returns the timestamp of the first minted NFT.

```javascript
	function claim() public {
        // No LoveToken for people who don't love their soulmates anymore.
@>      if (soulmateContract.isDivorced()) revert Airdrop__CoupleIsDivorced();

@>			//No validation here if the msg.sender has a Soulbound NFT

        // Calculating since how long soulmates are reunited
@>      uint256 numberOfDaysInCouple = (block.timestamp -
            soulmateContract.idToCreationTimestamp(
                soulmateContract.ownerToId(msg.sender)
            )) / daysInSecond;

     	...
     	...
	}
```

## Impact

An attacker lacking a Soulbound NFT can claim rewards, receiving them as if they have the first minted NFT.

Here's a test demonstrating two separate claims with a 10-day gap between each.

```javascript
	function testClaimNoSoulmate() public {
        console.log("Balance before: ", loveToken.balanceOf(soulmate1) / 10 ** loveToken.decimals());

        vm.warp(block.timestamp + 10 days + 1 seconds);

        vm.prank(soulmate1);
        airdropContract.claim();

        console.log("Balance after claim 1: ", loveToken.balanceOf(soulmate1) / 10 ** loveToken.decimals());

        vm.warp(block.timestamp + 20 days + 1 seconds);

        vm.prank(soulmate1);
        airdropContract.claim();

        console.log("Balance after claim 2: ", loveToken.balanceOf(soulmate1) / 10 ** loveToken.decimals());
   }

```

Output

```
Logs:
  Balance before:  0
  Balance after claim 1:  10
  Balance after claim 2:  30

```

## Tools Used

Manual review and Foundry

## Recommendations

Revise the `Airdrop::claim` function to include a check verifying that the caller possesses a minted Soulbound NFT, and start counting from nextID = 1, so an id of 0 will be used as invalid.
Adjust the `Soulmate::isDivorced` function to accurately validate the msg.sender.

```diff
/// @notice Claim tokens. Every person who have a Soulmate NFT token can claim 1 LoveToken per day.
	function claim() public {
        // No LoveToken for people who don't love their soulmates anymore.
-		if (soulmateContract.isDivorced()) revert Airdrop__CoupleIsDivorced();
+		if (soulmateContract.isDivorced(msg.sender)) revert Airdrop__CoupleIsDivorced();

++      require(soulmateContract.ownerToId(msg.sender) != 0);

        // Calculating since how long soulmates are reunited
        uint256 numberOfDaysInCouple = (block.timestamp -
            soulmateContract.idToCreationTimestamp(
                soulmateContract.ownerToId(msg.sender)
            )) / daysInSecond;

				...
				...
	}
```

Update the `isDivorced` function to correctly handle the msg.sender as a parameter.

```diff
--  function isDivorced() public view returns (bool) {
--      return divorced[msg.sender];
--  }

++  function isDivorced(address soulmate) public view returns (bool) {
++      return divorced[soulmate];
++  }
```

## <a id='H-03'></a>H-03. `Staking.claimRewards()` does not check that `msg.sender` have a soulmate leads to withdraw all LoveTokens minted for `StakingVault`

_Submitted by [shaflow01](/profile/clscnul8n00003fvj246ieozj), [64xprt](/profile/clkn2d0rl000gmb08ut2gan42), [0x4non](/profile/clk3udrho0004mb08dm6y7y17), [KiteWeb3](/profile/clk9pzw3j000smh08313lj91l), [shikhar229169](/profile/clk3yh639002emf08ywok1hzf), [0xlrivo](/profile/clqo26w180006cl27lldffow6), [Poor4ever](/profile/clqed8yto0000xs7svdgqz0cs), [wiasliaw](/profile/cllkdeq9r0000l608mqrmbi2j), [Turetos](/profile/clof0okll002ila08y4of251r), [GlitchicaL](/profile/clru42we4000t8usowjglteia), [maziXYZ](/profile/clkek71cx0000mg08jybn4zex), [Coffee](/profile/clln3vyj7000cml0877uhlb7j), [stefanlatinovic](/profile/clpbb43ek0014aevzt4shbvx2), [ceseshi](/profile/cln8zm3hz000gmf08kqdt7i5b), [Ritos](/profile/clqc7vjma0000jmo2axfqt88m). Selected submission by: [Ritos](/profile/clqc7vjma0000jmo2axfqt88m)._  


### Relevant GitHub Links

https://github.com/Cyfrin/2024-02-soulmate/blob/b3f9227942ffd5c443ce6bccaa980fea0304c38f/src/Staking.sol#L70C1-L99C6

## Vulnerability Details

Contract `Staking`
Function `claimRewards()`
Do not check for `msg.sender` is in `Soulmate.ownerToId` mapping before calculating [`amountToClaim`](https://github.com/Cyfrin/2024-02-soulmate/blob/b3f9227942ffd5c443ce6bccaa980fea0304c38f/src/Staking.sol#L90C1-L91C39) which leads `timeInWeeksSinceLastClaim` equal number of weeks since Jan 01 1970 to `block.timestamp`.  
This leads that any account without soulmate can claim as much tokens as much weeks since 1970 to `block.timestamp` multiplied by `userStakes`.

## POC

1. Attacker generates 9 accounts.
2. For each account call `Airdrop.claim()` which lead to withraw 19765 ether to each contract controlled by attacker
3. For each account call `loveToken.approve(address(stakingContract), 19765 ether)`
4. For each account except last call `stakingContract.deposit()` with 19765 ether  
   4. 1. for the last account `stakingContract.deposit()` with considering of remaining balance `stakingVault`

```javascript
uint stakeContractBalance = loveToken.balanceOf(address(stakingVault));
neededDeposit = stakeContractBalance / 2823;
```

5. For each contract call `stakingContract.claimRewards()`

which leads to withdraw all `LoveToken` the remainder of the division `stakeContractBalance % 2823` minted to `stakingVault` to accounts controlled by attacker.

Add this test to `StakingTest.t.sol` and run via `forge test --mt test_ClaimRewardsWithoutSoulmate()` to see it success.

```javascript
function test_ClaimRewardsWithoutSoulmate() public {
        string[9] memory arr = ["0", "1", "2", "3", "4", "5", "6", "7", "8"];

        vm.warp(1707770320); // timestamp for Feb-12-2024 08:34:59 PM +UTC, 2823 weeks since Jan 01 1970. (UTC)

        for (uint i = 0; i < 9; i++) {
            address lonely = makeAddr(string.concat("lonely", arr[i]));

            vm.prank(lonely);
            airdropContract.claim(); // loveToken.balanceOf(lonely) == 19765 ether

            vm.prank(lonely);
            loveToken.approve(address(stakingContract), 19765 ether);

            uint neededDeposit = 19765 ether;

            if (i == 8) {
                uint stakeContractBalance = loveToken.balanceOf(address(stakingVault));
                neededDeposit = stakeContractBalance / 2823; // stakeContractBalance % 2823 = 1955
            }

            vm.prank(lonely);
            stakingContract.deposit(neededDeposit);


            vm.prank(lonely);
            stakingContract.claimRewards();
        }

        assertTrue(loveToken.balanceOf(address(stakingVault)) == 1955);
    }
```

Output:

```text
Running 1 test for test/unit/StakingTest.t.sol:StakingTest
[PASS] test_ClaimRewardsWithoutSoulmate() (gas: 1245740)
Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 7.70ms

Ran 1 test suites: 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

## Tools Used

Manual review, foundry.

## Recommendations

Make the following changes in `Staking.claimRewards()`

https://github.com/Cyfrin/2024-02-soulmate/blob/b3f9227942ffd5c443ce6bccaa980fea0304c38f/src/Staking.sol#L70C1-L99C6

```diff
    function claimRewards() public {
+      require(soulmateContract.ownerToId(msg.sender) != 0);
        uint256 soulmateId = soulmateContract.ownerToId(msg.sender);
        // first claim
        if (lastClaim[msg.sender] == 0) {
            lastClaim[msg.sender] = soulmateContract.idToCreationTimestamp(
                soulmateId
            );
        }

        // How many weeks passed since the last claim.
        // Thanks to round-down division, it will be the lower amount possible until a week has completly pass.
        uint256 timeInWeeksSinceLastClaim = ((block.timestamp -
            lastClaim[msg.sender]) / 1 weeks);

        if (timeInWeeksSinceLastClaim < 1)
            revert Staking__StakingPeriodTooShort();

        lastClaim[msg.sender] = block.timestamp;

        // Send the same amount of LoveToken as the week waited times the number of token staked
        uint256 amountToClaim = userStakes[msg.sender] *
            timeInWeeksSinceLastClaim;
        loveToken.transferFrom(
            address(stakingVault),
            msg.sender,
            amountToClaim
        );

        emit RewardsClaimed(msg.sender, amountToClaim);
    }
```

## <a id='H-04'></a>H-04. Wrong check for divorce status in `Airdrop.claim()`

_Submitted by [abhishekthakur](/profile/clkaqh5590000k108p39ktfwl), [KiteWeb3](/profile/clk9pzw3j000smh08313lj91l), [spidy730](/profile/clsfy7l4s0000d3ccmm2yvbq9), [64xprt](/profile/clkn2d0rl000gmb08ut2gan42), [i3arba](/profile/clpbtp4g6000nyjx66awyzd9w), [robertodf99](/profile/clscdki8s0001sbgc9ukl8ewc), [azanux](/profile/clk45q9ry0000l5080kf923kw), [dimah7](/profile/clqqo7o2v000copafdu0zb10y), [shikhar229169](/profile/clk3yh639002emf08ywok1hzf), [Ritos](/profile/clqc7vjma0000jmo2axfqt88m), [happyformerlawyer](/profile/clmca6fy60000mp08og4j1koc), [GlitchicaL](/profile/clru42we4000t8usowjglteia), [Bube](/profile/clk3y8e9u000cjq08uw5phym7), [dougo](/profile/clnvz1v7u0000mg07ikf4cqcd), [Turetos](/profile/clof0okll002ila08y4of251r), [octeezy](/profile/clq3dzqi20000t9gtbga6fk0k), [mdasifahamed](/profile/clsbm5lfy0000ent4dux9f2bc), [merv](/profile/clrqlmc79000akrhqk2duq9e9), [Coffee](/profile/clln3vyj7000cml0877uhlb7j), [kryptonomousB](/profile/clmfopjnc0004mg08zhvhv7ue), [stefanlatinovic](/profile/clpbb43ek0014aevzt4shbvx2), [ceseshi](/profile/cln8zm3hz000gmf08kqdt7i5b), [0xe4669da](/profile/clpmlbj6r00037zrxiydwa3k6), [4th05](/profile/clrsi66ll0000o022xf5bcqfg), [Nocturnus](/profile/clk6gsllo0000mn08rjvbjy0x). Selected submission by: [Ritos](/profile/clqc7vjma0000jmo2axfqt88m)._  


### Relevant GitHub Links

https://github.com/Cyfrin/2024-02-soulmate/blob/b3f9227942ffd5c443ce6bccaa980fea0304c38f/src/Airdrop.sol#L53

https://github.com/Cyfrin/2024-02-soulmate/blob/b3f9227942ffd5c443ce6bccaa980fea0304c38f/src/Soulmate.sol#L131C1-L133C6

## Vulnerability Details

Wrong check for divorce status in `Airdrop.claim()` lead to possibility to collect LoveToken even if the user is divorced.

## POC

When we call `soulmateContract.isDivorced()` from `Airdrop.claim()` the `msg.sender` will be Airdrop contract address, who can never be in a `Soulmate.divorced` mapping.

Add this test to `AirdropTest.t.sol` and run via `forge test --mt test_Divorce` to see it success.

```javascript
function test_Divorce() public {
        _mintOneTokenForBothSoulmates();

        vm.prank(soulmate1);

        soulmateContract.getDivorced();
        assertTrue(soulmateContract.isDivorced() == true);

        vm.warp(block.timestamp + 200 days + 1 seconds);

        vm.prank(soulmate1);
        airdropContract.claim();

        assertTrue(loveToken.balanceOf(soulmate1) == 200 ether);

        vm.prank(soulmate2);
        airdropContract.claim();

        assertTrue(loveToken.balanceOf(soulmate2) == 200 ether);
    }
```

Output:

```text
Running 1 test for test/unit/AirdropTest.t.sol:AirdropTest
[PASS] test_Divorce() (gas: 403677)
Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.84ms
```

## Tools Used

Manual review, foundry.

## Recommendations

Make the following changes in `Solmate.isDivorced()`
https://github.com/Cyfrin/2024-02-soulmate/blob/b3f9227942ffd5c443ce6bccaa980fea0304c38f/src/Soulmate.sol#L131C1-L133C6

```diff
-    function isDivorced() public view returns (bool) {
+   function isDivorced(address soulmate) public view returns (bool) {
-        return divorced[msg.sender];
+        return divorced[soulmate];
    }
```

Make the following changes in `ISoulmate`
https://github.com/Cyfrin/2024-02-soulmate/blob/b3f9227942ffd5c443ce6bccaa980fea0304c38f/src/interface/ISoulmate.sol#L28

```diff
interface ISoulmate {
    ...

-    function isDivorced() external view returns (bool);
+   function isDivorced(address soulmate) external view returns (bool);

    ...
}
```

Make the following changes in `Airdrop.claim()`
https://github.com/Cyfrin/2024-02-soulmate/blob/b3f9227942ffd5c443ce6bccaa980fea0304c38f/src/Airdrop.sol#L53

```diff
function claim() public {
        // No LoveToken for people who don't love their soulmates anymore.
-        if (soulmateContract.isDivorced()) revert Airdrop__CoupleIsDivorced();
+       if (soulmateContract.isDivorced(msg.sender)) revert Airdrop__CoupleIsDivorced();

        // Calculating since how long soulmates are reunited
        uint256 numberOfDaysInCouple = (block.timestamp -
            soulmateContract.idToCreationTimestamp(
                soulmateContract.ownerToId(msg.sender)
            )) / daysInSecond;

        uint256 amountAlreadyClaimed = _claimedBy[msg.sender];

        if (
            amountAlreadyClaimed >=
            numberOfDaysInCouple * 10 ** loveToken.decimals()
        ) revert Airdrop__PreviousTokenAlreadyClaimed();

        uint256 tokenAmountToDistribute = (numberOfDaysInCouple *
            10 ** loveToken.decimals()) - amountAlreadyClaimed;

        // Dust collector
        if (
            tokenAmountToDistribute >=
            loveToken.balanceOf(address(airdropVault))
        ) {
            tokenAmountToDistribute = loveToken.balanceOf(
                address(airdropVault)
            );
        }
        _claimedBy[msg.sender] += tokenAmountToDistribute;

        emit TokenClaimed(msg.sender, tokenAmountToDistribute);
        loveToken.transferFrom(
            address(airdropVault),
            msg.sender,
            tokenAmountToDistribute
        );
    }

```

# Medium Risk Findings

## <a id='M-01'></a>M-01. **Anyone can send a message through the `Soulmate::writeMessageInSharedSpace` so the soulmates with the nft Id0 will have messages sent on his behalf or have override messages**

_Submitted by [abhishekthakur](/profile/clkaqh5590000k108p39ktfwl), [kiqo](/profile/clrrtiujv0000ld28mhpbea15), [shaflow01](/profile/clscnul8n00003fvj246ieozj), [heavenz52](/profile/clpz0w5io000011olqqwo7b3u), [64xprt](/profile/clkn2d0rl000gmb08ut2gan42), [jenniferSun](/profile/clsb1ozmi0000unx8zvm2lx40), [Poor4ever](/profile/clqed8yto0000xs7svdgqz0cs), [0x4non](/profile/clk3udrho0004mb08dm6y7y17), [i3arba](/profile/clpbtp4g6000nyjx66awyzd9w), [theirrationalone](/profile/clk46mun70016l5082te0md5t), [shikhar229169](/profile/clk3yh639002emf08ywok1hzf), [APex](/profile/clrmlw12f0000wvmrf4ya6wdv), [KiteWeb3](/profile/clk9pzw3j000smh08313lj91l), [SHJO](/profile/clsj69010000096lzk1vu462s), [octeezy](/profile/clq3dzqi20000t9gtbga6fk0k), [GlitchicaL](/profile/clru42we4000t8usowjglteia), [mahivasisth](/profile/clk86z1bk000olh08he15prja), [maziXYZ](/profile/clkek71cx0000mg08jybn4zex), [ceseshi](/profile/cln8zm3hz000gmf08kqdt7i5b), [Bube](/profile/clk3y8e9u000cjq08uw5phym7), [stefanlatinovic](/profile/clpbb43ek0014aevzt4shbvx2), [CarlosAlbaWork](/profile/clrw1ekq80007u9z6ke2mx6yr), [Zkillua](/profile/clrp7b7bo0000253492oe199q), [Berring](/profile/cls94h0bg0000gyor1gyfu4pt). Selected submission by: [i3arba](/profile/clpbtp4g6000nyjx66awyzd9w)._  


### Relevant GitHub Links

https://github.com/Cyfrin/2024-02-soulmate/blob/b3f9227942ffd5c443ce6bccaa980fea0304c38f/src/Soulmate.sol#L106

- **Anyone can send a message through the `Soulmate::writeMessageInSharedSpace` so the soulmates with the nft Id0 will have messages sent on his behalf or have override messages**

  - **Description:**
    - Anyone can call the function `Soulmate::writeMessageInSharedSpace` and override the message sent by the soulmate nft id0. Nevertheless, anyone can send an offensive message and cause a divorce leading to a loss of the airdrop rights. Considering the correction of "Soulmate being able to withdraw `LoveToken` from the period after divorce, leading to a loss of funds to protocol" finding, this can even be worse.
    - If an offensive message is sent through the breach, one of the soulmates sees, claims the withdrawal, and immediately gets divorced, the other soulmate will be directly private of his airdrop funds.
  - **Impact:**
    - Malicious users can send an offensive message leading to a divorce and loss of `LoveTokens`.
  - **Proof of Code:**

      <details>
      <summary>See the code below. PS: This code is already in your test suit</summary>

    ```javascript
      function test_WriteAndReadSharedSpace() public {
        vm.prank(soulmate1);
        soulmateContract.writeMessageInSharedSpace("Buy some eggs");

        vm.prank(soulmate2);
        string memory message = soulmateContract.readMessageInSharedSpace();

        string[4] memory possibleText = [
            "Buy some eggs, sweetheart",
            "Buy some eggs, darling",
            "Buy some eggs, my dear",
            "Buy some eggs, honey"
        ];
        bool found;
        for (uint i; i < possibleText.length; i++) {
            if (compare(possibleText[i], message)) {
                found = true;
                break;
            }
        }
        console2.log(message);
        assertTrue(found);
        }

    ```

      </details>

  - **Recommendation:**
  - Add a checker in the function. See the example below.

    <details>
    <summary>See the code suggestion below.</summary>

    ```diff
      function writeMessageInSharedSpace(string calldata message) external {
    +     if(soulmateOf[msg.sender] == address(0)) revert Soulmate__OnlyLoversCanSendMessages();
        uint256 id = ownerToId[msg.sender];
        sharedSpace[id] = message;
        emit MessageWrittenInSharedSpace(id, message);
      }
    ```

    </details>

## <a id='M-02'></a>M-02. User can couple with themself, being able to claim love token. soulmate::mintSoulmateToken

_Submitted by [abhishekthakur](/profile/clkaqh5590000k108p39ktfwl), [shaflow01](/profile/clscnul8n00003fvj246ieozj), [64xprt](/profile/clkn2d0rl000gmb08ut2gan42), [jenniferSun](/profile/clsb1ozmi0000unx8zvm2lx40), [Poor4ever](/profile/clqed8yto0000xs7svdgqz0cs), [toufikairane](/profile/clllaggdl002ql708zsctwwhm), [i3arba](/profile/clpbtp4g6000nyjx66awyzd9w), [robertodf99](/profile/clscdki8s0001sbgc9ukl8ewc), [theirrationalone](/profile/clk46mun70016l5082te0md5t), [shikhar229169](/profile/clk3yh639002emf08ywok1hzf), [wiasliaw](/profile/cllkdeq9r0000l608mqrmbi2j), [0xlrivo](/profile/clqo26w180006cl27lldffow6), [CarlosAlbaWork](/profile/clrw1ekq80007u9z6ke2mx6yr), [mdasifahamed](/profile/clsbm5lfy0000ent4dux9f2bc), [SHJO](/profile/clsj69010000096lzk1vu462s), [merv](/profile/clrqlmc79000akrhqk2duq9e9), [Awacs](/profile/clo47qxsq001dm808b0vjbta1), [DanielEth](/profile/clshml0b2000owdlalgjw9d0j), [Coffee](/profile/clln3vyj7000cml0877uhlb7j), [Sungyuk1](/profile/clqneoxm50003guj70byi2jtt), [Honour](/profile/clrc98bu4000011oz4po0q5dd), [0xloscar01](/profile/cllgowxgy0002la08qi9bhab4), [stefanlatinovic](/profile/clpbb43ek0014aevzt4shbvx2), [ceseshi](/profile/cln8zm3hz000gmf08kqdt7i5b), [paprikrumplikas](/profile/clrj23lnp006g14oum8qxro25), [EloiManuel](/profile/clq2hor730000b8vl1unuep88), [Nocturnus](/profile/clk6gsllo0000mn08rjvbjy0x), [Berring](/profile/cls94h0bg0000gyor1gyfu4pt). Selected submission by: [merv](/profile/clrqlmc79000akrhqk2duq9e9)._  


### Relevant GitHub Links

https://github.com/Cyfrin/2024-02-soulmate/blob/207db2f122ee4067d000d9f2bbf6b317ce194767/src/Soulmate.sol#L75-L76

## Summary

A user can couple with themselves. This can be on purpose or by accident. This allows the user to mint NFT and claim love tokens. This means they can also stake tokens and receive more tokens.

This will be shown in a test below.

## Vulnerability Details

User can cheat the protocol by coupling with themselfs.

## Impact

This will enable tokens to be sent to Users who are not entitled to them. It means that they will not need to couple with someone else. This defeats the purpose of the protocol.

```
**Proof of Concept**
function test_Himself1() public {
        address USER = makeAddr("user");
        vm.startPrank(USER);		       // broadcasting  User
        soulmateContract.mintSoulmateToken(); // press mint twice
        soulmateContract.mintSoulmateToken();
        address workCouple = soulmateContract.soulmateOf(USER); //shows soulmate is same user as couple
        console2.log("the partner of USER1 is:", workCouple);
        vm.stopPrank();
        vm.warp(86401);
        vm.prank(USER);
        airdropContract.claim();

        }
```

    result:

```
	624] Soulmate::soulmateOf(user: [0x6CA6d1e2D5347Bfab1d91e883F1915560e09129D]) [staticcall]
    │   └─ ← user: [0x6CA6d1e2D5347Bfab1d91e883F1915560e09129D] //shows partner is same user


    │   ├─ [2586] LoveToken::balanceOf(Vault: [0x8Ad159a275AEE56fb2334DBb69036E9c7baCEe9b])
    │   │   └─ ← 500000000000000000000000000 [5e26]
    │   ├─ emit TokenClaimed(user: user: [0x6CA6d1e2D5347Bfab1d91e883F1915560e09129D], amount: 1000000000000000000 [1e18])
    │   ├─ [33184] LoveToken::transferFrom(Vault: [0x8Ad159a275AEE56fb2334DBb69036E9c7baCEe9b], user: [0x6CA6d1e2D5347Bfab1d91e883F1915560e09129D], 1000000000000000000 [1e18])
    │   │   ├─ emit Transfer(from: Vault: [0x8Ad159a275AEE56fb2334DBb69036E9c7baCEe9b], to: user: [0x6CA6d1e2D5347Bfab1d91e883F1915560e09129D], amount: 1000000000000000000 [1e18])
    │   │   └─ ← true
    │   └─ ← ()								//!!RECEIVING FUNDS!!
    └─ ← ()
```

## Tools Used

forge, remix

## Recommendations

Add this line to soulmate::mintSoulmateToken line 75:

```diff

else if (soulmate2 == address(0)) {
+          require(idToOwners[nextID][0] != msg.sender, "you cant couple with yourself!");
            idToOwners[nextID][1] = msg.sender;
            // Once 2 soulmates are reunited, the token is minted
            ownerToId[msg.sender] = nextID;
            soulmateOf[msg.sender] = soulmate1;
            soulmateOf[soulmate1] = msg.sender;
            idToCreationTimestamp[nextID] = block.timestamp;

```

result:

```
[33015] Soulmate::mintSoulmateToken()
    │   ├─ emit SoulmateIsWaiting(soulmate: user: [0x6CA6d1e2D5347Bfab1d91e883F1915560e09129D])
    │   └─ ← 0
    ├─ [1375] Soulmate::mintSoulmateToken()
    │   └─ ← revert: you cant couple with yourself!
    └─ ← revert: you cant couple with yourself!
```

# Low Risk Findings

## <a id='L-01'></a>L-01. The event SoulmateAreReunited is triggered with incorrect parameters.

_Submitted by [shaflow01](/profile/clscnul8n00003fvj246ieozj), [0x4non](/profile/clk3udrho0004mb08dm6y7y17), [robertodf99](/profile/clscdki8s0001sbgc9ukl8ewc), [CarlosAlbaWork](/profile/clrw1ekq80007u9z6ke2mx6yr), [4th05](/profile/clrsi66ll0000o022xf5bcqfg), [akhilmanga](/profile/clk48iw7c0056l508gqk81x6a), [Nocturnus](/profile/clk6gsllo0000mn08rjvbjy0x). Selected submission by: [shaflow01](/profile/clscnul8n00003fvj246ieozj)._  


### Relevant GitHub Links

https://github.com/Cyfrin/2024-02-soulmate/blob/b3f9227942ffd5c443ce6bccaa980fea0304c38f/src/Soulmate.sol#L82

## Summary

The event SoulmateAreReunited is triggered with incorrect parameters.

## Vulnerability Details

```solidity
        else if (soulmate2 == address(0)) {
            idToOwners[nextID][1] = msg.sender;
            // Once 2 soulmates are reunited, the token is minted
            ownerToId[msg.sender] = nextID;
            soulmateOf[msg.sender] = soulmate1;
            soulmateOf[soulmate1] = msg.sender;
            idToCreationTimestamp[nextID] = block.timestamp;

            emit SoulmateAreReunited(soulmate1, soulmate2, nextID);

            _mint(msg.sender, nextID++);
        }
```

In the mintSoulmateToken() function, when entering the else if branch, soulmate2 is set to the address(0), causing soulmate2 to always be address(0) when triggering emit SoulmateAreReunited(soulmate1, soulmate2, nextID).

## Impact

The event SoulmateAreReunited is triggered with incorrect parameters.  
POC

```solidity
    function test_MintNewToken() public {
        uint tokenIdMinted = 0;

        vm.prank(soulmate1);
        soulmateContract.mintSoulmateToken();

        assertTrue(soulmateContract.totalSupply() == 0);

        vm.prank(soulmate2);
        soulmateContract.mintSoulmateToken();

        assertTrue(soulmateContract.totalSupply() == 1);
        assertTrue(soulmateContract.soulmateOf(soulmate1) == soulmate2);
        assertTrue(soulmateContract.soulmateOf(soulmate2) == soulmate1);
        assertTrue(soulmateContract.ownerToId(soulmate1) == tokenIdMinted);
        assertTrue(soulmateContract.ownerToId(soulmate2) == tokenIdMinted);
    }
```

result

```solidity
[PASS] test_MintNewToken() (gas: 210743)
Traces:
  [210743] SoulmateTest::test_MintNewToken()
    ├─ [0] VM::prank(soulmate1: [0x65629adcc2F9C857Aeb285100Cc00Fb41E78DC2f])
    │   └─ ← ()
    ├─ [33015] Soulmate::mintSoulmateToken()
    │   ├─ emit SoulmateIsWaiting(soulmate: soulmate1: [0x65629adcc2F9C857Aeb285100Cc00Fb41E78DC2f])
    │   └─ ← 0
    ├─ [393] Soulmate::totalSupply() [staticcall]
    │   └─ ← 0
    ├─ [0] VM::prank(soulmate2: [0xe93A5E9F20AF38E00a08b9109D20dEc1b965E891])
    │   └─ ← ()
    ├─ [157343] Soulmate::mintSoulmateToken()
    │   ├─ emit SoulmateAreReunited(soulmate1: soulmate1: [0x65629adcc2F9C857Aeb285100Cc00Fb41E78DC2f], soulmate2: 0x0000000000000000000000000000000000000000, tokenId: 0)
    │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: soulmate2: [0xe93A5E9F20AF38E00a08b9109D20dEc1b965E891], id: 0)
```

## Tools Used

Manual Review

## Recommendations

```solidity
    emit SoulmateAreReunited(soulmate1, msg.sender, nextID);
```

## <a id='L-02'></a>L-02. Staking Rewards Forfeited Upon Deposit Withdrawal

_Submitted by [KiteWeb3](/profile/clk9pzw3j000smh08313lj91l), [wiasliaw](/profile/cllkdeq9r0000l608mqrmbi2j), [Honour](/profile/clrc98bu4000011oz4po0q5dd), [paprikrumplikas](/profile/clrj23lnp006g14oum8qxro25), [Joshuajee](/profile/clp8tjvhb0000r61pfr2owzuy). Selected submission by: [paprikrumplikas](/profile/clrj23lnp006g14oum8qxro25)._  


### Relevant GitHub Links

https://github.com/Cyfrin/2024-02-soulmate/blob/b3f9227942ffd5c443ce6bccaa980fea0304c38f/src/Staking.sol#L90

## Summary

`Staking.sol` is designed to reward users for locking their love tokens over a period. However, due to a flaw in the implementation, users who withdraw their deposited funds before claiming their accrued rewards forfeit these rewards. This behavior deviates from the expected outcome where rewards accrued during the staking period should be claimable until the moment of withdrawal.

## Vulnerability Details

The vulnerability stems from the Staking contract's handling of reward claims and withdrawals. Specifically, the contract does not account for or preserve the rewards accrued by a user's stake when they perform a withdrawal. As demonstrated in the provided test code, after advancing time to allow for reward accrual and then withdrawing the initial deposit, the user's balance before and after attempting to claim rewards remains unchanged, indicating that the rewards were forfeited upon withdrawal.

```javascript
   function test_rewardsForfeitedUponDepositWithdrawal() public {
        uint256 balanceUser1;
        uint256 balanceUser1_beforeRewardsClaim;
        uint256 balanceUser1_afterRewardsClaim;
        uint256 totalAmountDeposited = 0;

        // pair up user 1 and 2
        vm.prank(user1);
        soulmateContract.mintSoulmateToken();
        vm.prank(user2);
        soulmateContract.mintSoulmateToken();

        vm.warp(block.timestamp + 4 weeks); // fast fwd time with 1 month
        vm.startPrank(user1);
        airdropContract.claim(); // claim airdrop
        balanceUser1 = loveToken.balanceOf(user1);
        totalAmountDeposited += balanceUser1;
        loveToken.approve(address(stakingContract), type(uint256).max); // approve spending
        stakingContract.deposit(balanceUser1); // deposit all
        vm.stopPrank();

        vm.warp(block.timestamp + 4 weeks); // fast fwd time with 1 month (due to another bug, not really neccesary)

        vm.startPrank(user1);
        stakingContract.withdraw(totalAmountDeposited);
        balanceUser1_beforeRewardsClaim = loveToken.balanceOf(user1); // no rewards to claim
        stakingContract.claimRewards();
        balanceUser1_afterRewardsClaim = loveToken.balanceOf(user1);
        vm.stopPrank();

        assertEq(balanceUser1_beforeRewardsClaim, balanceUser1_afterRewardsClaim);
    }
```

## Impact

1. Loss of Expected Rewards: Users lose potential rewards they have rightfully earned through staking, which can lead to dissatisfaction and reduced participation in the staking mechanism.
2. Misalignment with Staking Incentives: The fundamental incentive for staking — earning rewards over time — is undermined if users risk losing rewards by withdrawing their stake.

## Tools Used

Manual review.

## Recommendations

Automate the rewards claim process during withdrawal to eliminate the need for users to manually claim rewards in a separate transaction:

```diff
    function withdraw(uint256 amount) public {
+     claimRewards();
        // No require needed because of overflow protection
        userStakes[msg.sender] -= amount;
        loveToken.transfer(msg.sender, amount);
        emit Withdrew(msg.sender, amount);
    }
```

Additionally, do clearly document and communicate this change to users, explaining how the automatic reward claim process works and its benefits, to maintain transparency and trust.

## <a id='L-03'></a>L-03. Misleading Total Souls Count due to Unpaired and Self-Paired Users

_Submitted by [KiteWeb3](/profile/clk9pzw3j000smh08313lj91l), [paprikrumplikas](/profile/clrj23lnp006g14oum8qxro25). Selected submission by: [paprikrumplikas](/profile/clrj23lnp006g14oum8qxro25)._  


### Relevant GitHub Links

https://github.com/Cyfrin/2024-02-soulmate/blob/b3f9227942ffd5c443ce6bccaa980fea0304c38f/src/Soulmate.sol#L139

## Summary

`Soulmate::totalSouls` inaccurately calculates the total number of paired souls. This function currently returns a value that assumes all minting actions result in successful pairings, thereby doubling the `nextID` to represent total souls. However, this calculation does not account for users who are still awaiting pairing or those who have been incorrectly paired with themselves, leading to a misrepresented count of active soulmate pairs within the system.

## Vulnerability Details

The `totalSouls` function's simplistic calculation method (return nextID \* 2;) overlooks two critical scenarios:

1. Unpaired Souls: Users who have initiated the minting process but are not yet paired. These users should not contribute to the total count of souls until their pairing is confirmed.
2. Self-Paired Users: The current logic does not prevent a user from being paired with themselves.

The provided test case illustrates this issue by demonstrating that the `totalSouls` count can be inaccurate immediately after a minting request and can also reflect an incorrect increment when a user is allowed to pair with themselves.

Proof of code:

```javascript
    function test_numberOfSouls() public {
        uint256 totalSouls = soulmateContract.totalSouls();
        console2.log(totalSouls);
        assertEq(totalSouls, 0);

        vm.prank(user1);
        soulmateContract.mintSoulmateToken();
        totalSouls = soulmateContract.totalSouls();
        console2.log(totalSouls);
        assertLt(totalSouls, 1);

        vm.prank(user1);
        soulmateContract.mintSoulmateToken();
        totalSouls = soulmateContract.totalSouls();
        console2.log(totalSouls);
        assertGt(totalSouls, 1);
    }
```

## Impact

The inaccurate reporting of total souls impacts the transparency and reliability of the protocol's metrics. It could mislead users and stakeholders about the platform's activity level and the actual number of successful pairings, potentially affecting user trust and engagement.

## Tools Used

Manual review.

## Recommendations

The following modifications are recommended:

1. Prevent Self-Pairing: Implement checks within the minting function to prevent users from being paired with themselves, ensuring that all pairings are between distinct users:

```diff
  function mintSoulmateToken() public returns (uint256) {
        // Check if people already have a soulmate, which means already have a token
        address soulmate = soulmateOf[msg.sender];
        if (soulmate != address(0)) {
            revert Soulmate__alreadyHaveASoulmate(soulmate);
        }
        address soulmate1 = idToOwners[nextID][0];
        address soulmate2 = idToOwners[nextID][1];
        if (soulmate1 == address(0)) {
            idToOwners[nextID][0] = msg.sender;
            ownerToId[msg.sender] = nextID;
            emit SoulmateIsWaiting(msg.sender);
        } else if (soulmate2 == address(0)) {
+          require(msg.sender != soulmate1, "Can't be your own soulmate!");
            idToOwners[nextID][1] = msg.sender;
            // Once 2 soulmates are reunited, the token is minted
            ownerToId[msg.sender] = nextID;
            soulmateOf[msg.sender] = soulmate1;
            soulmateOf[soulmate1] = msg.sender;
            idToCreationTimestamp[nextID] = block.timestamp;

            emit SoulmateAreReunited(soulmate1, soulmate2, nextID);

            _mint(msg.sender, nextID++);
        }

        return ownerToId[msg.sender];
    }
```

2. Rename the function and change its implementation so that it returns the number of pairs, not the number of souls:

```diff
-    function totalSouls() external view returns (uint256) {
-        return nextID * 2;
+    function totalPairs() external view returns (uint256) {
+        return nextID;
    }
```

## <a id='L-04'></a>L-04. `Soulmate::mintSoulmateToken` fails to validate recipient addresses in its `_mint` function, risking a DOS attack on the token.

_Submitted by [Aizen](/profile/clrep1f8r0008b3fvl30ma1cw), [Owenzo](/profile/cls0la11f0000127hfng6wpzl), [dimah7](/profile/clqqo7o2v000copafdu0zb10y), [Bube](/profile/clk3y8e9u000cjq08uw5phym7), [Nocturnus](/profile/clk6gsllo0000mn08rjvbjy0x). Selected submission by: [Owenzo](/profile/cls0la11f0000127hfng6wpzl)._  


### Relevant GitHub Links

https://github.com/Cyfrin/2024-02-soulmate/blob/b3f9227942ffd5c443ce6bccaa980fea0304c38f/src/Soulmate.sol#L62

Description: The attack vector described involves malicious actors exploiting the lack of recipient addresss validation of the `onERC721Received` function within the `Soulmate::mintSoulmateToken` function, specifically through the use of the `_mint` function.

```javascript
function mintSoulmateToken() public returns (uint256) {
        address soulmate = soulmateOf[msg.sender];
        if (soulmate != address(0)) {
            revert Soulmate__alreadyHaveASoulmate(soulmate);
        }

        address soulmate1 = idToOwners[nextID][0];
        address soulmate2 = idToOwners[nextID][1];
        if (soulmate1 == address(0)) {
            idToOwners[nextID][0] = msg.sender;
            ownerToId[msg.sender] = nextID;
            emit SoulmateIsWaiting(msg.sender);
        } else if (soulmate2 == address(0)) {
            idToOwners[nextID][1] = msg.sender;
            ownerToId[msg.sender] = nextID;
            soulmateOf[msg.sender] = soulmate1;
            soulmateOf[soulmate1] = msg.sender;

            idToCreationTimestamp[nextID] = block.timestamp;

            emit SoulmateAreReunited(soulmate1, soulmate2, nextID);

@>            _mint(msg.sender, nextID++);
        }

        return ownerToId[msg.sender];
    }

```

```javascript

@> function _mint(address to, uint256 id) internal virtual {
        require(to != address(0), "INVALID_RECIPIENT");

        require(_ownerOf[id] == address(0), "ALREADY_MINTED");

        unchecked {
            _balanceOf[to]++;
        }

        _ownerOf[id] = to;

        emit Transfer(address(0), to, id);
    }

```

Impact: As a consequence of the attack, tokens sent to addresses incapable of handling them become irretrievably lost within the Ethereum blockchain. Since blockchain transactions are immutable, once tokens are sent to an address, they cannot be reversed or recovered thus resulting in a permanent loss of tokens from the token supply, adversely affecting `Soulmate` token holders and the overall ecosystem stability.

Proof of Concept:

1. A malicious actor may attempt to mint a `Soulmate` token by pairing a non-NFT compatible smart contract address with another address presumed to support NFT functionality.
2. A denial-of-service (DOS) attack transpires, resulting in the loss of the token within the Ethereum blockchain.

<details>
<summary>Proof Of Code</summary>

Place the following into the `SoulmateTest.t.sol`.

```javascript
function test_NonNftContractrevertswhenMintingSoulmateContract() public {
        vm.prank(soulmate1);
        soulmateContract.mintSoulmateToken();

        assertTrue(soulmateContract.totalSupply() == 0);

        vm.prank(address(nonNftContractaddress));
        vm.expectRevert();
        soulmateContract.mintSoulmateToken();

// Both Soulmate 1 and the NonNftContract address dont have a token mainly because of the exploit in the NonNftContract.
        assertTrue(soulmateContract.totalSupply() == 0);
    }
}


contract NonNftContract {
//  Contract that doesnt implement the IERC721Receiver.onERC721Received()
}

```

</details>

Recommended Mitigation: Use the `_safeMint` function instead of the `_mint` function since it carries out checks for the `onERC721Received` function which reverts back with the selector thus proving whether the recipient address is compatible to handle ERC-721 tokens or not.

```diff

-function _mint(address to, uint256 id) internal virtual {
-        require(to != address(0), "INVALID_RECIPIENT");

-       require(_ownerOf[id] == address(0), "ALREADY_MINTED");


-        unchecked {
-            _balanceOf[to]++;
-        }

-        _ownerOf[id] = to;

-        emit Transfer(address(0), to, id);
-    }


+function _safeMint(address to, uint256 id) internal virtual {
+        _mint(to, id);

+        require(
+            to.code.length == 0 ||
+                ERC721TokenReceiver(to).onERC721Received(msg.sender, address(0), id, "") ==
+                ERC721TokenReceiver.onERC721Received.selector,
+            "UNSAFE_RECIPIENT"
+        );
+    }



function mintSoulmateToken() public returns (uint256) {
        address soulmate = soulmateOf[msg.sender];
        if (soulmate != address(0)) {
            revert Soulmate__alreadyHaveASoulmate(soulmate);
        }

        address soulmate1 = idToOwners[nextID][0];
        address soulmate2 = idToOwners[nextID][1];
        if (soulmate1 == address(0)) {
            idToOwners[nextID][0] = msg.sender;
            ownerToId[msg.sender] = nextID;
            emit SoulmateIsWaiting(msg.sender);
        } else if (soulmate2 == address(0)) {
            idToOwners[nextID][1] = msg.sender;
            ownerToId[msg.sender] = nextID;
            soulmateOf[msg.sender] = soulmate1;
            soulmateOf[soulmate1] = msg.sender;
            idToCreationTimestamp[nextID] = block.timestamp;

            emit SoulmateAreReunited(soulmate1, soulmate2, nextID);

-            _mint(msg.sender, nextID++);
+            _safeMint(msg.sender, nextID++);
        }

        return ownerToId[msg.sender];
    }


```

## <a id='L-05'></a>L-05. Wrong data emitted in events of `LoveToken`

_Submitted by [0xTheBlackPanther](/profile/clnca1ftl0000lf08bfytq099)._  


### Relevant GitHub Links

https://github.com/Cyfrin/2024-02-soulmate/blob/b3f9227942ffd5c443ce6bccaa980fea0304c38f/src/LoveToken.sol#L46-L56

## Summary

The `initVault` function in the `LoveToken` contract emits both the `AirdropInitialized` and `StakingInitialized` events with incorrect data. The event data indicates that the parameters passed should be `airdropContract` and `stakingContract`, respectively. Still, the function emits both events with the parameter `managerContract`, resulting in inaccurate event logs.

## Vulnerability Details

Both the `AirdropInitialized` and `StakingInitialized` events are expected to log the addresses of the corresponding contracts (`airdropContract` and `stakingContract`). However, the `initVault` function incorrectly emits both events with the parameter `managerContract`, leading to a discrepancy between the event's data and the actual initialized contracts.

## Impact

The incorrect event data poses a challenge for developers and external systems relying on event logs, as the information provided does not accurately reflect the initialized contracts. This discrepancy may result in confusion and hinder proper tracking of contract initialization events.

## Tools Used

Manual review.

## Recommendations

**Correct Event Data:** Update the `initVault` function to emit both the `AirdropInitialized` and `StakingInitialized` events with the correct parameters, reflecting the addresses of the corresponding contracts (`airdropContract` and `stakingContract`).
