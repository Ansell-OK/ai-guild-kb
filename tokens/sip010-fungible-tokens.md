# SIP-010 Fungible Token Implementation Guide

## 📖 Overview

Fungible tokens are interchangeable digital assets where each unit is identical and has the same value. On Stacks, fungible tokens follow the SIP-010 standard, enabling compatibility across wallets, exchanges, and DeFi applications.

### What You'll Learn
- Understanding the SIP-010 trait
- Implementing a basic fungible token
- Token transfers and balance tracking
- Minting and burning mechanisms
- Best practices for token economics

### Prerequisites
- Basic Clarity syntax
- Understanding of principals and responses
- Familiarity with maps and variables

---

## 🎯 Key Concepts

### What is SIP-010?

SIP-010 defines the standard interface for fungible tokens on Stacks, similar to ERC-20 on Ethereum.

**Core Functions Required:**
- `transfer` - Transfer tokens between accounts
- `get-name` - Return token name
- `get-symbol` - Return token symbol
- `get-decimals` - Return decimal places
- `get-balance` - Get balance of an address
- `get-total-supply` - Get total tokens in circulation
- `get-token-uri` - Optional metadata URI

### Token Characteristics

1. **Fungible**: All units are identical and interchangeable
2. **Divisible**: Can be split into smaller units (based on decimals)
3. **Transferable**: Can be sent between addresses
4. **Supply Tracked**: Total supply is always known

---

## 💻 Basic Fungible Token Contract

### Complete Implementation

```clarity
;; SIP-010 Fungible Token Standard Implementation
;; Simple token with minting and transfer capabilities

;; Import the fungible token trait
(impl-trait 'SP3FBR2AGK5H9QBDH3EEN6DF8EK8JY7RX8QJ5SVTE.sip-010-trait-ft-standard.sip-010-trait)

;; Define the token
(define-fungible-token my-token)

;; Token metadata
(define-constant token-name "My Token")
(define-constant token-symbol "MTK")
(define-constant token-decimals u6)  ;; 6 decimals like USDC
(define-data-var token-uri (optional (string-utf8 256)) none)

;; Contract owner
(define-constant contract-owner tx-sender)

;; Error codes
(define-constant err-owner-only (err u100))
(define-constant err-not-token-owner (err u101))
(define-constant err-insufficient-balance (err u102))
(define-constant err-invalid-amount (err u103))

;; -----------------------------------------------------------
;; SIP-010 Required Functions
;; -----------------------------------------------------------

;; Transfer tokens
(define-public (transfer 
    (amount uint) 
    (sender principal) 
    (recipient principal) 
    (memo (optional (buff 34)))
)
    (begin
        ;; Ensure sender is tx-sender
        (asserts! (is-eq tx-sender sender) err-not-token-owner)
        
        ;; Transfer the tokens
        (try! (ft-transfer? my-token amount sender recipient))
        
        ;; Print memo if provided
        (match memo to-print (print to-print) 0x)
        
        (ok true)
    )
)

;; Get token name
(define-read-only (get-name)
    (ok token-name)
)

;; Get token symbol
(define-read-only (get-symbol)
    (ok token-symbol)
)

;; Get token decimals
(define-read-only (get-decimals)
    (ok token-decimals)
)

;; Get balance of address
(define-read-only (get-balance (account principal))
    (ok (ft-get-balance my-token account))
)

;; Get total supply
(define-read-only (get-total-supply)
    (ok (ft-get-supply my-token))
)

;; Get token URI
(define-read-only (get-token-uri)
    (ok (var-get token-uri))
)

;; -----------------------------------------------------------
;; Additional Functions (Minting, Burning, Admin)
;; -----------------------------------------------------------

;; Mint new tokens (only owner)
(define-public (mint (amount uint) (recipient principal))
    (begin
        ;; Only contract owner can mint
        (asserts! (is-eq tx-sender contract-owner) err-owner-only)
        
        ;; Amount must be positive
        (asserts! (> amount u0) err-invalid-amount)
        
        ;; Mint tokens
        (try! (ft-mint? my-token amount recipient))
        
        (ok true)
    )
)

;; Burn tokens from sender
(define-public (burn (amount uint))
    (begin
        ;; Amount must be positive
        (asserts! (> amount u0) err-invalid-amount)
        
        ;; Burn tokens from sender
        (try! (ft-burn? my-token amount tx-sender))
        
        (ok true)
    )
)

;; Set token URI (only owner)
(define-public (set-token-uri (new-uri (optional (string-utf8 256))))
    (begin
        (asserts! (is-eq tx-sender contract-owner) err-owner-only)
        (ok (var-set token-uri new-uri))
    )
)

;; -----------------------------------------------------------
;; Helper Functions
;; -----------------------------------------------------------

;; Convert amount to display format (with decimals)
(define-read-only (to-display-amount (raw-amount uint))
    ;; For 6 decimals: 1000000 raw units = 1.0 display units
    (/ raw-amount (pow u10 token-decimals))
)

;; Convert display amount to raw format
(define-read-only (to-raw-amount (display-amount uint))
    ;; For 6 decimals: 1.0 display = 1000000 raw units
    (* display-amount (pow u10 token-decimals))
)

;; Check if account has sufficient balance
(define-read-only (has-balance (account principal) (amount uint))
    (>= (ft-get-balance my-token account) amount)
)
```

