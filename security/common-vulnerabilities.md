# Common Vulnerabilities - Complete Security Guide

## 📖 Overview

Understanding common security vulnerabilities is essential before deploying smart contracts to mainnet. Unlike traditional software, blockchain bugs can result in permanent, irrecoverable loss of funds.

### What You'll Learn
- Common Clarity-specific vulnerabilities
- How to identify security issues
- Real-world attack examples
- Prevention techniques
- Safe coding patterns

### Prerequisites
- Understanding of Clarity fundamentals
- Knowledge of contract interactions
- Familiarity with access control

---

## 🚨 Critical Vulnerabilities

### 1. Unauthorized Access

**Problem:** Functions that should be restricted are accessible to anyone.

#### ❌ Vulnerable Code

```clarity
;; BAD: Anyone can mint tokens!
(define-public (mint (amount uint) (recipient principal))
    (ft-mint? my-token amount recipient)
)

;; BAD: Anyone can change contract settings!
(define-public (set-fee (new-fee uint))
    (ok (var-set fee-percentage new-fee))
)
```

#### ✅ Secure Code

```clarity
(define-constant contract-owner tx-sender)
(define-constant err-owner-only (err u100))

;; GOOD: Only owner can mint
(define-public (mint (amount uint) (recipient principal))
    (begin
        (asserts! (is-eq tx-sender contract-owner) err-owner-only)
        (ft-mint? my-token amount recipient)
    )
)

;; GOOD: Protected admin function
(define-public (set-fee (new-fee uint))
    (begin
        (asserts! (is-eq tx-sender contract-owner) err-owner-only)
        (ok (var-set fee-percentage new-fee))
    )
)
```

#### Real-World Impact
- Unauthorized minting = infinite token supply
- Setting fees to 0 = loss of revenue
- Changing parameters = protocol manipulation

---

### 2. Integer Overflow/Underflow

**Problem:** Arithmetic operations that exceed type limits.

#### ❌ Vulnerable Code

```clarity
;; BAD: Subtraction can underflow
(define-public (transfer (amount uint))
    (let
        (
            (sender-balance (get-balance tx-sender))
            ;; This will fail if sender-balance < amount, but doesn't handle it
            (new-balance (- sender-balance amount))
        )
        (set-balance tx-sender new-balance)
        (ok true)
    )
)

;; BAD: Addition can overflow (though Clarity prevents this, still bad practice)
(define-public (compound-rewards)
    (let
        (
            (balance (get-balance tx-sender))
            (rewards (calculate-rewards tx-sender))
        )
        ;; What if balance + rewards > max-uint?
        (set-balance tx-sender (+ balance rewards))
        (ok true)
    )
)
```

#### ✅ Secure Code

```clarity
(define-constant err-insufficient-balance (err u102))
(define-constant err-overflow (err u103))

;; GOOD: Check before subtraction
(define-public (transfer (amount uint))
    (let
        (
            (sender-balance (get-balance tx-sender))
        )
        (asserts! (>= sender-balance amount) err-insufficient-balance)
        (set-balance tx-sender (- sender-balance amount))
        (ok true)
    )
)

;; GOOD: Safe addition with checks
(define-public (compound-rewards)
    (let
        (
            (balance (get-balance tx-sender))
            (rewards (calculate-rewards tx-sender))
            (new-balance (+ balance rewards))
        )
        ;; Clarity prevents overflow, but explicit check is good practice
        (asserts! (>= new-balance balance) err-overflow)
        (set-balance tx-sender new-balance)
        (ok true)
    )
)
```

**Note:** Clarity prevents overflow by default, but explicit checks make code clearer and more maintainable.

---

### 3. Unprotected State Changes

**Problem:** State can be changed without proper validation.

#### ❌ Vulnerable Code

```clarity
;; BAD: No validation before state change
(define-public (set-price (new-price uint))
    (ok (var-set current-price new-price))
)

;; BAD: Anyone can change critical parameters
(define-public (update-threshold (new-threshold uint))
    (ok (var-set approval-threshold new-threshold))
)
```

