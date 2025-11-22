# Clarity Data Types - Complete Guide

## 📖 Overview

Clarity is a strongly-typed language, meaning every value has a specific type. Understanding types is crucial for writing correct and secure smart contracts.

### What You'll Learn
- All Clarity data types
- When to use each type
- Type conversion
- Common patterns and pitfalls
- Working with optional and response types

---

## 🔢 Numeric Types

### Unsigned Integers (uint)

Non-negative whole numbers (0, 1, 2, 3, ...).

```clarity
;; Define unsigned integers with 'u' prefix
(define-constant max-supply u1000000)
(define-data-var counter uint u0)

;; Common uses
u0          ;; Zero
u1          ;; One
u100        ;; One hundred
u1000000    ;; One million (useful for token amounts with decimals)

;; Range: 0 to 2^128 - 1
;; Maximum value
u340282366920938463463374607431768211455
```

**When to use uint:**
- Token amounts
- Counts and IDs
- Block heights
- Timestamps
- Anything that can't be negative

**Common operations:**
```clarity
(+ u10 u20)         ;; u30
(- u20 u10)         ;; u10
(* u5 u4)           ;; u20
(/ u20 u4)          ;; u5
(pow u2 u8)         ;; u256
(mod u10 u3)        ;; u1
```

### Signed Integers (int)

Can be positive or negative.

```clarity
;; Define signed integers (no prefix or use 'i')
(define-constant temperature -5)
(define-constant profit 1000)

;; Examples
-100    ;; Negative one hundred
0       ;; Zero (can be int or uint depending on context)
42      ;; Positive forty-two
```

**When to use int:**
- Rarely needed in smart contracts
- Financial calculations with debits/credits
- Coordinate systems
- Percentage changes

**Range:** -2^127 to 2^127 - 1

---

## 📝 String Types

### ASCII Strings

Limited to ASCII characters (a-z, A-Z, 0-9, common symbols).

```clarity
;; Define ASCII strings
(define-constant app-name "My DApp")
(define-constant error-message "Invalid input")

;; Type annotation
(define-data-var username (string-ascii 50) "")

;; Maximum length is part of the type
(string-ascii 10)   ;; Max 10 characters
(string-ascii 256)  ;; Max 256 characters
```

**When to use:**
- Usernames
- Error messages
- Simple labels and identifiers
- URLs
- Contract names

**String operations:**
```clarity
;; Concatenate
(concat "Hello" " " "World")  ;; "Hello World"

;; Get length
(len "Hello")  ;; u5

;; Get substring (from-index, to-index)
(slice? "Hello World" u0 u5)  ;; (some "Hello")
(slice? "Hello World" u6 u11) ;; (some "World")
```

### UTF-8 Strings

Support international characters and emojis.

```clarity
;; UTF-8 strings
(define-constant greeting u"Hello 👋")
(define-constant japanese u"こんにちは")

;; Type annotation
(string-utf8 100)  ;; Max 100 UTF-8 characters
```

**When to use:**
- User-generated content
- International applications
- Emojis and special characters

**Note:** UTF-8 strings are more expensive (gas cost) than ASCII strings.

---

## 🔡 Buffer Type

Raw bytes, useful for hashes and binary data.

```clarity
;; Hex notation with 0x prefix
(define-constant hash 0x1234567890abcdef)
(define-constant empty-buffer 0x)

;; Type annotation
(buff 32)  ;; 32-byte buffer (common for hashes)
(buff 64)  ;; 64-byte buffer
```

**Common uses:**
- Cryptographic hashes (SHA-256, HASH160)
- Signatures
- Raw binary data
- Merkle roots

**Buffer operations:**
```clarity
;; Concatenate buffers
(concat 0x1234 0x5678)  ;; 0x12345678

;; Get length
(len 0x1234)  ;; u2

;; Convert to uint (for small buffers)
(buff-to-uint-be 0x0001)  ;; u1
(buff-to-uint-le 0x0100)  ;; u1
```

---

## ✅ Boolean Type

True or false values.

```clarity
;; Boolean values
true
false

;; Using booleans
(define-data-var is-active bool true)
(define-data-var is-paused bool false)

;; Boolean operations
(and true true)         ;; true
(or false true)         ;; true
(not true)             ;; false

;; Comparisons return booleans
(> u10 u5)             ;; true
(is-eq "a" "a")        ;; true
```

**Common patterns:**
```clarity
;; Toggle a boolean
(define-public (toggle-active)
    (ok (var-set is-active (not (var-get is-active))))
)

;; Conditional logic
(if (var-get is-active)
    (ok "Active")
    (ok "Inactive")
)

;; Guard clauses
(asserts! (var-get is-active) (err u100))
```

---

## 👤 Principal Type

Represents addresses (wallets or contracts).