---

## 🔍 Code Explanation

### 1. Token Definition

```clarity
(define-fungible-token my-token)
```

Creates the fungible token. Unlike NFTs which track unique IDs, fungible tokens track quantities.

### 2. Token Metadata

```clarity
(define-constant token-name "My Token")
(define-constant token-symbol "MTK")
(define-constant token-decimals u6)
```

- **name**: Full token name
- **symbol**: Trading symbol (ticker)
- **decimals**: Divisibility (6 = divide by 1,000,000)

**Example with 6 decimals:**
- Raw amount: `1000000` = Display: `1.0`
- Raw amount: `500000` = Display: `0.5`

### 3. Transfer Function

```clarity
(define-public (transfer 
    (amount uint) 
    (sender principal) 
    (recipient principal) 
    (memo (optional (buff 34)))
)
    (begin
        (asserts! (is-eq tx-sender sender) err-not-token-owner)
        (try! (ft-transfer? my-token amount sender recipient))
        (match memo to-print (print to-print) 0x)
        (ok true)
    )
)
```

**Security:**
- Verifies `tx-sender` matches `sender` parameter
- Prevents unauthorized transfers
- Optional memo for transaction notes

### 4. Balance Tracking

```clarity
(define-read-only (get-balance (account principal))
    (ok (ft-get-balance my-token account))
)
```

Uses Clarity's built-in `ft-get-balance` to retrieve token balance for any address.

### 5. Minting Function

```clarity
(define-public (mint (amount uint) (recipient principal))
    (begin
        (asserts! (is-eq tx-sender contract-owner) err-owner-only)
        (asserts! (> amount u0) err-invalid-amount)
        (try! (ft-mint? my-token amount recipient))
        (ok true)
    )
)
```

**Controls:**
- Only contract owner can mint
- Amount must be greater than zero
- Updates total supply automatically

### 6. Burning Function

```clarity
(define-public (burn (amount uint))
    (begin
        (asserts! (> amount u0) err-invalid-amount)
        (try! (ft-burn? my-token amount tx-sender))
        (ok true)
    )
)
```

Anyone can burn their own tokens, reducing total supply.

---

## 🌐 Frontend Integration

### React Component for Token Operations