#### ✅ Secure Code

```clarity
(define-constant min-price u1000000) ;; 1 STX minimum
(define-constant max-price u1000000000000) ;; 1M STX maximum
(define-constant err-invalid-price (err u200))

;; GOOD: Validate price range
(define-public (set-price (new-price uint))
    (begin
        (asserts! (is-eq tx-sender contract-owner) err-owner-only)
        (asserts! (>= new-price min-price) err-invalid-price)
        (asserts! (<= new-price max-price) err-invalid-price)
        (ok (var-set current-price new-price))
    )
)

;; GOOD: Validate threshold is reasonable
(define-public (update-threshold (new-threshold uint))
    (let
        (
            (owner-count (len (var-get owners)))
        )
        (asserts! (is-eq tx-sender contract-owner) err-owner-only)
        (asserts! (> new-threshold u0) err-invalid-threshold)
        (asserts! (<= new-threshold owner-count) err-invalid-threshold)
        (ok (var-set approval-threshold new-threshold))
    )
)
```

---

### 4. Missing Input Validation

**Problem:** Functions accept invalid inputs that break contract logic.

#### ❌ Vulnerable Code

```clarity
;; BAD: No validation
(define-public (stake (amount uint))
    (begin
        (transfer-to-contract amount)
        (record-stake tx-sender amount)
        (ok true)
    )
)

;; BAD: Accepts zero or negative
(define-public (create-pool (token-a principal) (token-b principal) (amount uint))
    (map-set pools { token-a: token-a, token-b: token-b } amount)
    (ok true)
)
```

#### ✅ Secure Code

```clarity
(define-constant min-stake u1000000) ;; 1 token
(define-constant err-invalid-amount (err u104))
(define-constant err-same-token (err u105))

;; GOOD: Validate amount
(define-public (stake (amount uint))
    (begin
        (asserts! (>= amount min-stake) err-invalid-amount)
        (asserts! (> amount u0) err-invalid-amount)
        (transfer-to-contract amount)
        (record-stake tx-sender amount)
        (ok true)
    )
)

;; GOOD: Validate pool parameters
(define-public (create-pool (token-a principal) (token-b principal) (amount uint))
    (begin
        (asserts! (not (is-eq token-a token-b)) err-same-token)
        (asserts! (> amount u0) err-invalid-amount)
        (map-set pools { token-a: token-a, token-b: token-b } amount)
        (ok true)
    )
)
```

---

### 5. Reentrancy (Less Common in Clarity)

**Problem:** While Clarity is protected against reentrancy by design, understanding the concept is important.

**Why Clarity is Safe:**
- No external calls during execution
- Read-only functions cannot modify state
- Public functions complete before calling others

#### Example of What Would Be Vulnerable (Not Possible in Clarity)

```clarity
;; This pattern would be vulnerable in Solidity but isn't possible in Clarity
;; Clarity's execution model prevents this

;; Solidity-style vulnerable pattern (for educational purposes):
;; 1. User calls withdraw
;; 2. Contract sends funds
;; 3. During send, user's receive function calls withdraw again
;; 4. Repeat until contract drained

;; Clarity prevents this by:
;; - Not allowing state changes during external calls
;; - Completing all state changes before value transfers
;; - Using post-conditions to verify transfers
```

---

### 6. Front-Running

**Problem:** Transactions visible in mempool can be exploited by miners or other users.

#### ❌ Vulnerable Code

```clarity
;; BAD: Price can be front-run
(define-public (buy-nft (token-id uint))
    (let
        (
            (price (get-nft-price token-id))
        )
        ;; Price fetched, but could change before execution
        (try! (stx-transfer? price tx-sender (get-owner token-id)))
        (try! (transfer-nft token-id))
        (ok true)
    )
)
```

#### ✅ Secure Code

