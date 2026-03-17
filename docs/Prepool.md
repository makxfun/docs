# Prepool

## Overview

The `Prepool` is Makx's community launch mechanism. Instead of one developer being the first buyer of their own token, a group of people pool their ETH together, and the entire pool buys the token at the same time. Everyone enters at the same price — including the dev. No one gets an unfair advantage, and the LP is never burned: it stays in the Prepool contract, earning trading fees forever.

---

## For Users — The Big Picture

### The Problem with Normal Launches

On most launchpads, the dev buys first. Then they get impatient, dump, and the chart is dead. Everyone else loses. Early investors who sell make money; people who believed in the project lose.

### How Prepool Fixes This

With a Prepool:

- Contributors send ETH **before** the token launches.
- The Prepool holds the ETH until the launch conditions are met (a minimum raised, a time window, or both).
- When the Prepool deploys, **all the pooled ETH buys the token at once** — dev included. Nobody is ahead of anyone else.
- The Uniswap LP NFT goes to the Prepool, not the dev's wallet. The dev cannot rug the liquidity.
- Every trade on the token generates fees. Those fees flow back to the Prepool and are split 50/50 between the Makx team and contributors, **proportional to how much each person put in**.
- Contributors keep earning fees **for life** as long as they leave their contribution in the Prepool.
- When the token graduates to Uniswap (bonding curve migrates), withdrawing tokens shares the **entire current token balance** of the Prepool proportionally — including any tokens that accumulated after migration.

### The Token Lockup

After launch, tokens are locked for **48 hours**. This prevents a dump the moment the token goes live and gives everyone an equal chance to see the token in action before deciding to hold or exit.

### Exiting the Prepool

After the 48-hour lockup, a contributor can call `withdrawTokens()` to take their token share. But there's a trade-off:

> **Withdrawing tokens permanently removes you from future fee earnings.**

You keep the tokens you received, but all future trading fees will be distributed among the people who stayed. If everyone exits, all fees go to the Makx team.

**Before migration** (bonding curve still active): you receive your proportional share of the initial token snapshot. No fee.

**After migration** (token has graduated to Uniswap): you receive your proportional share of the Prepool's entire current token balance, calculated against `totalActiveContribution` (the pool of contributors who haven't withdrawn yet). This means the denominator shrinks as people leave — your share stays fair regardless of withdrawal order.

### Cancellation & Refunds

If a Prepool never meets its deployment conditions, or if the creator/Makx team cancels it, contributors get a full refund.

---

## For Integrators & DEX Terminals

### Contract Pattern

Each `Prepool` is a minimal proxy clone initialized by `LaunchFactory`. It also acts as a **Uniswap V4 hook** — its address must satisfy specific hook flag bits, which is why `createPrepool` validates the hook address before deploying.

### Lifecycle States

```
PENDING → (deployed = true) → ACTIVE → contributors can claim fees / withdraw tokens
        → (cancelled = true) → CANCELLED → contributors can refund
```

### Deployment Conditions

At least one condition must be set. Both can be set (AND logic):

| Parameter | Behavior |
|---|---|
| `minBalance > 0` | Requires `totalContributed >= minBalance` |
| `deployTime > 0` | Requires `block.timestamp >= deployTime` |

### Fee Distribution

Fee sources after graduation:
1. **Bonding curve platform fees** — claimed via `claimPlatformFees()`
2. **Uniswap V4 LP fees** — collected via `collectLPFees(positionManager)`
3. **Uniswap V4 hook fees** — 1% of every swap output, captured in `afterSwap`

All ETH fees are split:
- **50%** → `makxTeam`
- **50%** → active contributors, proportional to contribution

Distribution uses the **Synthetix reward-per-token** pattern (`rewardPerTokenStored`), so it's efficient regardless of how many contributors there are.

Token-side fees (tokens received from LP position) accumulate in `accumulatedTokenFees`. Post-migration, once all contributors have withdrawn their tokens (`totalActiveContribution == 0`), the makxTeam can sweep the entire remaining token balance via `withdrawTokenFees()`.

### Key State

```solidity
address public factory;
address public creator;
string  public tokenName;
string  public tokenSymbol;

uint256 public minBalance;
uint256 public deployTime;
uint256 public minContribution;
address public makxTeam;

mapping(address => uint256) public contributions;
uint256 public totalContributed;
uint256 public contributorCount;

bool    public deployed;
bool    public cancelled;
address public bondingCurve;
address public token;
uint256 public lpTokenId;         // Uniswap V4 position NFT token ID
uint256 public deployedAt;
uint256 public initialTokenBalance; // snapshot of tokens received at deployment (buy + prepool allocation)

// Synthetix reward tracking
uint256 public ethRewardPerTokenStored;
uint256 public totalActiveContribution; // only contributors who haven't withdrawn tokens
mapping(address => uint256) public userRewardPerTokenPaid;
mapping(address => uint256) public pendingRewards;
mapping(address => bool)    public hasWithdrawnTokens;

uint256 public undistributedEthFees;  // ETH fees pending distribution (hook fees, bonding curve fees, etc.)
uint256 public accumulatedTokenFees;  // token-side fees from LP position (swept by makxTeam post-migration)

uint256 public constant TOKEN_LOCKUP   = 48 hours;
uint256 public constant MAKX_FEE_BPS   = 5000;   // 50%
uint256 public constant HOOK_FEE_BPS   = 100;    // 1%
uint256 public constant FEE_DENOMINATOR = 10000;
```

