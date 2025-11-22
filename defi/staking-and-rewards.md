# Staking and Rewards - Complete Guide

## 📖 Overview

Staking allows users to lock their tokens and earn rewards over time. This guide covers building a complete staking system with reward distribution, time-locked staking, and flexible reward mechanisms.

### What You'll Learn
- Staking pool architecture
- Reward calculation methods
- Time-locked staking
- Stake withdrawal
- Reward distribution patterns
- Staking pool management

### Prerequisites
- Understanding of SIP-010 tokens
- Time-based calculations
- Maps and data structures

---

## 🎯 Key Concepts

### Staking Components

**1. Stake Pool**
- Holds staked tokens
- Tracks individual stakes
- Calculates rewards

**2. Reward Distribution**
- Time-based rewards
- Proportional distribution
- Compound staking

**3. Lock Periods**
- Minimum stake duration
- Early withdrawal penalties
- Lock tiers with bonuses

---

## 💻 Complete Staking Contract

### Implementation

```clarity
;; Staking Contract
;; Users stake tokens to earn rewards over time

(use-trait ft-trait 'SP3FBR2AGK5H9QBDH3EEN6DF8EK8JY7RX8QJ5SVTE.sip-010-trait-ft-standard.sip-010-trait)

;; Constants
(define-constant contract-owner tx-sender)
(define-constant err-not-authorized (err u300))
(define-constant err-already-staked (err u301))
(define-constant err-no-stake (err u302))
(define-constant err-insufficient-balance (err u303))
(define-constant err-lock-period-active (err u304))
(define-constant err-invalid-amount (err u305))
(define-constant err-pool-not-found (err u306))

;; Reward rate: 10% APY = 10000 basis points
(define-constant reward-rate u1000) ;; 10% in basis points
(define-constant basis-points u10000)
(define-constant blocks-per-year u52560) ;; Approx 144 blocks/day * 365

;; Pool configuration
(define-map staking-pools
    principal  ;; Staking token
    {
        total-staked: uint,
        reward-token: principal,
        reward-rate: uint,  ;; Basis points per year
        min-stake: uint,
        lock-period: uint  ;; In blocks
    }
)

;; Individual stakes
(define-map stakes
    { pool: principal, staker: principal }
    {
        amount: uint,
        start-block: uint,
        last-claim-block: uint,
        rewards-earned: uint
    }
)

;; -----------------------------------------------------------
;; Pool Management
;; -----------------------------------------------------------

;; Create staking pool
(define-public (create-pool
    (staking-token principal)
    (reward-token principal)
    (reward-rate-bps uint)
    (min-stake uint)
    (lock-period-blocks uint)
)
    (begin
        (asserts! (is-eq tx-sender contract-owner) err-not-authorized)
        
        (ok (map-set staking-pools staking-token {
            total-staked: u0,
            reward-token: reward-token,
            reward-rate: reward-rate-bps,
            min-stake: min-stake,
            lock-period: lock-period-blocks
        }))
    )
)

;; -----------------------------------------------------------
;; Staking Functions
;; -----------------------------------------------------------

;; Stake tokens
(define-public (stake
    (token-trait <ft-trait>)
    (token principal)
    (amount uint)
)
    (let
        (
            (pool (unwrap! (map-get? staking-pools token) err-pool-not-found))
            (existing-stake (map-get? stakes { pool: token, staker: tx-sender }))
        )
        ;; Validate
        (asserts! (>= amount (get min-stake pool)) err-invalid-amount)
        
        ;; Transfer tokens to contract
        (try! (contract-call? token-trait transfer 
            amount tx-sender (as-contract tx-sender) none
        ))
        
        ;; Create or update stake
        (match existing-stake
            current-stake
                ;; Add to existing stake
                (let
                    (
                        (pending-rewards (calculate-rewards
                            (get amount current-stake)
                            (- block-height (get last-claim-block current-stake))
                            (get reward-rate pool)
                        ))
                    )
                    (map-set stakes
                        { pool: token, staker: tx-sender }
                        {
                            amount: (+ (get amount current-stake) amount),
                            start-block: (get start-block current-stake),
                            last-claim-block: block-height,
                            rewards-earned: (+ (get rewards-earned current-stake) pending-rewards)
                        }
                    )
                )
            ;; Create new stake
            (map-set stakes
                { pool: token, staker: tx-sender }
                {
                    amount: amount,
                    start-block: block-height,
                    last-claim-block: block-height,
                    rewards-earned: u0
                }
            )
        )
        
        ;; Update total staked
        (map-set staking-pools token
            (merge pool { total-staked: (+ (get total-staked pool) amount) })
        )
        
        (print {
            event: "stake",
            staker: tx-sender,
            token: token,
            amount: amount
        })
        
        (ok true)
    )
)

;; Unstake tokens
(define-public (unstake
    (staking-token-trait <ft-trait>)
    (reward-token-trait <ft-trait>)
    (staking-token principal)
    (amount uint)
)
    (let
        (
            (pool (unwrap! (map-get? staking-pools staking-token) err-pool-not-found))
            (stake (unwrap! (map-get? stakes { pool: staking-token, staker: tx-sender }) err-no-stake))
            (blocks-staked (- block-height (get start-block stake)))
        )
        ;; Check lock period
        (asserts! (>= blocks-staked (get lock-period pool)) err-lock-period-active)
        
        ;; Check sufficient balance
        (asserts! (>= (get amount stake) amount) err-insufficient-balance)
        
        ;; Calculate and claim pending rewards
        (let
            (
                (pending-rewards (calculate-rewards
                    (get amount stake)
                    (- block-height (get last-claim-block stake))
                    (get reward-rate pool)
                ))
                (total-rewards (+ (get rewards-earned stake) pending-rewards))
            )
            ;; Return staked tokens
            (try! (as-contract (contract-call? staking-token-trait transfer
                amount tx-sender tx-sender none
            )))
            
            ;; Pay out rewards
            (if (> total-rewards u0)
                (try! (as-contract (contract-call? reward-token-trait transfer
                    total-rewards tx-sender tx-sender none
                )))
                true
            )
            
            ;; Update or remove stake
            (if (is-eq amount (get amount stake))
                ;; Remove stake completely
                (map-delete stakes { pool: staking-token, staker: tx-sender })
                ;; Update stake
                (map-set stakes
                    { pool: staking-token, staker: tx-sender }
                    {
                        amount: (- (get amount stake) amount),
                        start-block: (get start-block stake),
                        last-claim-block: block-height,
                        rewards-earned: u0
                    }
                )
            )
            
            ;; Update total staked
            (map-set staking-pools staking-token
                (merge pool { total-staked: (- (get total-staked pool) amount) })
            )
            
            (print {
                event: "unstake",
                staker: tx-sender,
                token: staking-token,
                amount: amount,
                rewards: total-rewards
            })
            
            (ok { amount: amount, rewards: total-rewards })
        )
    )
)

;; Claim rewards without unstaking
(define-public (claim-rewards
    (reward-token-trait <ft-trait>)
    (staking-token principal)
)
    (let
        (
            (pool (unwrap! (map-get? staking-pools staking-token) err-pool-not-found))
            (stake (unwrap! (map-get? stakes { pool: staking-token, staker: tx-sender }) err-no-stake))
            (pending-rewards (calculate-rewards
                (get amount stake)
                (- block-height (get last-claim-block stake))
                (get reward-rate pool)
            ))
            (total-rewards (+ (get rewards-earned stake) pending-rewards))
        )
        ;; Pay out rewards
        (asserts! (> total-rewards u0) err-invalid-amount)
        
        (try! (as-contract (contract-call? reward-token-trait transfer
            total-rewards tx-sender tx-sender none
        )))
        
        ;; Update stake
        (map-set stakes
            { pool: staking-token, staker: tx-sender }
            (merge stake {
                last-claim-block: block-height,
                rewards-earned: u0
            })
        )
        
        (print {
            event: "claim-rewards",
            staker: tx-sender,
            rewards: total-rewards
        })
        
        (ok total-rewards)
    )
)

;; -----------------------------------------------------------
;; Compound Staking
;; -----------------------------------------------------------

;; Compound rewards (stake earned rewards)
(define-public (compound-rewards
    (staking-token principal)
)
    (let
        (
            (pool (unwrap! (map-get? staking-pools staking-token) err-pool-not-found))
            (stake (unwrap! (map-get? stakes { pool: staking-token, staker: tx-sender }) err-no-stake))
            (pending-rewards (calculate-rewards
                (get amount stake)
                (- block-height (get last-claim-block stake))
                (get reward-rate pool)
            ))
            (total-rewards (+ (get rewards-earned stake) pending-rewards))
        )
        (asserts! (> total-rewards u0) err-invalid-amount)
        
        ;; Add rewards to staked amount
        (map-set stakes
            { pool: staking-token, staker: tx-sender }
            {
                amount: (+ (get amount stake) total-rewards),
                start-block: (get start-block stake),
                last-claim-block: block-height,
                rewards-earned: u0
            }
        )
        
        ;; Update total staked
        (map-set staking-pools staking-token
            (merge pool { total-staked: (+ (get total-staked pool) total-rewards) })
        )
        
        (print {
            event: "compound",
            staker: tx-sender,
            amount-compounded: total-rewards
        })
        
        (ok total-rewards)
    )
)

;; -----------------------------------------------------------
;; Reward Calculation
;; -----------------------------------------------------------

;; Calculate rewards earned
(define-read-only (calculate-rewards
    (staked-amount uint)
    (blocks-elapsed uint)
    (rate-bps uint)
)
    (let
        (
            ;; Annual reward = staked * rate / 10000
            (annual-reward (/ (* staked-amount rate-bps) basis-points))
            
            ;; Reward per block = annual / blocks-per-year
            (reward-per-block (/ annual-reward blocks-per-year))
            
            ;; Total reward = reward-per-block * blocks-elapsed
            (total-reward (* reward-per-block blocks-elapsed))
        )
        total-reward
    )
)

;; Get pending rewards
(define-read-only (get-pending-rewards
    (staking-token principal)
    (staker principal)
)
    (match (map-get? staking-pools staking-token)
        pool
            (match (map-get? stakes { pool: staking-token, staker: staker })
                stake (ok (+
                    (get rewards-earned stake)
                    (calculate-rewards
                        (get amount stake)
                        (- block-height (get last-claim-block stake))
                        (get reward-rate pool)
                    )
                ))
                err-no-stake
            )
        err-pool-not-found
    )
)

;; -----------------------------------------------------------
;; View Functions
;; -----------------------------------------------------------

;; Get stake info
(define-read-only (get-stake
    (staking-token principal)
    (staker principal)
)
    (ok (map-get? stakes { pool: staking-token, staker: staker }))
)

;; Get pool info
(define-read-only (get-pool-info (token principal))
    (ok (map-get? staking-pools token))
)

;; Calculate APY for display
(define-read-only (get-apy (staking-token principal))
    (match (map-get? staking-pools staking-token)
        pool (ok (get reward-rate pool))
        err-pool-not-found
    )
)

;; Get time until unlock
(define-read-only (get-time-until-unlock
    (staking-token principal)
    (staker principal)
)
    (match (map-get? staking-pools staking-token)
        pool
            (match (map-get? stakes { pool: staking-token, staker: staker })
                stake
                    (let
                        (
                            (blocks-staked (- block-height (get start-block stake)))
                            (lock-period (get lock-period pool))
                        )
                        (ok (if (>= blocks-staked lock-period)
                            u0
                            (- lock-period blocks-staked)
                        ))
                    )
                err-no-stake
            )
        err-pool-not-found
    )
)
```

