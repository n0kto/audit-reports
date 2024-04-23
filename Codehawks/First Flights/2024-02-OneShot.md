# First Flight #10: One Shot - Findings Report

# Table of contents

- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
  - ### [H-01. A challenger can use any rapper to fight](#H-01)
  - ### [H-02. Challengers cannot lose if they don't approve token](#H-02)
  - ### [H-03. Validators can choose the winner](#H-03)
- ## Low Risk Findings
  - ### [L-01. Defenders can fight against themselves and prevent any defeat](#L-01)
  - ### [L-02. Defender has always more chances to win than expected](#L-02)
  - ### [L-03. `battleWon` attribute of a rapper is never increased](#L-03)

# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #10

### Dates: Feb 22nd, 2024 - Feb 29th, 2024

[See more contest details here](https://www.codehawks.com/contests/clstf5qd2000rakskkj0lkm33)

# <a id='results-summary'></a>Results Summary

### Number of findings:

- High: 3
- Medium: 0
- Low: 3

# High Risk Findings

## <a id='H-01'></a>H-01. A challenger can use any rapper to fight

### Relevant GitHub Links

https://github.com/Cyfrin/2024-02-one-shot/blob/main/src/RapBattle.sol#L38C1-L52C6

## Description

`RapBattle::goOnStageOrBattle` allows one rapper to be stored in the contract waiting for a battle. The second user calling this function will fight against the rapper on stage. However, since the second rapper is not sent to the contract, no checks are made to verify if `msg.sender` is the owner of the challenger. This permits sending an arbitrary challenger to have more chances to win. Moreover, due to another bug that doesn't verify if a rapper exists, the challenger can use a non-existing rapper which has better attributes than any non-trained rappers.

```javascript
    function goOnStageOrBattle(uint256 _tokenId, uint256 _credBet) external {
@>
        if (defender == address(0)) {
            defender = msg.sender;
            defenderBet = _credBet;
            defenderTokenId = _tokenId;

            emit OnStage(msg.sender, _tokenId, _credBet);

            oneShotNft.transferFrom(msg.sender, address(this), _tokenId);
            credToken.transferFrom(msg.sender, address(this), _credBet);
        } else {
@>
            // credToken.transferFrom(msg.sender, address(this), _credBet);
            _battle(_tokenId, _credBet);
        }
    }
```

## Risk

Likelyhood: High

- Any challenger can use any rapper.

Impact: High

- A challenger can use the best existing rapper to have more chances to win.

## Proof of Concept

<details>

<summary>Foundry PoC to add in `OneShotTest.t.sol`</summary>

```javascript
    function testChallengerCanUseAnyRapper() public twoSkilledRappers {
        // Rapper go on stage
        vm.startPrank(user);
        oneShot.approve(address(rapBattle), 0);
        cred.approve(address(rapBattle), 3);

        rapBattle.goOnStageOrBattle(0, 3);

        vm.stopPrank();

        // timestamp where the challenger wins
        vm.warp(
            83398898792747926133958085108077828029129109360118067488968676214425609451248
        );

        // A third user use the challenger's rapper
        // Won't revert
        vm.startPrank(address(3));
        rapBattle.goOnStageOrBattle(1, 3);
        vm.stopPrank();

        // Third user earns the cred
        assert(cred.balanceOf(address(3)) == 3);
    }
```

</details>

## Recommended Mitigation

Verify that a rapper is owned by the message sender.

```diff
    function goOnStageOrBattle(uint256 _tokenId, uint256 _credBet) external {
+        require(oneShotNft.ownerOf(_tokenId) == msg.sender);
        if (defender == address(0)) {
            defender = msg.sender;
            defenderBet = _credBet;
            defenderTokenId = _tokenId;

            emit OnStage(msg.sender, _tokenId, _credBet);

            oneShotNft.transferFrom(msg.sender, address(this), _tokenId);
            credToken.transferFrom(msg.sender, address(this), _credBet);
        } else {
            // credToken.transferFrom(msg.sender, address(this), _credBet);
            _battle(_tokenId, _credBet);
        }
    }
```

## <a id='H-02'></a>H-02. Challengers cannot lose if they don't approve token

### Relevant GitHub Links

https://github.com/Cyfrin/2024-02-one-shot/blob/main/src/RapBattle.sol#L73

## Description

`RapBattle::goOnStageOrBattle` allows one rapper to be stored in the contract waiting for a battle. The second user calling this function will fight against the rapper on stage. However, only the Credibility token of the first user is transferred to the contract as a prize for the win. Challengers have to approve their token, and `transferFrom` is used if they lose. A challenger can just not approve any token to be transferred, and they will never lose.

```javascript
    function _battle(uint256 _tokenId, uint256 _credBet) internal {
        ...
        if (random <= defenderRapperSkill) {
            // We give them the money the defender deposited, and the challenger's bet
            credToken.transfer(_defender, defenderBet);
@>            credToken.transferFrom(msg.sender, _defender, _credBet);
        }
        ...
    }
```

## Risk

Likelyhood:

- Every time a challenger doesn't approve tokens before fighting.

Impact:

- Challenger can only win: The function will revert if they lose and pass if they win.

## Proof of Concept

<details>

<summary>Foundry PoC to add in `OneShotTest.t.sol`</summary>

```javascript
    function testDefenderCannotWin() public twoSkilledRappers {
        // Rapper go on stage
        vm.startPrank(user);
        oneShot.approve(address(rapBattle), 0);
        cred.approve(address(rapBattle), 3);

        rapBattle.goOnStageOrBattle(0, 3);
        vm.stopPrank();

        // Challenger go on battle
        vm.startPrank(challenger);

        // cred.approve(address(rapBattle), 3);

        // Random time where the defender wins
        vm.warp(
            83398898792747926133958085108077828029129109360118067488968676214425609451240
        );
        // Reverts: ERC20InsufficientAllowance
        rapBattle.goOnStageOrBattle(1, 3);
        vm.stopPrank();
    }
```

</details>

## Recommended Mitigation

As for the defender, transfer token of the challenger before the fight and send `totalPrize` to the winner.

## <a id='H-03'></a>H-03. Validators can choose the winner

### Relevant GitHub Links

https://github.com/Cyfrin/2024-02-one-shot/blob/main/src/RapBattle.sol#L63

## Description

In `RapBattle::goOnStageOrBattle`, the winner is decided using the `random` variable which uses `block.timestamp`, `block.prevrandao`, and `msg.sender` to be random. The problem is that validators can slightly move these variables during block validation. A malicious validator is able to manipulate the randomness and choose a winner.

```javascript
        uint256 random = uint256(
            keccak256(
@>                abi.encodePacked(block.timestamp, block.prevrandao, msg.sender)
            )
        ) % totalBattleSkill;
```

## Risk

Likelyhood: Low

- Only validators can manipulate the randomness.

Impact: High

- Validors are able to manipulate the randomness and choose a winner.

## Recommended Mitigation

Use an oracle like Chainlink to obtain real randomness.

# Medium Risk Findings

## <a id='M-01'></a>M-01. A Defender can block the stage with excessive betting

### Relevant GitHub Links

https://github.com/Cyfrin/2024-02-one-shot/blob/main/src/RapBattle.sol#L56

## Description

A rapper going on stage with a large bet can prevent other users from fighting against him. Since any user can mint multiple rappers and transfer credibility, a denial of service is possible.

```javascript
    function _battle(uint256 _tokenId, uint256 _credBet) internal {
        address _defender = defender;
@>        require(defenderBet == _credBet, "RapBattle: Bet amounts do not match");
        ...
    }
```

## Risk

Likelyhood: Low

- The attacker has to pay a considerable amount of gas.

Impact: High

- Denial of service of the RapBattle contract.

## Proof of concept

- Mint a lot of rappers.
- Stake them during 4 days.
- Transfer their Credibility tokens to one rapper.
- Go on stage with this rapper and all the tokens collected.

## Recommended Mitigation

Add a maximum amount to bet.
Alternatively, implement a maximum time for a rapper to be on stage.

# Low Risk Findings

## <a id='L-01'></a>L-01. Defenders can fight against themselves and prevent any defeat

### Relevant GitHub Links

https://github.com/Cyfrin/2024-02-one-shot/blob/main/src/RapBattle.sol#L38C1-L52C6

## Description

`RapBattle::goOnStageOrBattle` allows one rapper to be stored in the contract waiting for a battle. The second user calling this function will fight against the rapper on stage. However, nothing prevents defenders from being the second caller of this function and "fight against themselves". In that case, the defender can only win. With this bug, any defender can front-run a challenger who will win to prevent a defeat.

## Risk

Likelyhood: High

- Every time a defender calls the function twice before a different rapper calls it.

Impact: High

- Defenders always win, which can be used to front-run a defeat in the mempool.

## Proof of Concept

<details>

<summary>Foundry PoC to add in `OneShotTest.t.sol`</summary>

```javascript
    function testDefendersCanFightAgainstThemselves() public twoSkilledRappers {
        // Rapper go on stage
        vm.startPrank(user);
        oneShot.approve(address(rapBattle), 0);
        cred.approve(address(rapBattle), 3);

        rapBattle.goOnStageOrBattle(0, 3);

        // Rapper calls the same function to fight against themself
        cred.approve(address(rapBattle), 3);

        // Won't revert.
        rapBattle.goOnStageOrBattle(0, 3);
        vm.stopPrank();
    }
```

</details>

## Recommended Mitigation

Add this require at the top of the `else` condition:

```diff
function goOnStageOrBattle(uint256 _tokenId, uint256 _credBet) external {
        if (defender == address(0)) {
            ...
        } else {
            require(
+                _tokenId != defenderTokenId,
+                "Defender cannot be also the challenger"
+            );
            // credToken.transferFrom(msg.sender, address(this), _credBet);
            _battle(_tokenId, _credBet);
        }
    }
```

## <a id='L-02'></a>L-02. Defender has always more chances to win than expected

### Relevant GitHub Links

https://github.com/Cyfrin/2024-02-one-shot/blob/main/src/RapBattle.sol#L70

## Description

`RapBattle:_battle` does the fight and calculates the victory. This is calculated with the sum of both rappers' points, and a random number is chosen below this maximum. If the score is lower than the defender's points, the defender wins; otherwise, the challenger wins. The problem is that the condition checks if the random number is lower OR EQUAL to the defender points.

Concrete example:

- Defender and attacker have both 50 points.
- totalBattleSkill = 100.
- Defender win if random is between 0 and 50 : 51/100 chances.
- Challenger win if random is between 51 and 99 : 49/100 chances.

```javascript
    function _battle(uint256 _tokenId, uint256 _credBet) internal {
        ...
        uint256 totalBattleSkill = defenderRapperSkill + challengerRapperSkill;
        uint256 totalPrize = defenderBet + _credBet;

        uint256 random = uint256(
            keccak256(
                abi.encodePacked(block.timestamp, block.prevrandao, msg.sender)
            )
        ) % totalBattleSkill;

        ...
        // If random <= defenderRapperSkill -> defenderRapperSkill wins, otherwise they lose
@>        if (random <= defenderRapperSkill) {
            ...
        }
        ...
    }
```

## Risk

Likelyhood: High

- Every battle.

Impact: High

- Always more chance to win than expected for defenders.

## Recommended Mitigation

Correct the check to be strictly lower.

```diff
    function _battle(uint256 _tokenId, uint256 _credBet) internal {
        ...
        uint256 totalBattleSkill = defenderRapperSkill + challengerRapperSkill;
        uint256 totalPrize = defenderBet + _credBet;

        uint256 random = uint256(
            keccak256(
                abi.encodePacked(block.timestamp, block.prevrandao, msg.sender)
            )
        ) % totalBattleSkill;

        ...
        // If random <= defenderRapperSkill -> defenderRapperSkill wins, otherwise they lose
-        if (random <= defenderRapperSkill) {
+        if (random < defenderRapperSkill) {
            ...
        }
        ...
    }
```

## <a id='L-03'></a>L-03. `battleWon` attribute of a rapper is never increased

## Description

A rapper has several statistics, and one of them is `battleWon`. However, this variable is never used in the protocol. It would be impossible to know if a rapper has already won many battles or not. However, it is possible to know this information through events.

## Risk

Likelyhood: High

- The attribute will always be incorrect, except if a fighter didn't participate in any fight.

Impact: Very Low

- This variable is not used anywhere.
- The real information is accessible through emitted events.

## Recommended Mitigation

Increment and update `battleWon` attribute for each fight.
Consider adding `battleLose` attribute to be more transparent about the real stats of a rapper.
