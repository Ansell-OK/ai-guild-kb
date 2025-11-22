# Error Handling - Complete Guide

## 📖 Overview

Proper error handling is crucial for writing secure and maintainable smart contracts. Clarity's response type and error handling patterns help you build robust applications.

### What You'll Learn
- Error codes and messages
- Response types and patterns
- Error propagation
- Best practices for error handling
- Debugging and troubleshooting

---

## 🎯 Understanding Errors in Clarity

### The Response Type

Every public function must return a response:

```clarity
;; Success
(ok value)

;; Error
(err error-code)
```

**Response type annotation:**
```clarity
(response success-type error-type)

;; Examples:
(response bool uint)              ;; Success: bool, Error: uint
(response uint uint)              ;; Success: uint, Error: uint
(response principal (string-ascii 50))  ;; Success: principal, Error: string
```

---

## 🔢 Error Codes

### Defining Error Constants

```clarity
;; Use constants for error codes
(define-constant err-owner-only (err u100))
(define-constant err-not-found (err u101))
(define-constant err-already-exists (err u102))
(define-constant err-insufficient-balance (err u103))
(define-constant err-invalid-amount (err u104))
(define-constant err-not-authorized (err u105))
(define-constant err-contract-paused (err u106))
(define-constant err-invalid-recipient (err u107))
(define-constant err-transfer-failed (err u108))
(define-constant err-expired (err u109))
```

**Why use constants:**
- Self-documenting code
- Easy to update
- Consistent across contract
- Easier to test
- Better for frontend integration

### Error Code Organization

```clarity
;; Group errors by category

;; Authorization errors (100-199)
(define-constant err-owner-only (err u100))
(define-constant err-not-authorized (err u101))
(define-constant err-invalid-caller (err u102))

;; Validation errors (200-299)
(define-constant err-invalid-amount (err u200))
(define-constant err-invalid-recipient (err u201))
(define-constant err-invalid-parameter (err u202))

;; State errors (300-399)
(define-constant err-not-found (err u300))
(define-constant err-already-exists (err u301))
(define-constant err-contract-paused (err u302))

;; Business logic errors (400-499)
(define-constant err-insufficient-balance (err u400))
(define-constant err-transfer-failed (err u401))
(define-constant err-minting-disabled (err u402))
```

---

## 🛠️ Error Handling Patterns

### Pattern 1: asserts!

Check condition, return error if false.

```clarity
;; Basic assertion
(define-public (owner-only-function)
    (begin
        (asserts! (is-eq tx-sender contract-owner) err-owner-only)
        (ok true)
    )
)

;; Multiple assertions
(define-public (transfer (recipient principal) (amount uint))
    (begin
        (asserts! (> amount u0) err-invalid-amount)
        (asserts! (not (is-eq recipient tx-sender)) err-invalid-recipient)
        (asserts! (has-balance tx-sender amount) err-insufficient-balance)
        
        ;; Transfer logic
        (ok true)
    )
)

;; Complex assertions
(asserts! 
    (and
        (> amount min-amount)
        (<= amount max-amount)
        (is-authorized tx-sender)
    )
    err-invalid-amount
)
```

### Pattern 2: unwrap!

Extract value from optional, return error if none.

```clarity
;; Unwrap map value
(define-public (get-user-score)
    (let
        (
            (user-data (unwrap! (map-get? users tx-sender) err-not-found))
        )
        (ok (get score user-data))
    )
)

;; Unwrap with custom error
(define-public (update-listing (listing-id uint))
    (let
        (
            (listing (unwrap! (map-get? listings listing-id) err-not-found))
        )
        (asserts! (is-eq (get owner listing) tx-sender) err-not-authorized)
        (ok true)
    )
)

;; Multiple unwraps
(define-public (complex-operation)
    (let
        (
            (user (unwrap! (map-get? users tx-sender) err-not-found))
            (balance (unwrap! (get-balance tx-sender) err-not-found))
            (settings (unwrap! (map-get? settings tx-sender) err-not-found))
        )
        (ok { user: user, balance: balance, settings: settings })
    )
)
```

### Pattern 3: try!

Unwrap response, propagate error if err.

```clarity
;; Basic try
(define-public (multi-step)
    (begin
        (try! (step-one))
        (try! (step-two))
        (try! (step-three))
        (ok true)
    )
)

;; If step-one returns (err u100), entire function returns (err u100)

;; Try with contract calls
(define-public (forward-call)
    (let
        (
            (result (try! (contract-call? .other-contract do-something)))
        )
        (print { result: result })
        (ok result)
    )
)

;; Try with value extraction
(define-public (process-payment (amount uint))
    (let
        (
            (fee (try! (calculate-fee amount)))
            (net-amount (- amount fee))
        )
        (try! (transfer-tokens net-amount))
        (try! (collect-fee fee))
        (ok net-amount)
    )
)
```

### Pattern 4: match

Handle different response outcomes.