---

## 🌐 Staking Dashboard Frontend

```javascript
import React, { useState, useEffect } from 'react';
import { useWallet } from './WalletContext';
import {
  openContractCall,
  uintCV,
  principalCV
} from '@stacks/transactions';

function StakingDashboard() {
  const { address, isConnected } = useWallet();
  const [pools, setPools] = useState([]);
  const [userStakes, setUserStakes] = useState([]);
  const [selectedPool, setSelectedPool] = useState(null);

  return (
    <div className="staking-dashboard">
      <h1>Staking Pools</h1>

      {isConnected ? (
        <>
          <PoolsList pools={pools} onSelect={setSelectedPool} />
          
          {selectedPool && (
            <StakeInterface pool={selectedPool} address={address} />
          )}
          
          <UserStakes stakes={userStakes} address={address} />
        </>
      ) : (
        <p>Connect wallet to start staking</p>
      )}
    </div>
  );
}

function StakeInterface({ pool, address }) {
  const [amount, setAmount] = useState('');
  const [pendingRewards, setPendingRewards] = useState(0);

  const stake = async () => {
    const amountRaw = parseFloat(amount) * 1000000;

    const options = {
      network: new StacksTestnet(),
      contractAddress: 'ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM',
      contractName: 'staking-pool',
      functionName: 'stake',
      functionArgs: [
        principalCV(pool.stakingToken),
        principalCV(pool.stakingToken),
        uintCV(amountRaw)
      ],
      postConditionMode: PostConditionMode.Deny,
      onFinish: (data) => {
        alert(`Tokens staked! TX: ${data.txId}`);
      },
    };

    await openContractCall(options);
  };

  const claimRewards = async () => {
    const options = {
      network: new StacksTestnet(),
      contractAddress: 'ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM',
      contractName: 'staking-pool',
      functionName: 'claim-rewards',
      functionArgs: [
        principalCV(pool.rewardToken),
        principalCV(pool.stakingToken)
      ],
      postConditionMode: PostConditionMode.Deny,
      onFinish: (data) => {
        alert(`Rewards claimed! TX: ${data.txId}`);
      },
    };

    await openContractCall(options);
  };

  const compound = async () => {
    const options = {
      network: new StacksTestnet(),
      contractAddress: 'ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM',
      contractName: 'staking-pool',
      functionName: 'compound-rewards',
      functionArgs: [principalCV(pool.stakingToken)],
      postConditionMode: PostConditionMode.Deny,
      onFinish: (data) => {
        alert(`Rewards compounded! TX: ${data.txId}`);
      },
    };

    await openContractCall(options);
  };

  return (
    <div className="stake-interface">
      <div className="pool-info">
        <h2>{pool.name} Pool</h2>
        <div className="pool-stats">
          <div className="stat">
            <span>APY:</span>
            <span className="value">{pool.apy}%</span>
          </div>
          <div className="stat">
            <span>Total Staked:</span>
            <span className="value">{pool.totalStaked} Tokens</span>
          </div>
          <div className="stat">
            <span>Lock Period:</span>
            <span className="value">{pool.lockDays} days</span>
          </div>
        </div>
      </div>

      <div className="stake-form">
        <h3>Stake Tokens</h3>
        <input
          type="number"
          value={amount}
          onChange={(e) => setAmount(e.target.value)}
          placeholder="Amount to stake"
        />
        <button onClick={stake}>Stake</button>
      </div>

      <div className="rewards-section">
        <h3>Your Rewards</h3>
        <div className="pending-rewards">
          <span>Pending:</span>
          <span className="amount">{pendingRewards.toFixed(6)}</span>
        </div>
        <div className="reward-actions">
          <button onClick={claimRewards}>Claim Rewards</button>
          <button onClick={compound}>Compound</button>
        </div>
      </div>
    </div>
  );
}

function UserStakes({ stakes, address }) {
  return (
    <div className="user-stakes">
      <h2>Your Stakes</h2>
      <table>
        <thead>
          <tr>
            <th>Pool</th>
            <th>Staked</th>
            <th>Rewards</th>
            <th>Unlock</th>
            <th>Actions</th>
          </tr>
        </thead>
        <tbody>
          {stakes.map((stake, idx) => (
            <tr key={idx}>
              <td>{stake.poolName}</td>
              <td>{stake.amount}</td>
              <td>{stake.pendingRewards}</td>
              <td>{stake.unlockTime}</td>
              <td>
                <button disabled={!stake.canUnstake}>Unstake</button>
              </td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
}

export default StakingDashboard;
```

---

## 📊 Advanced Staking Features

### 1. Tiered Staking

```clarity
;; Different lock periods with different rates
(define-map lock-tiers
    uint  ;; Tier ID
    { lock-blocks: uint, bonus-rate: uint }
)

;; Initialize tiers
;; Tier 1: 30 days, 5% APY
;; Tier 2: 90 days, 12% APY
;; Tier 3: 365 days, 25% APY
```

### 2. Early Unstake Penalty

```clarity
(define-constant early-unstake-penalty u2000) ;; 20%

(define-public (emergency-unstake (amount uint))
    (let
        (
            (penalty (/ (* amount early-unstake-penalty) basis-points))
            (amount-after-penalty (- amount penalty))
        )
        ;; Return amount minus penalty
        ;; Penalty goes to remaining stakers
        (ok amount-after-penalty)
    )
)
```

---

## 🎓 Next Steps

**Related Guides:**
- → Yield Farming Strategies
- → DAO Governance with Staking
- → Liquidity Mining