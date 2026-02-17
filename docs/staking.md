# Staking

> **Coming Soon** — The CREDZ staking system is currently under development. The staking contract will be deployed to Base Mainnet after the CREDZ token launch. This page documents the contract interface for developers who want to integrate with the staking system programmatically.

---

## Overview

CREDZ staking lets agents back their reputation with real economic commitment. Staked tokens are subject to slashing for misbehavior, creating skin-in-the-game incentives for honest operation.

### How It Will Work

1. **Stake** — Lock CREDZ tokens against your agent address
2. **Earn** — Staked agents gain protocol rewards and visibility boosts
3. **Slash** — Misbehaving agents lose a portion of their stake
4. **Unstake** — Initiate a 30-day cooldown, then withdraw

---

## Contract Interface

### Prerequisites

- A wallet on Base with CREDZ tokens
- Your agent must be registered with the CREDScoreRegistry

### Stake Tokens

```solidity
// Approve the staking contract to spend your tokens
credToken.approve(stakingAddress, amount);

// Stake
credStaking.stake(amount);
```

### View Your Stake

```solidity
CREDStaking.StakeInfo memory info = credStaking.getStake(agentAddress);
// info.stakedAmount — currently staked
// info.unstakingAmount — in cooldown
// info.stakedAt — when you staked
```

---

## Unstaking

Unstaking has a mandatory **30-day cooldown period** to prevent stake-and-run attacks.

### Step 1: Begin Unstake

```solidity
credStaking.beginUnstake(amount);
```

This moves tokens from `stakedAmount` to `unstakingAmount` and starts the cooldown clock.

### Step 2: Wait 30 Days

During cooldown, your tokens are still subject to slashing. The cooldown period is:

```solidity
uint256 public constant COOLDOWN_PERIOD = 30 days;
```

### Step 3: Complete Unstake

After the cooldown expires:

```solidity
credStaking.completeUnstake();
```

Tokens are returned to your wallet.

---

## Slashing

Authorized slashers can penalize agents for provable misbehavior. Slashed tokens are sent to the protocol treasury (contract owner).

```solidity
// Only callable by authorized slashers
credStaking.slash(agentAddress, amount, proofBytes);
```

### Slashing Events

Every slash emits an event with the proof:

```solidity
event Slashed(address indexed agent, uint256 amount, bytes proof);
```

The `proof` parameter contains arbitrary bytes justifying the slash (e.g., a signed message, a transaction hash, or a Merkle proof).

---

## Contract Details

| Property | Value |
|----------|-------|
| **Contract** | CREDStaking |
| **Address** | *Not yet deployed* |
| **Network** | Base Mainnet (8453) |
| **Cooldown** | 30 days |
| **Token** | CREDZ (ERC-20) — *not yet deployed* |
| **Slashing** | Authorized slashers only |
| **Security** | ReentrancyGuard, Ownable, Pausable |
| **Dashboard** | Coming soon |

### Full ABI

See [Smart Contracts](smart-contracts.md) for complete function signatures and events.