### Write Functions

#### `contribute`

```solidity
function contribute(address to) external payable
```

Send ETH to join the prepool. `to` is the address credited (allows contributing on behalf of another). Reverts if `msg.value < minContribution`, or if already deployed/cancelled.

#### `deploy`

```solidity
function deploy(bytes32 curveSalt) external
```

Permissionless once conditions are met. Calls `LaunchFactory.createPrepoolLaunch` with all pooled ETH. The `curveSalt` is used to derive the bonding curve clone address — must be chosen such that the resulting bonding curve address is valid.

After deployment, `initialTokenBalance` is set from the value returned by `createPrepoolLaunch` — it includes both the tokens received from the initial buy and any direct prepool allocation minted at launch.

#### `withdrawTokens`

```solidity
function withdrawTokens() external
```

Available after `deployedAt + 48 hours`. Permanently marks the caller as exited — no more fee claims. Behavior depends on migration state:

| State | Token share calculation |
|---|---|
| Not migrated (`lpTokenId == 0`) | `initialTokenBalance × userContrib / totalContributed` — snapshot-based, no fee |
| Migrated (`lpTokenId > 0`) | `balanceOf(prepool) × userContrib / totalActiveContribution` — live balance, `totalActiveContribution` is decremented after the transfer to keep future withdrawals fair |

#### `claimFees`

```solidity
function claimFees() external
```

Claims accumulated ETH fees for the caller. Only available to active contributors (not exited). Uses the Synthetix pattern — safe to call at any time.

#### `cancel`

```solidity
function cancel() external
```

- **Anyone** can cancel if `deployTime` has passed but `minBalance` was not reached.
- **Creator or makxTeam** can cancel at any time before deployment.

#### `refund`

```solidity
function refund() external
```

Available only after cancellation. Returns the contributor's full ETH contribution.

#### `collectLPFees`

```solidity
function collectLPFees(address positionManager) external
```

Collects ETH + token fees from the Uniswap V4 LP position. ETH fees are immediately distributed (50/50). Token fees accumulate in `accumulatedTokenFees` for `makxTeam`.

#### `distributeAccumulatedFees`

```solidity
function distributeAccumulatedFees() external
```

Distributes any hook fees that were captured but not yet split (e.g. if `collectLPFees` hasn't been called yet).

### View Functions

```solidity
// Earned ETH fees for a contributor (0 if exited)
function earned(address user) external view returns (uint256)

// Whether the prepool can be deployed right now
function canDeploy() external view returns (bool)
```

### Events

```solidity
event Contributed(address indexed contributor, uint256 amount, uint256 totalContributed)
event Deployed(address indexed token, address indexed bondingCurve, uint256 ethUsed)
event TokensWithdrawn(address indexed contributor, uint256 tokenAmount)
event FeesClaimed(address indexed contributor, uint256 amount)
event Cancelled()
event Refunded(address indexed contributor, uint256 amount)
event FeesReceived(uint256 amount)
event TokenFeesWithdrawn(address indexed recipient, uint256 amount)
```

### Errors

| Error | Condition |
|---|---|
| `AlreadyDeployed` | Action not valid after deployment |
| `NotDeployed` | Action requires deployment first |
| `AlreadyCancelled` | Prepool already cancelled |
| `NotCancelled` | Refund attempted before cancellation |
| `BelowMinContribution` | `msg.value < minContribution` |
| `DeployConditionsNotMet` | Conditions not satisfied, or no conditions configured |
| `TokensLocked` | 48-hour lockup not yet passed |
| `AlreadyWithdrawn` | Contributor already withdrew tokens |
| `NoContribution` | Caller has no contribution |
| `CannotCancel` | Caller is not authorized to cancel |
| `ContributorsStillActive` | `withdrawTokenFees` called while contributors still hold positions |
| `NotMigrated` | `withdrawTokenFees` called before bonding curve has migrated |

### Uniswap V4 Hook

The Prepool implements `IHooks` with only `afterSwap` active. All other hooks revert with `HookNotImplemented`.

`afterSwap` captures **1% of every swap output** (ETH or token side) and holds it in the contract:
- ETH fees: tracked in `undistributedEthFees`, distributed on next `collectLPFees` or `distributeAccumulatedFees` call.
- Token fees: held in the contract, swept by `makxTeam` via `withdrawTokenFees()` only after all contributors have exited.

Note: hook and LP fees only exist post-migration — `accumulatedTokenFees` is always zero before `lpTokenId` is set.

Required hook flags (checked by `LaunchFactory.createPrepool`):
```
bit 6: AFTER_SWAP                  (0x0040)
bit 2: AFTER_SWAP_RETURNS_DELTA    (0x0004)
```
