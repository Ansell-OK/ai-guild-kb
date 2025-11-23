# Common Errors & Troubleshooting Guide

## 📖 Overview

This guide covers the most common errors encountered when developing Clarity smart contracts and their solutions. Based on real-world issues from building production contracts.

---

## 🔴 Clarity Compilation Errors

### 1. Line Ending Errors (Windows)

**Error**:
```
error: unsupported line-ending '\r'
```

**Cause**: Clarity requires LF (Line Feed) line endings, not CRLF (Carriage Return + Line Feed) used by Windows.

**Solution - PowerShell**:
```powershell
$path = "contracts\my-contract.clar"
$content = [IO.File]::ReadAllText($path)
[IO.File]::WriteAllText($path, $content.Replace("`r`n", "`n"))
```

**Solution - VS Code**:
1. Open the file
2. Click "CRLF" in bottom-right status bar
3. Select "LF"
4. Save file

**Solution - Git (Prevention)**:
```bash
# Prevent automatic CRLF conversion
git config core.autocrlf false
```

Add to `.gitattributes`:
```
*.clar text eol=lf
```

---

### 2. Undefined Function: block-height

**Error**:
```
error: use of unresolved name 'block-height'
```

**Wrong Code**:
```clarity
(define-public (create-escrow (expiry-blocks uint))
  (let ((expiry (+ block-height expiry-blocks)))  ;; ❌ Wrong
    (ok expiry)
  )
)
```

**Correct Code**:
```clarity
(define-public (create-escrow (expiry-blocks uint))
  (let ((expiry (+ stacks-block-height expiry-blocks)))  ;; ✅ Correct
    (ok expiry)
  )
)
```

**Explanation**: In Clarity, the correct built-in is `stacks-block-height`, not `block-height`.

---

### 3. Contract Not Found in Tests

**Error**:
```
Contract not found: my-contract
```

**Cause**: Contract not registered in `Clarinet.toml`

**Solution**: Add contract to `Clarinet.toml`:
```toml
[contracts.my-contract]
path = 'contracts/my-contract.clar'
clarity_version = 3
epoch = 'latest'
```

**For contracts with dependencies**:
```toml
[contracts.nft-minter]
path = 'contracts/nft-minter.clar'
depends_on = ['sip009-nft-trait']  # ← Critical!
clarity_version = 3
```

---

##🧪 Testing Errors

### 4. Wrong var-set Return Value

**Error**: Test fails expecting value, but gets boolean

**Wrong Test**:
```typescript
const { result } = simnet.callPublicFn(
  "counter",
  "increment",
  [],
  deployer
);
expect(result).toBeOk(Cl.uint(1));  // ❌ Fails!
```

**Correct Test**:
```typescript
const { result } = simnet.callPublicFn(
  "counter",
  "increment",
  [],
  deployer
);
expect(result).toBeOk(Cl.bool(true));  // ✅ Correct
```

**Explanation**: `var-set` returns `(ok true)` on success, not the new value.

---

### 5. Test State Not Resetting

**Problem**: Tests fail because state persists from previous test

**Wrong Approach**:
```typescript
it("increments counter", () => {
  simnet.callPublicFn("counter", "increment", [], deployer);
  // Counter is now 1
});

