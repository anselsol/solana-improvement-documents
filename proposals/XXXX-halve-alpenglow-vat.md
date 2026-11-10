---
simd: 'XXXX'
title: Halve the Validator Admission Ticket
authors:
  - Ansel (anselsol)
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
been revisited since Alpenglow activated in epoch 1064 (2026-10-21) [1].

Two things have changed.

1. The subsidy layer for small validators is shrinking. Under Solana
   Foundation Delegation Program (SFDP) rules effective 2026-11-06, membership
   lasts at most 12 months, the program delegates to at most 100 validators,
   and members exit when non-SFDP stake exceeds 300,000 SOL [2]. At epoch 1077,
   337 validators held 23.74M SOL of Foundation stake [2]. That is 48% of the
   698 staked vote accounts at epoch 1082 [3]. As this support ends, the VAT
   is the largest protocol-level fixed cost a small validator must cover from
   its own revenue.

2. The admitted set is far from its cap. The VAT exists partly to keep the
   set from growing too large (SIMD-0357). Alpenglow separately enforces a hard
   cap: at most 2,000 vote accounts, selected by stake (SIMD-0326, SIMD-0357).
   Mainnet has 698 staked vote accounts [3], down from roughly 2,500 in early
   2023 [4]. A lower VAT cannot push the set past the cap, and stake-ordered
   selection keeps the cap binding if it is reached.

A flat fee weighs most on low-stake validators. Appendix A shows the stake at
which commission covers the VAT: halving the VAT halves that stake.

Validator diversity also matters for liveness. On 2026-08-12 a routing failure
at one hosting provider (AS20326, 27.34% of stake) took 28.83% of stake
delinquent, 4.51 points from the 33.34% finality threshold [5].

## New Terminology

None. Uses VAT as defined in SIMD-0326 and SIMD-0357.

