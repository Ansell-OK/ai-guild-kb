# Liquidity Pools - Complete Implementation Guide

## 📖 Overview

Liquidity pools are the foundation of decentralized finance (DeFi). They allow users to provide token liquidity and earn fees from trades, enabling automated market making without centralized order books.

### What You'll Learn
- How liquidity pools work
- Constant product formula (x * y = k)
- Pool creation and liquidity provision
- Swap mechanics
- LP token issuance and rewards
- Fee distribution

### Prerequisites
- Understanding of SIP-010 tokens
- Basic DeFi concepts
- Clarity functions and maps

---

## 🎯 Key Concepts

### What is a Liquidity Pool?

A liquidity pool is a smart contract that holds reserves of two tokens and allows users to trade between them. The price is determined algorithmically based on the ratio of tokens in the pool.

**Core Components:**
1. **Token Reserves**: Two tokens (e.g., STX/USDA)
2. **LP Tokens**: Represent ownership share in the pool
3. **Swap Function**: Allows trading between tokens
4. **Fee Mechanism**: Incentivizes liquidity providers

### Constant Product Formula

```
x * y = k

Where:
x = Reserve of Token A
y = Reserve of Token B
k = Constant product (must remain constant after swaps)
```

**Example:**
- Pool has 100 STX and 1000 USDA
- k = 100 * 1000 = 100,000
- If someone buys 10 STX, they must add enough USDA to maintain k

---

## 💻 Basic Liquidity Pool Contract

### Complete Implementation