```clarity
(define-constant err-price-changed (err u106))

;; GOOD: User specifies max price they'll pay
(define-public (buy-nft (token-id uint) (max-price uint))
    (let
        (
            (current-price (get-nft-price token-id))
        )
        ;; Protect user from price changes
        (asserts! (<= current-price max-price) err-price-changed)
        (try! (stx-transfer? current-price tx-sender (get-owner token-id)))
        (try! (transfer-nft token-id))
        (ok true)
    )
)

;; GOOD: Use slippage protection for swaps
(define-public (swap-tokens (amount-in uint) (min-amount-out uint))
    (let
        (
            (amount-out (calculate-swap amount-in))
        )
        (asserts! (>= amount-out min-amount-out) err-slippage-exceeded)
        ;; Execute swap
        (ok amount-out)
    )
)
```

---

### 7. Timestamp Dependence (Block Height Manipulation)

**Problem:** Logic depending on block-height can be slightly manipulated by miners.

#### ❌ Vulnerable Code

```clarity
;; BAD: Exact block height checks can be gamed
(define-public (claim-reward)
    (begin
        ;; Miner can choose to include tx in current or next block
        (asserts! (is-eq block-height u1000) err-wrong-block)
        (mint-reward tx-sender)
        (ok true)
    )
)
```

#### ✅ Secure Code

```clarity
;; GOOD: Use block ranges, not exact values
(define-public (claim-reward)
    (begin
        (asserts! (>= block-height u1000) err-too-early)
        (asserts! (< block-height u1100) err-too-late)
        (asserts! (not (has-claimed tx-sender)) err-already-claimed)
        (mint-reward tx-sender)
        (ok true)
    )
)

;; GOOD: Use sufficient time windows
(define-read-only (is-expired (expiry-block uint))
    ;; Add buffer to prevent manipulation
    (>= block-height (+ expiry-block u10))
)
```

---

### 8. Denial of Service (DoS)

**Problem:** Contract becomes unusable due to unbounded operations or gas exhaustion.

#### ❌ Vulnerable Code

```clarity
;; BAD: Unbounded loop
(define-public (distribute-rewards)
    (begin
        ;; This will fail if too many users
        (map reward-user (var-get all-users))
        (ok true)
    )
)

;; BAD: Single user can block everyone
(define-public (withdraw-all)
    (let
        (
            (users (var-get user-list))
        )
        ;; If one user's withdrawal fails, all fail
        (map force-withdraw users)
        (ok true)
    )
)
```

#### ✅ Secure Code

```clarity
;; GOOD: Users withdraw individually
(define-public (withdraw-my-rewards)
    (let
        (
            (amount (get-pending-rewards tx-sender))
        )
        (asserts! (> amount u0) err-no-rewards)
        (try! (transfer-rewards tx-sender amount))
        (mark-withdrawn tx-sender)
        (ok amount)
    )
)

;; GOOD: Batch processing with limits
(define-public (process-batch (start-index uint) (count uint))
    (begin
        (asserts! (<= count u50) err-batch-too-large)
        (process-users start-index count)
        (ok true)
    )
)
```

---

### 9. Missing Event Logging

**Problem:** No way to track critical state changes off-chain.

#### ❌ Vulnerable Code

```clarity
;; BAD: No logging
(define-public (transfer (recipient principal) (amount uint))
    (begin
        (try! (ft-transfer? my-token amount tx-sender recipient))
        (ok true)
    )
)
```

#### ✅ Secure Code

```clarity
;; GOOD: Log all important events
(define-public (transfer (recipient principal) (amount uint))
    (begin
        (try! (ft-transfer? my-token amount tx-sender recipient))
        
        ;; Log the transfer
        (print {
            event: "transfer",
            from: tx-sender,
            to: recipient,
            amount: amount,
            block: block-height
        })
        
        (ok true)
    )
)

;; GOOD: Log state changes
(define-public (update-settings (new-fee uint))
    (begin
        (asserts! (is-eq tx-sender contract-owner) err-owner-only)
        
        (print {
            event: "settings-updated",
            old-fee: (var-get fee),
            new-fee: new-fee,
            updated-by: tx-sender
        })
        
        (var-set fee new-fee)
        (ok true)
    )
)
```

---

### 10. Unsafe External Calls

**Problem:** Calling untrusted contracts without validation.

