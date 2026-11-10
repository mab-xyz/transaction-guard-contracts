# MetaMask Added Protection: source code analysis

Analysis of MetaMask "Added Protection" (internally: *enforced simulations*), shipped in the MetaMask browser extension v13.45 (September 2026). Read on 2026-11-10 from the sources pinned below; the analysis is from reading code, not from running tests.

## Sources

Off-chain (extension), [metamask-extension@25604583](https://github.com/MetaMask/metamask-extension/tree/25604583aa6bcaf09f79e7d187bcf32cd9906650):

- https://github.com/MetaMask/metamask-extension/blob/25604583aa6bcaf09f79e7d187bcf32cd9906650/app/scripts/lib/transaction/containers/enforced-simulations.ts — turns the simulation diff into caveats
- https://github.com/MetaMask/metamask-extension/blob/25604583aa6bcaf09f79e7d187bcf32cd9906650/app/scripts/lib/transaction/containers/util.ts — rewrites the transaction
- https://github.com/MetaMask/metamask-extension/blob/25604583aa6bcaf09f79e7d187bcf32cd9906650/shared/lib/transaction/enforced-simulations.ts — eligibility, slippage

On-chain, [delegation-framework@b4b8c41](https://github.com/MetaMask/delegation-framework/tree/b4b8c41fc912455a1af2300bfcd1276676b92a2e):

- https://github.com/MetaMask/delegation-framework/blob/b4b8c41fc912455a1af2300bfcd1276676b92a2e/src/enforcers/NativeBalanceChangeEnforcer.sol
- https://github.com/MetaMask/delegation-framework/blob/b4b8c41fc912455a1af2300bfcd1276676b92a2e/src/enforcers/ERC20BalanceChangeEnforcer.sol
- https://github.com/MetaMask/delegation-framework/blob/b4b8c41fc912455a1af2300bfcd1276676b92a2e/src/enforcers/ERC721BalanceChangeEnforcer.sol
- https://github.com/MetaMask/delegation-framework/blob/b4b8c41fc912455a1af2300bfcd1276676b92a2e/src/enforcers/ERC1155BalanceChangeEnforcer.sol
- https://github.com/MetaMask/delegation-framework/blob/b4b8c41fc912455a1af2300bfcd1276676b92a2e/src/enforcers/CaveatEnforcer.sol
- https://github.com/MetaMask/delegation-framework/blob/b4b8c41fc912455a1af2300bfcd1276676b92a2e/src/DelegationManager.sol

Deployed v1.3.0 (the extension's `PREFERRED_VERSION`), addresses from [Deployments.md](https://github.com/MetaMask/delegation-framework/blob/b4b8c41fc912455a1af2300bfcd1276676b92a2e/documents/Deployments.md), code presence on Ethereum mainnet checked with `eth_getCode`:

- DelegationManager: https://etherscan.io/address/0xdb9B1e94B5b69Df7e401DDbedE43491141047dB3
- NativeBalanceChangeEnforcer: https://etherscan.io/address/0xbD7B277507723490Cd50b12EaaFe87C616be6880
- ERC20BalanceChangeEnforcer: https://etherscan.io/address/0xcdF6aB796408598Cea671d79506d7D48E97a5437
- ERC721BalanceChangeEnforcer: https://etherscan.io/address/0x8aFdf96eDBbe7e1eD3f5Cd89C7E084841e12A09e
- ERC1155BalanceChangeEnforcer: https://etherscan.io/address/0x63c322732695cAFbbD488Fc6937A0A7B66fC001A

## Mechanism

1. MetaMask's (centralized) simulation produces `simulationData`: native and token balance changes of the sender.
2. `generateCaveats` emits one caveat per balance change: ETH, ERC-20, ERC-721, ERC-1155. If the simulation shows no native change, a zero-tolerance "native balance must not decrease" caveat is added.
3. The transaction is rewritten into `DelegationManager.redeemDelegations` on a self-delegation (delegator = delegatee = sender), using EIP-7702 to make the EOA a DeleGator smart account.
4. On-chain, `DelegationManager` calls every caveat's `beforeHook` (caches `balanceOf(recipient)` in storage and sets a lock), executes the call in default (reverting) mode, then calls every `afterHook` in reverse order, which requires:
   - increase: `after >= before + minIncrease`
   - decrease: `after >= before - maxDecrease`
5. Any failed requirement reverts the whole transaction; the user pays gas.

This is the same design as the `TransactionGuard` in our README (per-asset `minDelta` on before/after balances), split into reusable pre-deployed enforcers instead of a per-transaction guard.

## Parameters

- Slippage: one value from a remote feature flag, default `DEFAULT_ENFORCED_SIMULATIONS_SLIPPAGE = 10` (%). Increase: at least 90% of the simulated amount. Decrease: at most 110% of the simulated amount, capped at the previous balance.
- ERC-721 / ERC-1155: no slippage, exact simulated delta.
- Eligibility: chain supports EIP-7702, simulation complete, address screening supports the chain, and the contract targets are not all marked `Trusted` by MetaMask's trust signals.

## Weaknesses

- **ERC-721 is a count, not an identity.** `balanceOf(owner)` counts NFTs; receiving a different (worthless) token of the same collection passes.
- **Token contracts are trusted.** Balances come from the token's own `balanceOf`; a malicious token can report the simulated increase. Our guard has the same blind spot.
- **Approval-only drainers pass.** `approve` / `setApprovalForAll` change no balance; the drain happens in a later transaction.
- **Only the sender's balances are checked.** No other accounts, no contract state, no tokens absent from the simulation diff (except the native no-decrease caveat).
- **Net delta only.** Diligence (Sep 2024, Minor, acknowledged) on the predecessor enforcers: re-entrant flows inside the execution can move the balance; only the net change is observed.
- **Gas.** Cache and lock live in persistent storage (Solidity 0.8.23, no transient storage): several SSTOREs per caveat.
- **Key collision (not exploitable here).** `ERC20BalanceChangeEnforcer`'s hash key is `(caller, token, delegationHash)` without recipient: two ERC-20 caveats on the same token with different recipients collide and revert (DoS, not bypass). The extension always uses the sender as recipient.
- **Fail-safe underflow.** If the balance fell below the allowed decrease before execution, `before - amount` underflows and reverts.
- **Centralized expected outcome.** The simulation, and therefore the expected outcome encoded in the caveats, comes from a MetaMask server.

## Which accounts

Not any EIP-7702 account: only MetaMask's own DeleGator implementation.

- On-chain, `DelegationManager` calls `IDeleGatorCore(delegator).executeFromExecutor(...)` and, since a delegated EOA has code, validates the delegation signature with ERC-1271 `isValidSignature`. The account must implement `IDeleGatorCore` and accept calls from the DelegationManager.
- In the extension ([delegation.ts](https://github.com/MetaMask/metamask-extension/blob/25604583aa6bcaf09f79e7d187bcf32cd9906650/app/scripts/lib/transaction/delegation.ts)), a plain EOA (no 7702 delegation) gets a 7702 authorization to MetaMask's upgrade contract added to the same transaction: enabling Added Protection upgrades the EOA to a MetaMask smart account. An account already delegated to a MetaMask contract listed in the feature flags (`isAccountUpgradedToEIP7702` in [MetaMask/core](https://github.com/MetaMask/core/blob/main/packages/transaction-controller/src/utils/eip7702.ts)) is used as is.
- An account delegated to a third-party 7702 implementation (e.g. another wallet's) gets no authorization from the enforced-simulations path, so `executeFromExecutor` is not available; gas estimation fails and `applyTransactionContainers` throws before approval. Inferred from code, not tested.
- Only dapp-originated transactions (`origin` set and not MetaMask itself) are eligible.

## Safe

The enforcers work with any `IDeleGatorCore`. MetaMask's [DeleGatorModule](https://github.com/MetaMask/delegator-safe-module/blob/2428cdb727b4fd6476b86f001963284e5f444121/src/DeleGatorModule.sol) gives a Safe `executeFromExecutor` (Call only, default exec mode) and validates delegations through the Safe's ERC-1271 `isValidSignature`, so Safe threshold applies. A Safe with the module enabled can wrap its transactions with the same balance-change caveats; no Safe front end generates them from a simulation. The module grants the DelegationManager execution rights on the Safe, bounded by the caveats of each signed delegation. Added Protection itself only targets MetaMask smart accounts in the extension.

## Audits

- **Cyfrin, May 2025 part 2** ([pdf](https://github.com/MetaMask/delegation-framework/blob/b4b8c41fc912455a1af2300bfcd1276676b92a2e/audits/cyfrin/cyfrin-5-25-part2.pdf)): lead 0kage, Apr 28–30 2025, commit `42a2cb4c1d07`, manual review and stateful fuzzing. The four `*BalanceChangeEnforcer` contracts were in scope; no findings on them (2 informational findings on `LogicalOrWrapperEnforcer`). The four files are unchanged since `4aefa339c47f` (2025-04-24), so the deployed code is the audited code.
- **Consensys Diligence, 2024** ([Jun](https://github.com/MetaMask/delegation-framework/blob/b4b8c41fc912455a1af2300bfcd1276676b92a2e/audits/diligence/diligence-6-24.pdf), [Aug](https://github.com/MetaMask/delegation-framework/blob/b4b8c41fc912455a1af2300bfcd1276676b92a2e/audits/diligence/diligence-8-24.pdf), [Sep](https://github.com/MetaMask/delegation-framework/blob/b4b8c41fc912455a1af2300bfcd1276676b92a2e/audits/diligence/diligence-9-24.pdf)): predecessor `*BalanceGteEnforcer` contracts and DelegationManager. Medium (Jun 2024): weak lock allowed double counting in `ERC20BalanceGteEnforcer`; the current key includes token and delegation hash. Minor (Sep 2024, acknowledged): re-entrancy can affect the before/after balance pattern.
- **Cyfrin, Sep 2025** ([pdf](https://github.com/MetaMask/delegation-framework/blob/b4b8c41fc912455a1af2300bfcd1276676b92a2e/audits/cyfrin/cyfrin-9-25.pdf)): High-risk bypass in the separate `*TotalBalanceChangeEnforcer` family, not used by Added Protection.
- **Consensys Diligence, Nov 2025**: the Safe module ([pdf](https://github.com/MetaMask/delegator-safe-module/blob/2428cdb727b4fd6476b86f001963284e5f444121/audit/diligence-11-25.pdf)); not read.
- **Not audited (as far as found):** the extension's off-chain code that maps the simulation to caveats, where slippage and coverage are decided.
