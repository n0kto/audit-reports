# First Flight #6: Voting Booth - Findings Report

# Table of contents

- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
  - ### [H-01. `VotingBooth::_distributeRewards` doesn't resend always all the money and lead to loss of funds](#H-01)

# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #6

### Dates: Dec 15th, 2023 - Dec 22nd, 2023

[See more contest details here](https://www.codehawks.com/contests/clq5cx9x60001kd8vrc01dirq)

# <a id='results-summary'></a>Results Summary

### Number of validated findings:

- High: 1
- Medium: 0
- Low: 0

# High Risk Findings

## <a id='H-01'></a>H-01. `VotingBooth::_distributeRewards` doesn't resend always all the money and lead to loss of funds

### Relevant GitHub Links

https://github.com/Cyfrin/2023-12-Voting-Booth/blob/main/src/VotingBooth.sol#L192

## Description:

The `VotingBooth::_distributeRewards` function in the code divides the rewards among all the voters, but only distributes the rewards to the `VotingBooth::s_votersFor` array. As a result, if there are voters who vote "Against" and the vote is won by the "For" voters, a portion of the funds will be stuck in the contract.

```javascript
        uint256 totalVotesFor = s_votersFor.length;
        uint256 totalVotesAgainst = s_votersAgainst.length;

        uint256 totalVotes = totalVotesFor + totalVotesAgainst;
        uint256 totalRewards = address(this).balance;

        if (totalVotesAgainst >= totalVotesFor) {
            .
            .
            .
        }
        else {
@>          uint256 rewardPerVoter = totalRewards / totalVotes;

            for (uint256 i; i < totalVotesFor; ++i) {
                .
                .
                .
                if (i == totalVotesFor - 1) {
                    rewardPerVoter = Math.mulDiv(
                        totalRewards,
                        1,
@>                      totalVotes,
                        Math.Rounding.Ceil
                    );
                }
                _sendEth(s_votersFor[i], rewardPerVoter);
            }
        }
```

To be accurate, the amount stuck in the contract will be `rewardPerVoter * totalVotesAgainst`.

## Impact:

This bug occurs whenever the "For" voters win and one person votes against. It leads to a loss of money because the contract cannot be reused to vote against and withdraw the funds to the owner.

## Proof of Concept:

- Construct the contract with an allowed list of 5 addresses.
- Have 2 people vote "For" and 1 person vote "Against", triggering the end of the voting.
- Check the balance of ether remaining in the contract.

<details>

<summary>Foundry PoC</summary>

You can add the following test to `VotingBoothTest.t.sol` to test the bug :

```javascript
    function testBadRewardDistribution() public {
        vm.prank(address(0x1));
        booth.vote(true);

        vm.prank(address(0x2));
        booth.vote(true);

        vm.prank(address(0x3));
        booth.vote(false);

        assert(!booth.isActive());
        assert(address(booth).balance == 0); // will revert because ETH are stuck in the contract!
    }
```

</details>

## Recommended Mitigation:

Instead of using `totalVotes` which includes the "Against" voters, use `totalVotesFor` directly.

```diff
        uint256 totalVotesFor = s_votersFor.length;
        uint256 totalVotesAgainst = s_votersAgainst.length;

-        uint256 totalVotes = totalVotesFor + totalVotesAgainst;
        uint256 totalRewards = address(this).balance;

        if (totalVotesAgainst >= totalVotesFor) {
            .
            .
            .
        }
        else {
-          uint256 rewardPerVoter = totalRewards / totalVotes;
+          uint256 rewardPerVoter = totalRewards / totalVotesFor;

            for (uint256 i; i < totalVotesFor; ++i) {
                .
                .
                .
                if (i == totalVotesFor - 1) {
                    rewardPerVoter = Math.mulDiv(
                        totalRewards,
                        1,
-                       totalVotes,
+                       totalVotesFor,
                        Math.Rounding.Ceil
                    );
                }
                _sendEth(s_votersFor[i], rewardPerVoter);
            }
        }
```