```clarity
;; Liquidity Pool Contract
;; Automated Market Maker (AMM) with constant product formula

;; Use token traits
(use-trait ft-trait 'SP3FBR2AGK5H9QBDH3EEN6DF8EK8JY7RX8QJ5SVTE.sip-010-trait-ft-standard.sip-010-trait)

;; Constants
(define-constant contract-owner tx-sender)
(define-constant err-not-authorized (err u100))
(define-constant err-insufficient-liquidity (err u101))
(define-constant err-invalid-amount (err u102))
(define-constant err-slippage-exceeded (err u103))
(define-constant err-pool-exists (err u104))
(define-constant err-pool-not-found (err u105))
(define-constant err-zero-liquidity (err u106))

;; Fee: 0.3% (30 basis points)
(define-constant fee-basis-points u30)
(define-constant basis-points-denominator u10000)

;; Pool data structure
(define-map pools
    { token-x: principal, token-y: principal }
    {
        reserve-x: uint,
        reserve-y: uint,
        total-supply: uint,
        fee-to-protocol: uint
    }
)

;; LP token balances
(define-map lp-balances
    { pool-id: { token-x: principal, token-y: principal }, holder: principal }
    uint
)

;; -----------------------------------------------------------
;; Pool Creation
;; -----------------------------------------------------------

;; Create a new liquidity pool
(define-public (create-pool 
    (token-x-trait <ft-trait>)
    (token-y-trait <ft-trait>)
    (token-x principal)
    (token-y principal)
    (amount-x uint)
    (amount-y uint)
)
    (let
        (
            (pool-id { token-x: token-x, token-y: token-y })
            (initial-liquidity (sqrt (* amount-x amount-y)))
        )
        ;; Validate inputs
        (asserts! (> amount-x u0) err-invalid-amount)
        (asserts! (> amount-y u0) err-invalid-amount)
        (asserts! (is-none (map-get? pools pool-id)) err-pool-exists)
        
        ;; Transfer tokens from user to contract
        (try! (contract-call? token-x-trait transfer 
            amount-x tx-sender (as-contract tx-sender) none
        ))
        (try! (contract-call? token-y-trait transfer 
            amount-y tx-sender (as-contract tx-sender) none
        ))
        
        ;; Create pool
        (map-set pools pool-id {
            reserve-x: amount-x,
            reserve-y: amount-y,
            total-supply: initial-liquidity,
            fee-to-protocol: u0
        })
        
        ;; Mint LP tokens to creator
        (map-set lp-balances
            { pool-id: pool-id, holder: tx-sender }
            initial-liquidity
        )
        
        (print {
            event: "pool-created",
            token-x: token-x,
            token-y: token-y,
            amount-x: amount-x,
            amount-y: amount-y,
            liquidity: initial-liquidity
        })
        
        (ok initial-liquidity)
    )
)

;; -----------------------------------------------------------
;; Add Liquidity
;; -----------------------------------------------------------

;; Add liquidity to existing pool
(define-public (add-liquidity
    (token-x-trait <ft-trait>)
    (token-y-trait <ft-trait>)
    (token-x principal)
    (token-y principal)
    (amount-x-desired uint)
    (amount-y-desired uint)
    (amount-x-min uint)
    (amount-y-min uint)
)
    (let
        (
            (pool-id { token-x: token-x, token-y: token-y })
            (pool (unwrap! (map-get? pools pool-id) err-pool-not-found))
            (reserve-x (get reserve-x pool))
            (reserve-y (get reserve-y pool))
            (total-supply (get total-supply pool))
            
            ;; Calculate optimal amounts
            (amount-y-optimal (/ (* amount-x-desired reserve-y) reserve-x))
            (amount-x-optimal (/ (* amount-y-desired reserve-x) reserve-y))
            
            ;; Determine actual amounts
            (amount-x (if (<= amount-y-optimal amount-y-desired)
                amount-x-desired
                amount-x-optimal
            ))
            (amount-y (if (<= amount-y-optimal amount-y-desired)
                amount-y-optimal
                amount-y-desired
            ))
            
            ;; Calculate liquidity to mint
            (liquidity-x (/ (* amount-x total-supply) reserve-x))
            (liquidity-y (/ (* amount-y total-supply) reserve-y))
            (liquidity (if (< liquidity-x liquidity-y) liquidity-x liquidity-y))
        )
        ;; Validate amounts
        (asserts! (>= amount-x amount-x-min) err-slippage-exceeded)
        (asserts! (>= amount-y amount-y-min) err-slippage-exceeded)
        (asserts! (> liquidity u0) err-zero-liquidity)
        
        ;; Transfer tokens from user
        (try! (contract-call? token-x-trait transfer 
            amount-x tx-sender (as-contract tx-sender) none
        ))
        (try! (contract-call? token-y-trait transfer 
            amount-y tx-sender (as-contract tx-sender) none
        ))
        
        ;; Update pool reserves
        (map-set pools pool-id {
            reserve-x: (+ reserve-x amount-x),
            reserve-y: (+ reserve-y amount-y),
            total-supply: (+ total-supply liquidity),
            fee-to-protocol: (get fee-to-protocol pool)
        })
        
        ;; Mint LP tokens
        (let
            (
                (current-balance (default-to u0 
                    (map-get? lp-balances { pool-id: pool-id, holder: tx-sender })
                ))
            )
            (map-set lp-balances
                { pool-id: pool-id, holder: tx-sender }
                (+ current-balance liquidity)
            )
        )
        
        (print {
            event: "liquidity-added",
            token-x: token-x,
            token-y: token-y,
            amount-x: amount-x,
            amount-y: amount-y,
            liquidity: liquidity
        })
        
        (ok liquidity)
    )
)

;; -----------------------------------------------------------
;; Remove Liquidity
;; -----------------------------------------------------------

;; Remove liquidity from pool
(define-public (remove-liquidity
    (token-x-trait <ft-trait>)
    (token-y-trait <ft-trait>)
    (token-x principal)
    (token-y principal)
    (liquidity-amount uint)
    (amount-x-min uint)
    (amount-y-min uint)
)
    (let
        (
            (pool-id { token-x: token-x, token-y: token-y })
            (pool (unwrap! (map-get? pools pool-id) err-pool-not-found))
            (reserve-x (get reserve-x pool))
            (reserve-y (get reserve-y pool))
            (total-supply (get total-supply pool))
            (user-balance (default-to u0 
                (map-get? lp-balances { pool-id: pool-id, holder: tx-sender })
            ))
            
            ;; Calculate amounts to return
            (amount-x (/ (* liquidity-amount reserve-x) total-supply))
            (amount-y (/ (* liquidity-amount reserve-y) total-supply))
        )
        ;; Validate
        (asserts! (>= user-balance liquidity-amount) err-insufficient-liquidity)
        (asserts! (>= amount-x amount-x-min) err-slippage-exceeded)
        (asserts! (>= amount-y amount-y-min) err-slippage-exceeded)
        
        ;; Update LP balance
        (map-set lp-balances
            { pool-id: pool-id, holder: tx-sender }
            (- user-balance liquidity-amount)
        )
        
        ;; Update pool reserves
        (map-set pools pool-id {
            reserve-x: (- reserve-x amount-x),
            reserve-y: (- reserve-y amount-y),
            total-supply: (- total-supply liquidity-amount),
            fee-to-protocol: (get fee-to-protocol pool)
        })
        
        ;; Transfer tokens back to user
        (try! (as-contract (contract-call? token-x-trait transfer 
            amount-x tx-sender tx-sender none
        )))
        (try! (as-contract (contract-call? token-y-trait transfer 
            amount-y tx-sender tx-sender none
        )))
        
        (print {
            event: "liquidity-removed",
            token-x: token-x,
            token-y: token-y,
            amount-x: amount-x,
            amount-y: amount-y,
            liquidity: liquidity-amount
        })
        
        (ok { amount-x: amount-x, amount-y: amount-y })
    )
)

;; -----------------------------------------------------------
;; Swap Functions
;; -----------------------------------------------------------

;; Swap token X for token Y
(define-public (swap-x-for-y
    (token-x-trait <ft-trait>)
    (token-y-trait <ft-trait>)
    (token-x principal)
    (token-y principal)
    (amount-x-in uint)
    (amount-y-out-min uint)
)
    (let
        (
            (pool-id { token-x: token-x, token-y: token-y })
            (pool (unwrap! (map-get? pools pool-id) err-pool-not-found))
            (reserve-x (get reserve-x pool))
            (reserve-y (get reserve-y pool))
            
            ;; Calculate amount out with fee
            (amount-x-in-with-fee (- amount-x-in (/ (* amount-x-in fee-basis-points) basis-points-denominator)))
            (amount-y-out (get-amount-out amount-x-in-with-fee reserve-x reserve-y))
        )
        ;; Validate
        (asserts! (> amount-x-in u0) err-invalid-amount)
        (asserts! (>= amount-y-out amount-y-out-min) err-slippage-exceeded)
        
        ;; Transfer token X from user
        (try! (contract-call? token-x-trait transfer 
            amount-x-in tx-sender (as-contract tx-sender) none
        ))
        
        ;; Transfer token Y to user
        (try! (as-contract (contract-call? token-y-trait transfer 
            amount-y-out tx-sender tx-sender none
        )))
        
        ;; Update reserves
        (map-set pools pool-id {
            reserve-x: (+ reserve-x amount-x-in),
            reserve-y: (- reserve-y amount-y-out),
            total-supply: (get total-supply pool),
            fee-to-protocol: (get fee-to-protocol pool)
        })
        
        (print {
            event: "swap",
            token-in: token-x,
            token-out: token-y,
            amount-in: amount-x-in,
            amount-out: amount-y-out
        })
        
        (ok amount-y-out)
    )
)

;; Swap token Y for token X
(define-public (swap-y-for-x
    (token-x-trait <ft-trait>)
    (token-y-trait <ft-trait>)
    (token-x principal)
    (token-y principal)
    (amount-y-in uint)
    (amount-x-out-min uint)
)
    (let
        (
            (pool-id { token-x: token-x, token-y: token-y })
            (pool (unwrap! (map-get? pools pool-id) err-pool-not-found))
            (reserve-x (get reserve-x pool))
            (reserve-y (get reserve-y pool))
            
            ;; Calculate amount out with fee
            (amount-y-in-with-fee (- amount-y-in (/ (* amount-y-in fee-basis-points) basis-points-denominator)))
            (amount-x-out (get-amount-out amount-y-in-with-fee reserve-y reserve-x))
        )
        ;; Validate
        (asserts! (> amount-y-in u0) err-invalid-amount)
        (asserts! (>= amount-x-out amount-x-out-min) err-slippage-exceeded)
        
        ;; Transfer token Y from user
        (try! (contract-call? token-y-trait transfer 
            amount-y-in tx-sender (as-contract tx-sender) none
        ))
        
        ;; Transfer token X to user
        (try! (as-contract (contract-call? token-x-trait transfer 
            amount-x-out tx-sender tx-sender none
        )))
        
        ;; Update reserves
        (map-set pools pool-id {
            reserve-x: (- reserve-x amount-x-out),
            reserve-y: (+ reserve-y amount-y-in),
            total-supply: (get total-supply pool),
            fee-to-protocol: (get fee-to-protocol pool)
        })
        
        (print {
            event: "swap",
            token-in: token-y,
            token-out: token-x,
            amount-in: amount-y-in,
            amount-out: amount-x-out
        })
        
        (ok amount-x-out)
    )
)

;; -----------------------------------------------------------
;; Read-Only Functions
;; -----------------------------------------------------------

;; Get pool info
(define-read-only (get-pool 
    (token-x principal)
    (token-y principal)
)
    (map-get? pools { token-x: token-x, token-y: token-y })
)

;; Get LP balance
(define-read-only (get-lp-balance
    (token-x principal)
    (token-y principal)
    (holder principal)
)
    (default-to u0 (map-get? lp-balances {
        pool-id: { token-x: token-x, token-y: token-y },
        holder: holder
    }))
)

;; Calculate amount out (constant product formula)
(define-read-only (get-amount-out 
    (amount-in uint)
    (reserve-in uint)
    (reserve-out uint)
)
    (let
        (
            (numerator (* amount-in reserve-out))
            (denominator (+ reserve-in amount-in))
        )
        (/ numerator denominator)
    )
)

;; Get price (how much Y for 1 X)
(define-read-only (get-price
    (token-x principal)
    (token-y principal)
)
    (match (map-get? pools { token-x: token-x, token-y: token-y })
        pool (ok (/ (get reserve-y pool) (get reserve-x pool)))
        err-pool-not-found
    )
)

;; -----------------------------------------------------------
;; Helper Functions
;; -----------------------------------------------------------

;; Square root (simplified implementation)
(define-private (sqrt (x uint))
    (if (<= x u1)
        x
        (/ (+ (sqrt (/ x u2)) (/ x (sqrt (/ x u2)))) u2)
    )
)
```

