# Security Best Practices - Complete Guide

## 📖 Overview

Following security best practices from the start prevents vulnerabilities and makes contracts more maintainable. This guide provides actionable patterns for writing secure Clarity smart contracts.

### What You'll Learn
- Secure coding patterns
- Access control strategies
- Safe arithmetic operations
- State management best practices
- Testing strategies
- Deployment procedures

---

## 🔐 Access Control

### Pattern 1: Owner-Based Access

```clarity
;; Define owner at deployment
(define-constant contract-owner tx-sender)
(define-constant err-owner-only (err u100))

;; Helper function for owner checks
(define-private (is-owner)
    (is-eq tx-sender contract-owner)
)

;; Use in protected functions
(define-public (admin-function)
    (begin
        (asserts! (is-owner) err-owner-only)
        ;; Admin logic
        (ok true)
    )
)
```

### Pattern 2: Multi-Level Access Control

```clarity
;; Different permission levels
(define-constant role-admin u1)
(define-constant role-moderator u2)
(define-constant role-user u3)

(define-map user-roles principal uint)

;; Initialize admin
(map-set user-roles contract-owner role-admin)

;; Check roles
(define-private (has-role (user principal) (role uint))
    (match (map-get? user-roles user)
        user-role (is-eq user-role role)
        false
    )
)

;; Admin functions
(define-public (grant-role (user principal) (role uint))
    (begin
        (asserts! (has-role tx-sender role-admin) err-not-authorized)
        (ok (map-set user-roles user role))
    )
)

;; Protected function example
(define-public (moderate-content (content-id uint))
    (begin
        (asserts! (or 
            (has-role tx-sender role-admin)
            (has-role tx-sender role-moderator)
        ) err-not-authorized)
        ;; Moderation logic
        (ok true)
    )
)
```

### Pattern 3: Transferable Ownership

```clarity
(define-data-var contract-owner principal tx-sender)
(define-data-var pending-owner (optional principal) none)

;; Transfer ownership (two-step process)
(define-public (transfer-ownership (new-owner principal))
    (begin
        (asserts! (is-eq tx-sender (var-get contract-owner)) err-owner-only)
        (var-set pending-owner (some new-owner))
        (ok true)
    )
)

;; New owner must accept
(define-public (accept-ownership)
    (let
        (
            (pending (unwrap! (var-get pending-owner) err-no-pending-owner))
        )
        (asserts! (is-eq tx-sender pending) err-not-pending-owner)
        (var-set contract-owner pending)
        (var-set pending-owner none)
        (ok true)
    )
)
```

---

## 🔢 Safe Arithmetic

### Always Validate Before Operations

```clarity
;; Subtraction - check sufficient balance
(define-private (safe-subtract (a uint) (b uint))
    (if (>= a b)
        (ok (- a b))
        (err u"insufficient-balance")
    )
)

;; Division - check for zero
(define-private (safe-divide (a uint) (b uint))
    (if (> b u0)
        (ok (/ a b))
        (err u"division-by-zero")
    )
)

;; Usage in functions
(define-public (transfer (amount uint))
    (let
        (
            (sender-balance (get-balance tx-sender))
            (new-balance (try! (safe-subtract sender-balance amount)))
        )
        (set-balance tx-sender new-balance)
        (ok true)
    )
)
```

### Percentage Calculations

```clarity
;; Safe percentage calculation using basis points
(define-constant basis-points u10000)

(define-private (calculate-percentage (amount uint) (percentage-bps uint))
    ;; percentage-bps: 100 = 1%, 10000 = 100%
    (/ (* amount percentage-bps) basis-points)
)

;; Example: Calculate 2.5% fee
(define-public (calculate-fee (amount uint))
    (ok (calculate-percentage amount u250))  ;; 250 bps = 2.5%
)
```

---

## 🛡️ Input Validation

### Validate Everything