```javascript
import { useState, useEffect } from 'react';
import {
  openContractCall,
  showConnect,
  AppConfig,
  UserSession
} from '@stacks/connect';
import {
  PostConditionMode,
  FungibleConditionCode,
  makeStandardFungiblePostCondition,
  uintCV,
  standardPrincipalCV,
  someCV,
  bufferCV
} from '@stacks/transactions';
import { StacksTestnet } from '@stacks/network';

function TokenDashboard() {
  const [userSession] = useState(() =>
    new UserSession({ appConfig: new AppConfig(['store_write']) })
  );
  const [userData, setUserData] = useState(null);
  const [balance, setBalance] = useState(0);
  const [totalSupply, setTotalSupply] = useState(0);

  const CONTRACT_ADDRESS = 'ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM';
  const CONTRACT_NAME = 'my-token';

  // Connect wallet
  const connectWallet = () => {
    showConnect({
      appDetails: {
        name: 'Token Dashboard',
        icon: window.location.origin + '/logo.png',
      },
      userSession,
      onFinish: () => {
        setUserData(userSession.loadUserData());
      },
    });
  };

  // Get token balance
  useEffect(() => {
    if (userData) {
      fetchBalance();
      fetchTotalSupply();
    }
  }, [userData]);

  const fetchBalance = async () => {
    if (!userData) return;

    const { callReadOnlyFunction, cvToValue } = await import('@stacks/transactions');
    
    const options = {
      contractAddress: CONTRACT_ADDRESS,
      contractName: CONTRACT_NAME,
      functionName: 'get-balance',
      functionArgs: [standardPrincipalCV(userData.profile.stxAddress.testnet)],
      network: new StacksTestnet(),
      senderAddress: userData.profile.stxAddress.testnet,
    };

    try {
      const result = await callReadOnlyFunction(options);
      const balanceValue = cvToValue(result);
      setBalance(balanceValue.value / 1000000); // Convert from raw to display (6 decimals)
    } catch (error) {
      console.error('Error fetching balance:', error);
    }
  };

  const fetchTotalSupply = async () => {
    const { callReadOnlyFunction, cvToValue } = await import('@stacks/transactions');
    
    const options = {
      contractAddress: CONTRACT_ADDRESS,
      contractName: CONTRACT_NAME,
      functionName: 'get-total-supply',
      functionArgs: [],
      network: new StacksTestnet(),
      senderAddress: CONTRACT_ADDRESS,
    };

    try {
      const result = await callReadOnlyFunction(options);
      const supplyValue = cvToValue(result);
      setTotalSupply(supplyValue.value / 1000000);
    } catch (error) {
      console.error('Error fetching supply:', error);
    }
  };

  // Transfer tokens
  const transferTokens = async (recipient, amount) => {
    if (!userData) return;

    const rawAmount = amount * 1000000; // Convert to raw amount (6 decimals)
    const senderAddress = userData.profile.stxAddress.testnet;

    // Create post condition to protect sender
    const postCondition = makeStandardFungiblePostCondition(
      senderAddress,
      FungibleConditionCode.Equal,
      rawAmount,
      `${CONTRACT_ADDRESS}.${CONTRACT_NAME}::my-token`
    );

    const functionArgs = [
      uintCV(rawAmount),
      standardPrincipalCV(senderAddress),
      standardPrincipalCV(recipient),
      someCV(bufferCV(Buffer.from('Transfer via UI', 'utf-8')))
    ];

    const options = {
      contractAddress: CONTRACT_ADDRESS,
      contractName: CONTRACT_NAME,
      functionName: 'transfer',
      functionArgs: functionArgs,
      network: new StacksTestnet(),
      postConditions: [postCondition],
      postConditionMode: PostConditionMode.Deny,
      onFinish: (data) => {
        console.log('Transfer TX:', data.txId);
        alert(`Tokens transferred! TX: ${data.txId}`);
        fetchBalance(); // Refresh balance
      },
    };

    await openContractCall(options);
  };

  // Mint tokens (owner only)
  const mintTokens = async (recipient, amount) => {
    const rawAmount = amount * 1000000;

    const functionArgs = [
      uintCV(rawAmount),
      standardPrincipalCV(recipient)
    ];

    const options = {
      contractAddress: CONTRACT_ADDRESS,
      contractName: CONTRACT_NAME,
      functionName: 'mint',
      functionArgs: functionArgs,
      network: new StacksTestnet(),
      postConditionMode: PostConditionMode.Deny,
      onFinish: (data) => {
        console.log('Mint TX:', data.txId);
        alert(`Tokens minted! TX: ${data.txId}`);
        fetchBalance();
        fetchTotalSupply();
      },
    };

    await openContractCall(options);
  };

  // Burn tokens
  const burnTokens = async (amount) => {
    const rawAmount = amount * 1000000;

    const functionArgs = [uintCV(rawAmount)];

    const options = {
      contractAddress: CONTRACT_ADDRESS,
      contractName: CONTRACT_NAME,
      functionName: 'burn',
      functionArgs: functionArgs,
      network: new StacksTestnet(),
      postConditionMode: PostConditionMode.Deny,
      onFinish: (data) => {
        console.log('Burn TX:', data.txId);
        alert(`Tokens burned! TX: ${data.txId}`);
        fetchBalance();
        fetchTotalSupply();
      },
    };

    await openContractCall(options);
  };

  return (
    <div className="p-6 max-w-2xl mx-auto">
      <h1 className="text-3xl font-bold mb-6">Token Dashboard</h1>

      {!userData ? (
        <button
          onClick={connectWallet}
          className="bg-blue-500 text-white px-6 py-3 rounded-lg"
        >
          Connect Wallet
        </button>
      ) : (
        <div className="space-y-6">
          <div className="bg-gray-100 p-4 rounded-lg">
            <p className="text-sm text-gray-600">Connected Address</p>
            <p className="font-mono text-sm">{userData.profile.stxAddress.testnet}</p>
          </div>

          <div className="grid grid-cols-2 gap-4">
            <div className="bg-white p-4 rounded-lg shadow">
              <p className="text-sm text-gray-600">Your Balance</p>
              <p className="text-2xl font-bold">{balance.toFixed(2)} MTK</p>
            </div>
            <div className="bg-white p-4 rounded-lg shadow">
              <p className="text-sm text-gray-600">Total Supply</p>
              <p className="text-2xl font-bold">{totalSupply.toFixed(2)} MTK</p>
            </div>
          </div>

          <div className="bg-white p-6 rounded-lg shadow">
            <h2 className="text-xl font-bold mb-4">Transfer Tokens</h2>
            <TransferForm onSubmit={transferTokens} />
          </div>

          <div className="grid grid-cols-2 gap-4">
            <div className="bg-white p-6 rounded-lg shadow">
              <h2 className="text-xl font-bold mb-4">Mint Tokens</h2>
              <MintForm onSubmit={mintTokens} />
            </div>
            <div className="bg-white p-6 rounded-lg shadow">
              <h2 className="text-xl font-bold mb-4">Burn Tokens</h2>
              <BurnForm onSubmit={burnTokens} />
            </div>
          </div>
        </div>
      )}
    </div>
  );
}

// Sub-components for forms
function TransferForm({ onSubmit }) {
  const [recipient, setRecipient] = useState('');
  const [amount, setAmount] = useState('');

  const handleSubmit = (e) => {
    e.preventDefault();
    onSubmit(recipient, parseFloat(amount));
    setRecipient('');
    setAmount('');
  };

  return (
    <form onSubmit={handleSubmit} className="space-y-4">
      <input
        type="text"
        placeholder="Recipient address"
        value={recipient}
        onChange={(e) => setRecipient(e.target.value)}
        className="w-full p-2 border rounded"
      />
      <input
        type="number"
        step="0.000001"
        placeholder="Amount"
        value={amount}
        onChange={(e) => setAmount(e.target.value)}
        className="w-full p-2 border rounded"
      />
      <button
        type="submit"
        className="w-full bg-blue-500 text-white py-2 rounded"
      >
        Transfer
      </button>
    </form>
  );
}

function MintForm({ onSubmit }) {
  const [recipient, setRecipient] = useState('');
  const [amount, setAmount] = useState('');

  const handleSubmit = (e) => {
    e.preventDefault();
    onSubmit(recipient, parseFloat(amount));
    setRecipient('');
    setAmount('');
  };

  return (
    <form onSubmit={handleSubmit} className="space-y-4">
      <input
        type="text"
        placeholder="Recipient address"
        value={recipient}
        onChange={(e) => setRecipient(e.target.value)}
        className="w-full p-2 border rounded"
      />
      <input
        type="number"
        step="0.000001"
        placeholder="Amount"
        value={amount}
        onChange={(e) => setAmount(e.target.value)}
        className="w-full p-2 border rounded"
      />
      <button
        type="submit"
        className="w-full bg-green-500 text-white py-2 rounded"
      >
        Mint
      </button>
    </form>
  );
}

function BurnForm({ onSubmit }) {
  const [amount, setAmount] = useState('');

  const handleSubmit = (e) => {
    e.preventDefault();
    onSubmit(parseFloat(amount));
    setAmount('');
  };

  return (
    <form onSubmit={handleSubmit} className="space-y-4">
      <input
        type="number"
        step="0.000001"
        placeholder="Amount to burn"
        value={amount}
        onChange={(e) => setAmount(e.target.value)}
        className="w-full p-2 border rounded"
      />
      <button
        type="submit"
        className="w-full bg-red-500 text-white py-2 rounded"
      >
        Burn
      </button>
    </form>
  );
}

export default TokenDashboard;
```