```clarity
;; Standard principals (wallet addresses)
'ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM

;; Contract principals (contract addresses)
'ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM.my-contract

;; Built-in principals
tx-sender           ;; Transaction sender
contract-caller     ;; Immediate caller
```

**Using principals:**
```clarity
;; Store principals
(define-data-var admin principal tx-sender)
(define-constant contract-owner tx-sender)

;; Map with principal keys
(define-map balances principal uint)

;; Compare principals
(is-eq tx-sender (var-get admin))

;; Functions with principal parameters
(define-public (transfer (recipient principal) (amount uint))
    (ok true)
)
```

**Principal operations:**
```clarity
;; Check if principal is standard (not contract)
(is-standard tx-sender)  ;; true or false

;; Get principal from contract
(as-contract tx-sender)  ;; Returns contract's principal
```

---

## 📦 Tuple Type

Collections of named values (like objects/structs).

```clarity
;; Define a tuple
{
    name: "Alice",
    age: u25,
    active: true
}

;; Tuple type annotation
{
    name: (string-ascii 50),
    age: uint,
    active: bool
}

;; Using tuples in maps
(define-map users
    principal  ;; key
    {
        name: (string-ascii 50),
        balance: uint,
        registered: uint
    }
)
```

**Tuple operations:**
```clarity
;; Get value from tuple
(get name { name: "Bob", age: u30 })  ;; "Bob"

;; Merge tuples (update specific fields)
(merge 
    { name: "Alice", age: u25, score: u100 }
    { score: u150 }
)
;; Result: { name: "Alice", age: u25, score: u150 }

;; Create tuple in function
(define-public (create-user (name (string-ascii 50)))
    (let
        (
            (user-data {
                name: name,
                balance: u0,
                registered: block-height
            })
        )
        (map-set users tx-sender user-data)
        (ok user-data)
    )
)
```

---

## 📋 List Type

Ordered collections of items (all same type).

```clarity
;; Define lists
(list u1 u2 u3 u4 u5)
(list "apple" "banana" "cherry")
(list true false true)

;; Empty list
(list)

;; Type annotation with max length
(list 10 uint)           ;; List of up to 10 uints
(list 5 principal)       ;; List of up to 5 principals
(list 20 (string-ascii 50))  ;; List of up to 20 strings
```

**List operations:**
```clarity
;; Get length
(len (list u1 u2 u3))  ;; u3

;; Access element by index (0-based)
(element-at (list u10 u20 u30) u0)  ;; (some u10)
(element-at (list u10 u20 u30) u5)  ;; none

;; Append element
(append (list u1 u2) u3)  ;; (list u1 u2 u3)

;; Concatenate lists
(concat (list u1 u2) (list u3 u4))  ;; (list u1 u2 u3 u4)

;; Map over list
(map + 
    (list u1 u2 u3) 
    (list u10 u20 u30)
)  ;; (list u11 u22 u33)

;; Filter list
(filter not (list true false true false))  ;; (list false false)

;; Fold (reduce) list
(fold + (list u1 u2 u3 u4) u0)  ;; u10
```

**Practical example:**
```clarity
;; Store a list of authorized users
(define-data-var authorized-users (list 100 principal) (list))

;; Add user to list
(define-public (add-authorized-user (user principal))
    (ok (var-set authorized-users 
        (append (var-get authorized-users) user)
    ))
)

;; Check if user is in list
(define-private (is-authorized (user principal))
    (is-some (index-of (var-get authorized-users) user))
)
```

---

## ❓ Optional Type

Represents a value that might or might not exist.

```clarity
;; Optional values
(some u10)      ;; Has a value: 10
none           ;; No value

;; Type annotation
(optional uint)
(optional principal)
(optional (string-ascii 50))
```

**When you get optionals:**
- `map-get?` returns optional
- `element-at` returns optional
- `index-of` returns optional

**Working with optionals:**
```clarity
;; Get value with default
(default-to u0 (some u10))  ;; u10
(default-to u0 none)        ;; u0

;; Check if has value
(is-some (some u10))  ;; true
(is-none none)        ;; true

;; Unwrap or error
(unwrap! (some u10) (err u404))  ;; u10
(unwrap! none (err u404))        ;; Returns (err u404)

;; Unwrap with panic (use carefully!)
(unwrap-panic (some u10))  ;; u10
(unwrap-panic none)        ;; Contract execution fails!
```

**Practical example:**
```clarity
(define-map user-scores principal uint)

(define-public (get-user-score (user principal))
    (ok (default-to u0 (map-get? user-scores user)))
)

(define-public (update-score (new-score uint))
    (let
        (
            ;; Get existing score or error if doesn't exist
            (current (unwrap! (map-get? user-scores tx-sender) (err u404)))
        )
        (ok (map-set user-scores tx-sender new-score))
    )
)
```

---

## ✨ Response Type

Represents success or failure (like Result in Rust).

