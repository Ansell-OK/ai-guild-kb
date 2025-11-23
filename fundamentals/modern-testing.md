# Modern Testing with Vitest and Clarinet SDK

## 📖 Overview

Modern Clarinet projects use **Vitest** with the **@hirosystems/clarinet-sdk** for testing smart contracts. This guide covers the current recommended approach for testing Clarity contracts.

### What You'll Learn
- Setting up Vitest with Clarinet SDK
- Using the `simnet` API
- Writing tests for different contract types
- Common patterns and best practices
- Differences from legacy Deno-based testing

### Prerequisites
- Node.js and npm installed
- Clarinet CLI installed
- Basic understanding of Clarity syntax

---

## 🚀 Project Setup

### Initialize a New Project

```bash
clarinet new my-project
cd my-project
```

**Modern Clarinet projects automatically include:**
- `package.json` with Vitest and Clarinet SDK dependencies
- `vitest.config.js` configuration
- Pre-configured test environment

### Install Dependencies

```bash
npm install
```

### Verify Setup

Your `package.json` should include:

```json
{
  "scripts": {
    "test": "vitest run",
    "test:watch": "vitest"
  },
  "devDependencies": {
    "@hirosystems/clarinet-sdk": "^2.0.0",
    "@stacks/transactions": "^6.0.0",
    "vitest": "^2.0.0"
  }
}
```

---

## 🧪 Writing Tests

### Test File Structure

**Naming Convention**: `contract-name.test.ts` (not `_test.ts`)

**Location**: `tests/` directory

**Basic Template**:

```typescript
import { describe, expect, it } from "vitest";
import { Cl } from "@stacks/transactions";

const accounts = simnet.getAccounts();
const deployer = accounts.get("deployer")!;
const wallet1 = accounts.get("wallet_1")!;
const wallet2 = accounts.get("wallet_2")!;

describe("My Contract", () => {
  it("test description", () => {
    // Test logic here
  });
});
```

---

## 🔧 Simnet API Reference

### Get Accounts

```typescript
const accounts = simnet.getAccounts();
const deployer = accounts.get("deployer")!;
const wallet1 = accounts.get("wallet_1")!;
```

**Available accounts:**
- `deployer` - Contract deployer
- `wallet_1` through `wallet_9` - Test wallets

### Call Public Functions

```typescript
const { result } = simnet.callPublicFn(
  "contract-name",    // Contract name (from Clarinet.toml)
  "function-name",    // Function to call
  [Cl.uint(100)],     // Arguments (Clarity values)
  deployer            // Caller address
);

// Check result
expect(result).toBeOk(Cl.uint(100));
```

### Call Read-Only Functions

```typescript
const { result } = simnet.callReadOnlyFn(
  "contract-name",
  "get-balance",
  [Cl.principal(wallet1)],
  deployer
);

expect(result).toBeUint(0);
```

### Get Current Block Height

```typescript
const height = simnet.blockHeight;
console.log(`Current block: ${height}`);
```

### Mine Blocks

```typescript
simnet.mineEmptyBlocks(10); // Advance 10 blocks
```

---

## 📝 Clarity Value Helpers

Use `Cl` from `@stacks/transactions` to create Clarity values:

```typescript
import { Cl } from "@stacks/transactions";

// Numbers
Cl.uint(100)           // Unsigned integer
Cl.int(-50)            // Signed integer

// Strings
Cl.stringAscii("hello")
Cl.stringUtf8("👋")

// Principals (addresses)
Cl.principal("ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM")

// Booleans
Cl.bool(true)

// Optionals
Cl.some(Cl.uint(42))
Cl.none()

// Tuples
Cl.tuple({
  amount: Cl.uint(1000),
  sender: Cl.principal(wallet1)
})

// Lists
Cl.list([Cl.uint(1), Cl.uint(2), Cl.uint(3)])

// Buffers
Cl.bufferFromHex("0x1234")
```

---

## ✅ Assertions

### Response Types

```typescript
// Success responses
expect(result).toBeOk(Cl.uint(100));
expect(result).toBeOk(Cl.bool(true));

// Error responses
expect(result).toBeErr(Cl.uint(404));
```

### Optional Types

```typescript
// Some value
expect(result).toBeSome(Cl.uint(42));

// None
expect(result).toBeNone();
```

### Basic Types

```typescript
expect(result).toBeUint(100);
expect(result).toBeBool(true);
expect(result).toBePrincipal("ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM");
```

---

## 📚 Complete Examples

### Example 1: Simple Counter

**Contract**: `simple-counter.clar`

```clarity
(define-data-var counter uint u0)

(define-read-only (get-count)
  (var-get counter)
)

(define-public (increment)
  (ok (var-set counter (+ (var-get counter) u1)))
)

(define-public (reset)
  (ok (var-set counter u0))
)
```

**Test**: `simple-counter.test.ts`