---

## 🧪 Testing with Clarinet

### Test File: `tests/token_test.ts`

```typescript
import { Clarinet, Tx, Chain, Account, types } from 'https://deno.land/x/clarinet@v1.0.0/index.ts';
import { assertEquals } from 'https://deno.land/std@0.90.0/testing/asserts.ts';

Clarinet.test({
  name: "Token has correct metadata",
  async fn(chain: Chain, accounts: Map<string, Account>) {
    const deployer = accounts.get('deployer')!;
    
    // Check name
    let name = chain.callReadOnlyFn(
      'my-token',
      'get-name',
      [],
      deployer.address
    );
    name.result.expectOk().expectUtf8("My Token");
    
    // Check symbol
    let symbol = chain.callReadOnlyFn(
      'my-token',
      'get-symbol',
      [],
      deployer.address
    );
    symbol.result.expectOk().expectUtf8("MTK");
    
    // Check decimals
    let decimals = chain.callReadOnlyFn(
      'my-token',
      'get-decimals',
      [],
      deployer.address
    );
    decimals.result.expectOk().expectUint(6);
  }
});

Clarinet.test({
  name: "Can mint tokens",
  async fn(chain: Chain, accounts: Map<string, Account>) {
    const deployer = accounts.get('deployer')!;
    const wallet1 = accounts.get('wallet_1')!;
    
    let block = chain.mineBlock([
      Tx.contractCall(
        'my-token',
        'mint',
        [types.uint(1000000), types.principal(wallet1.address)],
        deployer.address
      )
    ]);
    
    block.receipts[0].result.expectOk().expectBool(true);
    
    // Check balance
    let balance = chain.callReadOnlyFn(
      'my-token',
      'get-balance',
      [types.principal(wallet1.address)],
      deployer.address
    );
    balance.result.expectOk().expectUint(1000000);
    
    // Check total supply
    let supply = chain.callReadOnlyFn(
      'my-token',
      'get-total-supply',
      [],
      deployer.address
    );
    supply.result.expectOk().expectUint(1000000);
  }
});

Clarinet.test({
  name: "Can transfer tokens",
  async fn(chain: Chain, accounts: Map<string, Account>) {
    const deployer = accounts.get('deployer')!;
    const wallet1 = accounts.get('wallet_1')!;
    const wallet2 = accounts.get('wallet_2')!;
    
    // Mint to wallet1
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
  }
});

Clarinet.test({
  name: "Non-owner cannot mint",
  async fn(chain: Chain, accounts: Map<string, Account>) {
    const wallet1 = accounts.get('wallet_1')!;
    const wallet2 = accounts.get('wallet_2')!;
    
    let block = chain.mineBlock([
      Tx.contractCall(
        'my-token',
        'mint',
        [types.uint(1000000), types.principal(wallet2.address)],
        wallet1.address // Not owner
      )
    ]);
    
    block.receipts[0].result.expectErr().expectUint(100);
  }
});

Clarinet.test({
  name: "Cannot transfer more than balance",
  async fn(chain: Chain, accounts: Map<string, Account>) {
    const deployer = accounts.get('deployer')!;
    const wallet1 = accounts.get('wallet_1')!;
    const wallet2 = accounts.get('wallet_2')!;
    
    // Mint 1.0 token (1000000 raw)
    chain.mineBlock([
      Tx.contractCall(
        'my-token',
        'mint',
        [types.uint(1000000), types.principal(wallet1.address)],
        deployer.address
      )
    ]);
    
    // Try to transfer 2.0 tokens (2000000 raw)
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
  }
});

Clarinet.test({
  name: "Can burn tokens",
  async fn(chain: Chain, accounts: Map<string, Account>) {
    const deployer = accounts.get('deployer')!;
    const wallet1 = accounts.get('wallet_1')!;
    
    // Mint tokens
    chain.mineBlock([
      Tx.contractCall(
        'my-token',
        'mint',
        [types.uint(1000000), types.principal(wallet1.address)],
        deployer.address
      )
    ]);
    
    // Burn half
    let block = chain.mineBlock([
      Tx.contractCall(
        'my-token',
        'burn',
        [types.uint(500000)],
        wallet1.address
      )
    ]);
    
    block.receipts[0].result.expectOk().expectBool(true);
    
    // Check remaining balance
    let balance = chain.callReadOnlyFn(
      'my-token',
      'get-balance',
      [types.principal(wallet1.address)],
      deployer.address
    );
    balance.result.expectOk().expectUint(500000);
    
    // Check total supply reduced
    let supply = chain.callReadOnlyFn(
      'my-token',
      'get-total-supply',
      [],
      deployer.address
    );
    supply.result.expectOk().expectUint(500000);
  }
});
```

