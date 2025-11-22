# Testing with Clarinet - Complete Guide

## 📖 Overview

Clarinet is the official development tool for Clarity smart contracts. It provides local blockchain simulation, testing framework, and deployment tools.

### What You'll Learn
- Installing and setting up Clarinet
- Project structure
- Writing tests
- Test assertions
- Mocking and simulations
- Best testing practices

---

## 🚀 Getting Started with Clarinet

### Installation

```bash
# macOS and Linux
brew install clarinet

# Windows (using installer)
# Download from: https://github.com/hirosystems/clarinet/releases

# Verify installation
clarinet --version
```

### Creating a New Project

```bash
# Create new project
clarinet new my-project
cd my-project

# Project structure
my-project/
├── Clarinet.toml          # Project configuration
├── settings/
│   └── Devnet.toml       # Network settings
├── contracts/            # Your Clarity contracts
│   └── counter.clar
└── tests/               # Test files
    └── counter_test.ts
```

### Adding a Contract

```bash
# Generate new contract
clarinet contract new my-token

# This creates:
# - contracts/my-token.clar
# - tests/my-token_test.ts
```

---

## 📝 Writing Your First Test

### Basic Test Structure

```typescript
import { Clarinet, Tx, Chain, Account, types } from 'https://deno.land/x/clarinet@v1.0.0/index.ts';
import { assertEquals } from 'https://deno.land/std@0.90.0/testing/asserts.ts';

Clarinet.test({
    name: "Test description",
    async fn(chain: Chain, accounts: Map<string, Account>) {
        // Test code here
    },
});
```

### Simple Counter Test

```typescript
// tests/counter_test.ts

import { Clarinet, Tx, Chain, Account, types } from 'https://deno.land/x/clarinet@v1.0.0/index.ts';
import { assertEquals } from 'https://deno.land/std@0.90.0/testing/asserts.ts';

Clarinet.test({
    name: "Counter starts at zero",
    async fn(chain: Chain, accounts: Map<string, Account>) {
        const deployer = accounts.get('deployer')!;
        
        // Call read-only function
        let count = chain.callReadOnlyFn(
            'counter',
            'get-count',
            [],
            deployer.address
        );
        
        // Assert the value
        count.result.expectUint(0);
    },
});

Clarinet.test({
    name: "Can increment counter",
    async fn(chain: Chain, accounts: Map<string, Account>) {
        const deployer = accounts.get('deployer')!;
        
        // Mine a block with transaction
        let block = chain.mineBlock([
            Tx.contractCall(
                'counter',
                'increment',
                [],
                deployer.address
            )
        ]);
        
        // Check transaction succeeded
        block.receipts[0].result.expectOk().expectBool(true);
        
        // Verify new count
        let count = chain.callReadOnlyFn(
            'counter',
            'get-count',
            [],
            deployer.address
        );
        count.result.expectUint(1);
    },
});
```

---

## 🔍 Test Assertions

### Result Expectations

```typescript
// Expect ok response
result.expectOk()

// Expect ok with specific value
result.expectOk().expectBool(true)
result.expectOk().expectUint(100)
result.expectOk().expectPrincipal('ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM')

// Expect error
result.expectErr()

// Expect error with specific code
result.expectErr().expectUint(100)

// Expect specific type
result.expectUint(42)
result.expectBool(true)
result.expectPrincipal(address)
result.expectAscii("hello")
```

### Optional Expectations

```typescript
// Expect some
result.expectSome().expectUint(10)

// Expect none
result.expectNone()
```

### Tuple Expectations

```typescript
// Expect tuple
result.expectTuple()

// Access tuple fields
let tuple = result.expectOk().expectTuple();
tuple['amount'].expectUint(1000);
tuple['recipient'].expectPrincipal(recipient.address);
```

### List Expectations

```typescript
// Expect list
result.expectList()

// Check list length
let list = result.expectList();
assertEquals(list.length, 3);

// Check list elements
list[0].expectUint(1);
list[1].expectUint(2);
list[2].expectUint(3);
```

---

## 💻 Complete Testing Examples

### Example 1: Token Contract Tests

