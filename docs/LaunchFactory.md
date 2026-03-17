# LaunchFactory

## Overview

The `LaunchFactory` is the central contract of the Makx launchpad. It is the single entry point for creating new meme tokens — whether through a standard launch or a community-pooled **Prepool** launch. Every token created through Makx is tracked here, and each token gets a unique on-chain identity via ENS (Ethereum Name Service) subdomains.

---

## For Users — What Does It Do?

When someone wants to launch a token on Makx, they call the factory. The factory takes care of everything in one transaction:

1. **Checks the ticker is unique** — No two tokens can have the same symbol. First come, first served, forever.
2. **Deploys the token** — A new ERC-20 token contract is created with the full supply minted upfront.
3. **Deploys the bonding curve** — A dedicated price curve contract is created for this token. This is where people buy and sell the token before it graduates to Uniswap.
4. **Registers an ENS subdomain** — The token gets a name like `pepe.makx.eth` or `pepe.makx.bnb`, making it discoverable at `makx.fun/pepe`.
5. **Optionally does a dev buy** — The creator can send ETH with the launch transaction to buy the first tokens immediately, in the same block, preventing snipers.

### Two Launch Types

| | Standard Launch | Prepool Launch |
|---|---|---|
| Who sets it up | A single dev | Community contributors |
| First buy | Dev buys in same tx | All contributors buy together |
| Liquidity after graduation | Locked forever (burned LP) | Held by Prepool — earns fees forever |
| Fees | Go to platform | Shared between contributors and platform |

---

## For Integrators & DEX Terminals

### Deployment

`LaunchFactory` is deployed once per chain. It is chain-aware — `positionManager`, `permit2`, and `weth` are set at construction and are immutable.

Chains supported:
- **Base** — ENS parent: `makx.base.eth`
- **BNB Chain** — ENS parent: `makx.bnb`
- **Ethereum** — ENS parent: `makx.eth`

### Key State

```solidity
uint256 public launchCount;

struct LaunchInfo {
    address token;        // ERC-20 token address
    address bondingCurve; // BondingCurve clone address
    address creator;      // deployer EOA
    uint256 createdAt;    // block.timestamp
}

mapping(uint256 => LaunchInfo) public launches;
mapping(address => uint256)    public tokenToLaunchId;
mapping(address => bool)       public isBondingCurve;
mapping(address => bool)       public isPrepool;
mapping(bytes32 => bool)       public symbolTaken;  // keccak256(symbol) => taken

// Launch parameters
uint16 public prepoolAllocationBps; // % of total supply sent directly to prepool at mint (taken from LP reserve)
```

### Core Write Functions

#### `createLaunch`

```solidity
function createLaunch(
    string calldata name,
    string calldata symbol,
    bytes32 salt,
    TokenENS.TokenMetadata calldata meta,
    uint256 minDevBuyTokens  // slippage protection; ignored if msg.value == 0
) external payable returns (address tokenAddr, address curveAddr)
```

Deploys a standard launch. Pass `msg.value > 0` for an atomic dev buy (sniper-proof, same tx).

- Reverts `SymbolTaken` if the ticker is already registered.
- `salt` is combined with `msg.sender` to make clone addresses deterministic and front-run-resistant.

#### `createPrepool`

```solidity
function createPrepool(
    bytes32 salt,
    string calldata name,
    string calldata symbol,
    TokenENS.TokenMetadata calldata meta,
    uint256 minBalance,       // 0 = no minimum ETH required
    uint256 deployTime,       // 0 = no time-based trigger
    uint256 minContribution,  // minimum ETH per contributor
) external payable returns (address prepoolAddr)
```

Deploys a Prepool contract. The salt **must** produce an address whose lower 14 bits satisfy the Uniswap V4 hook flags:

- Bit 6: `AFTER_SWAP`
- Bit 2: `AFTER_SWAP_RETURNS_DELTA`

Use `predictPrepoolAddress(salt)` to find a valid salt off-chain before calling.

#### `createPrepoolLaunch`

```solidity
function createPrepoolLaunch(
    string calldata name,
    string calldata symbol,
    bytes32 curveSalt,
    TokenENS.TokenMetadata calldata meta
) external payable returns (address tokenAddr, address curveAddr, uint256 tokensOut)
```

Called **only** by a registered Prepool contract (enforced). Deploys the token + bonding curve for a prepool. The prepool address becomes the LP recipient, the Uniswap V4 hook, and the bonding curve fee recipient.

Token supply is split at mint:
- `prepoolAllocationBps` of total supply is sent **directly to the prepool** (taken from the LP reserve, not the sale).
- The remainder goes to the bonding curve for sale + LP.

`tokensOut` is the number of tokens the prepool received from its initial buy (used by the prepool to snapshot `initialTokenBalance`).

### View Functions

```solidity
// Predict the bonding curve clone address before deployment
function predictCurveAddress(bytes32 salt) external view returns (address)

// Predict the prepool clone address before deployment
function predictPrepoolAddress(bytes32 salt) external view returns (address)
```

### Admin Functions (owner only)

```solidity
function setParams(
    uint256 _totalSupply,
    uint16 _platformFeeBps,       // max 1000 (10%)
    uint256 _virtualEth,
    uint256 _virtualTokens,
    uint256 _salePercentBps,      // must be 1–10000
    uint16 _prepoolAllocationBps  // salePercentBps + prepoolAllocationBps must be ≤ 10000
) external onlyOwner

function setTokenENS(TokenENS _tokenENS) external onlyOwner
```

### Events

```solidity
event LaunchCreated(
    uint256 indexed launchId,
    address indexed token,
    address indexed bondingCurve,
    address creator,
    string name,
    string symbol
)

event PrepoolCreated(
    address indexed prepool,
    address indexed creator,
    string symbol
)

event DevBuy(
    uint256 indexed launchId,
    address indexed creator,
    uint256 ethSpent
)

event ParamsUpdated()
```

### Errors

| Error | Condition |
|---|---|
| `SymbolTaken` | Token ticker already registered |
| `InvalidParams` | Bad param values in `setParams` |
| `OnlyPrepool` | `createPrepoolLaunch` called by non-prepool |
| `InvalidHookAddress` | Prepool salt produces wrong V4 hook flag bits |
