# Functions and Control Flow - Complete Guide

## 📖 Overview

Functions are the building blocks of Clarity contracts. Understanding the different function types and control flow patterns is essential for writing effective smart contracts.

### What You'll Learn
- Three types of functions (public, read-only, private)
- Function parameters and return values
- Control flow (if, match, let, begin)
- Assertions and guards
- Advanced patterns

---

## 🎯 Three Types of Functions

### 1. Public Functions

Can be called by anyone, write to blockchain, cost gas, must return `response`.

```clarity
;; Basic public function
(define-public (say-hello)
    (ok "Hello!")
)

;; With parameters
(define-public (transfer (recipient principal) (amount uint))
    (begin
        ;; Validation
        (asserts! (> amount u0) (err u100))
        
        ;; Transfer logic here
        (ok true)
    )
)

;; Complex operations
(define-public (mint-tokens (amount uint) (recipient principal))
    (begin
        ;; Multiple checks
        (asserts! (is-eq tx-sender contract-owner) (err u101))
        (asserts! (> amount u0) (err u102))
        
        ;; Mint tokens
        (try! (ft-mint? my-token amount recipient))
        
        ;; Return success
        (ok amount)
    )
)
```

**Key characteristics:**
- Anyone can call them
- Write to the blockchain
- Cost gas
- Must return `(ok ...)` or `(err ...)`
- Shown in contract ABI

### 2. Read-Only Functions

Can read data but cannot modify state, free to call, no gas cost.

```clarity
;; Basic read-only
(define-read-only (get-balance (user principal))
    (default-to u0 (map-get? balances user))
)

;; With computation
(define-read-only (calculate-fee (amount uint))
    (/ (* amount u25) u10000)  ;; 0.25% fee
)

;; Multiple reads
(define-read-only (get-user-info (user principal))
    (let
        (
            (balance (default-to u0 (map-get? balances user)))
            (name (default-to "" (map-get? usernames user)))
        )
        {
            user: user,
            balance: balance,
            name: name
        }
    )
)

;; No response wrapper needed
(define-read-only (get-contract-name)
    "My Contract"
)
```

**Key characteristics:**
- Free to call (no gas)
- Cannot modify blockchain state
- Can be called from public functions
- Return any type (no response wrapper)
- Perfect for getting data

### 3. Private Functions

Only callable from within the contract, helper functions.

```clarity
;; Basic private function
(define-private (is-owner)
    (is-eq tx-sender contract-owner)
)

;; With parameters
(define-private (calculate-reward (amount uint) (multiplier uint))
    (* amount multiplier)
)

;; Used in public functions
(define-public (admin-function)
    (begin
        (asserts! (is-owner) (err u100))
        (ok true)
    )
)

;; Complex helper
(define-private (validate-transfer (sender principal) (amount uint))
    (let
        (
            (sender-balance (default-to u0 (map-get? balances sender)))
        )
        (and 
            (>= sender-balance amount)
            (> amount u0)
        )
    )
)
```

**Key characteristics:**
- Only callable from within contract
- Not exposed in ABI
- Used for code organization
- Can return any type
- Don't cost extra gas

---

## 🎛️ Control Flow

### if Expressions

```clarity
;; Basic if
(if (> amount u100)
    "Large amount"
    "Small amount"
)

;; If in functions
(define-public (check-amount (amount uint))
    (ok (if (> amount u100)
        "large"
        "small"
    ))
)

;; Nested if
(if (> amount u1000)
    "huge"
    (if (> amount u100)
        "large"
        "small"
    )
)

;; If with complex condition
(if (and (> amount u0) (is-eq sender tx-sender))
    (ok "valid")
    (err u100)
)
```

### match Expressions

Handle optional and response types.

