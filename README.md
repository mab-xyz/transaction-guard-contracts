# Transaction Guard Contracts for Ethereum

Authors: Martin Monperrus, Raphina Liu, Javier Ron [mab.xyz](http://mab.xyz), March 2026

---

## The Problem

What users approve in their wallet is not necessarily what executes on-chain. Modern wallets simulate transactions and display expected outcomes — token transfers, balance changes — before the user signs. But blockchain state can change between simulation and execution. The user consents to a simulated outcome; the chain enforces only the raw transaction.

This is the **simulation-execution gap**.

### Attack Vectors

The simulation-execution gap allows multiple attacks or bugs to place.

**Front-running.** A user simulates a swap: 1 ETH for 2,000 USDC. An attacker observes the pending transaction, manipulates pool liquidity, and the user receives 1,500 USDC. The transaction succeeds. The wallet showed 2,000; the chain delivered 1,500.

**Time-dependent exploits.** Simulation shows a safe protocol interaction. Between simulation and execution, a governance proposal changes protocol parameters — fees, withdrawal limits, ownership. The user's transaction executes under hostile conditions.

**Oracle manipulation.** Simulation uses the current oracle price. The oracle is manipulated before execution. A lending protocol liquidates the user's position instead of processing the expected deposit.

**Approval exploitation.** A user simulates a token approval for a legitimate DEX. The contract upgrades to a malicious implementation before execution. The approval now grants access to a compromised contract.

### Why Existing Defenses Are Insufficient

There are techniques, which protects a little bit against the simulation-execution gap.

- **Slippage parameters** are application-level and protect only against price movement, not arbitrary state changes.
- **Deadline parameters** prevent stale transactions but do not enforce expected outcomes.
- **Post-execution verification** is too late — funds are already gone.

The fundamental issue: users consent to simulated outcomes, but the blockchain enforces only the input transaction, not its expected effects.

---

## Solution: Transaction Guard Contracts

We propose "Transaction Guard Contracts". This mechanism works in three steps:

1. Capture the comprehensive side effects of a transaction during simulation (balance changes, state transitions, token transfers).
2. Encode these side effects as on-chain requirements in a one-shot guard contract.
3. Wrap the user's intended transaction inside this contract, which executes atomically: either all expected side effects hold, or the entire transaction reverts.

There is no off-chain trust assumption. Enforcement happens entirely on-chain, at execution time.

### Flow

```
┌──────────┐    ┌───────────────────┐    ┌──────────────────────┐    ┌──────────────┐
│  Wallet   │───▶│  Simulate tx via  │───▶│  Generate guard      │───▶│ Submit wrapped│
│  initiates│    │  eth_call / trace │    │  contract from diff  │    │  tx to mempool│
│  tx       │    │                   │    │                      │    │               │
└──────────┘    └───────────────────┘    └──────────────────────┘    └──────┬───────┘
                                                                           │
                                                                           ▼
                                                                    ┌──────────────┐
                                                                    │  On-chain:    │
                                                                    │  execute tx + │
                                                                    │  verify side  │
                                                                    │  effects      │
                                                                    │              │
                                                                    │  PASS → commit│
                                                                    │  FAIL → revert│
                                                                    └──────────────┘
```

The wallet simulates the transaction, extracts the expected state diff, generates a guard contract encoding those expectations, and submits the wrapped transaction. On-chain, the guard contract executes the original transaction, checks all side effects, and reverts atomically if any expectation is violated.

### Example

A user wants to swap 1 ETH for USDC on Uniswap. Simulation shows: -1 ETH, +2,000 USDC.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

/// @notice One-shot guard contract. Deployed per-transaction via CREATE2,
/// executes the user's intended call, verifies side effects, then self-destructs.
contract TransactionGuard {
    struct Expectation {
        address token;    // address(0) for native ETH
        address account;
        int256 minDelta;  // minimum acceptable balance change (negative = spending)
    }

    function enforce(
        address target,
        bytes calldata payload,
        Expectation[] calldata expectations
    ) external payable {
        // Snapshot balances before execution
        uint256[] memory before = new uint256[](expectations.length);
        for (uint256 i = 0; i < expectations.length; i++) {
            before[i] = _balanceOf(expectations[i].token, expectations[i].account);
        }

        // Execute the user's intended transaction
        (bool success, bytes memory returnData) = target.call{value: msg.value}(payload);
        if (!success) {
            assembly { revert(add(returnData, 32), mload(returnData)) }
        }

        // Verify every expected side effect
        for (uint256 i = 0; i < expectations.length; i++) {
            uint256 balanceAfter = _balanceOf(expectations[i].token, expectations[i].account);
            int256 delta = int256(balanceAfter) - int256(before[i]);
            require(
                delta >= expectations[i].minDelta,
                "Side effect below expectation"
            );
        }
    }

    function _balanceOf(address token, address account) internal view returns (uint256) {
        if (token == address(0)) return account.balance;
        return IERC20(token).balanceOf(account);
    }
}
```

The contract is generic: the wallet populates the `expectations` array from the simulation diff. For the swap example, expectations would encode "user's ETH balance decreases by at most 1 ETH" and "user's USDC balance increases by at least 1,900 USDC" (with tolerance applied).

If an attacker front-runs the swap and manipulates liquidity, the enforcement contract detects the deviation and reverts the entire transaction. The user loses nothing but gas.

### Tolerance Model

The guard contract does not require exact matches — it enforces minimum acceptable outcomes derived from the simulation. Tolerance accounts for legitimate variance between simulation and execution: minor price movement, rounding in AMM math, gas refund differences.

Tolerance is determined per side effect:

- **Token swaps**: percentage-based (e.g., 5% below simulated output). Configurable by the user or wallet, analogous to slippage but enforced on-chain even for protocol with no slippage support.
- **Native ETH transfers**: absolute threshold (e.g., simulated amount minus gas overhead).
- **Approval operations**: zero tolerance — the approved spender and amount must match exactly.
- **State-changing calls** (governance votes, parameter updates): zero tolerance on the target contract's state where applicable.

The wallet's simulation engine proposes default tolerances based on the transaction type. The user can tighten or loosen them before signing. This makes the mechanism adaptive: a large swap on a low-liquidity pair warrants wider tolerance than a stablecoin transfer.

### Gas Overhead

The enforcement wrapper adds gas cost proportional to the number of side effects being verified. Approximate overhead per component:

| Component | Gas cost |
|-----------|----------|
| Deployment of guard contract (CREATE2) | ~32,000 |
| Balance snapshot per token (SLOAD + external call) | ~2,600 per expectation |
| Post-execution balance check per token | ~2,600 per expectation |
| Loop and comparison logic | ~200 per expectation |

For a typical swap with 2 expectations (ETH out, token in), total overhead is approximately **43,000 gas** — roughly $0.05 at 30 gwei and 2,000 USD/ETH. For complex DeFi operations with 5–10 side effects, overhead reaches ~85,000 gas.

This is comparable to the gas overhead of existing DEX router safety checks and is negligible relative to the cost of a successful exploit.

---

## Relationship to EIP-7730

[EIP-7730](https://eips.ethereum.org/EIPS/eip-7730) solves the **"what you see"** problem: it enriches transaction data with formatting metadata so wallets can display human-readable descriptions instead of raw hex. It eliminates blind signing.

This proposal solves the **"what you get"** problem: it guarantees that on-chain execution matches the simulated outcome the user approved.

The two are complementary layers:

| Layer | Function | Protects against |
|-------|----------|-----------------|
| EIP-7730 | Clear signing — user sees what the transaction intends | Blind signing, UI spoofing |
| Transaction guard | Execution guarantee — chain enforces what was simulated | Front-running, state manipulation, oracle attacks |

**With EIP-7730 alone:** User sees "Swap 1 ETH for 2,000 USDC" clearly displayed, signs it, attacker front-runs, user receives 1,500 USDC, transaction succeeds.

**With both:** User sees "Swap 1 ETH for 2,000 USDC", signs it, attacker front-runs, guard contract requires at least 1,900 USDC, transaction reverts. User loses nothing.

EIP-7730 made transactions readable. Transaction guards make them enforceable.

---

## Threat Model

The mechanism operates under the following assumptions:

**Trusted components:**

- **The simulation environment.** The wallet's simulation (via `eth_call` or trace APIs) must faithfully report the transaction's side effects. A compromised simulation layer could feed false expectations to the guard contract. In practice, this means the user must trust their RPC endpoint, the simulation code and/or the simulation provider (eg Tenderly).
- **The wallet software.** The wallet generates the guard contract and populates expectations. A malicious wallet could omit side effects or set permissive tolerances. This is the same trust assumption users already place in their wallet for transaction construction.

**Untrusted components:**

- **The mempool.** Transactions are visible to searchers and validators. The guard contract does not hide transaction intent — it makes exploitation unprofitable by reverting manipulated outcomes.
- **The target contract.** The protocol being interacted with may behave adversarially (upgradeable proxies, admin key exploits). The guard contract protects the user regardless of the target's behavior.
- **Block producers.** Validators can reorder, include, or exclude transactions. The guard contract cannot prevent censorship, but it guarantees that *if* the transaction is included, the outcome matches expectations.

---

## Limitations

Side-effect enforcement is not a universal defense. It has clear boundaries.

**Phishing and social engineering.** If a user is tricked into signing a transaction that does what the attacker wants, enforcement will confirm the malicious outcome, not prevent it. The mechanism protects against divergence between simulation and execution, not against a user approving the wrong transaction.

**Censorship.** If a validator refuses to include the wrapped transaction, the mechanism cannot help. The user's funds are safe (the transaction simply doesn't execute), but the desired action doesn't happen. Private mempools (Flashbots Protect, MEV Blocker) address censorship and are complementary.

**Gas cost on revert.** When enforcement detects a violation, the transaction reverts but the user still pays gas for the failed execution. This is a cost of safety, not a vulnerability, but it means repeated attacks can drain gas from a user who retries without changing conditions.

**Multi-transaction workflows.** Some DeFi operations span multiple transactions (e.g., approve + swap, or deposit + stake). Enforcement operates per-transaction. Cross-transaction invariants require higher-level coordination that this mechanism does not provide.

---

## Related Work

**Flashbots Protect and MEV Blocker.** Private transaction submission services that hide pending transactions from the public mempool, reducing front-running exposure. Side-effect enforcement ensures correct outcomes regardless of mempool visibility. A user can submit a wrapped enforcement transaction through Flashbots Protect for layered protection.

**Intent-based protocols (UniswapX, CoW Protocol, 1inch Fusion).** Users express desired outcomes ("swap X for at least Y") and solvers compete to fill the order. This shares the goal of outcome enforcement but requires an off-chain solver network, introduces solver trust assumptions, and is limited to swap-like operations. Side-effect enforcement is protocol-agnostic and requires no off-chain infrastructure beyond standard RPC.

**ERC-4337 (Account Abstraction).** Enables programmable transaction validation through smart contract wallets. Account abstraction provides the infrastructure for custom validation logic; side-effect enforcement provides a specific use case for that infrastructure. The two are architecturally complementary — an ERC-4337 wallet could embed enforcement logic in its validation step.

**Transaction simulation services (Tenderly, Blowfish, Blockaid).** Provide pre-signing simulation and risk assessment. These are the "input" to the enforcement mechanism: they produce the expected side effects that the enforcement contract then verifies on-chain. Current simulation services show users what should happen; this proposal ensures it does happen.

---

## About mab.xyz

[mab.xyz](http://mab.xyz) is a Web3 security company, spinoff from Prof. Martin Monperrus' [crypto research](https://www.monperrus.net/martin/album?keywords=crypto) at KTH Royal Institute of Technology.