```typescript
import { describe, expect, it } from "vitest";
import { Cl } from "@stacks/transactions";

const accounts = simnet.getAccounts();
const deployer = accounts.get("deployer")!;

describe("Simple Counter", () => {
  it("starts at zero", () => {
    const { result } = simnet.callReadOnlyFn(
      "simple-counter",
      "get-count",
      [],
      deployer
    );
    expect(result).toBeUint(0);
  });

  it("can increment", () => {
    const { result } = simnet.callPublicFn(
      "simple-counter",
      "increment",
      [],
      deployer
    );
    expect(result).toBeOk(Cl.bool(true));

    const count = simnet.callReadOnlyFn(
      "simple-counter",
      "get-count",
      [],
      deployer
    );
    expect(count.result).toBeUint(1);
  });

  it("can reset", () => {
    // Increment first
    simnet.callPublicFn("simple-counter", "increment", [], deployer);
    
    // Then reset
    const { result } = simnet.callPublicFn(
      "simple-counter",
      "reset",
      [],
      deployer
    );
    expect(result).toBeOk(Cl.bool(true));

    const count = simnet.callReadOnlyFn(
      "simple-counter",
      "get-count",
      [],
      deployer
    );
    expect(count.result).toBeUint(0);
  });
});
```

### Example 2: Token Transfer

**Contract**: `my-token.clar`

```clarity
(define-map balances principal uint)

(define-public (transfer (recipient principal) (amount uint))
  (let ((sender-balance (default-to u0 (map-get? balances tx-sender))))
    (asserts! (>= sender-balance amount) (err u100))
    (map-set balances tx-sender (- sender-balance amount))
    (map-set balances recipient 
      (+ (default-to u0 (map-get? balances recipient)) amount))
    (ok true)
  )
)

(define-read-only (get-balance (account principal))
  (default-to u0 (map-get? balances account))
)
```

**Test**: `my-token.test.ts`

```typescript
import { describe, expect, it } from "vitest";
import { Cl } from "@stacks/transactions";

const accounts = simnet.getAccounts();
const deployer = accounts.get("deployer")!;
const wallet1 = accounts.get("wallet_1")!;
const wallet2 = accounts.get("wallet_2")!;

describe("Token Transfer", () => {
  it("can transfer tokens", () => {
    // Setup: give wallet1 some tokens
    simnet.callPublicFn(
      "my-token",
      "mint",
      [Cl.principal(wallet1), Cl.uint(1000)],
      deployer
    );

    // Transfer from wallet1 to wallet2
    const { result } = simnet.callPublicFn(
      "my-token",
      "transfer",
      [Cl.principal(wallet2), Cl.uint(500)],
      wallet1
    );
    expect(result).toBeOk(Cl.bool(true));

    // Verify balances
    const balance1 = simnet.callReadOnlyFn(
      "my-token",
      "get-balance",
      [Cl.principal(wallet1)],
      deployer
    );
    expect(balance1.result).toBeUint(500);

    const balance2 = simnet.callReadOnlyFn(
      "my-token",
      "get-balance",
      [Cl.principal(wallet2)],
      deployer
    );
    expect(balance2.result).toBeUint(500);
  });

  it("fails with insufficient balance", () => {
    const { result } = simnet.callPublicFn(
      "my-token",
      "transfer",
      [Cl.principal(wallet2), Cl.uint(999999)],
      wallet1
    );
    expect(result).toBeErr(Cl.uint(100));
  });
});
```

### Example 3: NFT Minting

**Test**: `nft-minter.test.ts`

```typescript
import { describe, expect, it } from "vitest";
import { Cl } from "@stacks/transactions";

const accounts = simnet.getAccounts();
const deployer = accounts.get("deployer")!;
const wallet1 = accounts.get("wallet_1")!;

describe("NFT Minter", () => {
  it("can mint NFT", () => {
    const { result } = simnet.callPublicFn(
      "nft-minter",
      "mint",
      [Cl.principal(wallet1)],
      deployer
    );
    expect(result).toBeOk(Cl.uint(1)); // Returns token ID

    // Verify ownership
    const owner = simnet.callReadOnlyFn(
      "nft-minter",
      "get-owner",
      [Cl.uint(1)],
      deployer
    );
    expect(owner.result).toBeOk(
      Cl.some(Cl.principal(wallet1))
    );
  });

  it("only owner can mint", () => {
    const { result } = simnet.callPublicFn(
      "nft-minter",
      "mint",
      [Cl.principal(wallet1)],
      wallet1 // Non-owner trying to mint
    );
    expect(result).toBeErr(Cl.uint(100));
  });
});
```

---

## 🎯 Best Practices

### 1. State Resets Between Tests

**Important**: Simnet resets state between each test (`it` block).

```typescript
describe("Counter", () => {
  it("test 1", () => {
    simnet.callPublicFn("counter", "increment", [], deployer);
    // Counter is now 1
  });

  it("test 2", () => {
    // Counter is back to 0 (state reset)
    const { result } = simnet.callReadOnlyFn(
      "counter",
      "get-count",
      [],
      deployer
    );
    expect(result).toBeUint(0); // Passes!
  });
});
```

### 2. Setup Data in Each Test