```clarity
;; Comprehensive validation pattern
(define-public (stake-tokens (amount uint) (duration-blocks uint))
    (begin
        ;; Validate amount
        (asserts! (> amount u0) err-invalid-amount)
        (asserts! (>= amount min-stake) err-below-minimum)
        (asserts! (<= amount max-stake) err-above-maximum)
        
        ;; Validate duration
        (asserts! (>= duration-blocks min-duration) err-duration-too-short)
        (asserts! (<= duration-blocks max-duration) err-duration-too-long)
        
        ;; Validate user has funds
        (asserts! (>= (get-balance tx-sender) amount) err-insufficient-balance)
        
        ;; Execute stake
        (execute-stake tx-sender amount duration-blocks)
    )
)
```

### Principal Validation

```clarity
;; Validate principals
(define-private (is-valid-address (addr principal))
    (and
        (not (is-eq addr contract-owner))  ;; Not contract owner
        (is-standard addr)  ;; Standard principal (not contract)
    )
)

;; Validate not self
(define-private (is-not-self (addr principal))
    (not (is-eq addr tx-sender))
)

;; Use in functions
(define-public (transfer (recipient principal) (amount uint))
    (begin
        (asserts! (is-valid-address recipient) err-invalid-recipient)
        (asserts! (is-not-self recipient) err-self-transfer)
        ;; Transfer logic
        (ok true)
    )
)
```

---

## 📝 State Management

### Use Descriptive Variable Names

```clarity
;; Bad
(define-data-var a uint u0)
(define-data-var b bool true)

;; Good
(define-data-var total-staked uint u0)
(define-data-var contract-paused bool false)
```

### Initialize Variables Properly

```clarity
;; Initialize with safe defaults
(define-data-var initialized bool false)

(define-public (initialize (params {...}))
    (begin
        ;; Prevent re-initialization
        (asserts! (not (var-get initialized)) err-already-initialized)
        
        ;; Set up contract
        (var-set initialized true)
        ;; ... other initialization
        (ok true)
    )
)
```

### Use Maps for Scalability

```clarity
;; Don't use lists for large datasets
;; Bad: (define-data-var users (list 1000 principal) (list))

;; Good: Use maps
(define-map user-data
    principal
    {
        balance: uint,
        joined-at: uint,
        active: bool
    }
)
```

---

## 🔍 Error Handling

### Comprehensive Error System

```clarity
;; Group errors by category
;; Authorization errors (100-199)
(define-constant err-owner-only (err u100))
(define-constant err-not-authorized (err u101))
(define-constant err-invalid-caller (err u102))

;; Validation errors (200-299)
(define-constant err-invalid-amount (err u200))
(define-constant err-invalid-address (err u201))
(define-constant err-invalid-parameter (err u202))

;; State errors (300-399)
(define-constant err-not-found (err u300))
(define-constant err-already-exists (err u301))
(define-constant err-contract-paused (err u302))

;; Business logic errors (400-499)
(define-constant err-insufficient-balance (err u400))
(define-constant err-transfer-failed (err u401))
```

### Always Handle Errors

```clarity
;; Bad: Ignoring potential errors
(define-public (complex-operation)
    (begin
        (operation-one)
        (operation-two)
        (ok true)
    )
)

;; Good: Proper error propagation
(define-public (complex-operation)
    (begin
        (try! (operation-one))
        (try! (operation-two))
        (ok true)
    )
)
```

---

## 📊 Event Logging

### Log All Critical Operations

```clarity
;; Template for event logging
(define-private (log-event (event-type (string-ascii 50)) (data {...}))
    (print (merge { event: event-type, block: block-height } data))
)

;; Usage examples
(define-public (transfer (recipient principal) (amount uint))
    (begin
        (try! (execute-transfer tx-sender recipient amount))
        
        (log-event "transfer" {
            from: tx-sender,
            to: recipient,
            amount: amount
        })
        
        (ok true)
    )
)

(define-public (update-settings (new-fee uint))
    (begin
        (asserts! (is-owner) err-owner-only)
        
        (log-event "settings-updated" {
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

## 🧩 Function Design

### Keep Functions Small and Focused

```clarity
;; Bad: One huge function
(define-public (complex-trade (params ...))
    ;; 100 lines of logic
)