```clarity
;; Match response
(match (some-function)
    success (ok success)
    error (err error)
)

;; Match with different handling
(match (risky-operation)
    success 
        (begin
            (print { success: success })
            (ok success)
        )
    error
        (begin
            (print { error: error })
            (ok u0)  ;; Return default instead of error
        )
)

;; Match with error transformation
(match (external-call)
    success (ok success)
    error (if (is-eq error u404)
        err-not-found
        err-external-failure
    )
)

;; Nested match
(match (map-get? users tx-sender)
    user
        (match (get-balance tx-sender)
            balance (ok { user: user, balance: balance })
            (err u500)
        )
    err-not-found
)
```

### Pattern 5: default-to

Provide default for optional values.

```clarity
;; Basic default
(default-to u0 (map-get? balances user))

;; Default in function
(define-read-only (get-balance (user principal))
    (default-to u0 (map-get? balances user))
)

;; Default for complex types
(define-read-only (get-user-profile (user principal))
    (default-to 
        { name: "Unknown", score: u0, active: false }
        (map-get? profiles user)
    )
)

;; Default with calculation
(let
    (
        (balance (default-to u0 (map-get? balances user)))
        (multiplier (default-to u1 (map-get? multipliers user)))
    )
    (* balance multiplier)
)
```

---

## 🔄 Error Propagation

### Automatic Propagation with try!

```clarity
(define-public (process-order (order-id uint))
    (begin
        ;; If validate returns error, process-order returns that error
        (try! (validate-order order-id))
        
        ;; If charge returns error, process-order returns that error
        (try! (charge-customer order-id))
        
        ;; If fulfill returns error, process-order returns that error
        (try! (fulfill-order order-id))
        
        (ok true)
    )
)

;; Helper functions
(define-private (validate-order (order-id uint))
    (let
        (
            (order (unwrap! (map-get? orders order-id) err-not-found))
        )
        (asserts! (is-eq (get status order) status-pending) err-invalid-state)
        (ok true)
    )
)
```

### Manual Error Handling

```clarity
;; Check and handle errors explicitly
(define-public (careful-operation)
    (match (risky-call)
        success 
            (ok success)
        error 
            (if (is-eq error u404)
                ;; Handle not-found specifically
                (begin
                    (try! (create-new-entry))
                    (ok "Created new")
                )
                ;; Propagate other errors
                (err error)
            )
    )
)
```

### Error Recovery

```clarity
;; Try operation, fall back to alternative
(define-public (transfer-with-fallback (recipient principal) (amount uint))
    (match (direct-transfer recipient amount)
        success (ok success)
        error 
            (begin
                (print { fallback: true, error: error })
                (indirect-transfer recipient amount)
            )
    )
)

;; Retry with different parameters
(define-public (flexible-mint (amount uint))
    (match (mint-exact amount)
        success (ok success)
        error 
            ;; Try with maximum available instead
            (let
                (
                    (max-available (get-max-mintable))
                )
                (mint-exact max-available)
            )
    )
)
```

---

## 📋 Comprehensive Error Handling Example

```clarity
;; Complete example with proper error handling

;; Error definitions
(define-constant err-owner-only (err u100))
(define-constant err-not-found (err u101))
(define-constant err-insufficient-balance (err u102))
(define-constant err-invalid-amount (err u103))
(define-constant err-self-transfer (err u104))
(define-constant err-transfer-failed (err u105))
(define-constant err-paused (err u106))

;; State
(define-map balances principal uint)
(define-data-var paused bool false)
(define-constant contract-owner tx-sender)

;; Main transfer function with comprehensive error handling
(define-public (transfer (recipient principal) (amount uint))
    (begin
        ;; Guard clauses
        (asserts! (not (var-get paused)) err-paused)
        (asserts! (> amount u0) err-invalid-amount)
        (asserts! (not (is-eq recipient tx-sender)) err-self-transfer)
        
        ;; Get balances with error handling
        (let
            (
                (sender-balance (default-to u0 (map-get? balances tx-sender)))
                (recipient-balance (default-to u0 (map-get? balances recipient)))
            )
            ;; Validate balance
            (asserts! (>= sender-balance amount) err-insufficient-balance)
            
            ;; Execute transfer
            (map-set balances tx-sender (- sender-balance amount))
            (map-set balances recipient (+ recipient-balance amount))
            
            ;; Log event
            (print {
                event: "transfer",
                from: tx-sender,
                to: recipient,
                amount: amount
            })
            
            (ok true)
        )
    )
)

;; Helper function with error handling
(define-private (validate-transfer (sender principal) (recipient principal) (amount uint))
    (ok (and
        (not (var-get paused))
        (> amount u0)
        (not (is-eq sender recipient))
        (>= (default-to u0 (map-get? balances sender)) amount)
    ))
)
```

---

## 🐛 Debugging and Troubleshooting

### Using print for Debugging

