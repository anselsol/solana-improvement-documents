---
simd: 'XXXX'
title: Halve the Validator Admission Ticket
authors:
  - Ansel (@ansel_sol)
category: Standard
type: Core
status: Idea
created: 2026-11-10
feature: (fill in with feature key and github tracking issues once accepted)
extends: SIMD-0525
---

## Summary

Reduce the Validator Admission Ticket (VAT) to 50% of its current value at
every slot-time stage. On mainnet today (200ms stage) this lowers the VAT from
0.8 SOL to 0.4 SOL per epoch, or from ~0.8 SOL/day to ~0.4 SOL/day. The VAT
remains flat, charged per admitted vote account, and fully burned. Activation
is behind a feature gate.

## Motivation

SIMD-0326 set the VAT at 80% of pre-Alpenglow vote fees, "initially about 0.8
SOL per day", to "maintain the present economic equilibrium", and left
"proposals to make this mechanism adaptive and more fitting to future SIMDs".
SIMD-0357 describes the VAT as "only a temporary solution". SIMD-0525 kept the
wall-clock cost at ~0.8 SOL/day across slot-time stages. The amount has not
been revisited since Alpenglow
[activated in epoch 1064](https://solanacompass.com/news/solana-alpenglow-is-live-on-mainnet-as-anza-retires-towerbft-at-slot-459653000)
(2026-10-21).

Two things have changed.

1. The subsidy layer for small validators is shrinking. Under
   [Solana Foundation Delegation Program (SFDP) rules](https://solanacompass.com/news/solana-foundation-puts-a-12-month-limit-and-300000-sol-exit-on-its-validator-delegation-program)
   effective 2026-11-06, membership lasts at most 12 months, the program
   delegates to at most 100 validators, and members exit when non-SFDP stake
   exceeds 300,000 SOL. At epoch 1077, 337 validators held 23.74M SOL of
   Foundation stake. That is 48% of the 698 staked vote accounts at epoch 1082
   (mainnet RPC `getVoteAccounts`). As this support ends, the VAT is the
   largest protocol-level fixed cost a small validator must cover from its own
   revenue.

2. The admitted set is far from its cap. The VAT exists partly to keep the
   set from growing too large (SIMD-0357). Alpenglow separately enforces a hard
   cap: at most 2,000 vote accounts, selected by stake (SIMD-0326, SIMD-0357).
   Mainnet has 698 staked vote accounts, down from
   [roughly 2,500 in early 2023](https://www.theblock.co/news/ecosystems/2026-01-28-solana-validator-count-below-800-vote-transactions-drop-40-387108).
   A lower VAT cannot push the set past the cap, and stake-ordered selection
   keeps the cap binding if it is reached. Max Resnick's analysis, cited by
   [Placeholder](https://www.placeholder.vc/blog/2025/9/15/leveling-the-stakes-on-solana),
   puts a safe VAT at 0.135 SOL/day, so a 50% cut stays well above that floor.

A flat fee weighs most on low-stake validators. Halving the VAT halves the
stake at which commission covers it (see Impact).

Diversity is also a liveness issue. On 2026-08-12 a routing failure at one
hosting provider (AS20326, 27.34% of stake) took 28.83% of stake delinquent,
4.51 points from the 33.34% finality threshold
([Forklog](https://forklog.com/en/news/solana-nears-transaction-finalization-halt-due-to-teraswitch-outage)).

The author ran a mainnet validator for 2 years (shut down 2026-11-09): 1,144
SOL in rewards, 790 SOL in vote fees, 265 SOL in servers, 329 SOL net, year 2
at -0.7 SOL. Vote fees were the largest cost line.

## New Terminology

None. Uses VAT as defined in SIMD-0326 and SIMD-0357.

## Detailed Design

Introduce feature gate `halve_validator_admission_ticket` (name open to
reviewers).

When the feature is effective, the VAT table in SIMD-0525 ("Validator Admission
Ticket Scaling") MUST be replaced by:

| Effective target | Epoch | Current VAT (lamports) | New VAT (lamports) |
|------------------|-------|------------------------|--------------------|
| 400ms | 48h | 1_600_000_000 | 800_000_000 |
| 350ms | 42h | 1_400_000_000 | 700_000_000 |
| 300ms | 36h | 1_200_000_000 | 600_000_000 |
| 250ms | 30h | 1_000_000_000 | 500_000_000 |
| 200ms | 24h | 800_000_000 | 400_000_000 |

All other rules from SIMD-0357 and SIMD-0525 are unchanged: collection at the
epoch boundary, the VAT plus rent-exempt balance check, the 2,000 cap, sort
order and tie handling, the BLS public key requirement, the vote account as the
funding source, and burn to the incinerator.

Only the VAT constant applied at the epoch boundary changes. Admission rules,
consensus, and all other validator components are unchanged.

### Activation Timing

The feature MUST follow the same delay as the SIMD-0525 slot-time gates. If the
feature is activated during the epoch `E -> E+1` transition, the new amount is
first used on the `E+1 -> E+2` transition, for admission into `E+3`. Earlier
transitions use the current amount.

### Edge Cases

- A vote account holding at least the new VAT plus rent, but less than the
  current VAT plus rent, is admitted after the feature is effective and
  excluded before it.
- If a later SIMD-0525-style gate adds a new slot-time stage, its VAT MUST
  be 50% of the amount that stage would have under the current rule.
- Clients SHOULD test a vote account funded between the new and current VAT
  across activation: excluded before the effective epoch, admitted and charged
  the new amount from it.

## Alternatives Considered

Nil

## Impact

- Validators: VAT falls from 292.2 to 146.1 SOL/year per validator; at 5%
  commission, the stake needed for commission to cover the VAT drops from
  ~112k to ~56k SOL.
- Delegators: no direct impact. The VAT is charged to the vote account.
- Supply: at 698 staked vote accounts, annual burn falls by 101,976 SOL/year,
  ~0.45% of annual issuance (mainnet RPC `getVoteAccounts`, `getSupply`,
  `getInflationRate` at epoch 1082).
- Core contributors: a constant table change behind a feature gate in Agave
  and Firedancer, plus updated VAT tests.

## Security Considerations

- Sybil admission: bounded by the 2,000 cap and stake-ordered selection. A
  lower VAT makes it cheaper to hold a seat below the cap, which is the intent.
  It does not let low-stake vote accounts displace high-stake ones once the cap
  is reached.
- A lower VAT may increase the number of low-stake vote accounts below the cap.
- The VAT SHOULD be revisited if the admitted set exceeds 1,500 vote accounts,
  or when Rotor or MCP ship, whichever comes first.

## Backwards Compatibility

Requires a feature gate. Vote accounts funded for the current VAT remain
funded for the new one. No action is needed from operators.