## Detailed Design

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in [RFC
2119](https://www.ietf.org/rfc/rfc2119.txt) and [RFC
8174](https://www.ietf.org/rfc/rfc8174.txt).

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

### Validator Components Affected

| Validator Component             | Impact                              |
|---------------------------------|-------------------------------------|
| Transaction Execution (Runtime) | VAT constant at epoch boundary      |
| Virtual Machine                 | None                                |
| Block Packing                   | None                                |
| Consensus                       | None (admission rules unchanged)    |
| Gossip                          | None                                |
| Turbine                         | None                                |
| Snapshots                       | None                                |
| On-Chain Core BPF Programs      | None                                |
| Other (please describe)         | None                                |

## Alternatives Considered

- Keep the VAT at ~0.8 SOL/day. Keeps the burn; the fixed cost stays the same
  as SFDP support shrinks.
- Remove the VAT. Removes the only economic admission control. Out of scope.
- Stake-weighted or tiered VAT, raised in the SIMD-0326 forum thread [6]. A
  SIMD-0326 author noted that tiers encourage splitting validators and that a
  stake-weighted VAT lets large operators split stake to push out small ones
  [6]. A flat cut avoids both issues.
- A larger cut. Placeholder cites an analysis by Max Resnick that the VAT "can
  be safely reduced to 0.135 SOL/day" [7]. A bigger change, and harder to agree
  on. A 50% cut is a smaller step that can be measured first.
- An adaptive VAT tied to admitted-set size relative to the 2,000 cap. A
  strong long-term option, proposed as follow-up work, especially once Rotor or
  MCP change the cost of a large set.

## Impact

- Validators: VAT falls by 0.4 SOL per epoch at the 200ms stage, or 146.1
  SOL/year at the SIMD-0525 nominal 24h epoch (Appendix A).
- Delegators: no direct impact. The VAT is charged to the vote account.
- Supply: at 698 staked vote accounts, annual burn falls from ~203,951 SOL to
  ~101,976 SOL. That 101,976 SOL difference is ~0.45% of current annual
  issuance of ~22.71M SOL (Appendix B).
- Core contributors: a constant table change behind a feature gate in Agave
  and Firedancer, plus updated VAT tests.

## Security Considerations

- Sybil admission: bounded by the 2,000 cap and stake-ordered selection. A
  lower VAT makes it cheaper to hold a seat below the cap, which is the intent.
  It does not let low-stake vote accounts displace high-stake ones once the cap
  is reached.
- The VAT SHOULD be revisited if the admitted set exceeds 1,500 vote accounts,
  or when Rotor or MCP ship, whichever comes first.

## Drawbacks

- Less SOL burned (see Impact).
- Lower cost to run a minimal-stake validator, which may increase the number of
  low-stake vote accounts below the cap.

## Backwards Compatibility

Requires a feature gate. Vote accounts funded for the current VAT remain
funded for the new one. No action is needed from operators.

## Conformance

Each client will include tests that, at the epoch boundary after the feature
is effective, deduct the new VAT amount for each stage in the table above. A
localnet ledger will show admission and burn amounts before activation, at
activation, and after the effective epoch, including a vote account funded
between the new and current thresholds.

## References

1. Solana Compass, "Solana Alpenglow Is Live on Mainnet as Anza Retires
   TowerBFT at Slot 459,653,000":
   https://solanacompass.com/news/solana-alpenglow-is-live-on-mainnet-as-anza-retires-towerbft-at-slot-459653000
2. Solana Compass, SFDP rule change (citing the Solana Foundation delegation
   dashboard):
   https://solanacompass.com/news/solana-foundation-puts-a-12-month-limit-and-300000-sol-exit-on-its-validator-delegation-program
3. Mainnet RPC `getVoteAccounts`, `getSupply`, `getInflationRate`, epoch 1082
   (slot ~467,823,000): 698 vote accounts with active stake (657 current, 41
   delinquent), 434.98M SOL active stake.
4. The Block, 2026-01-28:
   https://www.theblock.co/news/ecosystems/2026-01-28-solana-validator-count-below-800-vote-transactions-drop-40-387108
5. Forklog, 2026-08-12:
   https://forklog.com/en/news/solana-nears-transaction-finalization-halt-due-to-teraswitch-outage
6. SIMD-0326 forum thread:
   https://forum.solana.com/t/simd-0326-proposal-for-the-new-alpenglow-consensus-protocol/4236
7. Placeholder, "Leveling the Stakes on Solana", 2025-09-15:
   https://www.placeholder.vc/blog/2025/9/15/leveling-the-stakes-on-solana

## Appendix A: VAT Cost and Break-Even Stake

Annual VAT uses the SIMD-0525 200ms value of 157,784,629.968 slots/year and
432,000 slots/epoch, giving 365.24 epochs/year. Measured mainnet slot time was
~214.5ms over the last 30 performance samples at epoch 1082, or ~340.6
epochs/year; the nominal figure is used below.

| | Current | Proposed |
|---|---|---|
| VAT per epoch (200ms) | 0.8 SOL | 0.4 SOL |
| VAT per year (365.24 epochs) | 292.2 SOL | 146.1 SOL |

Stake needed for commission income alone to cover the VAT:

```
staking yield y = annual issuance / active stake
                = 22.71M / 434.98M = 5.22%
break-even stake = annual VAT / (commission * y)
```

| Commission | Current VAT | Proposed VAT |
|------------|-------------|--------------|
| 5% | ~111,900 SOL | ~56,000 SOL |
| 10% | ~56,000 SOL | ~28,000 SOL |

This excludes block rewards, priority fees, MEV, and hardware costs, so it is a
lower bound on the stake a validator needs. It shows the scale of the change,
not a full P&L.

## Appendix B: Burn and Issuance

From mainnet RPC at epoch 1082 [3]: total supply 637.31M SOL, inflation rate
3.563%.

```
annual issuance       = 637.31M * 3.563%            = 22.71M SOL
current annual burn   = 698 * 0.8 SOL * 365.24      = 203,951 SOL
proposed annual burn  = 698 * 0.4 SOL * 365.24      = 101,976 SOL
reduction / issuance  = 101,976 / 22.71M            = 0.45%
```
