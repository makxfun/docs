# BondingCurve

## Overview

The `BondingCurve` contract is the trading engine for every Makx token. It controls buying and selling before the token is ready for a real DEX, and automatically migrates liquidity to Uniswap V4 when the token sells out. Each token has its own bonding curve — a lightweight clone deployed by `LaunchFactory`.

---

## For Users — How Does It Work?

Think of the bonding curve as an automatic market maker with a built-in price ladder. The more people buy, the higher the price goes. If someone wants to sell back, the price goes down.

### Buying

You send ETH, you get tokens. The price you pay depends on how many tokens have already been sold — early buyers get cheaper prices. A small platform fee is deducted from every buy.

### Selling

You can sell tokens back to the curve at any time before the token graduates. The curve sends you ETH back, again minus a small platform fee.

### Graduation

When all the tokens available on the curve are sold out, the token **graduates**. The ETH raised and the remaining token supply are automatically sent to Uniswap V4 to create a permanent trading pool. After graduation, you trade on Uniswap — the bonding curve is done.

For standard launches, the Uniswap LP position is sent to a dead address (permanently locked). For Prepool launches, the LP goes to the Prepool contract so contributors can collect fees forever.

---

## For Integrators & DEX Terminals

### Contract Pattern

Each `BondingCurve` is a minimal proxy clone (EIP-1167) of a shared implementation. It is initialized once by the factory and cannot be re-initialized.

### Curve Mathematics

Uses a **constant-product AMM** (pump.fun style) with virtual reserves:

```
price = ethReserve / tokenReserve

ethReserve  = virtualEth  + ethRaised
tokenReserve = virtualTokens - tokensSold
```

`virtualEth` and `virtualTokens` are set by the factory and define the initial price floor. Only `salePercentBps` of the total supply is available on the curve; the rest is reserved for the Uniswap LP.

Math is in [`CurveLib`](../contracts/libraries/CurveLib.sol).

### Key State

```solidity
IERC20  public token;
uint256 public totalSupply;
uint16  public platformFeeBps;
uint256 public virtualEth;
uint256 public virtualTokens;
uint256 public maxTokensForSale;  // salePercentBps of totalSupply

uint256 public tokensSold;
uint256 public ethRaised;
bool    public graduated;

address public lpRecipient;    // dead address (normal) or prepool address (prepool)
address public poolHook;       // address(0) (normal) or prepool address (prepool)
address public feeRecipient;   // receives platform fees immediately at trade time
```

### Write Functions

#### `buy`

```solidity
function buy(
    address recipient,
    uint256 minTokensOut,  // slippage protection
    uint256 deadline       // unix timestamp
) external payable returns (uint256 tokensOut)
```

Buys tokens with ETH. Deducts platform fee (sent immediately to `feeRecipient`), calculates output via constant product, transfers tokens to `recipient`. Auto-graduates if `tokensSold >= maxTokensForSale` after the buy.

Excess ETH is refunded if the buy would overshoot the graduation limit. The fee is adjusted proportionally before the refund.

#### `sell`

```solidity
function sell(
    address recipient,
    uint256 tokenAmount,
    uint256 minEthOut,  // slippage protection
    uint256 deadline
) external returns (uint256 ethOut)
```

Sells tokens back to the curve. Pulls tokens from `msg.sender` (requires approval). Platform fee is sent immediately to `feeRecipient`. Not callable after graduation.

#### `graduate` (manual fallback)

```solidity
function graduate() external
```

Permissionless. Triggers migration to Uniswap V4 if `tokensSold >= maxTokensForSale`. Normally called automatically by the last buy.

### View / Quote Functions

```solidity

// Current spot price (ETH per token, scaled 1e18)
function getPrice() external view returns (uint256);

// Tokens you'd receive for ethIn (after fee)
function getBuyQuote(uint256 ethIn) external view returns (uint256);

// ETH you'd receive for tokenAmount (after fee)
function getSellQuote(uint256 tokenAmount) external view returns (uint256);

// Fetch creator/metadata from factory
function getLaunchInfo() external view returns (
    address tokenAddr,
    address bondingCurve,
    address creator,
    uint256 createdAt
)
```

### Events

```solidity
event TokensPurchased(
    address indexed buyer,
    uint256 ethSpent,
    uint256 tokensReceived,
    uint256 newPrice
)

event TokensSold(
    address indexed seller,
    uint256 tokensReturned,
    uint256 ethReceived,
    uint256 newPrice
)

event Graduated(
    uint256 ethToPool,
    uint256 tokensToPool,
    uint256 lpTokenId
)

```

### Errors

| Error | Condition |
|---|---|
| `AlreadyGraduated` | Buy/sell attempted after graduation |
| `NotGraduated` | Manual `graduate()` call before conditions met |
| `SlippageExceeded` | Output below `minTokensOut` / `minEthOut` |
| `DeadlineExpired` | `block.timestamp > deadline` |
| `InsufficientTokens` | Selling more tokens than were ever sold |
| `ZeroAmount` | ETH or token amount is zero |

### Uniswap V4 Pool Parameters

```solidity
uint24 public constant POOL_FEE     = 3000;  // 0.3%
int24  public constant TICK_SPACING = 60;
```

Pool pair: `currency0 = ETH (address(0))`, `currency1 = token address`.

Full-range liquidity position: ticks `[-887220, 887220]` (aligned to tick spacing of 60).

### Platform Fees

Platform fees are sent **immediately** at trade time to `feeRecipient`. There is no accumulated balance to claim.

- For **normal launches**: `feeRecipient` is the factory owner.
- For **prepool launches**: `feeRecipient` is the prepool contract, which distributes them 50/50 to makxTeam and contributors via its ETH fee mechanism.
