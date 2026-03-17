# TokenENS

## Overview

`TokenENS` is Makx's on-chain identity layer for launched tokens. Every token launched through Makx gets a human-readable ENS subdomain (e.g. `pepe.makx.eth`, `pepe.makx.bnb`). This makes tokens discoverable, linkable, and machine-readable without a centralized database. The contract is deployed once per chain and operated by `LaunchFactory`.

---

## For Users — What Does It Mean?

Every Makx token has a unique ticker — and that ticker becomes its web address. If you launch a token called `PEPE`:

- On Ethereum: `pepe.makx.eth` → `makx.fun/pepe`
- On Base: `pepe.makx.base.eth` → `makx.fun/pepe`
- On BNB: `pepe.makx.bnb` → `makx.fun/pepe`

Because the ticker is unique on the launchpad (enforced by `LaunchFactory`), there can only ever be one `pepe.makx.eth`. You can also resolve the subdomain directly in any ENS-compatible wallet or explorer to get the token's contract address, social links, and description — all stored on-chain.

---

## For Integrators & DEX Terminals

### Architecture

`TokenENS` is a wrapper around the ENS Registry and Public Resolver. It:
1. Creates an ENS subdomain under the Makx parent node.
2. Points it to the Public Resolver.
3. Sets the ETH address record → token contract address.
4. Batches all text records in a single `multicall`.

The `parentNode` is set at construction and is chain-specific (e.g. `namehash("makx.eth")` on Ethereum).

### ENS Text Records

Every registered token sets the following ENS text records:

#### Standard ENSIP Records

| Key | Value |
|---|---|
| `avatar` | Logo/avatar image URL or IPFS/NFT reference |
| `description` | Token or project description |
| `url` | Project website URL |
| `com.twitter` | Twitter/X handle |
| `org.telegram` | Telegram handle |

#### Makx-Specific Records

| Key | Value |
|---|---|
| `name` | Token display name |
| `io.makx.token.symbol` | Token ticker symbol |
| `io.makx.token.decimals` | Always `"18"` |
| `io.makx.token.totalSupply` | Total supply as a decimal string (no decimals) |
| `io.makx.token.address` | Token contract address (lowercase hex, `0x`-prefixed) |
| `io.makx.launchpad.address` | BondingCurve contract address (lowercase hex, `0x`-prefixed) |

#### ETH Address Record

The subdomain's ETH address (`addr(node)`) is set to the **token contract address**, not the bonding curve.

### Resolving a Token

Given a token symbol `sym`, resolve metadata via standard ENS calls on node `namehash(sym + ".makx.eth")` (or chain equivalent):

```js
const node = namehash(`${symbol.toLowerCase()}.makx.eth`)
const tokenAddress  = resolver.addr(node)
const curveAddress  = resolver.text(node, "io.makx.launchpad.address")
const name          = resolver.text(node, "name")
const description   = resolver.text(node, "description")
const twitter       = resolver.text(node, "com.twitter")
const telegram      = resolver.text(node, "org.telegram")
const website       = resolver.text(node, "url")
const avatar        = resolver.text(node, "avatar")
```

### Contract Interface

```solidity
struct TokenMetadata {
    string avatar;       // logo/avatar image URL or IPFS/NFT ref
    string description;  // project description
    string comTwitter;   // Twitter/X handle
    string comTelegram;  // Telegram handle
    string url;          // project website URL
}
```

#### `registerToken` (called by factory only)

```solidity
function registerToken(
    address tokenAddress,
    address bondingCurveAddress,
    string calldata label,       // subdomain label (e.g. "pepe")
    string calldata name,        // display name
    string calldata symbol,      // ticker
    string calldata totalSupply, // as decimal string
    TokenMetadata calldata meta
) external onlyAuthorized
```

Registers a subdomain and sets all records atomically. Reverts `SubdomainTaken` if the node already exists.

#### `updateRecord` / `updateRecords` (owner only)

```solidity
function updateRecord(address tokenAddress, string calldata key, string calldata value) external onlyOwner

function updateRecords(address tokenAddress, string[] calldata keys, string[] calldata values) external onlyOwner
```

Post-launch metadata updates (e.g. updating avatar or description). Only callable by the `TokenENS` owner (Makx team), not the token creator.

#### `transferSubdomain` (owner only)

```solidity
function transferSubdomain(address tokenAddress, address newOwner) external onlyOwner
```

Transfers ENS subdomain ownership to a new address. For migrations or token team handoffs.

### Key Mappings

```solidity
mapping(address => bytes32) public tokenNode;       // token address → ENS node
mapping(bytes32 => bytes32) public nodeLabelHash;   // node → label hash
```

### Events

```solidity
event TokenRegistered(address indexed tokenAddress, bytes32 indexed node, string label)
event TokenRecordUpdated(address indexed tokenAddress, string key, string value)
```

### Errors

| Error | Condition |
|---|---|
| `NotOwner` | `onlyOwner` function called by non-owner |
| `NotAuthorized` | `onlyAuthorized` function called by non-owner and non-factory |
| `TokenNotRegistered` | Update attempted for an unregistered token |
| `LengthMismatch` | `keys` and `values` arrays have different lengths in `updateRecords` |
| `SubdomainTaken` | Token address or subdomain label already registered |