---

## ⚠️ Common Pitfalls

### 1. **Decimal Confusion**

❌ **Wrong:** Sending display amount instead of raw
```javascript
transfer(1, sender, recipient) // Will transfer 0.000001 tokens!
```

✅ **Correct:** Convert to raw amount
```javascript
const rawAmount = 1 * 1000000; // 1.0 token
transfer(rawAmount, sender, recipient)
```

### 2. **Missing Transfer Authorization**

❌ **Wrong:**
```clarity
(define-public (transfer (amount uint) (sender principal) (recipient principal))
    (ft-transfer? my-token amount sender recipient)
)
```

✅ **Correct:**
```clarity
(define-public (transfer (amount uint) (sender principal) (recipient principal))
    (begin
        (asserts! (is-eq tx-sender sender) err-not-token-owner)
        (ft-transfer? my-token amount sender recipient)
    )
)
```

### 3. **No Post Conditions**

Always use post conditions in frontend to protect users:

```javascript
const postCondition = makeStandardFungiblePostCondition(
  senderAddress,
  FungibleConditionCode.Equal, // or LessEqual
  amount,
  `${CONTRACT_ADDRESS}.${CONTRACT_NAME}::my-token`
);
```

### 4. **Ignoring Error Responses**

❌ **Wrong:**
```clarity
(ft-transfer? my-token amount sender recipient)
```

