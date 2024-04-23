# First Flight #1: PasswordStore - Findings Report

# Table of contents

- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
  - ### [H-01. No check for setPassword](#H-01)
  - ### [H-02. password store unencrypted in a private variable](#H-02)

# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #1

### Dates: Oct 18th, 2023 - Oct 25th, 2023

[See more contest details here](https://www.codehawks.com/contests/clnuo221v0001l50aomgo4nyn)

# <a id='results-summary'></a>Results Summary

### Number of validated findings:

- High: 2
- Medium: 0
- Low: 0

# High Risk Findings

## <a id='H-01'></a>H-01. No check for setPassword

### Relevant GitHub Links

https://github.com/Cyfrin/2023-10-PasswordStore/blob/main/src/PasswordStore.sol

## Summary

No authentication control in the `setPassword()` function.

## Vulnerability Details

Password can be changed thanks to

```
function setPassword(string memory newPassword) external {
  s_password = newPassword;
  emit SetNetPassword();
}
```

## Impact

Anyone can overwrite/modify the password.

## Tools Used

Foundry cast :
`cast send <CONTRACT_ADDRESS> "setPassword(string memory)" test --private-key $PRIVATE_KEY`

## Recommendations

Add a modifier to verify the owner address:

```
modifier onlyOwner() {
  if (msg.sender != s_owner) {
  revert PasswordStore__NotOwner();
  }
}

function setPassword(string memory newPassword) external onlyOwner() {
  s_password = newPassword;
  emit SetNetPassword();
}
```

## <a id='H-02'></a>H-02. password store unencrypted in a private variable

### Relevant GitHub Links

https://github.com/Cyfrin/2023-10-PasswordStore/blob/main/src/PasswordStore.sol

## Summary

The password is stored in s_password which is publicly stored in spite of the "private" keyword

## Vulnerability Details

The password is stored in s_password which is publicly stored in the contract at the second storage
slot (index 1).

## Impact

Anyone can access it

## Tools Used

foundry cast:
`cast storage <CONTRACT_ADRRESS> 1`

## Recommendations

• Do not put sensitive data on blockchain.
• If it is really needed, at least encrypt it with a secure algorithm and a key only you know.
• In case of password before encryption, hash it to do compararison
• Use a privacy librairy like zama fhEVM