```clarity
;; Match optional
(match (map-get? users tx-sender)
    user-data (ok user-data)
    (err u404)  ;; If none
)

;; Match response
(match (contract-call? .other-contract some-function)
    success (ok success)
    error (err error)
)

;; Complex match
(match (map-get? listings token-id)
    listing 
        (if (is-eq (get seller listing) tx-sender)
            (ok "You own this")
            (ok "Someone else owns this")
        )
    (err u404)
)

;; Nested match
(match (map-get? users user)
    user-data
        (match (map-get? scores user)
            score (ok { user: user-data, score: score })
            (ok { user: user-data, score: u0 })
        )
    (err u404)
)
```

### let Bindings

Create local variables.

```clarity
;; Basic let
(let
    (
        (x u10)
        (y u20)
    )
    (+ x y)  ;; u30
)

;; Let in functions
(define-public (transfer (recipient principal) (amount uint))
    (let
        (
            (sender-balance (default-to u0 (map-get? balances tx-sender)))
            (recipient-balance (default-to u0 (map-get? balances recipient)))
        )
        (asserts! (>= sender-balance amount) (err u100))
        
        (map-set balances tx-sender (- sender-balance amount))
        (map-set balances recipient (+ recipient-balance amount))
        
        (ok true)
    )
)

;; Multiple lets
(let
    (
        (price u1000000)
        (fee (/ (* price u25) u10000))
        (total (+ price fee))
    )
    total  ;; Returns total
)

;; Let with destructuring
(let
    (
        (user-data (unwrap! (map-get? users tx-sender) (err u404)))
        (user-name (get name user-data))
        (user-balance (get balance user-data))
    )
    (ok { name: user-name, balance: user-balance })
)
```

### begin Expressions

Execute multiple expressions in sequence.

```clarity
;; Basic begin
(begin
    (var-set counter u1)
    (var-set active true)
    (ok "Done")
)

;; Begin in public functions
(define-public (initialize)
    (begin
        (asserts! (is-eq tx-sender contract-owner) (err u100))
        (var-set initialized true)
        (var-set admin tx-sender)
        (ok true)
    )
)

;; Begin with try!
(define-public (multi-step-operation)
    (begin
        (try! (step-one))
        (try! (step-two))
        (try! (step-three))
        (ok "All steps completed")
    )
)

;; Begin with prints
(begin
    (print { event: "transfer", amount: u100 })
    (map-set balances recipient new-balance)
    (ok true)
)
```

---

## 🛡️ Assertions and Guards

### asserts!

Check conditions, return error if false.

```clarity
;; Basic assertion
(asserts! (> amount u0) (err u100))

;; Multiple assertions
(define-public (transfer (amount uint))
    (begin
        (asserts! (> amount u0) (err u100))
        (asserts! (is-eq tx-sender sender) (err u101))
        (asserts! (has-sufficient-balance sender amount) (err u102))
        (ok true)
    )
)

;; Assertion with complex condition
(asserts! 
    (and 
        (> amount u0)
        (<= amount max-transfer)
        (is-authorized tx-sender)
    )
    (err u100)
)
```

### unwrap!

Extract value from optional or return error.

```clarity
;; Unwrap optional
(let
    (
        (user (unwrap! (map-get? users tx-sender) (err u404)))
    )
    (ok user)
)

;; Unwrap response
(let
    (
        (result (unwrap! (contract-call? .other-contract get-data) (err u500)))
    )
    (ok result)
)

;; Multiple unwraps
(define-public (complex-operation)
    (let
        (
            (user (unwrap! (map-get? users tx-sender) (err u404)))
            (balance (unwrap! (get-balance tx-sender) (err u500)))
        )
        (ok { user: user, balance: balance })
    )
)
```

### try!

Unwrap response or return its error.

```clarity
;; Basic try
(try! (some-function))

;; Chain operations
(define-public (multi-call)
    (begin
        (try! (call-one))
        (try! (call-two))
        (try! (call-three))
        (ok true)
    )
)

;; If any function returns (err ...), entire function stops and returns that error

;; Try with contract calls
(define-public (forward-call)
    (let
        (
            (result (try! (contract-call? .other-contract do-something)))
        )
        (ok result)
    )
)
```