✅ **Correct:**
```clarity
(try! (ft-transfer? my-token amount sender recipient))
```

---

## 🚀 Advanced Features

### 1. **Allowance System**

Allow third parties to spend on your behalf:

```clarity
(define-map allowances 
    { owner: principal, spender: principal } 
    { amount: uint }
)

(define-public (approve (spender principal) (amount uint))
    (begin
        (map-set allowances 
            { owner: tx-sender, spender: spender }
            { amount: amount }
        )
        (ok true)
    )
)

(define-public (transfer-from 
    (amount uint) 
    (owner principal) 
    (recipient principal)
)
    (let
        (
            (allowance (default-to u0 
                (get amount (map-get? allowances 
                    { owner: owner, spender: tx-sender }
                ))
            ))
        )
        (asserts! (>= allowance amount) err-insufficient-allowance)
        (try! (ft-transfer? my-token amount owner recipient))
        (map-set allowances 
            { owner: owner, spender: tx-sender }
            { amount: (- allowance amount) }
        )
        (ok true)
    )
)
```

### 2. **Transfer Fees**

Implement transaction fees:

```clarity
(define-constant fee-percentage u100) ;; 1% (100/10000)
(define-constant fee-recipient 'ST1234...')

(define-public (transfer-with-fee 
    (amount uint) 
    (sender principal) 
    (recipient principal)
)
    (let
        (
            (fee (/ (* amount fee-percentage) u10000))
            (amount-after-fee (- amount fee))
        )
        (asserts! (is-eq tx-sender sender) err-not-token-owner)
        
        ;; Transfer fee
        (try! (ft-transfer? my-token fee sender fee-recipient))
        
        ;; Transfer remaining to recipient
        (try! (ft-transfer? my-token amount-after-fee sender recipient))
        
        (ok true)
    )
)
```

### 3. **Pausable Transfers**

Emergency pause mechanism:

```clarity
(define-data-var paused bool false)

(define-public (pause)
    (begin
        (asserts! (is-eq tx-sender contract-owner) err-owner-only)
        (ok (var-set paused true))
    )
)

(define-public (unpause)
    (begin
        (asserts! (is-eq tx-sender contract-owner) err-owner-only)
        (ok (var-set paused false))
    )
)

(define-public (transfer 
    (amount uint) 
    (sender principal) 
    (recipient principal) 
    (memo (optional (buff 34)))
)
    (begin
        (asserts! (not (var-get paused)) (err u104))
        (asserts! (is-eq tx-sender sender) err-not-token-owner)
        (try! (ft-transfer? my-token amount sender recipient))
        (match memo to-print (print to-print) 0x)
        (ok true)
    )
)
```

