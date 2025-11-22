# Clarity Basics - Complete Beginner's Guide

## 📖 Overview

Clarity is a decidable smart contract language for Stacks blockchain. Unlike other smart contract languages, Clarity is interpreted (not compiled) and the code you write is exactly what executes on-chain. This makes it more secure and predictable.

### What You'll Learn
- What makes Clarity unique
- Basic syntax and structure
- Writing your first contract
- Variables and constants
- Basic operations
- Understanding principals (addresses)

### Prerequisites
- Basic programming knowledge
- Understanding of blockchain concepts
- Clarinet installed (optional but recommended)

---

## 🎯 What Makes Clarity Special?

### 1. **Decidable**
You can know, before execution, what a program will do. This prevents infinite loops and unexpected behavior.

### 2. **Interpreted, Not Compiled**
The code you write is the code that runs. No compilation step means no hidden bugs from compiler optimizations.

### 3. **No Reentrancy**
Built-in protection against reentrancy attacks that plague other blockchains.

### 4. **Post Conditions**
Users can specify what should happen in a transaction, protecting against malicious contracts.

### 5. **Optimized for Bitcoin**
Clarity contracts can interact with Bitcoin through Stacks' unique architecture.

---

## 💻 Your First Clarity Contract

### Hello World

```clarity
;; hello-world.clar
;; The simplest Clarity contract

;; Define a public function that returns a string
(define-public (say-hello)
    (ok "Hello, World!")
)

;; Define a read-only function
(define-read-only (get-greeting)
    "Hello, Clarity!"
)
```

**Breaking it down:**

```clarity
;; This is a comment - anything after ;; is ignored
```

```clarity
(define-public (say-hello)
    (ok "Hello, World!")
)
```
- `define-public` - Creates a function that can be called by anyone and writes to the blockchain
- `say-hello` - The function name
- `ok` - Returns a successful response
- Return value is wrapped in `(ok ...)` for public functions

