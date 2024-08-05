# AutoBit Report

# Summary

A quick and dirty code review of the AutoBitStack contract. It has several functionalities designed to create and manage Dollar-Cost Averaging (DCA) orders and limit orders.

This contract was looked at without access to the paired off-chain code as it was not working. The scope is solely the AutoBitStack.sol contract.

## Files Summary

| Key        | Value |
| ---------- | ----- |
| .sol Files | 144   |

# High Issues

## H-1: No Check Implemented for the priceTarget Variable:

The priceTarget variable is not checked for correctness in the createLimitOrder function. If \_priceTarget is not scaled correctly before being passed to functions like createLimitOrder it could lead to loss of funds. There is also no check on priceTarget in the fulfillLimitOrder function - leading to incorrectly fulfilled or rejected orders. priceTarget is assumed to already be scaled correctly .

**Impact:** If priceTarget is not correctly scaled, it could lead to orders being incorrectly fulfilled or rejected.

**Fix:** Ensure priceTarget is correctly scaled in createLimitOrder, and fulfillLimitOrder. Ensure that all interactions with priceTarget account for the 4-decimal scaling.


## H-2: fulfillLimitOrder Function Does Not Check if the priceTarget Has Been Met

The fulfillLimitOrder function does not verify whether the priceTarget has been met. This omission could lead to orders being filled regardless of the current market price, defeating the purpose of limit orders. This may be handled off chain but in its current state it does not work.

**Impact:** Without a price check, limit orders can be fulfilled incorrectly, leading to potential financial losses and trust issues with the contract.

**Fix:** Integrate a mechanism to check the current price against the priceTarget before fulfilling the order.
Since there is no on-chain oracle in the current implementation, this is apparently checked off-chain, but I have not tested this.


## H-3: (MAYBE) Incorrect Calculation of remainingAmount in DCA Struct

The remainingAmount in the DCA struct is currently initialized using the modulus operator:`remainingAmount: _totalAmount % _totalFrequency`, this approach may not accurately reflect the true remaining funds.
There are two options here as there are no dev comments:

- If the intent is to handle any leftover amount after evenly distributing \_totalAmount into \_totalFrequency swaps, then using the modulus operator is appropriate.
- If the intent is to keep track of the total remaining amount that has not yet been swapped, initializing remainingAmount to \_totalAmount and then decrementing it as swaps are executed would be more appropriate.

**Impact:** If the second option, in its current form the miscalculation of remainingAmount can lead to incorrect processing of future swaps, potentially resulting in under or over-allocation of funds.

**Fix:** If this is not the intended fucntionality, initialize remainingAmount to \_totalAmount, then deduct the amount swapped from remainingAmount as swaps are executed.

## H-4: Reentrancy in Certain Functions

The updateDCAOrder and fulfillLimitOrder functions lack the nonReentrant modifier, making them susceptible to reentrancy attacks - this is mitigated somewhat with the onlyOwner modifier, but this could still potentially be called through the owner contract. **See informational finding I-2**.

The onlyOwner modifier ensures that only the contract owner can call the function, but it does not prevent the contract from being called multiple times within a single transaction due to reentrancy.

**Impact:** Reentrancy attacks can allow malicious actors to repeatedly call these functions within a single transaction, potentially manipulating the contract’s state and draining funds.

**Attack Scenarios -** updateDCAOrder:

- A malicious contract calls updateDCAOrder.
- During the execution of transferTokensFrom, the malicious contract’s fallback function gets called.
- The fallback function re-calls updateDCAOrder, manipulating the state before the first call finishes.

**Attack Scenarios -** fulfillLimitOrder:

- An attacker deploys a malicious contract and creates a LIMIT order in the AutoBitStack contract.
- The attacker initiates a call to fulfillLimitOrder on the AutoBitStack contract.
- Within the transferTokensFrom function, the malicious contract's fallback function gets executed.
- The fallback function of the malicious contract re-calls fulfillLimitOrder before the state update in the AutoBitStack - contract is completed.

**Potential Impact:**
The attacker could repeatedly call fulfillLimitOrder, causing the LIMIT order to be fulfilled multiple times, potentially draining funds from the contract.

**Fix:** Add the nonReentrant modifier to both functions & ensure state changes occur before making external calls to further prevent reentrancy.

# Informational Issues

## I-1: Lack of Event Emission for Token Transfers

The transferTokens and transferTokensFrom functions do not emit events upon successful transfers.

**Impact:** The lack of event emissions makes it difficult to track and debug token transfers.

**Fix:** Add events to these functions to improve traceability.

## I-2: Lack of Authorization Checks in updateDCAOrder and fulfillLimitOrder

**Description:** While the onlyOwner modifier restricts access to these functions, it does not fully prevent unauthorized actions by ensuring that the msg.sender is the correct entity expected to perform these actions.

**Attack Scenarios**

- Delegation to Malicious Contract: If the owner is a contract, and that contract has a bug or a vulnerability, it might unintentionally allow unauthorized calls to these critical functions.
- Misuse by Owner: Even the owner might unintentionally misuse these functions due to a lack of specific role-based checks, leading to unintended consequences.

**Impact:** The lack of finer authorization controls could lead to misuse or unintended actions by the owner or another authorized entity.

_Fix:_ Implement role-based access control using OpenZeppelin's AccessControl library or a custom modifier.

# Gas Issues

## G-1: Gas Optimisation by Struct Packing

The DCA and LIMIT structs can be optimized by packing variables of similar types together to reduce storage costs.

**Impact:** Optimizing struct packing can save gas costs during contract deployment and interactions.

**Fix:** Rearrange the variables in the structs to achieve better packing.
