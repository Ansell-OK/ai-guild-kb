# Token Swaps and DEX - Complete Guide

## 📖 Overview

A Decentralized Exchange (DEX) allows users to swap tokens directly from their wallets without intermediaries. This guide covers building a complete DEX with swap routing, price quotes, and slippage protection.

### What You'll Learn
- DEX architecture
- Multi-hop swap routing
- Price quotation
- Slippage protection
- Router contract patterns
- Frontend DEX integration

### Prerequisites
- Understanding of liquidity pools
- Token standards (SIP-010)
- Clarity contract calls

---

## 🎯 Key Concepts

### DEX Components

**1. Router Contract**
- Routes swaps through optimal paths
- Handles multi-hop swaps
- Calculates best prices

**2. Price Oracle**
- Provides real-time price data
- Calculates optimal routes

**3. Slippage Protection**
- Protects users from price movement
- Set minimum acceptable output

---

## 💻 DEX Router Contract

### Complete Implementation

```clarity
;; DEX Router Contract
;; Handles token swaps and routing

(use-trait ft-trait 'SP3FBR2AGK5H9QBDH3EEN6DF8EK8JY7RX8QJ5SVTE.sip-010-trait-ft-standard.sip-010-trait)

;; Constants
(define-constant err-invalid-path (err u200))
(define-constant err-excessive-slippage (err u201))
(define-constant err-deadline-passed (err u202))
(define-constant err-insufficient-output (err u203))

;; Pool factory contract (where all pools are registered)
(define-constant pool-factory-contract .liquidity-pool-factory)

;; -----------------------------------------------------------
;; Direct Swap Functions
;; -----------------------------------------------------------

;; Swap exact input for minimum output
(define-public (swap-exact-tokens-for-tokens
    (amount-in uint)
    (amount-out-min uint)
    (path (list 5 principal))  ;; Token path (e.g., [STX, USDA, BTC])
    (deadline uint)
)
    (begin
        ;; Check deadline
        (asserts! (<= block-height deadline) err-deadline-passed)
        
        ;; Execute swap through path
        (let
            (
                (amounts (try! (get-amounts-out amount-in path)))
                (final-amount (unwrap! (element-at amounts (- (len amounts) u1)) err-invalid-path))
            )
            ;; Check slippage
            (asserts! (>= final-amount amount-out-min) err-excessive-slippage)
            
            ;; Execute swaps
            (try! (execute-swaps amount-in path amounts))
            
            (ok final-amount)
        )
    )
)

;; Swap for exact output with maximum input
(define-public (swap-tokens-for-exact-tokens
    (amount-out uint)
    (amount-in-max uint)
    (path (list 5 principal))
    (deadline uint)
)
    (begin
        (asserts! (<= block-height deadline) err-deadline-passed)
        
        (let
            (
                (amounts (try! (get-amounts-in amount-out path)))
                (required-input (unwrap! (element-at amounts u0) err-invalid-path))
            )
            (asserts! (<= required-input amount-in-max) err-excessive-slippage)
            
            (try! (execute-swaps required-input path amounts))
            
            (ok amount-out)
        )
    )
)

;; -----------------------------------------------------------
;; Multi-Hop Swap Execution
;; -----------------------------------------------------------

;; Execute series of swaps through path
(define-private (execute-swaps
    (amount-in uint)
    (path (list 5 principal))
    (amounts (list 5 uint))
)
    (let
        (
            (path-length (len path))
        )
        ;; Loop through path and execute each swap
        (ok (fold execute-single-swap
            (generate-swap-pairs path)
            { current-amount: amount-in, success: true }
        ))
    )
)

;; Execute a single swap between two tokens
(define-private (execute-single-swap
    (swap-pair { token-in: principal, token-out: principal, amount-in: uint, amount-out: uint })
    (state { current-amount: uint, success: bool })
)
    (if (get success state)
        (match (perform-swap 
            (get token-in swap-pair)
            (get token-out swap-pair)
            (get amount-in swap-pair)
            (get amount-out swap-pair)
        )
            success { current-amount: (get amount-out swap-pair), success: true }
            error { current-amount: u0, success: false }
        )
        state
    )
)

;; Perform actual swap via pool
(define-private (perform-swap
    (token-in principal)
    (token-out principal)
    (amount-in uint)
    (amount-out-min uint)
)
    (contract-call? pool-factory-contract swap-tokens
        token-in
        token-out
        amount-in
        amount-out-min
        tx-sender
    )
)

;; -----------------------------------------------------------
;; Price Calculation Functions
;; -----------------------------------------------------------

;; Calculate amounts out for exact input through path
(define-read-only (get-amounts-out
    (amount-in uint)
    (path (list 5 principal))
)
    (let
        (
            (path-length (len path))
        )
        (asserts! (>= path-length u2) err-invalid-path)
        
        (ok (fold calculate-next-amount-out
            (generate-pairs path)
            (list amount-in)
        ))
    )
)

;; Calculate amounts in for exact output through path
(define-read-only (get-amounts-in
    (amount-out uint)
    (path (list 5 principal))
)
    (let
        (
            (path-length (len path))
            (reversed-path (reverse-list path))
        )
        (asserts! (>= path-length u2) err-invalid-path)
        
        (ok (reverse-list (fold calculate-next-amount-in
            (generate-pairs reversed-path)
            (list amount-out)
        )))
    )
)

;; Calculate output amount for next hop
(define-private (calculate-next-amount-out
    (pair { token-in: principal, token-out: principal })
    (amounts (list 5 uint))
)
    (let
        (
            (current-amount (unwrap-panic (element-at amounts (- (len amounts) u1))))
            (reserves (unwrap-panic (get-pool-reserves 
                (get token-in pair)
                (get token-out pair)
            )))
            (next-amount (calculate-amount-out
                current-amount
                (get reserve-in reserves)
                (get reserve-out reserves)
            ))
        )
        (unwrap-panic (as-max-len? (append amounts next-amount) u5))
    )
)

;; Calculate input amount for previous hop
(define-private (calculate-next-amount-in
    (pair { token-in: principal, token-out: principal })
    (amounts (list 5 uint))
)
    (let
        (
            (current-amount (unwrap-panic (element-at amounts (- (len amounts) u1))))
            (reserves (unwrap-panic (get-pool-reserves 
                (get token-in pair)
                (get token-out pair)
            )))
            (next-amount (calculate-amount-in
                current-amount
                (get reserve-in reserves)
                (get reserve-out reserves)
            ))
        )
        (unwrap-panic (as-max-len? (append amounts next-amount) u5))
    )
)

;; -----------------------------------------------------------
;; AMM Math Functions
;; -----------------------------------------------------------

;; Calculate output amount (with 0.3% fee)
(define-read-only (calculate-amount-out
    (amount-in uint)
    (reserve-in uint)
    (reserve-out uint)
)
    (let
        (
            (amount-in-with-fee (* amount-in u997))  ;; 0.3% fee
            (numerator (* amount-in-with-fee reserve-out))
            (denominator (+ (* reserve-in u1000) amount-in-with-fee))
        )
        (/ numerator denominator)
    )
)

;; Calculate input amount required (with 0.3% fee)
(define-read-only (calculate-amount-in
    (amount-out uint)
    (reserve-in uint)
    (reserve-out uint)
)
    (let
        (
            (numerator (* (* reserve-in amount-out) u1000))
            (denominator (* (- reserve-out amount-out) u997))
        )
        (+ (/ numerator denominator) u1)
    )
)

;; -----------------------------------------------------------
;; Helper Functions
;; -----------------------------------------------------------

;; Get pool reserves for token pair
(define-read-only (get-pool-reserves
    (token-a principal)
    (token-b principal)
)
    (contract-call? pool-factory-contract get-reserves token-a token-b)
)

;; Generate consecutive pairs from path
(define-private (generate-pairs (path (list 5 principal)))
    (let
        (
            (token-0 (unwrap-panic (element-at path u0)))
            (token-1 (unwrap-panic (element-at path u1)))
        )
        ;; Simplified - in practice would generate all pairs
        (list { token-in: token-0, token-out: token-1 })
    )
)

;; Generate swap pairs with amounts
(define-private (generate-swap-pairs (path (list 5 principal)))
    ;; Simplified implementation
    (list)
)

;; Reverse a list
(define-private (reverse-list (items (list 5 principal)))
    ;; Implementation would reverse the list
    items
)

;; -----------------------------------------------------------
;; Price Quote Functions
;; -----------------------------------------------------------

;; Get quote for exact input
(define-read-only (quote-exact-input
    (amount-in uint)
    (token-in principal)
    (token-out principal)
)
    (match (get-pool-reserves token-in token-out)
        reserves (ok (calculate-amount-out 
            amount-in
            (get reserve-in reserves)
            (get reserve-out reserves)
        ))
        err-invalid-path
    )
)

;; Get quote for exact output
(define-read-only (quote-exact-output
    (amount-out uint)
    (token-in principal)
    (token-out principal)
)
    (match (get-pool-reserves token-in token-out)
        reserves (ok (calculate-amount-in
            amount-out
            (get reserve-in reserves)
            (get reserve-out reserves)
        ))
        err-invalid-path
    )
)

;; Get best route for swap
(define-read-only (find-best-route
    (amount-in uint)
    (token-in principal)
    (token-out principal)
)
    ;; Simplified - would check multiple routes and return best
    (ok (list token-in token-out))
)
```

