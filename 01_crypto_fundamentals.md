# 01 — Crypto Fundamentals

## What is a Blockchain

A blockchain is a distributed ledger. Instead of one bank keeping records, thousands of computers (nodes) all hold identical copies. To add a new record, majority of nodes must agree — this is **consensus**.

Key properties:
- **Immutable**: once written, data can't be changed without redoing all subsequent blocks
- **Transparent**: all transactions visible to all (public chains)
- **Permissionless**: anyone can participate without approval
- **Trustless**: rules enforced by math/code, not institutions

A **block** contains: list of transactions + hash of previous block + timestamp + nonce (for PoW). Chaining hashes is what makes it tamper-evident.

---

## Bitcoin

**Created**: 2009, Satoshi Nakamoto  
**Purpose**: Peer-to-peer electronic cash  
**Supply cap**: 21 million BTC, ever

### UTXO Model
Bitcoin doesn't have "accounts" — it has Unspent Transaction Outputs (UTXOs).

When you "have" 1 BTC, what you actually have is one or more UTXOs that sum to 1 BTC, locked to your public key.

To spend: consume UTXOs as inputs, create new UTXOs as outputs.

```
UTXO model:
  Input: [UTXO_A: 0.8 BTC] + [UTXO_B: 0.3 BTC]
  Output: [To Alice: 1.0 BTC] + [Change: 0.09 BTC] + [Fee: 0.01 BTC]
```

**Why it matters for quant**: UTXO chain analysis lets you trace coin flows. On-chain analytics firms (Chainalysis, Glassnode) build signals from this.

### Proof of Work (PoW)
Miners compete to find a nonce such that `hash(block + nonce) < target`. This is computationally expensive (energy cost = security). Winner adds the block and gets block reward.

- **Hashrate** = total computational power on network. Higher = harder to attack.
- **Difficulty** adjusts every 2016 blocks (~2 weeks) to keep block time ~10 min.
- **Halving**: block reward cuts in half every 210,000 blocks (~4 years). Creates supply shock narrative.

### Bitcoin as Quant Signal
- **Hash ribbon**: miner capitulation → accumulation signal
- **MVRV ratio**: market cap / realized cap — overvalued vs undervalued signal
- **Stock-to-Flow (S2F)**: scarcity model based on supply issuance rate

---

## Ethereum

**Created**: 2015, Vitalik Buterin  
**Purpose**: Programmable blockchain — a world computer  
**Current consensus**: Proof of Stake (since "The Merge", Sep 2022)

### Account Model
Unlike Bitcoin's UTXOs, Ethereum uses accounts:
- **Externally Owned Accounts (EOA)**: controlled by private key. Has ETH balance.
- **Contract Accounts**: controlled by code. Has ETH balance + code + storage.

When you send ETH or call a contract, you sign a transaction from your EOA.

### Gas
Every computation on Ethereum costs **gas**. Gas price (in Gwei, where 1 Gwei = 10⁻⁹ ETH) is bid by users.

```
Transaction fee = gas_used × gas_price
```

Since EIP-1559 (Aug 2021):
- **Base fee**: algorithmically set, burned (removed from supply)
- **Priority fee (tip)**: goes to validator
- **Max fee**: what you're willing to pay

Gas mechanics are critical for crypto quant — gas costs eat into arbitrage profits and smart contract execution costs.

### Proof of Stake (PoS)
Validators lock up ("stake") 32 ETH as collateral. Chosen pseudo-randomly (weighted by stake) to propose blocks. Others attest to blocks. Bad behavior → **slashing** (lose stake).

- More energy-efficient than PoW
- **Staking yield** ≈ 3-4% annually → ETH has a "risk-free rate" equivalent

---

## Tokens

### ERC-20 (Fungible Tokens)
Standard interface for fungible tokens on Ethereum. Every DeFi token (USDC, UNI, AAVE) is ERC-20.

Core functions:
```solidity
transfer(to, amount)
approve(spender, amount)
transferFrom(from, to, amount)
balanceOf(address) → uint256
totalSupply() → uint256
```

**For quant**: tokens can be created by anyone. Most are worthless. Market cap = circulating supply × price.

### ERC-721 (NFTs)
Non-fungible — each token has unique ID. No two are identical. Used for art, game items, domain names.

### ERC-1155
Hybrid: batch-transfer both fungible and non-fungible tokens in one transaction.

---

## Consensus Mechanisms Summary

| Mechanism | How security works | Examples | Notes |
|---|---|---|---|
| Proof of Work | Computational cost | Bitcoin, Litecoin | Energy-intensive, most battle-tested |
| Proof of Stake | Economic stake at risk | Ethereum, Cardano | Efficient, risk of centralization |
| Delegated PoS | Token holders vote for validators | EOS, TRON | Fast, but few validators |
| Proof of History | Cryptographic clock + PoS | Solana | Very fast (~65k TPS), complex |
| Proof of Authority | Pre-approved validators | BSC, private chains | Centralized, fast |

---

## Wallets

A wallet doesn't store coins — it stores **private keys**. Coins live on the blockchain.

