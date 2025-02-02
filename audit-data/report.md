---
title: Password Store Audit Report
author: Pluton
date: July 30, 2024
header-includes:
  - \usepackage{titling}
  - \usepackage{graphicx}
  - \usepackage{hyperref} 
---

\begin{titlepage}
    \centering
    \begin{figure}[h]
        \centering
        \includegraphics[width=0.8\textwidth]{logo.pdf} 
    \end{figure}
    \vspace{2cm}
    \noindent\rule{1\textwidth}{0.85pt}
    {\Huge\bfseries Password Store Audit Report\par}
    \noindent\rule{1\textwidth}{0.85pt}
    {\Large\itshape Prepared by Pluton \par}
    {\Large Version 1.0\par}
    \vspace{5cm}
    {\Large\bfseries Lead Auditor \par} 
    {\Large \href{https://herman-effendi.vercel.app/}{Herman Effendi} \par} 
    \vfill
    {\large \today\par}
\end{titlepage}

\newpage

# Table of Contents
- [Table of Contents](#table-of-contents)
- [Introduction](#introduction)
- [Protocol Summary](#protocol-summary)
- [Risk Classification](#risk-classification)
- [Audit Details](#audit-details)
  - [Scope](#scope)
  - [Roles](#roles)
  - [Findings](#findings)
    - [High](#high)
      - [\[H-1\] Storing password on-chain makes it visible to everyone and no longer private](#h-1-storing-password-on-chain-makes-it-visible-to-everyone-and-no-longer-private)
      - [\[H-2\] `PasswordStore::setPassword` has no access controls, meaning non-owner could change the password](#h-2-passwordstoresetpassword-has-no-access-controls-meaning-non-owner-could-change-the-password)
    - [Informational](#informational)
      - [\[I-1\] The `PasswordStore::getPassword` natspec indicates a parameter doesn't exist, causing the natspec to be incorrect.](#i-1-the-passwordstoregetpassword-natspec-indicates-a-parameter-doesnt-exist-causing-the-natspec-to-be-incorrect)

\newpage

# Introduction

The Pluton team has diligently worked to identify as many vulnerabilities as possible within the constraints of the allotted time frame. However, we disclaim any responsibility for the issues highlighted in this report. Please note that our security audit does not imply endorsement or approval of the business or product under review. The audit was conducted with a specific focus on the security aspects of the Solidity implementation of the contracts and was limited in scope.

# Protocol Summary

`PasswordStore` is a protocol dedicated to storage and retrieval of a user’s passwords. The protocol is
designed to be used by a single user, and is not designed to be used by multiple users. Only the owner
should be able to set and access this password

# Risk Classification

|            |        | Impact |        |     |
| ---------- | ------ | ------ | ------ | --- |
|            |        | High   | Medium | Low |
|            | High   | H      | H/M    | M   |
| Likelihood | Medium | H/M    | M      | M/L |
|            | Low    | M      | M/L    | L   |

We use the [CodeHawks](https://docs.codehawks.com/hawks-auditors/how-to-evaluate-a-finding-severity) severity matrix to determine severity. See the documentation for more details.

\newpage

# Audit Details 

## Scope 

```bash
src/
|--- PasswordStore.sol
```

## Roles

- Owner: Is the only one who should be able to set and access the password. For this contract, only the owner should be able to interact with the contract

## Findings

| Severity          | Number of issues found |
| ----------------- | ---------------------- |
| High              | 2                      |
| Medium            | 0                      |
| Low               | 0                      |
| Info              | 1                      |
| Gas Optimizations | 0                      |
| **Total**         | 3                      |


### High

#### [H-1] Storing password on-chain makes it visible to everyone and no longer private

**Description:** 

All data stored on-chain is visible for everyone, and can be read directly from blockchain. The `PasswordStore::s_password` variable is intended to be a private variable and only accessed through the `PasswordStore::getPassword()` function, which is intended to be only called by the owner of the contract.

we show one such method how to read the data off-chain below.

**Impact:** 

Anyone can read the private password, severly breaking the functionality of the protocol

**Proof of Concept:**

1. Create locally running on-chain
```bash
make anvil
```

2. Deploy the contract on anvil local chain
```bash
make deploy
```

3. Get the address contract and get value from contract storage through command `cast`
```bash
cast storage 0x5FbDB2315678afecb367f032d93F642f64180aa3 1 --rpc-url http://127.0.0.1:8545
```

4. Decode the value from, using command `cast`
```bash
cast parse-bytes32-string 0x6d7950617373776f726400000000000000000000000000000000000000000014

myPassword // output
```

5. finally we can read the actual password from local on-chain anvil.

**Recommended Mitigation:** 

Due to this, the overall the architecture of the contract should be rethought. One could encrypt the password off-chain, and store the encrypted password on-chain. This would require the user to remember another password off-chain to decrypt the password. however, you would also likely want to remove the view funtion as you wouldn't be want the user to accidently send a transaction with the password that decrypts the password.


#### [H-2] `PasswordStore::setPassword` has no access controls, meaning non-owner could change the password 

**Description:** 

The `Password::setPassword` function is set to be an `external` function. However, the natspec of the function and overall purpose of the smart contract is that `This function allows only the owner to set a new password.`

```javascript
    function setPassword(string memory newPassword) external {
@>      // @audit - there are no access controls
        s_password = newPassword;
        emit SetNetPassword();
    }
```

**Impact:**

 Anyone can set/change the password of the contract. severly breaking the contract intended functionality. 

**Proof of Concept:**

add the following to the `PasswordStore.t.sol` test file.

<details>

<summary>Code</summary>

```javascript
    function test_non_owner_can_setting_password(address random) public {
        vm.assume(random != owner);

        vm.prank(random);
        string memory expectedPassword = "myNewPassword";
        passwordStore.setPassword(expectedPassword);

        vm.prank(owner);
        string memory actualPassword = passwordStore.getPassword();
        
        assertEq(actualPassword, expectedPassword);
    }
```
</details>

**Recommended Mitigation:** 

Add an access control conditional to the `PasswordStore::setPassword` function.

```javascript
    if(msg.sender != owner){
        revert PasswordStore__NotOwner();
    }
```

### Informational

#### [I-1] The `PasswordStore::getPassword` natspec indicates a parameter doesn't exist, causing the natspec to be incorrect.

**Description:** 

```javascript
    /*
     * @notice This allows only the owner to retrieve the password.
@>   * @param newPassword The new password to set.
     */
    function getPassword() external view returns (string memory) {
        if (msg.sender != s_owner) {
            revert PasswordStore__NotOwner();
        }
        return s_password;
    }
```

The `PasswordStore::getPassword` function signature is `getPassword()` while the natspec says it should be `getPassword(string)`.

**Impact:** 

The natspec is incorrect.

**Proof of Concept:** 

Remove the incorrect natspec line.

**Recommended Mitigation:** 

Remove the incorrect natspec 

```diff
-   * @param newPassword The new password to set.
```