```typescript
// tests/my-token_test.ts

import { Clarinet, Tx, Chain, Account, types } from 'https://deno.land/x/clarinet@v1.0.0/index.ts';
import { assertEquals } from 'https://deno.land/std@0.90.0/testing/asserts.ts';

Clarinet.test({
    name: "Token has correct metadata",
    async fn(chain: Chain, accounts: Map<string, Account>) {
        const deployer = accounts.get('deployer')!;
        
        let name = chain.callReadOnlyFn(
            'my-token',
            'get-name',
            [],
            deployer.address
        );
        name.result.expectOk().expectAscii("My Token");
        
        let symbol = chain.callReadOnlyFn(
            'my-token',
            'get-symbol',
            [],
            deployer.address
        );
        symbol.result.expectOk().expectAscii("MTK");
        
        let decimals = chain.callReadOnlyFn(
            'my-token',
            'get-decimals',
            [],
            deployer.address
        );
        decimals.result.expectOk().expectUint(6);
    },
});

Clarinet.test({
    name: "Can mint tokens",
    async fn(chain: Chain, accounts: Map<string, Account>) {
        const deployer = accounts.get('deployer')!;
        const recipient = accounts.get('wallet_1')!;
        
        // Mint tokens
        let block = chain.mineBlock([
            Tx.contractCall(
                'my-token',
                'mint',
                [
                    types.uint(1000000), // 1 token with 6 decimals
                    types.principal(recipient.address)
                ],
                deployer.address
            )
        ]);
        
        // Check mint succeeded
        block.receipts[0].result.expectOk().expectBool(true);
        
        // Verify balance
        let balance = chain.callReadOnlyFn(
            'my-token',
            'get-balance',
            [types.principal(recipient.address)],
            deployer.address
        );
        balance.result.expectOk().expectUint(1000000);
        
        // Verify total supply
        let supply = chain.callReadOnlyFn(
            'my-token',
            'get-total-supply',
            [],
            deployer.address
        );
        supply.result.expectOk().expectUint(1000000);
    },
});

Clarinet.test({
    name: "Can transfer tokens",
    async fn(chain: Chain, accounts: Map<string, Account>) {
        const deployer = accounts.get('deployer')!;
        const wallet1 = accounts.get('wallet_1')!;
        const wallet2 = accounts.get('wallet_2')!;
        
        // Mint tokens to wallet1
        chain.mineBlock([
            Tx.contractCall(
                'my-token',
                'mint',
                [types.uint(1000000), types.principal(wallet1.address)],
                deployer.address
            )
        ]);
        
        // Transfer from wallet1 to wallet2
        let block = chain.mineBlock([
            Tx.contractCall(
                'my-token',
                'transfer',
                [
                    types.uint(500000),
                    types.principal(wallet1.address),
                    types.principal(wallet2.address),
                    types.none()
                ],
                wallet1.address
            )
        ]);
        
        block.receipts[0].result.expectOk().expectBool(true);
        
        // Check balances
        let balance1 = chain.callReadOnlyFn(
            'my-token',
            'get-balance',
            [types.principal(wallet1.address)],
            deployer.address
        );
        balance1.result.expectOk().expectUint(500000);
        
        let balance2 = chain.callReadOnlyFn(
            'my-token',
            'get-balance',
            [types.principal(wallet2.address)],
            deployer.address
        );
        balance2.result.expectOk().expectUint(500000);
    },
});

Clarinet.test({
    name: "Non-owner cannot mint",
    async fn(chain: Chain, accounts: Map<string, Account>) {
        const wallet1 = accounts.get('wallet_1')!;
        const wallet2 = accounts.get('wallet_2')!;
        
        // Try to mint as non-owner
        let block = chain.mineBlock([
            Tx.contractCall(
                'my-token',
                'mint',
                [types.uint(1000000), types.principal(wallet2.address)],
                wallet1.address // Not the owner
            )
        ]);
        
        // Should fail with error 100
        block.receipts[0].result.expectErr().expectUint(100);
    },
});

Clarinet.test({
    name: "Cannot transfer more than balance",
    async fn(chain: Chain, accounts: Map<string, Account>) {
        const deployer = accounts.get('deployer')!;
        const wallet1 = accounts.get('wallet_1')!;
        const wallet2 = accounts.get('wallet_2')!;
        
        // Mint 1 token to wallet1
        chain.mineBlock([
            Tx.contractCall(
                'my-token',
                'mint',
                [types.uint(1000000), types.principal(wallet1.address)],
                deployer.address
            )
        ]);
        
        // Try to transfer 2 tokens
        let block = chain.mineBlock([
            Tx.contractCall(
                'my-token',
                'transfer',
                [
                    types.uint(2000000),
                    types.principal(wallet1.address),
                    types.principal(wallet2.address),
                    types.none()
                ],
                wallet1.address
            )
        ]);
        
        // Should fail
        block.receipts[0].result.expectErr();
    },
});
```

### Example 2: NFT Contract Tests