### Key Pairs
- **Private key**: 256-bit random number. Secret. Generates public key via elliptic curve math.
- **Public key**: derived from private key. Used to derive your address.
- **Address**: shortened hash of public key. What you share.

```
Private key → (ECDSA secp256k1) → Public key → (Keccak-256 hash) → Address
```

Losing private key = losing funds. No recovery.

### HD Wallets (BIP-32/39/44)
**Hierarchical Deterministic** wallets derive all keys from one **seed phrase** (12 or 24 words).

```
Seed phrase → master private key → child keys (one per address)
```

This means one seed = infinite addresses. Back up the seed phrase = back up everything.

### Hot vs Cold
| Type | Internet connected | Security | Use case |
|---|---|---|---|
| Hot wallet | Yes | Lower | Active trading, DeFi |
| Cold wallet (hardware) | No | Higher | Long-term storage |
| Paper wallet | No | Physical risk | Archival |

---

## Exchanges

### Centralized Exchange (CEX)
Traditional order book model. You deposit funds, exchange holds custody. Exchange matches buy/sell orders.

Examples: Binance, Coinbase, Kraken, OKX, Bybit

**For quant**:
- REST APIs for order management
- WebSocket feeds for real-time order book
- Rate limits are a real constraint for HFT
- Withdraw/deposit latency is hours (blockchain confirmation)
- CEX order books look like equity markets — limit order books with bid/ask spread

### Decentralized Exchange (DEX)
Smart contracts hold funds. No custody risk. Trade directly from wallet.

Examples: Uniswap, Curve, dYdX, GMX

**For quant**:
- No KYC, no account
- Trades are on-chain transactions → gas cost + block time latency
- Price determined by automated market maker (AMM) formula, not order book
- Slippage is deterministic (function of pool depth and trade size)
- MEV (Miner Extractable Value) means your transactions can be front-run

### Key CEX Comparison (Crypto Quant Perspective)

| Exchange | API quality | Fee tier | Specialty |
|---|---|---|---|
| Binance | Excellent, low latency | Very low | Spot + futures + options |
| Bybit | Good | Low | Perpetuals focus |
| OKX | Good | Low | Options liquidity |
| Coinbase Advanced | Good | Higher | US regulated |
| Kraken | Good | Medium | EUR pairs, OTC |
| dYdX | On-chain/off-chain hybrid | Low | Decentralized perps |

---

## Crypto Market Structure

Unlike equities (one central exchange per country), crypto has **no central venue**. Same BTC trades on 50+ exchanges simultaneously.

### Spot vs Derivatives
- **Spot**: buy/sell actual asset for immediate delivery
- **Futures**: contract to buy/sell at future date at agreed price
- **Perpetual futures (perps)**: futures with no expiry date. Most liquid crypto derivative. Funding rate keeps price anchored to spot.
- **Options**: right but not obligation to buy/sell at strike price

### Funding Rate (Perpetual Futures)
Critical crypto concept. Mechanism to keep perp price ≈ spot price.

```
Every 8 hours:
  If perp_price > spot: longs pay shorts (positive funding)
  If perp_price < spot: shorts pay longs (negative funding)
```

Funding rate = signal of market sentiment. Persistently high positive funding = overleveraged longs = potential cascade liquidation.

### Liquidation Cascades
When leveraged traders get liquidated (margin call), exchange sells their position automatically → drives price down → liquidates more positions. This creates flash crashes.

**For quant**: monitoring liquidation levels is a risk signal. Open interest + funding rate + liquidation heatmaps are core crypto quant data.

---

## Stablecoins

Price-pegged tokens (usually to USD). Essential for:
- Avoiding crypto volatility while staying on-chain
- DeFi liquidity provision
- Cross-exchange arbitrage (move value without fiat wires)

| Type | Mechanism | Risk | Examples |
|---|---|---|---|
| Fiat-backed | 1:1 USD reserves | Counterparty / regulatory | USDC, USDT, BUSD |
| Crypto-backed | Overcollateralized with crypto | Liquidation cascade | DAI |
| Algorithmic | Seigniorage/rebasing | De-peg death spiral | UST (failed 2022) |

**USDT (Tether)**: largest by market cap, but opaque reserves. During stress, USDT depeg events happen.

**USDC (Circle)**: audited reserves, regulated. Preferred for institutional crypto quant.

---

## On-Chain Data as Signals

Unlike equities, crypto has transparent blockchain data. This is an extra alpha source.

| Signal | What it measures |
|---|---|
| Exchange inflows | Coins moving to exchanges → selling pressure |
| Exchange outflows | Coins leaving exchanges → long-term holding |
| Whale movements | Large wallet transfers |
| MVRV Z-score | Market value vs realized value → over/undervalued |
| SOPR | Profit ratio of moved coins → sentiment |
| Funding rate | Leveraged sentiment |
| Open interest | Total derivative exposure |
| Liquidation heatmap | Where forced selling/buying will happen |
| Miner flows | Miner sell pressure |

Data sources: Glassnode, CryptoQuant, Nansen, Dune Analytics, Token Terminal

---

## Next

Read [02_web3.md](02_web3.md) for DeFi, AMMs, and MEV — the crypto-native market structure layer.
