# Wallets and systems with on-chain transaction outcome checks

Survey of which wallets do what [MetaMask Added Protection](metamask-added-protection.md) does: turn the outcome of a transaction (from simulation, or written by someone) into conditions checked on-chain at execution time, reverting on divergence. Done on 2026-11-10 by web search and GitHub code search; star counts and commit dates from the GitHub API that day. "Not found" means not found by search, not proven absent.

## Tier 1: generated automatically by the wallet from its simulation

**MetaMask Added Protection (EVM).** DelegationManager + `{Native,ERC20,ERC721,ERC1155}BalanceChangeEnforcer` on EIP-7702 DeleGator accounts; sender balance changes with 10% default slippage; on by default for dapp transactions; extension v13.45, September 2026. See [metamask-added-protection.md](metamask-added-protection.md).

**Phantom Guard Instructions (Solana), built on Lighthouse.**

- Program: [Jac0xb/lighthouse](https://github.com/Jac0xb/lighthouse), id `L2TExMFKdjpN9kozasaurPirfHy9P8sbXoAN1qA3S95` (mainnet and devnet). OtterSec audit, 2024-11-16 (`audits/lighthouse_audit_final.pdf`).
- Phantom's [developer docs](https://docs.phantom.com/developer-powertools/lighthouse): transactions "may be augmented with Lighthouse assertion instructions" to protect "against unexpected outcomes like simulation spoofing or malicious transaction manipulation"; a failing assertion fails the whole transaction; "You don't need to do anything special". Blog: [anti-spoofing security](https://phantom.com/learn/blog/anti-spoofing-security) (undated, about early 2026).
- Lighthouse can assert account info and data, account deltas against a snapshot written earlier in the same transaction (`MemoryWrite` + `AssertAccountDelta`), token accounts (amount, delegate), mints and stake accounts. Phantom only documents the token balance case for swaps.
- Live in production: integrators allowlist the injected instructions, e.g. [cowprotocol/services#4960](https://github.com/cowprotocol/services/pull/4960), [x402#828](https://github.com/x402-foundation/x402/issues/828), [x402#2097](https://github.com/x402-foundation/x402/issues/2097) ("about 3 Lighthouse instructions"), the [x402 SVM spec](https://github.com/x402-foundation/x402/blob/main/specs/schemes/exact/scheme_exact_svm.md). Phantom's SDK has `UNSUPPORTED_LIGHTHOUSE_PROGRAM_ID` ([simulation.ts](https://github.com/phantom/phantom-connect-sdk/blob/main/packages/cli/src/utils/simulation.ts)).
- Predates MetaMask's feature; MetaMask's "first self-custody wallet" claim can only hold for EVM.
- Solflare: unverified (one integrator PR mentions it, no payload captured). Backpack: no evidence.

**Prototype only.** [Bitfinding "state constraints"](https://bitfinding.com/blog/enforcing-transaction-simulation) (2025-05-29): same idea as MetaMask, design and prototype, not shipped.

## Tier 2: on-chain conditions written by a developer, protocol or user

- **Stacks post-conditions.** Protocol-native: the dapp declares allowed STX/FT/NFT transfers; in Deny mode any uncovered outflow aborts the transaction ([docs](https://docs.stacks.co/post-conditions/overview), check in [stacks-core transactions.rs](https://github.com/stacks-network/stacks-core/blob/main/stackslib/src/chainstate/stacks/db/transactions.rs)). Leather and Xverse display them; SIP-040 adds an originator mode ([sips#257](https://github.com/stacksgov/sips/pull/257)). No wallet generates them from simulation.
- **Biconomy ERC-8211 Smart Batching.** Parameters fetched at execution time and checked with EQ/GTE/LTE/IN constraints; a target-less entry is a pure predicate that reverts the batch. Absolute values, not deltas. `executeComposable` is in [Nexus.sol](https://github.com/bcnmy/nexus/blob/main/contracts/Nexus.sol); contracts in [bcnmy/composability](https://github.com/bcnmy/composability) and [bcnmy/erc8211-contracts](https://github.com/bcnmy/erc8211-contracts); SDK [bcnmy/smart-batching-sdk](https://github.com/bcnmy/smart-batching-sdk). Audits: Pashov (2025-03, 2026-05), Zenith (2025-03).
- **Blossom Labs Assertions.** ERC-8211 predicate contracts for DAO proposals, Safe batches and upgrades, hand-written or via EVMcrispr `assert` ([BlossomLabs/Assertions](https://github.com/BlossomLabs/Assertions)); not audited per its docs.
- **Porto (Ithaca) GuardedExecutor.** Reads ERC-20 balances before and after a batch, spent = `max(sum(outgoing), before - after)`, reverts on `ExceededSpendLimit` ([GuardedExecutor.sol](https://github.com/ithacaxyz/account/blob/main/src/GuardedExecutor.sol)). Session-key spend limits set by the user; EOA and super-admin keys skip it.
- **Gearbox v3.** `storeExpectedBalances` / `compareBalances` in credit-account multicalls: balance + signed minimum delta per token, reverted at the end of the multicall ([ICreditFacadeV3Multicall.sol](https://github.com/Gearbox-protocol/core-v3/blob/main/contracts/interfaces/ICreditFacadeV3Multicall.sol)). Closest code to our `TransactionGuard`, but protocol-internal.
- **Uniswap UniversalRouter `BALANCE_CHECK_ERC20`.** `balanceOf(owner) >= minBalance` as a router command ([Dispatcher.sol](https://github.com/Uniswap/universal-router/blob/main/contracts/base/Dispatcher.sol)).
- **NaniDAO AccountURL.** ETH-only: reverts if `after*100/before < rugPercent` ([AccountURL.sol](https://github.com/NaniDAO/accounts/blob/main/src/AccountURL.sol)).
- **boedegoat/safewallet.** EIP-7702 delegate `executePostChecked(txs, states)` checking ETH, ERC-20, ERC-721 holdings ([PostCheckedDelegationContract.sol](https://github.com/boedegoat/safewallet/blob/main/src/wallet/PostCheckedDelegationContract.sol)); hackathon scale.
- **Rhinestone Resource Lock Hook.** Balance must stay above the locked amount, for intents ([modules](https://docs.rhinestone.dev/home/resources/modules)).
- **Safe guards.** `checkAfterExecution` makes it possible ([docs](https://docs.safe.global/advanced/smart-account-guards)); existing guards are policies, not outcome checks: [Fiducia](https://github.com/safe-research/fiducia) (allowlists, limits, cosigner), [Safe Shield Guardian](https://help.safe.global/en/articles/470314-understanding-safe-shield-guardian-by-hypernative) (Hypernative off-chain risk policies).
- **Phylax Credible Layer.** Protocol-written Solidity assertions enforced by the block builder, which drops violating transactions; live on Linea ([docs](https://docs.phylax.systems/credible/architecture), [credible-std](https://github.com/phylaxsystems/credible-std)). Not a wallet feature.
- **Swap-only.** CoW Protocol limit price at settlement ([hooks](https://docs.cow.fi/cow-protocol/concepts/order-types/cow-hooks)); Enso `_checkMinAmountOut`.

## Tier 3: protocol-level

- **EIP-7906** transaction assertions on EIP-8141 frame transactions (`TXTRACE`, net state diff). Ethereum Foundation [post, 2026-10-05](https://blog.ethereum.org/en/2026/10/05/transaction-assertions), names no wallet. Prototypes, none audited: [IntentGuardLabs/transaction-assertions-eip7906](https://github.com/IntentGuardLabs/transaction-assertions-eip7906), [forshtat/eip7906-assertions-workshop](https://github.com/forshtat/eip7906-assertions-workshop) (ethrex Hegota devnet), [levyflux/tx-assertion-lab](https://github.com/levyflux/tx-assertion-lab).

## Checked: off-chain only

Simulation previews, scanning or off-chain policy, no on-chain outcome check found:

- Rabby, Ambire ("onchain simulation" is a preview), Coinbase Wallet / Base, Zerion, Rainbow, OKX Wallet, Trust Wallet, Ledger (Transaction Check), Argent / Ready, Zengo (ClearSign), Fireblocks (SGX policy engine), Privy (policy engine).
- Blockaid Cosigner: on-chain Safe co-signature, but the decision is Blockaid's off-chain simulation.
- Kerberus Sentinel3 (owns Fire, Pocket Universe): scanning plus insurance. Wallet Guard: acquired by Consensys, 2024. Harpie: shut down March 2025. Flashbots Protect: revert protection only. Kleidi: timelocks and policies.
- No evidence found, docs not read one by one: Uniswap Wallet, Bitget, 1inch, Thirdweb, Sequence, Openfort, Instadapp Avocado, Clave, Daimo, Frame, Gnosis Pay, imToken.

## Other chains

- Sui, Aptos: no assertion or post-condition mechanism found.
- Fuel predicates: spending authorization, not outcome checks.
- Cardano (eUTXO): outputs are explicit in the transaction, so the gap mostly does not exist.

## Takeaway

Only two wallets generate on-chain checks from their own simulation: MetaMask (EVM) and Phantom (Solana, via Lighthouse). Lighthouse is the oldest and most general deployed instance of the idea. On EVM, ERC-8211 and the delegation-framework enforcers provide the on-chain building blocks; the missing piece elsewhere is the wallet-side generation from simulation, which is what transaction guards specify.