```clarity
(define-read-only (get-greeting)
    "Hello, Clarity!"
)
```
- `define-read-only` - Creates a function that only reads data (doesn't modify state)
- No `ok` wrapper needed for read-only functions
- These functions are free to call (no gas cost)

---

## 📝 Variables and Constants

### Constants

Constants never change after deployment:

```clarity
;; Define constants
(define-constant contract-owner tx-sender)
(define-constant maximum-supply u1000000)
(define-constant app-name "My DApp")
(define-constant fee-percentage u250) ;; 2.5% (250/10000)

;; Using constants
(define-read-only (get-owner)
    contract-owner
)

(define-read-only (get-max-supply)
    maximum-supply
)
```

**Key Points:**
- Use `define-constant` for values that never change
- `tx-sender` is a built-in constant (the transaction sender)
- Constants use kebab-case naming
- No type declaration needed (inferred from value)

### Data Variables

Variables that can change:

```clarity
;; Define mutable variables
(define-data-var counter uint u0)
(define-data-var contract-active bool true)
(define-data-var admin principal tx-sender)

;; Reading variables
(define-read-only (get-counter)
    (var-get counter)
)

;; Writing variables
(define-public (increment-counter)
    (ok (var-set counter (+ (var-get counter) u1)))
)

;; Updating with logic
(define-public (set-active (active bool))
    (begin
        ;; Only admin can change
        (asserts! (is-eq tx-sender (var-get admin)) (err u100))
        (ok (var-set contract-active active))
    )
)
```

**Key Points:**
- Use `var-get` to read a variable
- Use `var-set` to update a variable
- Variables are private by default (need functions to access)
- Always specify the type when defining

---

## 🔢 Understanding Clarity Syntax

### S-Expressions (Lisp-like)

Clarity uses prefix notation (operator comes first):

```clarity
;; Math operations
(+ 1 2)           ;; Returns 3
(- 10 5)          ;; Returns 5
(* 3 4)           ;; Returns 12
(/ 20 5)          ;; Returns 4

;; Nested operations
(+ (* 2 3) (- 10 5))  ;; (2*3) + (10-5) = 11

;; Comparison
(> 10 5)          ;; true
(< 3 7)           ;; true
(is-eq 5 5)       ;; true
```

### Function Calls

```clarity
;; Format: (function-name arg1 arg2 arg3)

;; Built-in functions
(len "hello")                    ;; Returns u5
(concat "hello" " " "world")     ;; Returns "hello world"

;; Your custom functions
(define-private (add-numbers (a uint) (b uint))
    (+ a b)
)

;; Call it
(add-numbers u10 u20)  ;; Returns u30
```

---

## 🎨 Basic Contract Structure

### Complete Template

```clarity
;; Contract: My First Contract
;; Description: A beginner-friendly example

;; ============================================
;; Constants
;; ============================================

(define-constant contract-owner tx-sender)
(define-constant err-owner-only (err u100))
(define-constant err-not-found (err u101))

;; ============================================
;; Data Variables
;; ============================================

(define-data-var total-users uint u0)

;; ============================================
;; Data Maps (key-value storage)
;; ============================================

(define-map users
    principal  ;; Key: user address
    {
        name: (string-ascii 50),
        score: uint,
        registered-at: uint
    }
)

;; ============================================
;; Private Functions (internal use only)
;; ============================================

(define-private (is-owner)
    (is-eq tx-sender contract-owner)
)

;; ============================================
;; Read-Only Functions (free to call)
;; ============================================

(define-read-only (get-total-users)
    (var-get total-users)
)

(define-read-only (get-user (user principal))
    (map-get? users user)
)

;; ============================================
;; Public Functions (costs gas, writes data)
;; ============================================

(define-public (register-user (name (string-ascii 50)))
    (let
        (
            (new-user {
                name: name,
                score: u0,
                registered-at: block-height
            })
        )
        ;; Store user
        (map-set users tx-sender new-user)
        
        ;; Increment counter
        (var-set total-users (+ (var-get total-users) u1))
        
        ;; Return success
        (ok true)
    )
)

(define-public (update-score (new-score uint))
    (let
        (
            (user (unwrap! (map-get? users tx-sender) err-not-found))
        )
        ;; Update user's score
        (ok (map-set users tx-sender
            (merge user { score: new-score })
        ))
    )
)
```

---

## 🔑 Understanding Principals (Addresses)

### What is a Principal?

A principal is a Clarity address - either a wallet address or a contract address.

```clarity
;; Standard principal (wallet address)
'ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM

;; Contract principal (contract address)
'ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM.my-contract

;; tx-sender: Built-in constant for the transaction sender
tx-sender

;; contract-caller: Who called this contract
contract-caller
```

### Working with Principals

```clarity
;; Store a principal
(define-data-var admin principal tx-sender)

;; Check if two principals are equal
(define-read-only (is-admin (user principal))
    (is-eq user (var-get admin))
)

;; Principal in maps
(define-map balances principal uint)

;; Set balance for a principal
(define-public (set-balance (amount uint))
    (ok (map-set balances tx-sender amount))
)

;; Get balance for a principal
(define-read-only (get-balance (user principal))
    (default-to u0 (map-get? balances user))
)
```

### tx-sender vs contract-caller

```clarity
;; tx-sender: The wallet that signed the transaction
;; contract-caller: The immediate caller (could be another contract)

(define-public (example-function)
    (begin
        ;; These might be different if called via another contract
        (print tx-sender)         ;; The original wallet
        (print contract-caller)   ;; Could be a contract or wallet
        (ok true)
    )
)
```

---

## 🧮 Basic Operations

### Arithmetic

```clarity
;; Addition
(+ u10 u20)       ;; u30

;; Subtraction (careful with underflow!)
(- u20 u10)       ;; u10

;; Multiplication
(* u5 u4)         ;; u20

;; Division
(/ u20 u4)        ;; u5

;; Modulo (remainder)
(mod u10 u3)      ;; u1

;; Power
(pow u2 u3)       ;; u8 (2^3)
```

### Comparison

```clarity
;; Greater than
(> u10 u5)        ;; true

;; Less than
(< u5 u10)        ;; true

;; Greater or equal
(>= u10 u10)      ;; true

;; Less or equal
(<= u5 u10)       ;; true

;; Equality (use is-eq for all types)
(is-eq u5 u5)     ;; true
(is-eq "hello" "hello")  ;; true
```

### Boolean Logic

```clarity
;; AND
(and true true)   ;; true
(and true false)  ;; false

;; OR
(or true false)   ;; true
(or false false)  ;; false

;; NOT
(not true)        ;; false
(not false)       ;; true

;; Combining logic
(and (> u10 u5) (< u10 u20))  ;; true
```

---

## 📦 Working with Data Structures

### Tuples (Like Objects)

```clarity
;; Define a tuple
(define-read-only (get-user-info)
    {
        name: "Alice",
        age: u25,
        active: true
    }
)

;; Access tuple values
(define-read-only (get-user-name)
    (get name (get-user-info))  ;; Returns "Alice"
)

;; Tuples in functions
(define-public (create-profile (name (string-ascii 50)) (age uint))
    (let
        (
            (profile {
                name: name,
                age: age,
                created-at: block-height
            })
        )
        (ok profile)
    )
)

;; Merge tuples (update specific fields)
(define-public (example-merge)
    (let
        (
            (original { name: "Bob", age: u30, score: u100 })
            (updated (merge original { score: u150 }))
        )
        ;; updated is now { name: "Bob", age: u30, score: u150 }
        (ok updated)
    )
)
```

### Lists

```clarity
;; Define a list
(define-constant my-numbers (list u1 u2 u3 u4 u5))

;; List functions
(len my-numbers)              ;; u5
(element-at my-numbers u0)    ;; (some u1)
(element-at my-numbers u10)   ;; none

;; Append to list
(append (list u1 u2) u3)      ;; (list u1 u2 u3)

;; Concatenate lists
(concat (list u1 u2) (list u3 u4))  ;; (list u1 u2 u3 u4)

;; Map over list
(map + (list u1 u2 u3) (list u10 u20 u30))  
;; Result: (list u11 u22 u33)

;; Filter list
(filter even? (list u1 u2 u3 u4))  ;; (list u2 u4)
;; Note: even? would need to be defined

;; Fold (reduce) list
(fold + (list u1 u2 u3 u4) u0)  ;; u10 (sum of list)
```

---

## 🎓 Practical Examples

### Example 1: Simple Counter

```clarity
;; counter.clar
;; A simple incrementing counter

(define-data-var counter uint u0)

;; Get current count
(define-read-only (get-count)
    (var-get counter)
)

;; Increment by 1
(define-public (increment)
    (ok (var-set counter (+ (var-get counter) u1)))
)

;; Increment by specified amount
(define-public (increment-by (amount uint))
    (ok (var-set counter (+ (var-get counter) amount)))
)

;; Decrement (with safety check)
(define-public (decrement)
    (let ((current (var-get counter)))
        (asserts! (> current u0) (err u100))
        (ok (var-set counter (- current u1)))
    )
)

;; Reset to zero (owner only)
(define-public (reset)
    (begin
        (asserts! (is-eq tx-sender contract-owner) (err u101))
        (ok (var-set counter u0))
    )
)
```

### Example 2: Simple Storage

```clarity
;; storage.clar
;; Store and retrieve data by key

(define-map storage
    { key: (string-ascii 64) }
    { value: (string-ascii 256), owner: principal }
)

;; Store a value
(define-public (store (key (string-ascii 64)) (value (string-ascii 256)))
    (ok (map-set storage
        { key: key }
        { value: value, owner: tx-sender }
    ))
)

;; Retrieve a value
(define-read-only (retrieve (key (string-ascii 64)))
    (map-get? storage { key: key })
)

;; Update value (only owner can update)
(define-public (update (key (string-ascii 64)) (new-value (string-ascii 256)))
    (let
        (
            (existing (unwrap! (map-get? storage { key: key }) (err u100)))
        )
        ;; Check caller is owner
        (asserts! (is-eq tx-sender (get owner existing)) (err u101))
        
        ;; Update value
        (ok (map-set storage
            { key: key }
            { value: new-value, owner: tx-sender }
        ))
    )
)

;; Delete value (only owner can delete)
(define-public (delete (key (string-ascii 64)))
    (let
        (
            (existing (unwrap! (map-get? storage { key: key }) (err u100)))
        )
        (asserts! (is-eq tx-sender (get owner existing)) (err u101))
        (ok (map-delete storage { key: key }))
    )
)
```

### Example 3: Simple Voting

```clarity
;; voting.clar
;; A basic voting system

(define-map votes
    { proposal-id: uint, voter: principal }
    { vote: bool }  ;; true = yes, false = no
)

(define-map proposal-results
    uint  ;; proposal-id
    { yes-votes: uint, no-votes: uint, total-votes: uint }
)

;; Cast a vote
(define-public (cast-vote (proposal-id uint) (vote bool))
    (let
        (
            (existing-vote (map-get? votes 
                { proposal-id: proposal-id, voter: tx-sender }
            ))
            (results (default-to 
                { yes-votes: u0, no-votes: u0, total-votes: u0 }
                (map-get? proposal-results proposal-id)
            ))
        )
        ;; Check if already voted
        (asserts! (is-none existing-vote) (err u100))
        
        ;; Record vote
        (map-set votes
            { proposal-id: proposal-id, voter: tx-sender }
            { vote: vote }
        )
        
        ;; Update results
        (ok (map-set proposal-results proposal-id
            {
                yes-votes: (if vote (+ (get yes-votes results) u1) (get yes-votes results)),
                no-votes: (if vote (get no-votes results) (+ (get no-votes results) u1)),
                total-votes: (+ (get total-votes results) u1)
            }
        ))
    )
)

;; Get proposal results
(define-read-only (get-results (proposal-id uint))
    (map-get? proposal-results proposal-id)
)

;; Check if user voted
(define-read-only (has-voted (proposal-id uint) (voter principal))
    (is-some (map-get? votes 
        { proposal-id: proposal-id, voter: voter }
    ))
)
```

---

## 🚨 Common Mistakes for Beginners

### 1. **Forgetting to Use `ok` in Public Functions**

❌ **Wrong:**
```clarity
(define-public (do-something)
    "Hello"  ;; Error! Public functions need (ok ...) or (err ...)
)
```

✅ **Correct:**
```clarity
(define-public (do-something)
    (ok "Hello")
)
```

### 2. **Using `=` Instead of `is-eq`**

❌ **Wrong:**
```clarity
(if (= x y)  ;; Error! = doesn't exist in Clarity
    "equal"
    "not equal"
)
```

✅ **Correct:**
```clarity
(if (is-eq x y)
    "equal"
    "not equal"
)
```

### 3. **Not Using `u` Prefix for Unsigned Integers**

❌ **Wrong:**
```clarity
(define-constant max-supply 1000000)  ;; This is a signed int
```

✅ **Correct:**
```clarity
(define-constant max-supply u1000000)  ;; Unsigned int
```

### 4. **Forgetting `var-get` for Variables**

❌ **Wrong:**
```clarity
(define-data-var counter uint u0)

(define-read-only (get-counter)
    counter  ;; Error! Need var-get
)
```

✅ **Correct:**
```clarity
(define-read-only (get-counter)
    (var-get counter)
)
```

### 5. **Not Handling Optional Values**

❌ **Wrong:**
```clarity
(define-public (get-user-age (user principal))
    (let ((user-data (map-get? users user)))
        (ok (get age user-data))  ;; Error if user doesn't exist!
    )
)
```

✅ **Correct:**
```clarity
(define-public (get-user-age (user principal))
    (let ((user-data (unwrap! (map-get? users user) (err u100))))
        (ok (get age user-data))
    )
)
```

---

## 🎯 Quick Reference

### Common Patterns

```clarity
;; Check caller is owner
(asserts! (is-eq tx-sender contract-owner) err-owner-only)

;; Get value with default
(default-to u0 (map-get? balances user))

;; Unwrap or return error
(unwrap! (map-get? data key) err-not-found)

;; Conditional logic
(if (> amount u100)
    (ok "Large amount")
    (ok "Small amount")
)

;; Multiple conditions with begin
(begin
    (asserts! condition1 err1)
    (asserts! condition2 err2)
    (var-set my-var new-value)
    (ok true)
)
```

---

## 🎓 Practice Exercises

Try building these simple contracts:

1. **Guestbook**: Let users store messages
2. **Simple Ledger**: Track balances for users
3. **Poll System**: Create and vote on polls
4. **Name Registry**: Register and lookup names

---

## 📚 Next Steps

Now that you understand the basics:

1. **Learn Data Types** → Next guide covers all Clarity types in detail
2. **Master Functions** → Different function types and patterns
3. **Error Handling** → Proper error management
4. **Testing** → Write tests with Clarinet

**Related Guides:**
- → Data Types Deep Dive
- → Functions and Control Flow
- → Error Handling Patterns
- → Testing with Clarinet