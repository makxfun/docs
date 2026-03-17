# Makx Launchpad — Overview

Makx is a meme token launchpad that uses an AMM bonding curve to bootstrap liquidity, graduating tokens to Uniswap V4 once they sell out. Every token gets a unique ENS subdomain, and a community-pooled launch mode (Prepool) lets groups of people launch tokens together and share trading fees forever.

---

## Contracts at a Glance

| Contract | Purpose |
|---|---|
| [`LaunchFactory`](./LaunchFactory.md) | Entry point — deploys tokens, bonding curves, and prepools |
| [`BondingCurve`](./BondingCurve.md) | Per-token AMM trading engine, auto-graduates to Uniswap V4 |
| [`Prepool`](./Prepool.md) | Community pooled launch + lifelong fee-sharing mechanism |
| [`TokenENS`](./TokenENS.md) | On-chain identity — registers ENS subdomains for every token |

---

## How It Works

### Standard Launch

```
Creator → LaunchFactory.createLaunch()
              ↓
         Token deployed (full supply → BondingCurve)
         BondingCurve deployed (clone)
         ENS subdomain registered (e.g. pepe.makx.eth)
         Optional dev buy in same tx (sniper-proof)
              ↓
         Public trades via BondingCurve.buy() / .sell()
              ↓
         All tokens sold → auto-graduates to Uniswap V4
         LP NFT → dead address (locked forever)
```

### Prepool Launch

```
Creator → LaunchFactory.createPrepool()
              ↓
         Prepool contract deployed (clone, valid V4 hook address)
              ↓
         Contributors → Prepool.contribute()
              ↓
         Conditions met (minBalance and/or deployTime)
              ↓
         Anyone → Prepool.deploy()
              → LaunchFactory.createPrepoolLaunch()
              → Token + BondingCurve deployed
              → All pooled ETH buys tokens (everyone equal footing)
              → LP NFT → Prepool (not dev, not dead address)
              ↓
         After 48h → contributors can Prepool.withdrawTokens()
                      (forfeits future fee earnings)
              ↓
         Trading fees (hook 1% + LP fees + platform fees)
              → 50% Makx team
              → 50% active contributors (proportional to contribution)
              → earned forever while contribution stays in pool
```

---

## Chains

Makx is deployed on multiple chains. DEX and naming addresses are parameterized at factory construction:

| Chain | ENS Parent | DEX |
|---|---|---|
| Ethereum | `makx.eth` | Uniswap V4 |
| Base | `makx.base.eth` | Uniswap V4 |
| BNB Chain | `makx.bnb` | Uniswap V4 (BNB) |

---

## Key Properties

- **Unique tickers** — Each token symbol is registered once. `SymbolTaken` reverts any duplicate.
- **Deterministic addresses** — Bonding curves and prepools use `CREATE2` clones. Addresses are predictable before deployment.
- **No admin keys on liquidity** — Normal launch LP is permanently burned. Prepool LP is locked in the contract.
- **On-chain metadata** — All token info (name, social links, contract addresses) lives in ENS text records, queryable without a backend.
- **Sniper resistance** — Dev buy happens in the same transaction as token creation. There is no window to front-run.
- **Prepool equal footing** — The entire pooled ETH is the "dev buy". No single participant enters before any other.

---

## Fee Summary

| Fee | Rate | Recipient |
|---|---|---|
| Platform fee (buy/sell on curve) | Configurable (max 10%) | Factory owner (Makx) |
| Uniswap V4 pool fee | 0.3% | LP position holders |
| Prepool hook fee (post-graduation swaps) | 1% of output | 50% Makx / 50% contributors |
| Prepool LP fee share | From Uniswap LP position | 50% Makx / 50% contributors |