```typescript
// tests/my-nft_test.ts

Clarinet.test({
    name: "Can mint NFT",
    async fn(chain: Chain, accounts: Map<string, Account>) {
        const deployer = accounts.get('deployer')!;
        const recipient = accounts.get('wallet_1')!;
        
        let block = chain.mineBlock([
            Tx.contractCall(
                'my-nft',
                'mint',
                [types.principal(recipient.address)],
                deployer.address
            )
        ]);
        
        // Check mint returned token ID 1
        block.receipts[0].result.expectOk().expectUint(1);
        
        // Verify owner
        let owner = chain.callReadOnlyFn(
            'my-nft',
            'get-owner',
            [types.uint(1)],
            deployer.address
        );
        owner.result
            .expectOk()
            .expectSome()
            .expectPrincipal(recipient.address);
    },
});

Clarinet.test({
    name: "Can transfer NFT",
    async fn(chain: Chain, accounts: Map<string, Account>) {
        const deployer = accounts.get('deployer')!;
        const wallet1 = accounts.get('wallet_1')!;
        const wallet2 = accounts.get('wallet_2')!;
        
        // Mint NFT to wallet1
        chain.mineBlock([
            Tx.contractCall(
                'my-nft',
                'mint',
                [types.principal(wallet1.address)],
                deployer.address
            )
        ]);
        
        // Transfer to wallet2
        let block = chain.mineBlock([
            Tx.contractCall(
                'my-nft',
                'transfer',
                [
                    types.uint(1),
                    types.principal(wallet1.address),
                    types.principal(wallet2.address)
                ],
                wallet1.address
            )
        ]);
        
        block.receipts[0].result.expectOk();
        
        // Verify new owner
        let owner = chain.callReadOnlyFn(
            'my-nft',
            'get-owner',
            [types.uint(1)],
            deployer.address
        );
        owner.result
            .expectOk()
            .expectSome()
            .expectPrincipal(wallet2.address);
    },
});

Clarinet.test({
    name: "Non-owner cannot transfer NFT",
    async fn(chain: Chain, accounts: Map<string, Account>) {
        const deployer = accounts.get('deployer')!;
        const wallet1 = accounts.get('wallet_1')!;
        const wallet2 = accounts.get('wallet_2')!;
        const wallet3 = accounts.get('wallet_3')!;
        
        // Mint to wallet1
        chain.mineBlock([
            Tx.contractCall(
                'my-nft',
                'mint',
                [types.principal(wallet1.address)],
                deployer.address
            )
        ]);
        
        // Try to transfer as wallet2 (not owner)
        let block = chain.mineBlock([
            Tx.contractCall(
                'my-nft',
                'transfer',
                [
                    types.uint(1),
                    types.principal(wallet1.address),
                    types.principal(wallet3.address)
                ],
                wallet2.address
            )
        ]);
        
        // Should fail with error 101
        block.receipts[0].result.expectErr().expectUint(101);
    },
});
```

---

## 🎭 Advanced Testing Patterns

### Testing with Multiple Blocks

```typescript
Clarinet.test({
    name: "Multi-block scenario",
    async fn(chain: Chain, accounts: Map<string, Account>) {
        const deployer = accounts.get('deployer')!;
        
        // Block 1: Initialize
        let block1 = chain.mineBlock([
            Tx.contractCall('counter', 'initialize', [], deployer.address)
        ]);
        block1.receipts[0].result.expectOk();
        
        // Block 2: Multiple operations
        let block2 = chain.mineBlock([
            Tx.contractCall('counter', 'increment', [], deployer.address),
            Tx.contractCall('counter', 'increment', [], deployer.address),
            Tx.contractCall('counter', 'increment', [], deployer.address),
        ]);
        
        // Check all succeeded
        block2.receipts[0].result.expectOk();
        block2.receipts[1].result.expectOk();
        block2.receipts[2].result.expectOk();
        
        // Verify final state
        let count = chain.callReadOnlyFn(
            'counter',
            'get-count',
            [],
            deployer.address
        );
        count.result.expectUint(3);
    },
});
```

### Testing Events

```typescript
Clarinet.test({
    name: "Emits correct events",
    async fn(chain: Chain, accounts: Map<string, Account>) {
        const deployer = accounts.get('deployer')!;
        const recipient = accounts.get('wallet_1')!;
        
        let block = chain.mineBlock([
            Tx.contractCall(
                'my-token',
                'transfer',
                [
                    types.uint(1000000),
                    types.principal(deployer.address),
                    types.principal(recipient.address),
                    types.none()
                ],
                deployer.address
            )
        ]);
        
        // Check events
        assertEquals(block.receipts[0].events.length, 1);
        
        let event = block.receipts[0].events[0];
        assertEquals(event.type, 'ft_transfer_event');
        assertEquals(event.ft_transfer_event.sender, deployer.address);
        assertEquals(event.ft_transfer_event.recipient, recipient.address);
        assertEquals(event.ft_transfer_event.amount, '1000000');
    },
});
```