#### ❌ Vulnerable Code

```clarity
;; BAD: No validation of external contract
(define-public (call-external (contract principal))
    (contract-call? contract some-function)
)

;; BAD: Trusting external contract response
(define-public (get-price-from-oracle (oracle principal))
    (let
        (
            (price (unwrap-panic (contract-call? oracle get-price)))
        )
        ;; What if oracle returns manipulated price?
        (use-price price)
    )
)
```

#### ✅ Secure Code

```clarity
;; GOOD: Whitelist trusted contracts
(define-map trusted-contracts principal bool)

(define-public (call-external (contract principal))
    (begin
        (asserts! (is-trusted contract) err-untrusted-contract)
        (contract-call? contract some-function)
    )
)

;; GOOD: Validate oracle response
(define-public (get-price-from-oracle (oracle principal))
    (let
        (
            (price (unwrap! (contract-call? oracle get-price) err-oracle-failed))
        )
        ;; Validate price is reasonable
        (asserts! (> price min-price) err-invalid-price)
        (asserts! (< price max-price) err-invalid-price)
        (use-price price)
    )
)

;; GOOD: Use multiple oracles for critical data
(define-public (get-validated-price)
    (let
        (
            (price1 (unwrap! (contract-call? .oracle1 get-price) err-oracle-failed))
            (price2 (unwrap! (contract-call? .oracle2 get-price) err-oracle-failed))
            (price3 (unwrap! (contract-call? .oracle3 get-price) err-oracle-failed))
        )
        ;; Use median price
        (ok (median (list price1 price2 price3)))
    )
)
```

---

## 🛡️ Prevention Checklist

### Before Deploying

- [ ] All admin functions have access control
- [ ] All inputs are validated
- [ ] All arithmetic operations have overflow checks
- [ ] State changes have proper validation
- [ ] Critical operations emit events
- [ ] External calls are to trusted contracts only
- [ ] Front-running protection where needed
- [ ] DoS vectors identified and mitigated
- [ ] Error handling is comprehensive
- [ ] Post-conditions used in frontend

---

## 🧪 Security Testing

### Test Every Vulnerability

```typescript
// Test unauthorized access
Clarinet.test({
    name: "Non-owner cannot mint tokens",
    async fn(chain: Chain, accounts: Map<string, Account>) {
        const wallet1 = accounts.get('wallet_1')!;
        
        let block = chain.mineBlock([
            Tx.contractCall('my-token', 'mint',
                [types.uint(1000000), types.principal(wallet1.address)],
                wallet1.address  // Not owner
            )
        ]);
        
        block.receipts[0].result.expectErr().expectUint(100);
    }
});

// Test overflow protection
Clarinet.test({
    name: "Cannot transfer more than balance",
    async fn(chain: Chain, accounts: Map<string, Account>) {
        // Setup: Give wallet1 100 tokens
        // Try to transfer 200 tokens
        // Should fail
    }
});

// Test input validation
Clarinet.test({
    name: "Cannot stake zero amount",
    async fn(chain: Chain, accounts: Map<string, Account>) {
        const wallet1 = accounts.get('wallet_1')!;
        
        let block = chain.mineBlock([
            Tx.contractCall('staking', 'stake',
                [types.uint(0)],
                wallet1.address
            )
        ]);
        
        block.receipts[0].result.expectErr();
    }
});
```

---

## 📚 Real-World Examples

### Case Study: Unauthorized Minting

**Problem:** A token contract allowed anyone to call the mint function.

**Impact:** Attacker minted unlimited tokens, crashing the token price to zero.

**Fix:**
```clarity
;; Before
(define-public (mint (amount uint))
    (ft-mint? token amount tx-sender)
)

;; After
(define-public (mint (amount uint))
    (begin
        (asserts! (is-eq tx-sender contract-owner) err-owner-only)
        (ft-mint? token amount tx-sender)
    )
)
```

---

## 🎓 Next Steps

**Related Guides:**
- → Security Best Practices
- → Post-Conditions Guide
- → Security Audit Checklist