---

## 🌐 Complete DEX Frontend

```javascript
import React, { useState, useEffect } from 'react';
import { useWallet } from './WalletContext';
import {
  openContractCall,
  uintCV,
  listCV,
  principalCV
} from '@stacks/transactions';

function DEXInterface() {
  const { address, isConnected } = useWallet();
  const [tokenIn, setTokenIn] = useState('STX');
  const [tokenOut, setTokenOut] = useState('USDA');
  const [amountIn, setAmountIn] = useState('');
  const [amountOut, setAmountOut] = useState('');
  const [priceImpact, setPriceImpact] = useState(0);
  const [slippage, setSlippage] = useState(0.5); // 0.5%

  // Get quote when amount changes
  useEffect(() => {
    if (amountIn) {
      fetchQuote();
    }
  }, [amountIn, tokenIn, tokenOut]);

  const fetchQuote = async () => {
    // Call quote-exact-input
    const quote = await getQuote(amountIn, tokenIn, tokenOut);
    setAmountOut(quote);
    calculatePriceImpact();
  };

  const executeSwap = async () => {
    const amountInRaw = parseFloat(amountIn) * 1000000;
    const minAmountOut = parseFloat(amountOut) * (1 - slippage / 100) * 1000000;
    const deadline = getCurrentBlock() + 100; // 100 blocks from now

    const path = listCV([
      principalCV(getTokenAddress(tokenIn)),
      principalCV(getTokenAddress(tokenOut))
    ]);

    const options = {
      network: new StacksTestnet(),
      contractAddress: 'ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM',
      contractName: 'dex-router',
      functionName: 'swap-exact-tokens-for-tokens',
      functionArgs: [
        uintCV(amountInRaw),
        uintCV(minAmountOut),
        path,
        uintCV(deadline)
      ],
      postConditionMode: PostConditionMode.Deny,
      onFinish: (data) => {
        alert(`Swap successful! TX: ${data.txId}`);
      },
    };

    await openContractCall(options);
  };

  return (
    <div className="dex-interface">
      <h1>Swap Tokens</h1>

      <div className="swap-form">
        {/* Input Token */}
        <div className="token-input">
          <label>From</label>
          <select value={tokenIn} onChange={(e) => setTokenIn(e.target.value)}>
            <option value="STX">STX</option>
            <option value="USDA">USDA</option>
            <option value="BTC">BTC</option>
          </select>
          <input
            type="number"
            value={amountIn}
            onChange={(e) => setAmountIn(e.target.value)}
            placeholder="0.0"
          />
        </div>

        {/* Swap Direction Button */}
        <button className="swap-direction" onClick={() => {
          setTokenIn(tokenOut);
          setTokenOut(tokenIn);
        }}>
          ↓
        </button>

        {/* Output Token */}
        <div className="token-input">
          <label>To</label>
          <select value={tokenOut} onChange={(e) => setTokenOut(e.target.value)}>
            <option value="STX">STX</option>
            <option value="USDA">USDA</option>
            <option value="BTC">BTC</option>
          </select>
          <input
            type="number"
            value={amountOut}
            readOnly
            placeholder="0.0"
          />
        </div>

        {/* Swap Details */}
        {amountOut && (
          <div className="swap-details">
            <div className="detail-row">
              <span>Price Impact:</span>
              <span className={priceImpact > 5 ? 'warning' : ''}>
                {priceImpact.toFixed(2)}%
              </span>
            </div>
            <div className="detail-row">
              <span>Minimum Received:</span>
              <span>{(parseFloat(amountOut) * (1 - slippage / 100)).toFixed(6)}</span>
            </div>
            <div className="detail-row">
              <span>Slippage Tolerance:</span>
              <span>{slippage}%</span>
            </div>
          </div>
        )}

        {/* Slippage Settings */}
        <div className="slippage-settings">
          <label>Slippage Tolerance</label>
          <div className="slippage-options">
            {[0.1, 0.5, 1.0].map(val => (
              <button
                key={val}
                className={slippage === val ? 'active' : ''}
                onClick={() => setSlippage(val)}
              >
                {val}%
              </button>
            ))}
            <input
              type="number"
              step="0.1"
              value={slippage}
              onChange={(e) => setSlippage(parseFloat(e.target.value))}
              placeholder="Custom"
            />
          </div>
        </div>

        {/* Swap Button */}
        <button
          className="swap-button"
          onClick={executeSwap}
          disabled={!isConnected || !amountIn || !amountOut}
        >
          {!isConnected ? 'Connect Wallet' : 'Swap'}
        </button>
      </div>

      {/* Price Chart (optional) */}
      <PriceChart tokenIn={tokenIn} tokenOut={tokenOut} />

      {/* Recent Swaps */}
      <RecentSwaps />
    </div>
  );
}

function PriceChart({ tokenIn, tokenOut }) {
  return (
    <div className="price-chart">
      <h3>{tokenIn}/{tokenOut} Price Chart</h3>
      {/* Implement chart visualization */}
    </div>
  );
}

function RecentSwaps() {
  const [swaps, setSwaps] = useState([]);

  return (
    <div className="recent-swaps">
      <h3>Recent Swaps</h3>
      <table>
        <thead>
          <tr>
            <th>From</th>
            <th>To</th>
            <th>Amount</th>
            <th>Time</th>
          </tr>
        </thead>
        <tbody>
          {swaps.map((swap, idx) => (
            <tr key={idx}>
              <td>{swap.tokenIn}</td>
              <td>{swap.tokenOut}</td>
              <td>{swap.amount}</td>
              <td>{swap.timestamp}</td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
}

export default DEXInterface;
```