---

## 📊 Token Economics Examples

### Fixed Supply Token

```clarity
;; Set max supply at deployment
(define-constant max-supply u1000000000000) ;; 1 million with 6 decimals

(define-public (mint (amount uint) (recipient principal))
    (let
        (
            (current-supply (ft-get-supply my-token))
            (new-supply (+ current-supply amount))
        )
        (asserts! (is-eq tx-sender contract-owner) err-owner-only)
        (asserts! (<= new-supply max-supply) (err u105))
        (try! (ft-mint? my-token amount recipient))
        (ok true)
    )
)
```

### Vesting Schedule

```clarity
(define-map vesting-schedule
    principal
    {
        total-amount: uint,
        released-amount: uint,
        start-block: uint,
        duration-blocks: uint
    }
)

(define-public (release-vested-tokens (beneficiary principal))
    (let
        (
            (schedule (unwrap! (map-get? vesting-schedule beneficiary) (err u106)))
            (blocks-passed (- block-height (get start-block schedule)))
            (vested-amount (/ (* (get total-amount schedule) blocks-passed) 
                             (get duration-blocks schedule)))
            (releasable (- vested-amount (get released-amount schedule)))
        )
        (asserts! (> releasable u0) (err u107))
        
        ;; Update released amount
        (map-set vesting-schedule beneficiary
            (merge schedule { released-amount: vested-amount })
        )
        
        ;; Transfer tokens
        (try! (ft-transfer? my-token releasable tx-sender beneficiary))
        (ok releasable)
    )
)
```

---

## 🔒 Security Best Practices

1. **Always Validate Sender**
   ```clarity
   (asserts! (is-eq tx-sender sender) err-not-token-owner)
   ```

2. **Use Post Conditions in Frontend**
   ```javascript
   postConditions: [makeStandardFungiblePostCondition(...)]
   ```

3. **Check for Zero Amounts**
   ```clarity
   (asserts! (> amount u0) err-invalid-amount)
   ```

4. **Handle Overflow/Underflow**
   ```clarity
   ;; Clarity prevents overflow by default, but still validate
   (asserts! (>= balance amount) err-insufficient-balance)
   ```

5. **Emit Events for Tracking**
   ```clarity
   (print { 
       event: "transfer",
       sender: sender,
       recipient: recipient,
       amount: amount
   })
   ```

---

## 🚀 Deployment Checklist

### Pre-Deployment

- [ ] Token name, symbol, decimals finalized
- [ ] Max supply determined (if applicable)
- [ ] Minting strategy decided
- [ ] All tests passing
- [ ] Security audit completed
- [ ] Testnet deployment successful

### Deployment Steps

1. **Deploy to Testnet**
```bash
clarinet deploy --testnet
```

2. **Test All Functions**
   - Mint tokens
   - Transfer tokens
   - Check balances
   - Test edge cases

3. **Deploy to Mainnet**
```bash
clarinet deploy --mainnet
```

### Post-Deployment

- [ ] Contract verified on explorer
- [ ] Token listed on block explorers
- [ ] Frontend updated with contract address
- [ ] Initial token distribution completed
- [ ] Documentation published

---

## 📚 Use Cases

### 1. **Governance Token**
- Fixed supply
- No minting after initial distribution
- Used for DAO voting

### 2. **Utility Token**
- Pay for services in your dApp
- Can be earned through platform activity
- Burnable for premium features

### 3. **Stablecoin**
- Pegged to fiat currency
- Minting/burning controlled by oracle
- Requires collateral management

### 4. **Reward Token**
- Minted as rewards for users
- Can be staked for additional benefits
- Time-locked vesting

---

## 🎓 Next Steps

1. **Add Advanced Features**: Implement allowances, fees, or vesting
2. **Build DeFi Integration**: Create liquidity pools or staking
3. **Create Token Sale**: Build ICO/IDO mechanism
4. **Governance Integration**: Add voting capabilities

**Related Guides:**
- → Building Liquidity Pools
- → Token Staking Mechanisms
- → DAO Governance with Tokens
- → DEX Integration