### Testing Block Height

```typescript
Clarinet.test({
    name: "Time-based logic works",
    async fn(chain: Chain, accounts: Map<string, Account>) {
        const deployer = accounts.get('deployer')!;
        
        // Get current block height
        let initialHeight = chain.blockHeight;
        
        // Mine empty blocks to advance time
        chain.mineEmptyBlock(100); // Advance 100 blocks
        
        // Now test time-dependent function
        let block = chain.mineBlock([
            Tx.contractCall(
                'escrow',
                'refund-expired',
                [types.uint(1)],
                deployer.address
            )
        ]);
        
        block.receipts[0].result.expectOk();
    },
});
```

---

## 🧪 Testing Best Practices

### 1. Test Organization

```typescript
// Group related tests
Clarinet.test({ name: "Token: metadata is correct", ... });
Clarinet.test({ name: "Token: can mint tokens", ... });
Clarinet.test({ name: "Token: can transfer tokens", ... });

Clarinet.test({ name: "NFT: can mint NFT", ... });
Clarinet.test({ name: "NFT: can transfer NFT", ... });
```

### 2. Test Naming Convention

```typescript
// Good test names describe what they test
"Can mint new tokens"
"Non-owner cannot mint"
"Transfer fails with insufficient balance"
"NFT ownership changes after transfer"
```

### 3. Test Independence

```typescript
// Each test should be independent
Clarinet.test({
    name: "Test 1",
    async fn(chain: Chain, accounts: Map<string, Account>) {
        // This test doesn't depend on Test 2
        // Fresh state for each test
    },
});
```

### 4. Testing Edge Cases

```typescript
// Test boundary conditions
Clarinet.test({ name: "Transfers zero amount", ... });
Clarinet.test({ name: "Transfers maximum uint", ... });
Clarinet.test({ name: "Empty list handling", ... });
Clarinet.test({ name: "Non-existent token ID", ... });
```

### 5. Error Path Testing

```typescript
// Test all error conditions
Clarinet.test({ name: "Fails when not authorized", ... });
Clarinet.test({ name: "Fails when paused", ... });
Clarinet.test({ name: "Fails with invalid input", ... });
```

---

## 🎯 Complete Test Suite Template

```typescript
import { Clarinet, Tx, Chain, Account, types } from 'https://deno.land/x/clarinet@v1.0.0/index.ts';
import { assertEquals } from 'https://deno.land/std@0.90.0/testing/asserts.ts';

// ============================================
// Setup and Initialization Tests
// ============================================

Clarinet.test({
    name: "Contract initializes correctly",
    async fn(chain: Chain, accounts: Map<string, Account>) {
        // Test initial state
    },
});

// ============================================
// Happy Path Tests
// ============================================

Clarinet.test({
    name: "Basic operation succeeds",
    async fn(chain: Chain, accounts: Map<string, Account>) {
        // Test expected behavior
    },
});

// ============================================
// Authorization Tests
// ============================================

Clarinet.test({
    name: "Owner can perform admin action",
    async fn(chain: Chain, accounts: Map<string, Account>) {
        // Test owner permissions
    },
});

Clarinet.test({
    name: "Non-owner cannot perform admin action",
    async fn(chain: Chain, accounts: Map<string, Account>) {
        // Test access control
    },
});

// ============================================
// Validation Tests
// ============================================

Clarinet.test({
    name: "Rejects invalid inputs",
    async fn(chain: Chain, accounts: Map<string, Account>) {
        // Test input validation
    },
});

// ============================================
// State Transition Tests
// ============================================

Clarinet.test({
    name: "State updates correctly",
    async fn(chain: Chain, accounts: Map<string, Account>) {
        // Test state changes
    },
});

// ============================================
// Edge Case Tests
// ============================================

Clarinet.test({
    name: "Handles edge case",
    async fn(chain: Chain, accounts: Map<string, Account>) {
        // Test boundary conditions
    },
});

// ============================================
// Integration Tests
// ============================================

Clarinet.test({
    name: "Works with other contracts",
    async fn(chain: Chain, accounts: Map<string, Account>) {
        // Test contract interactions
    },
});
```

---

## 🚨 Running Tests

```bash
# Run all tests
clarinet test

# Run specific test file
clarinet test tests/my-token_test.ts

# Run with verbose output
clarinet test --watch

# Generate coverage report
clarinet test --coverage
```

---

## 🎓 Practice Exercises

1. Write a complete test suite for a counter contract
2. Test all error conditions in a token contract
3. Create integration tests for a marketplace
4. Test time-dependent escrow logic

---

## 📚 Next Steps

**Related Guides:**
- → Deploying Contracts
- → Contract Debugging
- → Integration Testing Patterns