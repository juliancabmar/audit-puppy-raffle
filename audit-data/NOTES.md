# Protocol Objectives:

1. This project is to enter a raffle to win a cute dog NFT
2. Call the enterRaffle function with the following parameters:
address[] participants: A list of addresses that enter. You can use this to enter yourself multiple times, or yourself and a group of your friends.  
3. Duplicate addresses are not allowed
//@audit says "... duplicate addresses are not allowed" but says what "...You can use this to enter yourself multiple times"
4. Users are allowed to get a refund of their ticket & value if they call the refund function
5. Every X seconds, the raffle will be able to draw a winner and be minted a random puppy
6. The owner of the protocol will set a feeAddress to take a cut of the value, and the rest of the funds will be sent to the winner of the puppy.

# Roles

Owner - Deployer of the protocol, has the power to change the wallet address to which fees are sent through the `changeFeeAddress` function.
Player - Participant of the raffle, has the power to enter the raffle with the `enterRaffle` function and refund value through `refund` function.

Audit notes:
//@audit Protocol Objectives says "... duplicate addresses are not allowed" but says what "...You can use this to enter yourself multiple times"
// @audit this repo says what the right commit is 2a47715b30cf11ca82db148704e67652ad679cd8, but the commit related repo says what the right commit is e30d199697bbc822b646d76533b66b7d529b8ef5
// @? Solc Version: 0.7.6 is hackeable?
