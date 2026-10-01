# Crypto Financial Crime Typologies — Reference Guide

Seven typologies, each with a definition, why it matters, a worked
example, a diagram, and the detection logic that catches it. This is
the kind of reference document a crypto compliance team keeps to
train new analysts and ground detection-rule design in a shared
vocabulary.

> **Data disclosure:** every wallet address, transaction hash, and
> the referenced "Bridge Protocol X" and scam report number in this
> document are fictional, generated for this demonstration.

---

## 1. Layering through crypto

**Definition:** Moving funds through a sequential chain of wallets,
each one forwarding nearly all of what it received to the next, to
put distance between the original source and the eventual
destination.

**Why it matters:** This is the crypto-native equivalent of layering
cash through shell companies. The transactions themselves usually
have no other economic purpose — their only function is to add hops
between origin and destination.

```mermaid
flowchart LR
    SRC["Unattributed Source"] -->|25.0 ETH| H1["Intermediate Hop 1"]
    H1 -->|24.9 ETH, 15 min| H2["Intermediate Hop 2"]
    H2 -->|24.8 ETH, 15 min| H3["Intermediate Hop 3"]
    H3 -->|24.7 ETH, 15 min| SINK["Final Sink"]
```

**Detection logic:** A wallet with exactly one inbound and one
outbound transaction, a dwell time under 60 minutes between them, and
over 90% of the inbound value retained in the outbound transfer.
Flagged individually, each hop looks unremarkable — the real
detection value comes from chaining flagged hops together into a
sequence (see `Wallet_Behavior_Analysis` in the workbook), which a
production system would do via graph traversal rather than treating
each hop in isolation.

---

## 2. Mule wallets

**Definition:** A wallet that collects many small amounts from
numerous unrelated sources, then forwards the consolidated total
onward — typically controlled by someone recruited (often
unknowingly) to move funds on behalf of others.

**Why it matters:** Mule wallets are frequently tied to romance
scams, job scams ("money mule" recruitment), or distributed fraud
proceeds being gathered before a single cash-out.

```mermaid
flowchart LR
    S1["Source Wallet 1"] -->|1.2 ETH| HUB["Mule Wallet (Hub)"]
    S2["Source Wallet 2"] -->|0.9 ETH| HUB
    S3["Source Wallet 3"] -->|1.5 ETH| HUB
    S4["Source Wallet 4"] -->|1.1 ETH| HUB
    S5["Source Wallet 5"] -->|0.8 ETH| HUB
    HUB -->|5.4 ETH, consolidated| OUT["Cash-Out Wallet"]
```

**Detection logic:** A wallet receiving from 4 or more distinct
counterparties, followed by a single (or very few) consolidated
outbound transfer. The key distinguishing feature from
consolidation/dispersal (typology 4) is the outbound side: a mule
wallet forwards to essentially **one** destination, while a
consolidation/dispersal hub fans back out to **several**.

---

## 3. Rapid movement of funds

**Definition:** Funds that pass through a wallet almost immediately
upon receipt, repeated across multiple cycles, rather than being
held as part of ordinary wallet activity.

**Why it matters:** A human managing a personal wallet doesn't
typically move funds out again within minutes of receiving them,
three separate times in one day, to three different destinations.
This pattern is far more consistent with scripted or automated
laundering infrastructure.

```mermaid
flowchart LR
    P["Source P"] -->|10.0 ETH| X["Pass-Through Wallet"]
    X -->|9.9 ETH, 4 min| Q["Destination Q"]
    S["Source S"] -->|8.0 ETH| X
    X -->|7.9 ETH, 3 min| T["Destination T"]
    U["Source U"] -->|12.0 ETH| X
    X -->|11.9 ETH, 5 min| V["Destination V"]
```

**Detection logic:** 3 or more inbound AND 3 or more outbound
transactions at the same wallet, with the dwell time between the
first inbound and first outbound under 10 minutes. The sub-10-minute
threshold is what separates this from consolidation/dispersal, which
shares the "fan-in and fan-out at one wallet" shape but unfolds over
a much longer window.

---

## 4. Wallet consolidation/dispersal

**Definition:** A two-phase pattern: multiple wallets funnel funds
INTO a hub (consolidation), which then, after an interval, sends
funds OUT to a new set of wallets (dispersal) — commonly used to
break the link between a group of source wallets and a group of
destination wallets.

**Why it matters:** This shape is genuinely ambiguous on its own —
legitimate exchange and custodian treasury operations look similar.
The distinguishing factor in practice is usually the absence of any
other legitimate business context for the wallet, combined with
other red flags elsewhere in the case.

```mermaid
flowchart LR
    C1["Collector Wallet 1"] -->|15.0 ETH| HUB["Consolidation Hub"]
    C2["Collector Wallet 2"] -->|18.0 ETH| HUB
    C3["Collector Wallet 3"] -->|12.0 ETH| HUB
    C4["Collector Wallet 4"] -->|20.0 ETH| HUB
    HUB -.->|300 min later| HUB
    HUB -->|13.0 ETH| D1["Dispersal Wallet 1"]
    HUB -->|13.0 ETH| D2["Dispersal Wallet 2"]
    HUB -->|13.0 ETH| D3["Dispersal Wallet 3"]
    HUB -->|13.0 ETH| D4["Dispersal Wallet 4"]
    HUB -->|13.0 ETH| D5["Dispersal Wallet 5"]
```