```clarity
(define-public (debug-transfer (recipient principal) (amount uint))
    (begin
        ;; Print at start
        (print { 
            function: "transfer",
            sender: tx-sender,
            recipient: recipient,
            amount: amount
        })
        
        (let
            (
                (sender-balance (default-to u0 (map-get? balances tx-sender)))
            )
            ;; Print state
            (print { sender-balance: sender-balance })
            
            (asserts! (>= sender-balance amount) err-insufficient-balance)
            
            ;; Print before critical operation
            (print { action: "updating-balances" })
            
            (map-set balances tx-sender (- sender-balance amount))
            
            ;; Print success
            (print { status: "success" })
            
            (ok true)
        )
    )
)
```

### Error Testing Pattern

```clarity
;; Create specific error scenarios for testing
(define-public (test-error-case (case uint))
    (if (is-eq case u1)
        err-owner-only
        (if (is-eq case u2)
            err-not-found
            (if (is-eq case u3)
                err-insufficient-balance
                (ok true)
            )
        )
    )
)
```

---

## ⚠️ Common Mistakes

### 1. **Not Using Error Constants**

❌ **Wrong:**
```clarity
(define-public (do-something)
    (begin
        (asserts! (is-eq tx-sender owner) (err u100))
        (asserts! (> amount u0) (err u100))  ;; Same error code!
        (ok true)
    )
)
```

✅ **Correct:**
```clarity
(define-constant err-not-owner (err u100))
(define-constant err-invalid-amount (err u101))

(define-public (do-something)
    (begin
        (asserts! (is-eq tx-sender owner) err-not-owner)
        (asserts! (> amount u0) err-invalid-amount)
        (ok true)
    )
)
```

### 2. **Ignoring Errors**

❌ **Wrong:**
```clarity
(define-public (chain-operations)
    (begin
        (operation-one)  ;; Error not handled!
        (operation-two)
        (ok true)
    )
)
```

✅ **Correct:**
```clarity
(define-public (chain-operations)
    (begin
        (try! (operation-one))
        (try! (operation-two))
        (ok true)
    )
)
```

### 3. **Using unwrap-panic in Production**

❌ **Wrong:**
```clarity
(define-public (get-user-data)
    (let
        (
            (user (unwrap-panic (map-get? users tx-sender)))  ;; DON'T DO THIS!
        )
        (ok user)
    )
)
```

✅ **Correct:**
```clarity
(define-public (get-user-data)
    (let
        (
            (user (unwrap! (map-get? users tx-sender) err-not-found))
        )
        (ok user)
    )
)
```

### 4. **Generic Error Messages**

❌ **Wrong:**
```clarity
(define-constant err-generic (err u100))  ;; What does this mean?

(define-public (do-many-things)
    (begin
        (asserts! condition1 err-generic)
        (asserts! condition2 err-generic)
        (asserts! condition3 err-generic)
        (ok true)
    )
)
```

✅ **Correct:**
```clarity
(define-constant err-invalid-state (err u100))
(define-constant err-insufficient-funds (err u101))
(define-constant err-unauthorized (err u102))

(define-public (do-many-things)
    (begin
        (asserts! condition1 err-invalid-state)
        (asserts! condition2 err-insufficient-funds)
        (asserts! condition3 err-unauthorized)
        (ok true)
    )
)
```

---

## 📝 Error Documentation Template

```clarity
;; ============================================
;; Error Codes
;; ============================================

;; Authorization Errors (100-199)
;; u100: Caller is not the contract owner
;; u101: Caller is not authorized for this action
;; u102: Invalid caller for this function

;; Validation Errors (200-299)
;; u200: Amount must be greater than zero
;; u201: Recipient address is invalid
;; u202: Parameter value out of range

;; State Errors (300-399)
;; u300: Resource not found
;; u301: Resource already exists
;; u302: Contract is paused

;; Business Logic Errors (400-499)
;; u400: Insufficient balance for operation
;; u401: Transfer operation failed
;; u402: Minting is currently disabled

(define-constant err-owner-only (err u100))
(define-constant err-not-authorized (err u101))
;; ... etc
```

---

## 🎯 Best Practices

1. **Use Descriptive Error Names**: `err-insufficient-balance` not `err-1`
2. **Group Error Codes**: Keep related errors in numerical ranges
3. **Document Errors**: Comment what each error means
4. **Be Specific**: Different errors for different failures
5. **Handle All Cases**: Never leave errors unhandled
6. **Test Error Paths**: Write tests for every error condition
7. **Log Context**: Use print to provide debugging context

---

## 🎓 Practice Exercises

1. Create a comprehensive error handling system for a token contract
2. Write error recovery logic for a marketplace
3. Build error tests for all failure scenarios
4. Document all errors with clear descriptions

---

## 📚 Next Steps

**Related Guides:**
- → Testing Error Conditions with Clarinet
- → Frontend Error Handling
- → Debugging Smart Contracts