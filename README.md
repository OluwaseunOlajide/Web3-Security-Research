# Web3 Security Portfolio

**Summary:** Security researcher specializing in DeFi protocol analysis with emphasis on economic logic vulnerabilities, access control flaws, and cross-contract interactions. Work focuses on manual code review complemented by property-based testing and exploit reproduction.

---

## Key Findings

| Protocol | Severity | Category | Status | Context | Date |
|----------|----------|----------|--------|---------|------|
| NADO Protocol | High    | Logic Error | Duplicate (confirmed) | Hackenproof - he Endpoint contract facilitates a "Slow Mode" mechanism to allow users to force transactions (such as deposits and withdrawals) when the Sequencer is down or censoring them.  | December 2025 |
| 1INCHAQUA | None | Fee on transger error | out of scope | HackenProof - The Aqua contract updates the internal credit ledger (balance.store) based on the input amount parameter, but executes the token transfer using safeTransferFrom. If the token deducts a transfer fee, the amount received by the Maker is strictly less than the amount credited in the Aqua ledger.| december 2025|

> **Note on Disputed Findings:** The 1INCHAQUA report was marked out-of-scope post-submission due to contest boundary definitions. The technical validity of the finding remains sound; documentation available in `/01-Competitive-Audits`.

---

## Repository Structure
```
/01-Competitive-Audits
├─ Live contest submissions demonstrating triage skills under time pressure
├─ Showcases ability to identify high-impact vulnerabilities in production-grade code
└─ Includes full written reports with PoC development

/02-Exploit-Simulations  
├─ Post-mortem analysis of historical exploits (Euler Finance, etc.)
├─ Demonstrates proficiency in writing Foundry-based exploit reproductions
└─ Focus on understanding attack vectors and root cause analysis

/03-Practice-Labs
├─ Continuous skill development through CTF challenges and guided scenarios
├─ AI-judged CodeHawks First Flights for rapid iteration
└─ Includes custom fuzzing harnesses and invariant test suites
```

---

## Methodology
After the debacle i experienced in {leaving space for blend} I decided to develop a reasonable method for auditing this took several months of iteration and abstraction which led to these core ideals: 
- **Manual Review First:** Static analysis tools (Slither, Aderyn) used for initial surface scanning; primary focus on understanding business logic and state transitions
- **Property-Based Testing:** Foundry invariant tests to validate protocol assumptions under adversarial conditions
- **Attack Modeling:** Threat modeling based on MEV interactions, oracle manipulation, and cross-protocol composability risks

---

## Technical Stack

| Category | Tools |
|----------|-------|
| Primary Framework | Foundry (Forge, Anvil, Cast) |
| Languages | Solidity, Python |
| Static Analysis | Slither, Aderyn |
| Specializations | Fuzzing, Invariant Testing, Exploit PoC Development |
| Auxiliary | Git, Markdown, Data Structure Implementation (BF-Tree) |

---

## Contact

Open to security research roles and audit collaborations.  
Feedback on methodology or findings is welcome via Issues.