```clarity
;; Success
(ok u100)
(ok "Success!")
(ok true)

;; Error
(err u404)
(err "Not found")

;; Type annotation
(response uint uint)           ;; Success: uint, Error: uint
(response bool (string-ascii 50))  ;; Success: bool, Error: string
```

**All public functions must return a response:**
```clarity
;; Correct
(define-public (do-something)
    (ok true)
)

;; Wrong - won't compile
(define-public (do-something)
    true
)
```

**Working with responses:**
```clarity
;; Unwrap response or return error
(try! (some-function))

;; Match response
(match (some-function)
    success (ok success)
    error (err error)
)

;; Unwrap with panic (testing only!)
(unwrap-panic (ok u10))
(unwrap! (ok u10) (err u999))
```

**Practical patterns:**
```clarity
;; Chain operations
(define-public (complex-operation)
    (begin
        (try! (step-one))
        (try! (step-two))
        (try! (step-three))
        (ok true)
    )
)

;; If any step returns (err ...), entire function returns that error

;; Handle different errors
(define-public (safe-operation)
    (match (risky-operation)
        success (ok success)
        error (if (is-eq error u404)
            (ok "Not found, created new")
            (err error)
        )
    )
)
```

---

## 🔄 Type Conversion

### String to Buffer
```clarity
(to-consensus-buff? "Hello")  ;; Converts to buffer
```

### Integer Conversions
```clarity
;; Unsigned to signed (if fits)
(to-int u100)  ;; 100

;; Signed to unsigned (if positive)
(to-uint 100)  ;; u100
(to-uint -50)  ;; Fails at runtime
```

### Buffer to Integer
```clarity
;; Big-endian
(buff-to-uint-be 0x0064)  ;; u100

;; Little-endian
(buff-to-uint-le 0x6400)  ;; u100
```

### Integer to Buffer
```clarity
;; Unsigned int to buffer
(uint-to-buff u100)  ;; 0x0000000000000064 (16 bytes)
```

---

## 📊 Type Annotations in Practice

### Function Parameters
```clarity
(define-public (transfer 
    (recipient principal)
    (amount uint)
    (memo (optional (buff 34)))
)
    (ok true)
)
```

### Map Definitions
```clarity
(define-map nft-owners
    uint  ;; token-id
    principal  ;; owner
)

(define-map complex-data
    { user: principal, id: uint }  ;; Tuple key
    {  ;; Tuple value
        balance: uint,
        name: (string-ascii 50),
        active: bool
    }
)
```

### Variable Definitions
```clarity
(define-data-var counter uint u0)
(define-data-var name (string-ascii 100) "")
(define-data-var users (list 100 principal) (list))
```

---

## ⚠️ Common Type Mistakes

### 1. **Mixing int and uint**
❌ **Wrong:**
```clarity
(+ u10 5)  ;; Error! Can't mix types
```
✅ **Correct:**
```clarity
(+ u10 u5)
```

### 2. **String length mismatch**
❌ **Wrong:**
```clarity
(define-data-var name (string-ascii 10) "This is a very long name")
;; Error! String too long
```
✅ **Correct:**
```clarity
(define-data-var name (string-ascii 50) "This is a very long name")
```

### 3. **Not handling optionals**
❌ **Wrong:**
```clarity
(define-read-only (get-balance (user principal))
    (map-get? balances user)  ;; Returns optional, not uint!
)
```
✅ **Correct:**
```clarity
(define-read-only (get-balance (user principal))
    (default-to u0 (map-get? balances user))
)
```

### 4. **Forgetting response wrapper**
❌ **Wrong:**
```clarity
(define-public (set-value (val uint))
    (var-set my-var val)  ;; Returns bool, not response!
)
```
✅ **Correct:**
```clarity
(define-public (set-value (val uint))
    (ok (var-set my-var val))
)
```

---

## 🎯 Type Selection Cheatsheet

| Use Case | Type | Example |
|----------|------|---------|
| Token amounts | `uint` | `u1000000` |
| NFT IDs | `uint` | `u1, u2, u3` |
| Usernames | `string-ascii` | `"alice"` |
| Descriptions | `string-utf8` | `u"Hello 👋"` |
| Addresses | `principal` | `tx-sender` |
| Hash values | `buff` | `0x1234...` |
| Flags/states | `bool` | `true/false` |
| User data | `tuple` | `{ name: ..., age: ... }` |
| Multiple items | `list` | `(list u1 u2 u3)` |
| Maybe exists | `optional` | `(some ...)` or `none` |
| Success/fail | `response` | `(ok ...)` or `(err ...)` |

---

## 🎓 Practice Exercises

1. Create a map that stores user profiles with name, age, and bio
2. Write a function that takes a list of numbers and returns their sum
3. Create a function that safely retrieves a value from a map with a default
4. Build a type-safe token transfer function

---

## 📚 Next Steps

**Related Guides:**
- → Functions and Control Flow
- → Error Handling Patterns
- → Working with Maps and Storage