**Detection logic:** 3 or more inbound AND 3 or more outbound
transactions at the same wallet — the same structural shape as rapid
movement — but with a dwell time of 60 minutes or more between the
first inbound and first outbound, consistent with the hub aggregating
funds over time before redistributing rather than passing them
straight through.

---

## 5. Mixer exposure

**Definition:** Direct interaction (deposit or withdrawal) with a
non-custodial mixing/tumbling service, which is specifically designed
to break deterministic on-chain traceability between the funds going
in and the funds coming out.

**Why it matters:** Legitimate privacy-seeking users of mixing
services exist, but mixer use is heavily overrepresented in
laundering flows and is treated as a significant risk indicator by
virtually every crypto compliance program.

```mermaid
flowchart LR
    SRC["Wallet Prior to Mixer"] -->|50.0 ETH| MIX["Known Mixer<br/>Deposit Address"]
    MIX -.->|"~5hr later, 48.5 ETH<br/>(inferred only)"| MIX2["Known Mixer<br/>Withdrawal Address"]
    MIX2 --> DST["Wallet After Mixer"]

    style MIX fill:#ffeb9c,stroke:#9c6500
    style MIX2 fill:#ffeb9c,stroke:#9c6500
```

**Detection logic:** Simple label lookup — does the counterparty
address match a known mixing service in a screening reference list?
**The critical honest caveat:** a withdrawal from a mixer can only be
*correlated* to a prior deposit by timing and amount, never provably
linked on-chain. Every project in this portfolio that touches a mixer
states this distinction explicitly rather than treating a plausible
correlation as proof.

---

## 6. Scam/fraud-related flows

**Definition:** Funds whose origin is a wallet that has been
independently identified — via a victim report, law enforcement
referral, or external database — as associated with a specific scam
or fraud scheme.

**Why it matters:** This typology is detected by **reference data**
(what's already known about a wallet), not by transaction pattern
analysis — which is exactly why keeping that reference data current
matters as much as the detection logic itself. A perfectly-designed
pattern-matching system catches nothing here if the scam-wallet list
behind it is stale.

```mermaid
flowchart LR
    V["Victim Wallet"] -->|7.5 ETH| SCAM["Reported Scam Wallet<br/>Investment Fraud<br/>Ext. Report #FR-2025-1182"]
    SCAM -->|7.4 ETH| L1["Post-Scam<br/>Layering Wallet 1"]
    L1 -->|7.3 ETH| L2["Post-Scam<br/>Layering Wallet 2"]

    style SCAM fill:#ffc7ce,stroke:#9c0006
```

**Detection logic:** Source (or, later in the chain, an upstream)
wallet matches an entry on an internal or external scam/fraud report
list, regardless of what the transaction pattern itself looks like.
Note how this case flows directly into typology 1 (layering) once the
scam proceeds start moving — in a real investigation, these typologies
compound rather than appearing in isolation.

---

## 7. Cross-chain movement

**Definition:** Funds that leave the chain being monitored via a
bridge protocol and reappear on a different blockchain, which breaks
single-chain analytics tooling unless the investigator specifically
correlates activity across chains.

**Why it matters:** This is an increasingly common evasion technique
precisely because most compliance tooling, and most analysts'
habitual workflow, is single-chain. Cross-chain movement exploits a
gap in *coverage*, not a gap in the blockchain's underlying
transparency — the data to follow the funds usually exists, just on
a different explorer than the one the analyst started with.

```mermaid
flowchart LR
    SRC["Wallet Prior to Bridge"] -->|22.0 ETH| BRIDGE["Cross-Chain Bridge Contract<br/>Bridge Protocol X"]
    BRIDGE -.->|"Correlated arrival on a<br/>different chain, net of fee<br/>(cross-chain correlation only)"| DST["Related Wallet<br/>on Destination Chain"]

    style BRIDGE fill:#ffeb9c,stroke:#9c6500
```

**Detection logic:** Outbound transaction to a known bridge contract
address. The real investigative work happens *off* the monitored
chain: confirming a correlated inbound arrival (by amount, net of
bridge fee, and timing) on the destination chain — which requires
either a multi-chain analytics tool or manually checking the
destination chain's own explorer.

---

## How these typologies compound in a real case

These seven rarely appear in isolation. The scam/fraud-related flow
example above (§6) shows proceeds moving directly into a layering
chain. A realistic investigation often finds several of these
typologies stacked in one case — see this portfolio's separate
**On-Chain Transaction Investigation** and **Crypto Transaction
Monitoring Case Study** projects for worked examples of exactly that:
multiple typologies compounding within a single fund-flow
investigation, rather than the clean, isolated examples shown here.
Those examples are deliberately kept simple and separated — this
document's job is to teach each pattern clearly on its own, not to
replicate the complexity of a live case.
