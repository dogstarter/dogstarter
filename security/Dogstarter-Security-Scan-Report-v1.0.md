# Dogstarter Contracts v1.0: Security Scan Report

Automated security scan results, fixes and testnet deployment · 23 November 2026

> **Important.** This report summarises automated security scans (FYEO Scanner, an AI-powered vulnerability scanner) together with our own tests and static analysis. It is **not** a professional manual audit. The contracts have not been through a professional manual audit yet; one is planned. Using the contracts carries smart-contract risk.

## 1. Summary

| Critical | High | Medium | Low | Info |
| :---: | :---: | :---: | :---: | :---: |
| 0 | 0 | 1 (accepted, mitigated) | 0 | 0 |

Final FYEO PR scan of commit `997c826` (head of PR #56, merged into main of the contracts repository). The only open item is the standard ERC-20 `approve()` allowance race (SWC-114) on the DogSwap LP token. `approve()` is kept because wallets, routers and aggregators require it; race-free `increaseAllowance`, `decreaseAllowance` and `safeApprove` were added, and the LP of the official (graduated) pools is locked forever in DogLockerV3. Every other finding from every scan is fixed and covered by a test.

## 2. Scope

Commit `997c826`, Solidity 0.8.24 (via-IR, optimizer on). In scope: the contracts that ship with Dogstarter Contracts v1.0.

| Area | Contracts |
| --- | --- |
| Launchpad (v1.0) | DogLaunchFactoryV3, DogLaunchV3Ext, DogLaunchV3Lib, DogBondingCurve, DogMemeTokenV3, DogTokenRegistryV3, DogLockerV3, DogQuoterV3, DogOrdersV3, DogSessionVaultV3 |
| Fees and governance | DogControlV3 (timelock, roles), DogFeeLedgerV3, DogHolderRewardsV3, DogMerkleRewardsV3, DogPaymentsV3, DogGiftsV3, DogHoldVaultV3 |
| Official pools | DogSwapV3 (DogSwapFactoryV3, DogSwapFeePairV3, DogSwapRouterV3, DogFeeTiersV3), DogSwapERC20 |
| Open DEX and tools | DogSwapOpenFactory, DogSwapOpenPair, DogSwapRouter, DogSwapAggregator, DogLPLocker, DogSwapOrders, DogSwapTrailing, DogDustSweeper, DogTokenProbe, DogLaunchGuard, WDOGE, libraries |
| Token | DogToken ($DOG), DogVesting, AirdropClaim |

## 3. Method

- FYEO Scanner full scans of commits `78b6ab0`, `d92e3d1` and `6a269ce`, then PR diff scans of every fix (`cc43ec1`, `f048cc7`, `2cbbbdf`, `a1273fe`, `997c826`).
- 151 unit, integration and stateful fuzz tests (Hardhat), including money invariants (ledger balances, curve reserves, fee tiers) over 9,000 random steps, and one test per scan finding.
- Slither static analysis (all results reviewed) and Solhint (0 errors). Contract sizes checked against EIP-170 (factory 22,026 bytes).
- A rehearsal of the full deployment on a local fork of DogeOS testnet before every real deployment, then a smoke test on testnet.

## 4. Findings and fixes

### FYEO full scan of 6a269ce (20 November 2026)

9 findings, all fixed in commit `4d04666`. Findings of the earlier full scans (`78b6ab0`, `d92e3d1`) are fixed in the same code line.

| Severity | Finding | Fix |
| --- | --- | --- |
| Medium | DogSwapTrailing: a permissionless poke could shape the confirmed peak; arm and fill could share a block | Every price is a pool TWAP over at least 5 minutes; arm and fill must be in different blocks; fill floor from the arming TWAP |
| Low | DogSwapAggregator took the official-pool fee only when WDOGE was one side of the pair | Fee taken on the pool's own quote token |
| Low | DogSwapTrailing: a long pump could inflate the peak so the stop never fills | Fill floor no longer derived from the peak; a fall re-arms at the new TWAP |
| Low | DogGiftsV3: a commit-reveal claim could be copied from a pending transaction | The claim code derives a one-time signing key; the code never goes on-chain |
| Low | DogPaymentsV3.pay did not check the amount against the price | Exact price enforced on-chain |
| Low | Rescue redemption: floor division left a rounding rest in the contract | The last redeemer receives the remainder |
| Info | Open DEX factory fee routing controlled by the deployer EOA | Ownership handed to the 48-hour timelock in the deploy step |
| Info | LP permit domain separator cached with the deploy chain id | Rebuilt when the chain id changes (EIP-712) |
| Info | LP permit accepted malleable ECDSA signatures | OpenZeppelin ECDSA (low-s only) |

### FYEO PR scans (21 to 23 November 2026)

| Scan of | Severity | Finding | Fix | Fix commit |
| --- | --- | --- | --- | --- |
| cc43ec1 | Medium | DogSwapTrailing fill(): the tip transfer assumed ERC-20 transfer returns bool | SafeERC20 for every arbitrary-token transfer | f4beae0 |
| cc43ec1 | Medium | DogSwapTrailing cancel(): assumed transfer returns bool (funds could lock) | SafeERC20; DogLPLocker withdraw as well | f4beae0, 2cbbbdf |
| f048cc7 | Medium | executeTokenBuyback priced the pool buy from spot reserves (flash-loan manipulable) | A pool TWAP of at least 5 minutes is required and the spot price must be within 3% of it, both ways | a1273fe |
| 2cbbbdf | Medium | DogSwapTrailing.place set the start peak from the spot price | No peak at placing; the first full 5-minute TWAP window sets it and never triggers | a1273fe |
| 2cbbbdf | Low | $DOG holder discount used an instantaneous balanceOf (flash-loanable; the feature is off) | New DogHoldVaultV3: deposits count from the next block; the discount accepts only the vault | a1273fe |
| cc43ec1 to 997c826 | Medium / Low | ERC-20 approve() allowance race (SWC-114) on the LP token | Mitigated: increaseAllowance, decreaseAllowance (reverts below zero), safeApprove (compare-and-set); approve() kept for ERC-20 compatibility | 997c826 |

**Accepted item: SWC-114 approve race.** A spender that is already approved could front-run an owner who changes a non-zero allowance to another non-zero value and spend both amounts. This applies to every standard ERC-20 token, including Uniswap V2 LP tokens; removing or restricting `approve()` would break wallets, routers and aggregators. Mitigation: the LP token offers `increaseAllowance`, `decreaseAllowance` (reverts below zero, so a front-run spend makes the decrease fail instead of granting more) and `safeApprove(spender, expectedCurrent, value)`; the LP of graduated (official) pools is locked forever. Covered by 6 tests, including the front-run scenario.

## 5. Admin controls

- One 48-hour timelock (DogControlV3) for every change of fees, wallets, limits, prices and roles; queued changes are public before they apply.
- A guardian can pause launches and buys (sells and claims keep working) and veto a pending holder-rewards list; only the owner can unpause.
- Hard limits in code: trading fee at most 1%, charity at least 2% of every fee, total buy fee at most 99%; pool fee tiers only go down.
- Fees are paid through a pull ledger; a failed payout keeps only its own credit.
- Mainnet: the owner will be a Safe multisig and the role wallets are separate from the deployer; the deployment script refuses mainnet without a Safe owner.

## 6. Testnet deployment (DogeOS Chikyu Testnet, chain 6281971)

Deployed on 23 November 2026 from the scanned code (start block 9717948). 23 of 23 contracts verified on Sourcify (exact match): `repo.sourcify.dev/6281971/<address>`. The smoke test passed: launch, buy, sell, open DEX launchpad guard, community pool liquidity and swaps, Boost purchase, holder-rewards funding, and a limit order filled by the keeper. The buyback TWAP guard was checked on a fork of testnet (after a dump, after a pump, at a fair price).

| Contract | Address |
| --- | --- |
| DogLaunchFactoryV3 | `0x7E6D018a10e0aA82912c8177d014e807B9a13C0D` |
| DogLaunchV3Ext (library) | `0x0143289C56431Fe228059df03F00b6aaC4cd37Ad` |
| DogControlV3 (48-hour timelock) | `0x9bc2861DD95ad8cC70D3e188E8574101d5A5016E` |
| DogFeeLedgerV3 | `0x3C0D230a78C3878E6aB736b36556818701C8Ee4E` |
| DogSwapFactoryV3 (official pools) | `0xBfD1E06b4cec6d768f31bcD9a0ea0e1B34e63845` |
| DogSwapRouterV3 | `0xf7B76092428012dDEE43DCEc38060545f7208734` |
| DogLockerV3 (LP locked forever) | `0x69B5563aDD3Ce10403A0C8Be78ea3E5aB6CcB2f4` |
| DogHolderRewardsV3 | `0xe0FD4aA1a0334Af2A04212A2F3e3D7Dd727E0763` |
| DogPaymentsV3 | `0xe659a33250f885CA7715eC2568e23eCd35e9bCb9` |
| DogHoldVaultV3 | `0x30b3D7e35fd97BF8Bb0dace3D00dFD56E07D39dF` |
| DogSwapOpenFactory | `0x59980f09C56C33B250CfFDaD53307EA12244b978` |
| DogSwapAggregator | `0x457af02E264EcE2345B79e8547BB16d31b114655` |

Website: [dogstarter.dog](https://dogstarter.dog). This report covers the smart contracts and automated analysis only; it does not cover the website, servers or key management.
