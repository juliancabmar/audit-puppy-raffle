### [H-1] All data on-chain is public data, so everyone can see `s_password` value.

**Description:**\
The `PasswordStore::s_password` variable is intented to be private and can only accessed by `PasswordStore::getPassword` function, but (how it show below) the value can be read directly from the blockchain data.

**Impact:**\
Anyone can read the private password, severaly breaking the protocol's functionality.

**Proof of Concept:**

1. Running a local chain
```bash
$ anvil
```

2. Compile and Deploy the contract
```bash
$ make deploy
```

3. Get the storage slot #1 data (`s_password` data is on slot #1) for the deployed contract
```bash
$ cast storage [CONTRACT ADDRESS] 1
```

You'll get a result like this:\
`0x6d7950617373776f726400000000000000000000000000000000000000000014` 

4. Parse the hex-bytes32 result to string
```bash
$ cast parse-bytes32-string 0x6d7950617373776f726400000000000000000000000000000000000000000014
```

Getting the password value:\
`myPassword`


**Recommended Mitigation:**\
The actual contract objetive can't be reached, because you can't store a plain text password on-chain without that can be reading by everyone. A viable solution could be encript the password off-chain before store it on-chain, but this implies an extra step by the user.

<br>
<hr>
<br>

### [H-2] `PasswordStore::setPassword` haven't access control, meaning a non-owner could set a password.

**Description:**\
The `PasswordStore:setPassword` function does not have any access verification mechanism that ensures only the owner can call it, this implies that everyone could set a password.

```javascript
function setPassword(string memory newPassword) external {
@>  // @audit - There are not access controls
    s_password = newPassword;
    emit SetNetPassword();
}
```

**Impact:**\
Anyone can set/change the current password of the contract, breaking severaly the intended contract functionality.   

**Proof of Concept:**\
Add the following to the `PasswordStore.t.sol`test file:
<details>
<summary>PoC test:</summary>

```javascript
function test_anyone_can_set_a_password(address anyone) public {
    string memory pass = "any password";
    vm.prank(anyone);
    passwordStore.setPassword(pass);

    vm.prank(address(owner));
    string memory actualPassword = passwordStore.getPassword();

    assertEq(pass, actualPassword);
}
```

</details>
<br>

**Recommended Mitigation:**\
Add an access control condition on `PasswordStore::setPassword` function.
```javascript
function setPassword(string memory newPassword) external {
    if (msg.sender != s_owner) {
        revert PasswordStore__NotOwner();
    }
    s_password = newPassword;
    emit SetNetPassword();
}
```

<br>
<hr>
<br>

### [I-1] `PasswordStore::setPassword` function not take any parameters, but the related natspect indicated one. 

**Description:**

```javasccript
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

The `PasswordStore::setPassword` function signature is `getPassword()`while the natspec says it should be `getPassword(string)`.

**Impact:**\
The natspec is incorrect.

**Recommended Mitigation:**

```diff
- * @param newPassword The new password to set.
```