---

## 🌐 Frontend Integration

### Complete Liquidity Pool UI

```javascript
import React, { useState, useEffect } from 'react';
import { useWallet } from './WalletContext';
import {
  openContractCall,
  uintCV,
  contractPrincipalCV,
  noneCV
} from '@stacks/transactions';
import { StacksTestnet } from '@stacks/network';

function LiquidityPoolDApp() {
  const { address, isConnected } = useWallet();
  const [pools, setPools] = useState([]);
  const [selectedPool, setSelectedPool] = useState(null);

  const POOL_CONTRACT = 'ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM';

  return (
    <div className="pool-dapp">
      <h1>Liquidity Pools</h1>
      
      {isConnected ? (
        <>
          <PoolList pools={pools} onSelect={setSelectedPool} />
          {selectedPool && (
            <>
              <AddLiquidityForm pool={selectedPool} address={address} />
              <RemoveLiquidityForm pool={selectedPool} address={address} />
              <SwapForm pool={selectedPool} address={address} />
            </>
          )}
        </>
      ) : (
        <p>Connect wallet to interact with pools</p>
      )}
    </div>
  );
}

function AddLiquidityForm({ pool, address }) {
  const [amountX, setAmountX] = useState('');
  const [amountY, setAmountY] = useState('');
  
  const addLiquidity = async () => {
    const amountXRaw = parseFloat(amountX) * 1000000;
    const amountYRaw = parseFloat(amountY) * 1000000;
    
    const options = {
      network: new StacksTestnet(),
      contractAddress: POOL_CONTRACT,
      contractName: 'liquidity-pool',
      functionName: 'add-liquidity',
      functionArgs: [
        contractPrincipalCV(pool.tokenX, 'token-x'),
        contractPrincipalCV(pool.tokenY, 'token-y'),
        contractPrincipalCV(pool.tokenX, 'token-x'),
        contractPrincipalCV(pool.tokenY, 'token-y'),
        uintCV(amountXRaw),
        uintCV(amountYRaw),
        uintCV(amountXRaw * 0.95), // 5% slippage
        uintCV(amountYRaw * 0.95)
      ],
      postConditionMode: PostConditionMode.Deny,
      onFinish: (data) => {
        alert(`Liquidity added! TX: ${data.txId}`);
      },
    };

    await openContractCall(options);
  };

  return (
    <div className="add-liquidity-form">
      <h2>Add Liquidity</h2>
      <input
        type="number"
        placeholder="Amount Token X"
        value={amountX}
        onChange={(e) => setAmountX(e.target.value)}
      />
      <input
        type="number"
        placeholder="Amount Token Y"
        value={amountY}
        onChange={(e) => setAmountY(e.target.value)}
      />
      <button onClick={addLiquidity}>Add Liquidity</button>
    </div>
  );
}

export default LiquidityPoolDApp;
```

---

## 📊 Pool Mathematics

### Price Impact Calculation

```clarity
(define-read-only (calculate-price-impact
    (amount-in uint)
    (reserve-in uint)
    (reserve-out uint)
)
    (let
        (
            (amount-out (get-amount-out amount-in reserve-in reserve-out))
            (price-before (/ reserve-out reserve-in))
            (new-reserve-in (+ reserve-in amount-in))
            (new-reserve-out (- reserve-out amount-out))
            (price-after (/ new-reserve-out new-reserve-in))
            (price-impact-percent (/ (* (- price-before price-after) u10000) price-before))
        )
        price-impact-percent
    )
)
```

---

## 🎓 Next Steps

**Related Guides:**
- → Token Swaps and DEX
- → Staking Mechanisms
- → Yield Farming