```typescript
it("can transfer", () => {
  // Always set up state explicitly
  simnet.callPublicFn("nft", "mint", [Cl.principal(wallet1)], deployer);
  
  // Now test transfer
  const { result } = simnet.callPublicFn(
    "nft",
    "transfer",
    [Cl.uint(1), Cl.principal(wallet1), Cl.principal(wallet2)],
    wallet1
  );
  expect(result).toBeOk(Cl.bool(true));
});
```

### 3. Test Error Cases

```typescript
it("prevents unauthorized access", () => {
  const { result } = simnet.callPublicFn(
    "vault",
    "admin-function",
    [],
    wallet1 // Not admin
  );
  expect(result).toBeErr(Cl.uint(100)); // err-not-authorized
});
```

### 4. Use Descriptive Test Names

```typescript
// Good ✅
it("prevents double voting on the same proposal", () => { ... });

// Bad ❌
it("test 1", () => { ... });
```

---

## ⚠️ Common Issues

### Issue 1: Contract Not Found

**Error**: `Contract not found`

**Solution**: Ensure contract is registered in `Clarinet.toml`:

```toml
[contracts.my-contract]
path = 'contracts/my-contract.clar'
clarity_version = 3
epoch = 'latest'
```

### Issue 2: Line Endings (Windows)

**Error**: `unsupported line-ending '\r'`

**Solution**: Clarity requires LF line endings (not CRLF). On Windows, convert files:

**PowerShell:**
```powershell
$path = "contracts\my-contract.clar"
$content = [IO.File]::ReadAllText($path)
[IO.File]::WriteAllText($path, $content.Replace("`r`n", "`n"))
```

**VS Code:**
1. Open the file
2. Click "CRLF" in bottom-right status bar
3. Select "LF"

**Git Configuration:**
```bash
# Prevent automatic CRLF conversion
git config core.autocrlf false
```

### Issue 3: Test File Not Found

**Error**: `No test files found`

**Solution**: Use `.test.ts` extension (not `_test.ts`):
- ✅ `my-contract.test.ts`
- ❌ `my-contract_test.ts`

### Issue 4: Wrong Block Height Function

**Error**: `use of unresolved name 'block-height'`

**Solution**: Use `stacks-block-height` (not `block-height`):

```clarity
// Correct ✅
(define-public (create-escrow (expiry-blocks uint))
  (let ((expiry (+ stacks-block-height expiry-blocks)))
    ...
  )
)

// Wrong ❌
(let ((expiry (+ block-height expiry-blocks))) ... )
```

---

## 🔄 Differences from Legacy Testing

If you've used older Clarinet documentation, here are the key changes:

| Legacy (Deno) | Modern (Vitest) |
|---------------|-----------------|
| `Clarinet.test()` | `describe()` + `it()` |
| Deno imports | npm packages |
| `chain.mineBlock()` | `simnet.callPublicFn()` |
| `types.uint()` | `Cl.uint()` |
| `_test.ts` files | `.test.ts` files |
| `clarinet test` command | `npm test` command |

**Migration Example:**

**Legacy Deno:**
```typescript
import { Clarinet, Tx, Chain, types } from 'clarinet';

Clarinet.test({
  name: "Can increment",
  async fn(chain: Chain, accounts: Map<string, Account>) {
    let block = chain.mineBlock([
      Tx.contractCall('counter', 'increment', [], accounts.get('deployer')!)
    ]);
    block.receipts[0].result.expectOk();
  }
});
```

**Modern Vitest:**
```typescript
import { describe, expect, it } from "vitest";
import { Cl } from "@stacks/transactions";

const accounts = simnet.getAccounts();
const deployer = accounts.get("deployer")!;

describe("Counter", () => {
  it("can increment", () => {
    const { result } = simnet.callPublicFn(
      "counter",
      "increment",
      [],
      deployer
    );
    expect(result).toBeOk(Cl.bool(true));
  });
});
```

---

## 🚀 Running Tests

### Run All Tests

```bash
npm test
```

### Watch Mode

```bash
npm run test:watch
```

### Run Specific File

```bash
npx vitest run tests/my-contract.test.ts
```

### Verbose Output

```bash
npm test -- --reporter=verbose
```

---

## 📦 Complete Project Example

See the [Clarity Project Suite](https://github.com/Ansell-OK/ai-guild-kb/tree/main/examples/clarity-suite) for a complete working example with:
- Simple Counter
- Voting System
- NFT Minter (SIP-009)
- Escrow Service

All contracts include comprehensive test coverage using Vitest and Clarinet SDK.

---

## 📚 Additional Resources

- [Clarinet Documentation](https://docs.hiro.so/clarinet)
- [Vitest Documentation](https://vitest.dev)
- [Stacks Transactions Library](https://github.com/hirosystems/stacks.js)
- [Clarity Language Reference](https://docs.stacks.co/clarity)

---

## 🎓 Next Steps

**Related Guides:**
- → [Clarity Basics](clarity-basics.md)
- → [Functions and Control Flow](functions-and-control-flow.md)
- → [Data Types](data-types.md)
- → [SIP-009 NFT Implementation](../tokens/sip009-nfts.md)
