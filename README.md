# Crypto Financial Crime Typologies — Reference & Detection Logic

A reference guide to seven core crypto financial crime typologies —
each with a definition, a worked example, a diagram, and formula-driven
detection logic that distinguishes it from the other six. Built
entirely in Excel formulas for the detection engine, with a Markdown
reference guide for the conceptual content. No Python, no SQL, no VBA.

This is the eighth project in an AML/KYC portfolio series, and the
one built specifically as a teaching/reference artifact — the kind of
document a crypto compliance team keeps to train new analysts and
ground detection-rule design in a shared vocabulary, rather than a
single case walkthrough.

## ⚠️ About the data

Every wallet address, transaction hash, and the referenced "Bridge
Protocol X" and scam report number are entirely fictional, generated
for this demonstration.

## What's in this repo

```
TYPOLOGIES.md                         ← the core reference document (read this first)
Crypto_Typologies_Reference.xlsx      ← the detection-logic workbook
README.md
data/
  typology_glossary.csv               ← definitions/red flags, standalone CSV
  transactions.csv                    ← all 31 example transactions, standalone CSV
  wallet_behavior_analysis_output.csv ← computed detection results, exported as CSV
screenshots/
  typology_glossary.png
  wallet_behavior_analysis.png
```

**Start with `TYPOLOGIES.md`** — it has the full definition, red
flags, real-world context, and a diagram for each of the seven
typologies. The workbook is the tested, working detection logic
behind it.

## The seven typologies

1. **Layering through crypto** — sequential wallet-to-wallet hops to
   add distance between source and destination.
2. **Mule wallets** — many small inbound sources consolidating at one
   wallet before a single forward.
3. **Rapid movement of funds** — funds passing through a wallet
   within minutes, repeated across cycles.
4. **Wallet consolidation/dispersal** — fan-in from several wallets,
   then fan-out to a new set, over a longer window.
5. **Mixer exposure** — direct interaction with a non-custodial
   mixing service.
6. **Scam/fraud-related flows** — funds originating from a
   wallet already on a scam/fraud report list.
7. **Cross-chain movement** — funds exiting the monitored chain via a
   bridge protocol.

## The hard part: telling four look-alike typologies apart

Mule wallets, rapid movement, consolidation/dispersal, and (each
single hop of) layering all look superficially similar — "funds pass
through a wallet." The real work in this project was designing each
mini-case so a simple formula genuinely distinguishes all four from
each other, not just from the three label-based typologies:

| Typology | In count | Out count | Dwell time | Value retention |
|---|---|---|---|---|
| Layering (one hop) | 1 | 1 | < 60 min | > 90% |
| Mule wallet | 4+ | 1 | < 24 hours | n/a |
| Rapid movement | 3+ | 3+ | < 10 min | n/a |
| Consolidation/dispersal | 3+ | 3+ | ≥ 60 min | n/a |

I tested every combination: all six "subject" wallets in the
workbook trigger **exactly one** of the four structural flags, and
every other flag correctly returns NO for them — 24 checks, zero
false positives, zero false negatives.

![Wallet behavior analysis](screenshots/wallet_behavior_analysis.png)

## Key formulas used

All standard Excel — no add-ins, no VBA:

- **`ISNUMBER` + `SEARCH`** — label-based detection for Mixer
  Exposure, Scam-Related Flows, and Cross-Chain Movement (does the
  counterparty label contain a known keyword?).
- **`COUNTIF` / `SUMIF`** — counting inbound/outbound transactions and
  total value per wallet.
- **`MINIFS`** — finding each wallet's earliest inbound and earliest
  outbound timestamp, to compute dwell time.
- **Nested `IF`/`AND`** — turning the four behavioral metrics (in
  count, out count, dwell time, value retention) into the four
  structural typology flags shown in the table above.

## Workbook structure

| Tab | Purpose |
|---|---|
| `README` | Methodology (same content as this file) |
| `Typology_Glossary` | Definition, red flags, and real-world context for all 7 typologies |
| `Transactions` | All 31 transactions across the 7 mini-cases, with the 3 label-based flags computed |
| `Wallet_Behavior_Analysis` | The 6 subject wallets, with the 4 structural flags computed and verified mutually exclusive |

![Typology glossary](screenshots/typology_glossary.png)

## How this connects to the rest of the portfolio

This project is deliberately the *reference layer* — each typology
here is shown in isolation, cleanly, for teaching purposes. Real
cases compound several of these at once: see this portfolio's
**On-Chain Transaction Investigation** (a peel chain combining
layering, mixer exposure, and exchange-side smurfing in one case) and
**Crypto Transaction Monitoring Case Study** (a single customer's
activity combining volume deviation, structuring, and mixer/darknet
wallet exposure) for worked examples of these typologies compounding
in a realistic investigation.

## Limitations (stated honestly)

- Synthetic data only — one small illustrative case per typology.
  Real detection systems validate thresholds (the 60-minute dwell
  cutoff, the 4-sender mule threshold, etc.) against an institution's
  actual transaction population, not a handwritten example.
- The four structural typologies are detected by a single wallet's
  aggregate in/out behavior. A production system would additionally
  need true distinct-counterparty deduplication (this dataset's
  counterparties are already unique, so a simple `COUNTIF` suffices
  here, but a wallet receiving repeat transactions from the same
  sender needs a proper distinct count — e.g. via Power Query or a
  pivot table — which plain `COUNTIF` does not provide).
- The label-based typologies (mixer, scam, cross-chain) are only as
  good as the reference data behind them — this is a stated design
  point in `TYPOLOGIES.md` §6, not just a limitation of this demo.
- No chain-linking logic to automatically string flagged layering
  hops into a confirmed multi-hop sequence — the workbook flags each
  hop individually, as noted in `TYPOLOGIES.md` §1.

## About

Built by [Your Name], CAMS-certified compliance analyst exploring
crypto-asset AML/compliance, as a portfolio piece. See also: [link to
SQL AML project], [link to Excel AML transaction monitoring project],
[link to Sanctions & PEP screening project], [link to Customer Risk
Rating Model project], [link to AML QA Review project], [link to
On-Chain Transaction Investigation project], [link to Crypto
Transaction Monitoring Case Study], and [link to Medium AML/crypto
compliance article series].