;; Good: Break into smaller functions
(define-public (complex-trade (params ...))
    (begin
        (try! (validate-trade params))
        (try! (check-balances params))
        (try! (execute-trade params))
        (try! (settle-trade params))
        (ok true)
    )
)

(define-private (validate-trade (params ...))
    ;; Focused validation logic
)

(define-private (check-balances (params ...))
    ;; Check sufficient funds
)
```

### Use Read-Only for Queries

```clarity
;; Read-only functions are free and safe
(define-read-only (get-user-balance (user principal))
    (default-to u0 (map-get? balances user))
)

(define-read-only (calculate-reward (stake-amount uint) (blocks uint))
    (* stake-amount (/ blocks u52560))  ;; Simple APY calculation
)

;; Use them in public functions
(define-public (claim-rewards)
    (let
        (
            (stake-info (get-stake-info tx-sender))
            (reward (calculate-reward 
                (get amount stake-info)
                (- block-height (get start-block stake-info))
            ))
        )
        (try! (mint-rewards tx-sender reward))
        (ok reward)
    )
)
```

---

## 🔒 Contract Pausing

### Emergency Stop Pattern

```clarity
(define-data-var contract-paused bool false)
(define-constant err-contract-paused (err u302))

;; Modifier-like function
(define-private (check-not-paused)
    (asserts! (not (var-get contract-paused)) err-contract-paused)
)

;; Use in all critical functions
(define-public (transfer (recipient principal) (amount uint))
    (begin
        (check-not-paused)
        ;; Transfer logic
        (ok true)
    )
)

;; Emergency pause (owner only)
(define-public (pause-contract)
    (begin
        (asserts! (is-owner) err-owner-only)
        (var-set contract-paused true)
        (print { event: "contract-paused", by: tx-sender })
        (ok true)
    )
)

(define-public (unpause-contract)
    (begin
        (asserts! (is-owner) err-owner-only)
        (var-set contract-paused false)
        (print { event: "contract-unpaused", by: tx-sender })
        (ok true)
    )
)
```

---

## 🧪 Testing Best Practices

### Test Coverage Requirements

```typescript
// 1. Test happy path
Clarinet.test({
    name: "Successfully transfers tokens",
    async fn(chain: Chain, accounts: Map<string, Account>) {
        // Normal successful transfer
    }
});

// 2. Test permissions
Clarinet.test({
    name: "Non-owner cannot mint",
    async fn(chain: Chain, accounts: Map<string, Account>) {
        // Should fail with err-owner-only
    }
});

// 3. Test edge cases
Clarinet.test({
    name: "Cannot transfer zero amount",
    async fn(chain: Chain, accounts: Map<string, Account>) {
        // Should fail with err-invalid-amount
    }
});

// 4. Test state transitions
Clarinet.test({
    name: "Balance updates correctly after transfer",
    async fn(chain: Chain, accounts: Map<string, Account>) {
        // Verify before and after balances
    }
});

// 5. Test error conditions
Clarinet.test({
    name: "Transfer fails with insufficient balance",
    async fn(chain: Chain, accounts: Map<string, Account>) {
        // Should fail with err-insufficient-balance
    }
});
```

### Integration Testing

```typescript
// Test contract interactions
Clarinet.test({
    name: "Swap works across multiple pools",
    async fn(chain: Chain, accounts: Map<string, Account>) {
        // 1. Create pools
        // 2. Add liquidity
        // 3. Execute swap
        // 4. Verify final balances
    }
});

