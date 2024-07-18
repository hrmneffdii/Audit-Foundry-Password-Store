## [H-1] Storing password on-chain makes it visible to everyone and no longer private

**Description:** All data stored on-chain is visible for everyone, and can be read directly from blockchain. The `PasswordStore::s_password` variable is intended to be a private variable and only accessed through the `PasswordStore::getPassword()` function, which is intended to be only called by the owner of the contract.

we show one such method how to read the data off-chain below.

**Impact:** Anyone can read the private password, severly breaking the functionality of the protocol

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

**Recommended Mitigation:** Due to this, the overall the architecture of the contract should be rethought. One could encrypt the password off-chain, and store the encrypted password on-chain. This would require the user to remember another password off-chain to decrypt the password. however, you would also likely want to remove the view funtion as you wouldn't be want the user to accidently send a transaction with the password that decrypts the password.


## [H-2] `PasswordStore::setPassword` has no access controls, meaning non-owner could change the password 

**Description:** The `Password::setPassword` function is set to be an `external` function. However, the natspec of the function and overall purpose of the smart contract is that `This function allows only the owner to set a new password.`

```javascript
    function setPassword(string memory newPassword) external {
@>      // @audit - there are no access controls
        s_password = newPassword;
        emit SetNetPassword();
    }
```

**Impact:** Anyone can set/change the password of the contract. severly breaking the contract intended functionality. 

**Proof of Concept:**add the following to the `PasswordStore.t.sol` test file.

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

**Recommended Mitigation:** Add an access control conditional to the `PasswordStore::setPassword` function.

```javascript
    if(msg.sender != owner){
        revert PasswordStore__NotOwner();
    }
```

## [I-1] The `PasswordStore::getPassword` natspec indicates a parameter doesn't exist, causing the natspec to be incorrect.

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

**Impact:** The natspec is incorrect.

**Proof of Concept:** Remove the incorrect natspec line.

**Recommended Mitigation:** Remove the incorrect natspec 

```diff
-   * @param newPassword The new password to set.
```