### unwrap-panic

Extract value or halt execution (testing/admin only).

```clarity
;; Unwrap-panic optional
(unwrap-panic (some u10))  ;; Returns u10
(unwrap-panic none)  ;; STOPS CONTRACT EXECUTION!

;; Unwrap-panic response
(unwrap-panic (ok u10))  ;; Returns u10
(unwrap-panic (err u404))  ;; STOPS EXECUTION!

;; Use case: Contract initialization (one-time only)
(define-public (initialize-admin)
    (begin
        (asserts! (is-none (var-get admin)) (err u100))
        (var-set admin (some tx-sender))
        (ok true)
    )
)

;; Use case: Testing
(define-private (test-helper)
    (unwrap-panic (map-get? test-data u1))
)
```

---

## 🎨 Advanced Function Patterns

### Pattern 1: Guard Clauses

```clarity
(define-public (privileged-action (amount uint))
    (begin
        ;; Guard clauses at the start
        (asserts! (is-eq tx-sender contract-owner) (err u100))
        (asserts! (> amount u0) (err u101))
        (asserts! (var-get contract-active) (err u102))
        
        ;; Main logic
        (var-set total-amount (+ (var-get total-amount) amount))
        (ok true)
    )
)
```

### Pattern 2: Early Returns

```clarity
(define-public (conditional-transfer (recipient principal) (amount uint))
    (let
        (
            (sender-balance (default-to u0 (map-get? balances tx-sender)))
        )
        ;; Early return if zero amount
        (if (is-eq amount u0)
            (ok true)
            (begin
                (asserts! (>= sender-balance amount) (err u100))
                ;; Transfer logic
                (ok true)
            )
        )
    )
)
```

### Pattern 3: Function Composition

```clarity
;; Helper functions
(define-private (calculate-fee (amount uint))
    (/ (* amount u25) u10000)
)

(define-private (apply-fee (amount uint))
    (- amount (calculate-fee amount))
)

;; Main function uses helpers
(define-public (transfer-with-fee (recipient principal) (amount uint))
    (let
        (
            (net-amount (apply-fee amount))
            (fee (calculate-fee amount))
        )
        (try! (transfer-tokens recipient net-amount))
        (try! (collect-fee fee))
        (ok net-amount)
    )
)
```

### Pattern 4: State Machines

```clarity
(define-constant state-pending u0)
(define-constant state-active u1)
(define-constant state-completed u2)
(define-constant state-cancelled u3)

(define-map orders
    uint
    { status: uint, amount: uint }
)

(define-public (transition-order (order-id uint) (new-status uint))
    (let
        (
            (order (unwrap! (map-get? orders order-id) (err u404)))
            (current-status (get status order))
        )
        ;; Validate state transition
        (asserts! 
            (or
                (and (is-eq current-status state-pending) (is-eq new-status state-active))
                (and (is-eq current-status state-active) (is-eq new-status state-completed))
                (and (is-eq current-status state-pending) (is-eq new-status state-cancelled))
            )
            (err u100)
        )
        
        ;; Update status
        (ok (map-set orders order-id (merge order { status: new-status })))
    )
)
```

### Pattern 5: Batch Operations

```clarity
;; Process list of recipients
(define-public (batch-transfer (recipients (list 100 principal)) (amount uint))
    (let
        (
            (total (+ u1 u1))  ;; Calculate total needed
        )
        (ok (fold transfer-to-recipient recipients (ok true)))
    )
)

(define-private (transfer-to-recipient (recipient principal) (previous-result (response bool uint)))
    (match previous-result
        success (transfer recipient u100)  ;; Continue if previous succeeded
        error (err error)  ;; Propagate error
    )
)
```

---

## 📊 Function Parameters and Return Types

### Parameter Types

