# UTXO — Deep Dive

> Supplement to [01_crypto_fundamentals.md](01_crypto_fundamentals.md)

---

## The Core Mental Model

Banks track balances: "Alice has $100." Bitcoin doesn't. Bitcoin tracks **coins**, not balances.

Think of UTXO like physical cash bills in your wallet. You don't have "a balance of $47" — you have a $20 bill, a $20 bill, a $5 bill, and two $1 coins. To pay $37, you hand over bills, get change back.

UTXOs are those bills. Each one is a **discrete chunk of bitcoin** locked to a public key.

---

## What a UTXO Actually Is

Every UTXO has exactly three properties:

```
UTXO = {
    txid:   "abc123...",   // which transaction created this output
    index:  0,             // which output in that transaction (0-indexed)
    value:  0.5 BTC,       // how much it's worth
    script: OP_DUP OP_HASH160 <pubkey_hash> OP_EQUALVERIFY OP_CHECKSIG
            // locking script: "only person with private key matching this hash can spend me"
}
```

The `script` is the lock. Your private key is the key.

---

## Spending: Consuming Inputs, Creating Outputs

A Bitcoin transaction does one thing: **destroy some UTXOs (inputs) and create new UTXOs (outputs)**.

```
TRANSACTION
├── Inputs (UTXOs being destroyed):
│     ├── Input 0: UTXO [txid=abc123, index=0] = 0.5 BTC
│     │            + unlocking script (signature proving you own it)
│     └── Input 1: UTXO [txid=def456, index=2] = 0.3 BTC
│                  + unlocking script
│
└── Outputs (new UTXOs being created):
      ├── Output 0: 0.7 BTC → Alice's address    (payment)
      └── Output 1: 0.09 BTC → your own address  (change)

      Fee (implicit): 0.5 + 0.3 - 0.7 - 0.09 = 0.01 BTC → miner
```

**Rules**:
- Every input must be a real, unspent UTXO
- Sum of inputs ≥ sum of outputs (difference = miner fee)
- You must provide valid signature to spend each input
- Once consumed, a UTXO is gone forever — can never be spent again

---

## The "Change" Problem

You have one UTXO worth 1 BTC. You want to pay 0.3 BTC.

You **cannot** split the UTXO. Must consume the whole thing as input, create two outputs:

```
Input:  [1.0 BTC UTXO]
Output 0: 0.3 BTC → Bob         (payment)
Output 1: 0.699 BTC → yourself  (change, your new UTXO)
Fee:    0.001 BTC               (to miner, implicit)
```

Your wallet handles this automatically. Your wallet contains a collection of UTXOs of various sizes — not a simple balance.

---

## Your "Balance" is Just a Sum

When your wallet shows "1.5 BTC", it's lying a little. What it actually has:

```
UTXO set for your keys:
  [txid=aaa, index=1]:  0.5 BTC
  [txid=bbb, index=0]:  0.8 BTC
  [txid=ccc, index=2]:  0.2 BTC
  ─────────────────────────────
  "Balance":            1.5 BTC
```

Wallet scans entire Bitcoin blockchain, finds all UTXOs locked to your keys, sums them → your balance.

---

## UTXO Selection (Coin Selection)

When you send BTC, wallet must choose **which UTXOs to use as inputs**. Non-trivial:

```
Goal: pay 0.4 BTC
Your UTXOs: [0.5], [0.8], [0.2]

Option A: use [0.5] → change = 0.099  (one input, small tx, low fee)
Option B: use [0.2] + [0.3]           (two inputs, bigger tx, higher fee)
Option C: use [0.8] → change = 0.399  (creates large change UTXO)
```

Bad coin selection → overpay fees OR create "dust" UTXOs (tiny amounts worth less than the fee to spend them).

Wallet algorithms (Branch and Bound) optimize this. Poor wallets do it badly → privacy leaks + wasted fees.

---

## Why UTXO Enables Parallel Validation

Bitcoin's architectural advantage. Each UTXO is independent.

```
Block has 3000 transactions. To validate:

UTXO model (Bitcoin):
  Tx1 uses [UTXO_A, UTXO_B] → validate A and B in parallel
  Tx2 uses [UTXO_C, UTXO_D] → validate C and D in parallel (independent of Tx1!)
  Tx3 uses [UTXO_E]         → parallel

  All 3000 transactions can be validated simultaneously IF they don't share inputs
  → massive parallelism possible

Account model (Ethereum):
  Tx1: Alice sends Bob
  Tx2: Alice sends Carol  ← must wait — Alice's nonce might conflict

  Sequential execution required for same account
```

Bitcoin UTXO validation parallelizes naturally. Ethereum needs complex schemes (Parallel EVM) to achieve same.

---

## UTXO vs Account Model

| Property | UTXO (Bitcoin) | Account (Ethereum) |
|---|---|---|
| State unit | Individual coins | Account balance |
| Spend atomicity | All-or-nothing per UTXO | Partial balance change |
| Parallelism | Natural | Complex |
| Privacy | Better (new address per tx) | Worse (persistent address) |
| Smart contracts | Hard (Script is limited) | Natural (Solidity has state) |
| Light client | Efficient (UTXO set) | Harder (need full state) |
| Double-spend prevention | UTXO spent or not | Nonce ordering |

---

## UTXO and Privacy

Each UTXO can go to a **different address**. Good wallets generate fresh address for every change output.

```
Bad practice (privacy leak):
  You pay Bob from address A
  Change goes back to address A
  → analyst knows: A paid Bob AND A still controls the change

Good practice:
  You pay Bob from address A
  Change goes to new address B (your fresh key)
  → analyst can't trivially link B to A without graph analysis
```