it("decrements counter", () => {
  // ❌ Expects counter to still be 1, but it's 0!
  simnet.callPublicFn("counter", "decrement", [], deployer);
});
```

**Correct Approach**:
```typescript
it("decrements counter", () => {
  // ✅ Set up state explicitly in each test
  simnet.callPublicFn("counter", "increment", [], deployer);
  
  const { result } = simnet.callPublicFn(
    "counter",
    "decrement",
    [],
    deployer
  );
  expect(result).toBeOk(Cl.bool(true));
});
```

**Explanation**: Simnet resets state between each `it()` block. Always set up required state within the test.

---

### 6. Test File Not Found

**Error**:
```
No test files found
```

**Cause**: Wrong file naming convention

**Wrong**: `my-contract_test.ts`
**Correct**: `my-contract.test.ts`

Vitest looks for `.test.ts` or `.spec.ts` files.

---

## 🌐 Frontend/Network Errors

### 7. Wrong Network Import

**Error**:
```typescript
'@stacks/network' has no exported member named 'StacksTestnet'
```

**Wrong Code**:
```typescript
import { StacksTestnet } from "@stacks/network";
const network = new StacksTestnet();  // ❌ Doesn't exist
```

**Correct Code**:
```typescript
import { STACKS_TESTNET } from "@stacks/network";
const network = STACKS_TESTNET;  // ✅ Correct
```

**Alternative (for custom nodes**:
```typescript
import { StacksNetwork } from "@stacks/network";
const network = new StacksNetwork({
  url: "https://api.testnet.hiro.so",
  chainId: ChainID.Testnet
});
```

---

### 8. showConnect is not a function

**Error**:
```
showConnect is not a function
```

**Wrong Code**:
```typescript
import { showConnect } from "@stacks/connect";
showConnect({ ... });  // ❌ Old API
```

**Correct Code**:
```typescript
import { connect } from "@stacks/connect";
await connect({ ... });  // ✅ Current API
```

**See**: [Stacks Wallet Authentication Guide](../wallets/stacks-connect-authentication.md)

---

### 9. Contract Call Fails Silently

**Problem**: `openContractCall` doesn't show errors

**Better Error Handling**:
```typescript
try {
  await openContractCall({
    network: STACKS_TESTNET,
    contractAddress: "ST...",
    contractName: "my-contract",
    functionName: "transfer",
    functionArgs: [Cl.uint(100)],
    onFinish: (data) => {
      console.log("Success:", data.txId);
    },
    onCancel: () => {
      console.log("User cancelled");
    },
  });
} catch (error) {
  console.error("Contract call failed:", error);
  alert("Transaction failed. Please try again.");
}
```

---

## 🚀 Deployment Errors

### 10. Invalid Mnemonic

**Error**:
```
mnemonic has an invalid word count: 5. 
Word count must be 12, 15, 18, 21, or 24
```

**Cause**: Invalid or incomplete seed phrase in `settings/Testnet.toml`

**Solution**: Use a valid 12 or 24-word mnemonic:
```toml
[accounts.deployer]
mnemonic = "board list obtain sugar hour worth raven scout denial thunder horse logic fury scorpion fold genuine phrase wealth news aim below celery when cabin"
```

**Generate new mnemonic**:
```bash
# Using Clarinet
clarinet deployments generate --testnet --low-cost
```

---

### 11. Cost Strategy Not Specified

**Error**:
```
error: cost strategy not specified
```

**Wrong**:
```bash
clarinet deployments generate --testnet
```

**Correct**:
```bash
clarinet deployments generate --testnet --low-cost
# or
clarinet deployments generate --testnet --medium-cost
# or
clarinet deployments generate --testnet --high-cost
```

---

### 12. Insufficient Funds for Deployment

**Error**:
```
insufficient funds
```

**Solution**: Get testnet STX from faucet

1. Visit: https://explorer.hiro.so/sandbox/faucet?chain=testnet
2. Enter your deployer address (from deployment plan)
3. Request testnet STX
4. Wait ~10 minutes for confirmation
5. Retry deployment

---

## 📦 Dependency Errors

### 13. Trait Not Found

**Error**:
```
trait sip009-nft-trait not found
```

**Wrong Clarinet.toml**:
```toml
[contracts.nft-minter]
path = 'contracts/nft-minter.clar'
# Missing dependency!
```

**Correct Clarinet.toml**:
```toml
[contracts.sip009-nft-trait]
path = 'contracts/sip009-nft-trait.clar'

[contracts.nft-minter]
path = 'contracts/nft-minter.clar'
depends_on = ['sip009-nft-trait']  # ← Add this!
```

**Contract Reference**:
```clarity
;; In nft-minter.clar
(impl-trait .sip009-nft-trait.nft-trait)  ;; Must match trait contract name
```

---

## 🔍 Debugging Tips

### Enable Verbose Logging

**Clarinet**:
```bash
clarinet check --verbose
clarinet test --verbose
```

**Tests**:
```typescript
it("debugging test", () => {
  const { result, events } = simnet.callPublicFn(...);
  console.log("Result:", result);
  console.log("Events:", events);
});
```

### Check Contract on Explorer

After deployment, verify on blockchain:
```
https://explorer.hiro.so/txid/YOUR_TX_ID?chain=testnet
```

### Use Clarinet Console

Interactive REPL for testing:
```bash
clarinet console
```

```clarity
>> (contract-call? .my-contract get-count)
u0
```

---

## 📋 Error Code Reference

### Common Clarity Error Codes

```clarity
;; Standard error codes
(define-constant err-owner-only (err u100))
(define-constant err-not-found (err u101))
(define-constant err-already-exists (err u102))
(define-constant err-insufficient-balance (err u103))
(define-constant err-unauthorized (err u104))
(define-constant err-invalid-input (err u105))
```

### HTTP Status Codes

- `200` - Success
- `400` - Bad request (invalid parameters)
- `401` - Unauthorized
- `404` - Contract not found
- `500` - Internal server error

---

## 🆘 Getting Help

**Stacks Discord**: https://discord.gg/stacks
**Stacks Forum**: https://forum.stacks.org
**GitHub Issues**: https://github.com/hirosystems/clarinet/issues

**When asking for help, include**:
1. Full error message
2. Clarity/Clarinet version
3. Code snippet (minimal reproduction)
4. What you've tried

---

## ✅ Prevention Checklist

Before deploying to testnet:

- [ ] All tests passing locally
- [ ] Contracts using LF line endings
- [ ] Dependencies correctly specified in Clarinet.toml
- [ ] Error codes documented
- [ ] Contract checked with `clarinet check`
- [ ] Deployment plan reviewed
- [ ] Testnet STX in deployer wallet

---

## 🔗 Related Guides

- [Modern Testing](modern-testing.md)
- [Deployment Workflow](../deployment/testnet-deployment.md)
- [Wallet Authentication](../wallets/stacks-connect-authentication.md)