```clarity
;; Simple types
(define-public (simple-params 
    (amount uint)
    (recipient principal)
    (active bool)
)
    (ok true)
)

;; Complex types
(define-public (complex-params
    (user-data { name: (string-ascii 50), age: uint })
    (token-ids (list 10 uint))
    (memo (optional (buff 34)))
)
    (ok true)
)

;; Trait parameters (interfaces)
(use-trait nft-trait .nft-trait.nft-trait)

(define-public (accept-nft 
    (nft-contract <nft-trait>)
    (token-id uint)
)
    (contract-call? nft-contract transfer token-id tx-sender (as-contract tx-sender))
)
```

### Return Types

```clarity
;; Public functions return response
(define-public (returns-uint)
    (ok u100)
)

(define-public (returns-bool)
    (ok true)
)

(define-public (returns-tuple)
    (ok { amount: u100, recipient: tx-sender })
)

;; Read-only returns any type
(define-read-only (returns-string)
    "Hello"
)

(define-read-only (returns-optional)
    (some u10)
)
```

---

## ⚠️ Common Mistakes

### 1. **Not Returning Response in Public Functions**

❌ **Wrong:**
```clarity
(define-public (do-something)
    true
)
```

✅ **Correct:**
```clarity
(define-public (do-something)
    (ok true)
)
```

### 2. **Forgetting try! for Responses**

❌ **Wrong:**
```clarity
(define-public (chain-calls)
    (begin
        (call-one)  ;; Returns response, not handled
        (call-two)
        (ok true)
    )
)
```

✅ **Correct:**
```clarity
(define-public (chain-calls)
    (begin
        (try! (call-one))
        (try! (call-two))
        (ok true)
    )
)
```

### 3. **Not Using let for Complex Logic**

❌ **Wrong:**
```clarity
(define-public (complex-calc)
    (ok (+ (- (* u10 u5) u20) (/ u100 u5)))
)
```

✅ **Correct:**
```clarity
(define-public (complex-calc)
    (let
        (
            (product (* u10 u5))
            (difference (- product u20))
            (quotient (/ u100 u5))
        )
        (ok (+ difference quotient))
    )
)
```

### 4. **Nested if Instead of Match**

❌ **Wrong:**
```clarity
(if (is-some (map-get? users tx-sender))
    (ok (unwrap-panic (map-get? users tx-sender)))
    (err u404)
)
```

✅ **Correct:**
```clarity
(match (map-get? users tx-sender)
    user (ok user)
    (err u404)
)
```

---

## 🎯 Function Design Best Practices

1. **Single Responsibility**: Each function should do one thing
2. **Guard Clauses First**: Validate inputs at the start
3. **Use Private Helpers**: Break complex logic into smaller functions
4. **Clear Naming**: Function names should describe what they do
5. **Consistent Returns**: Use same error codes for same failures

```clarity
;; Good function design
(define-public (transfer-tokens (recipient principal) (amount uint))
    (begin
        ;; Guards first
        (asserts! (> amount u0) err-invalid-amount)
        (asserts! (not (is-eq recipient tx-sender)) err-self-transfer)
        
        ;; Get data with let
        (let
            (
                (sender-balance (get-balance tx-sender))
                (recipient-balance (get-balance recipient))
            )
            ;; Validate
            (asserts! (>= sender-balance amount) err-insufficient-balance)
            
            ;; Execute
            (try! (update-balance tx-sender (- sender-balance amount)))
            (try! (update-balance recipient (+ recipient-balance amount)))
            
            ;; Log event
            (print { 
                event: "transfer",
                from: tx-sender,
                to: recipient,
                amount: amount
            })
            
            ;; Return
            (ok true)
        )
    )
)
```

---

## 🎓 Practice Exercises

1. Write a function that validates user input with multiple checks
2. Create a function that processes a list of operations
3. Build a state machine with valid transitions
4. Implement a batch operation function

---

## 📚 Next Steps

**Related Guides:**
- → Error Handling Patterns
- → Testing Functions with Clarinet
- → Advanced Contract Patterns