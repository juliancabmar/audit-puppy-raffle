### [M-#] The For Loop on `Puppyraffle::enterRaffle` function is a potential denial of service (DoS) attack.

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

**Recommended Mitigation:**\
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