This is why Bitcoin has HD wallets — derive unlimited addresses from one seed so every transaction uses fresh addresses.

---

## The UTXO Set

Full set of all unspent UTXOs on Bitcoin at any moment.

```
Bitcoin UTXO set (2025):
  ~85 million UTXOs
  ~5-6 GB in memory (compact)

This is the minimal state a full node must hold to validate new transactions.
(Compare: Ethereum state is ~100+ GB)
```

Every mined block:
- Consumed UTXOs → removed from set
- New UTXOs → added to set

Set grows when people receive and don't spend. Shrinks when people consolidate.

"Dust attacks": malicious actors deliberately create millions of tiny UTXOs to bloat the set and slow down nodes.

---

## UTXO as Quant Signal

On-chain UTXO analysis = extra alpha source unavailable in traditional markets.

### UTXO Age Bands (HODL Waves)

```
Categorize every UTXO by how long it's been unspent:
  < 1 day, 1d-1w, 1w-1m, 1m-3m, 3m-6m, 6m-1y, 1y-2y, 2y-5y, 5y+

Pattern:
  Long-dormant UTXOs (1y+) start moving → "old coins" waking up
  → holders selling into strength = distribution = bearish signal
  
  Short-term UTXOs growing (people buying recently) = new demand = bullish
```

### Realized Price & Realized Cap

```
For each UTXO:
  cost_basis = BTC price at time UTXO was last moved

Realized Cap = sum of (utxo_value × price_when_last_moved)
             = what the market collectively "paid" for all Bitcoin

MVRV = Market Cap / Realized Cap

MVRV > 3.5: market priced far above collective cost basis → historically overbought, sell zone
MVRV < 1.0: market below collective cost basis → capitulation, historically best buy zone
MVRV ≈ 1.0: fair value
```

Historical peaks:
- Dec 2017: MVRV = 4.8 (top of bull market)
- Dec 2018: MVRV = 0.7 (bear market bottom)
- Nov 2021: MVRV = 3.9 (top of bull market)
- Nov 2022: MVRV = 0.8 (FTX collapse bottom)

### SOPR (Spent Output Profit Ratio)

```
For each transaction output being spent:
  SOPR = value_at_spend_time / value_at_creation_time
       = current_price / price_when_UTXO_was_created

SOPR > 1: coins moving are in profit on average (holders selling at gain)
SOPR < 1: coins moving are at a loss (capitulation — people selling below cost)
SOPR = 1: breaking even → key support/resistance level
```

Pattern:
- Bull market: SOPR consistently above 1 (every dip gets bought, people sell profits)
- Bear market: SOPR dips below 1 (people selling at losses, capitulation)
- SOPR bouncing off 1.0 from above = strong support in bull market

### Exchange Flow Signals

```
Exchange inflows: UTXOs moving TO exchange wallets
  → holders preparing to sell → bearish pressure

Exchange outflows: UTXOs moving FROM exchange wallets
  → holders withdrawing to cold storage → long-term holding → bullish

Net flow = inflows - outflows
  Sustained negative net flow = supply leaving exchanges = bullish
```

This is why on-chain analysts track exchange wallet balances. Glassnode tags ~95% of all exchange UTXOs by fingerprinting known exchange addresses.

### CDD (Coin Days Destroyed)

```
Coin days = BTC_amount × days_unspent

CDD = coin days destroyed when UTXO spent

1 BTC unspent 100 days = 100 coin days
When that UTXO is spent: 100 coin days destroyed

High CDD spike = old coins moving = significant holder activity
  Usually precedes major price moves
```

---

## Script Types (How UTXOs Are Locked)

Different locking mechanisms create different UTXO types:

| Script Type | Format | Use |
|---|---|---|
| P2PKH | Pay to Public Key Hash | Standard address (1xxx) |
| P2SH | Pay to Script Hash | Multisig (3xxx) |
| P2WPKH | Pay to Witness PubKey Hash | SegWit (bc1q...) |
| P2WSH | Pay to Witness Script Hash | SegWit multisig (bc1q...) |
| P2TR | Pay to Taproot | Taproot (bc1p...) |
| Multisig | M-of-N signatures required | Exchanges, cold storage |
| Timelock | Can't spend until block height | HTLCs, Lightning |

**Taproot (P2TR, activated Nov 2021)**:
- All scripts look identical on-chain (privacy)
- Complex scripts (multisig, Lightning) look same as single-sig
- Enables more efficient smart contracts on Bitcoin (Ordinals uses this)

---

## UTXO in Lightning Network

Lightning = payment channels built on Bitcoin UTXOs.

```
Opening a channel:
  Alice + Bob create a 2-of-2 multisig UTXO together (the "funding UTXO")
  This UTXO stays on-chain, locked

Off-chain payments:
  Alice sends Bob 0.01 BTC → update off-chain balance sheet, no UTXO change
  Bob sends Alice 0.005 BTC → update off-chain, no UTXO
  
  Thousands of payments: zero on-chain transactions

Closing a channel:
  Broadcast final balance settlement → one on-chain UTXO created per party
  Funding UTXO consumed, two new UTXOs created
```

Lightning lets Bitcoin scale to millions of transactions per second while UTXO set stays small on-chain.

---

## Summary

```
UTXO = discrete coin, locked to key, must be consumed whole
Balance = sum of your UTXOs
Tx = consume input UTXOs, create output UTXOs, difference = fee
Change = new UTXO sent to yourself
Parallel validation = UTXOs independent → Bitcoin's speed advantage
On-chain signals = UTXO age + cost basis + exchange flows = alpha
```

Back to [01_crypto_fundamentals.md](01_crypto_fundamentals.md) or continue to [02_web3.md](02_web3.md).