---

## 📊 Advanced DEX Features

### 1. Route Optimization

```clarity
;; Find optimal route through multiple pools
(define-read-only (find-optimal-route
    (amount-in uint)
    (token-in principal)
    (token-out principal)
    (max-hops uint)
)
    (let
        (
            ;; Direct route
            (direct-output (quote-direct-swap amount-in token-in token-out))
            
            ;; Try via intermediate token (e.g., stable coin)
            (via-stable-output (quote-multi-hop 
                amount-in 
                (list token-in stable-coin token-out)
            ))
        )
        ;; Return route with best output
        (if (> via-stable-output direct-output)
            (ok (list token-in stable-coin token-out))
            (ok (list token-in token-out))
        )
    )
)
```

### 2. Price Impact Warning

```javascript
function calculatePriceImpact(amountIn, reserveIn, reserveOut) {
  const amountOut = getAmountOut(amountIn, reserveIn, reserveOut);
  const priceBeforeSwap = reserveOut / reserveIn;
  const priceAfterSwap = (reserveOut - amountOut) / (reserveIn + amountIn);
  const priceImpact = ((priceBeforeSwap - priceAfterSwap) / priceBeforeSwap) * 100;
  
  return {
    impact: priceImpact,
    warning: priceImpact > 5 ? 'High price impact!' : null
  };
}
```

---

## ⚠️ Common Pitfalls

### 1. **Insufficient Slippage Protection**
Always set reasonable slippage tolerance (0.5-1% for stable pairs, 1-5% for volatile pairs)

### 2. **Deadline Not Set**
Use block-based deadlines to prevent pending transactions from executing at bad prices

### 3. **Not Checking Price Impact**
Warn users when price impact exceeds 5%

---

## 🎓 Next Steps

**Related Guides:**
- → Liquidity Pools
- → Staking and Rewards
- → Advanced AMM Strategies