// Test complex workflows
Clarinet.test({
    name: "Complete staking lifecycle",
    async fn(chain: Chain, accounts: Map<string, Account>) {
        // 1. Stake tokens
        // 2. Wait some blocks
        // 3. Claim rewards
        // 4. Unstake
        // 5. Verify all balances correct
    }
});
```

---

## 🚀 Deployment Best Practices

### Pre-Deployment Checklist

```clarity
;; 1. Set correct initial values
(define-constant contract-owner tx-sender)  ;; Will be deployer

;; 2. Use constants for immutable values
(define-constant token-name "MyToken")
(define-constant token-symbol "MTK")

;; 3. Document all constants
;; Fee: 0.3% (30 basis points)
(define-constant fee-bps u30)

;; 4. Initialize with safe defaults
(define-data-var initialized bool false)
(define-data-var total-supply uint u0)
```

### Deployment Steps

```bash
# 1. Test thoroughly on testnet
clarinet test

# 2. Deploy to testnet
clarinet deploy --testnet

# 3. Verify contract on explorer
# Check all functions and constants

# 4. Test on testnet with real transactions
# Perform all critical operations

# 5. Security audit (for mainnet)
# Get professional audit if handling significant value

# 6. Deploy to mainnet
clarinet deploy --mainnet

# 7. Verify on mainnet explorer

# 8. Test with small amounts first
```

---

## 📋 Security Review Checklist

### Before Mainnet Deployment

**Access Control:**
- [ ] All admin functions have owner checks
- [ ] Owner can be transferred if needed
- [ ] No functions allow unauthorized access

**Input Validation:**
- [ ] All inputs are validated
- [ ] Amount checks prevent zero/negative
- [ ] Principal validation for addresses
- [ ] Parameter ranges are enforced

**Arithmetic:**
- [ ] All subtractions check for underflow
- [ ] Divisions check for zero
- [ ] Percentage calculations use basis points

**State Management:**
- [ ] State changes are validated
- [ ] No unbounded loops or arrays
- [ ] Maps used instead of large lists

**Error Handling:**
- [ ] All errors have unique codes
- [ ] Error messages are clear
- [ ] All operations use try! or proper handling

**Events:**
- [ ] Critical operations emit events
- [ ] Events include all relevant data
- [ ] Event names are descriptive

**Testing:**
- [ ] >90% code coverage
- [ ] All error paths tested
- [ ] Integration tests complete
- [ ] Edge cases covered

**Documentation:**
- [ ] All functions documented
- [ ] Constants explained
- [ ] Error codes documented
- [ ] Usage examples provided

---

## 🎓 Common Patterns Library

### Safe Transfer Pattern

```clarity
(define-public (safe-transfer 
    (token-trait <ft-trait>)
    (amount uint)
    (sender principal)
    (recipient principal)
)
    (begin
        ;; Validations
        (asserts! (is-eq sender tx-sender) err-unauthorized)
        (asserts! (> amount u0) err-invalid-amount)
        (asserts! (not (is-eq sender recipient)) err-self-transfer)
        
        ;; Execute transfer
        (try! (contract-call? token-trait transfer 
            amount sender recipient none
        ))
        
        ;; Log event
        (print {
            event: "transfer",
            token: (contract-of token-trait),
            amount: amount,
            from: sender,
            to: recipient
        })
        
        (ok true)
    )
)
```

### Withdraw Pattern (Pull over Push)

```clarity
;; Good: Users withdraw their own funds
(define-map pending-withdrawals principal uint)

(define-public (claim-withdrawal)
    (let
        (
            (amount (default-to u0 (map-get? pending-withdrawals tx-sender)))
        )
        (asserts! (> amount u0) err-no-withdrawal)
        
        ;; Clear pending before transfer (reentrancy protection)
        (map-delete pending-withdrawals tx-sender)
        
        ;; Transfer funds
        (try! (stx-transfer? amount (as-contract tx-sender) tx-sender))
        
        (ok amount)
    )
)
```

---

## 📚 Next Steps

**Related Guides:**
- → Post-Conditions Deep Dive
- → Security Audit Checklist
- → Common Vulnerabilities