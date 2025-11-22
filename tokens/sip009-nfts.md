# SIP-009 NFT Implementation Guide

## 📖 Overview

NFTs (Non-Fungible Tokens) on Stacks follow the SIP-009 standard, which defines a common interface for NFT contracts. This allows different applications and marketplaces to interact with your NFTs in a standardized way.

### What You'll Learn
- Understanding the SIP-009 trait
- Implementing basic NFT functionality
- Minting and transferring NFTs
- Adding metadata and URIs
- Common patterns and best practices

### Prerequisites
- Basic Clarity syntax knowledge
- Understanding of principals (addresses)
- Familiarity with response types

---

## 🎯 Key Concepts

### What is SIP-009?

SIP-009 is a standard trait (interface) that all NFTs on Stacks should implement. It ensures compatibility across the ecosystem.

**Core Functions Required:**
- `get-last-token-id` - Returns the highest token ID minted
- `get-token-uri` - Returns metadata URI for a token
- `get-owner` - Returns the owner of a token
- `transfer` - Transfers token ownership

### NFT Characteristics

1. **Unique**: Each token has a unique ID
2. **Ownable**: Each token has exactly one owner
3. **Transferable**: Owners can transfer to others
4. **Discoverable**: Metadata accessible via URI

---

## 💻 Basic NFT Contract

### Complete Implementation

```clarity
;; SIP-009 NFT Standard Implementation
;; This contract creates a basic NFT collection

;; Import the NFT trait
(impl-trait 'SP2PABAF9FTAJYNFZH93XENAJ8FVY99RRM50D2JG9.nft-trait.nft-trait)

;; Define the NFT
(define-non-fungible-token my-nft uint)

;; Storage for metadata
(define-data-var last-token-id uint u0)
(define-constant contract-owner tx-sender)
(define-constant base-token-uri "ipfs://QmExample/")

;; Error codes
(define-constant err-owner-only (err u100))
(define-constant err-not-token-owner (err u101))
(define-constant err-token-exists (err u102))
(define-constant err-token-not-found (err u103))

;; SIP-009 Required Functions

;; Get the last token ID
(define-read-only (get-last-token-id)
    (ok (var-get last-token-id))
)

;; Get token URI
(define-read-only (get-token-uri (token-id uint))
    (ok (some (concat base-token-uri (uint-to-string token-id))))
)

;; Get token owner
(define-read-only (get-owner (token-id uint))
    (ok (nft-get-owner? my-nft token-id))
)

;; Transfer function
(define-public (transfer (token-id uint) (sender principal) (recipient principal))
    (begin
        ;; Check that sender owns the token
        (asserts! (is-eq sender tx-sender) err-not-token-owner)
        
        ;; Transfer the NFT
        (match (nft-transfer? my-nft token-id sender recipient)
            success (ok success)
            error (err error)
        )
    )
)

;; Additional Minting Function

;; Mint new token (only contract owner)
(define-public (mint (recipient principal))
    (let
        (
            (new-token-id (+ (var-get last-token-id) u1))
        )
        ;; Only contract owner can mint
        (asserts! (is-eq tx-sender contract-owner) err-owner-only)
        
        ;; Mint the NFT
        (match (nft-mint? my-nft new-token-id recipient)
            success
                (begin
                    ;; Update last token ID
                    (var-set last-token-id new-token-id)
                    (ok new-token-id)
                )
            error (err error)
        )
    )
)

;; Helper function to convert uint to string
(define-read-only (uint-to-string (value uint))
    ;; Simplified - real implementation would handle conversion
    (if (is-eq value u0) "0"
    (if (is-eq value u1) "1"
    (if (is-eq value u2) "2"
    ;; ... more conditions
    "unknown")))
)
```

---

## 🔍 Code Explanation

### 1. Trait Implementation

```clarity
(impl-trait 'SP2PABAF9FTAJYNFZH93XENAJ8FVY99RRM50D2JG9.nft-trait.nft-trait)
```

This declares that your contract follows the SIP-009 standard. The trait is deployed on mainnet at this address.

### 2. NFT Definition

```clarity
(define-non-fungible-token my-nft uint)
```

Defines your NFT collection. Each token is identified by a `uint` (unsigned integer).

### 3. Storage Variables

```clarity
(define-data-var last-token-id uint u0)
(define-constant contract-owner tx-sender)
```

- `last-token-id`: Tracks the highest minted token ID
- `contract-owner`: Stores the deployer's address (immutable)

### 4. Error Handling

```clarity
(define-constant err-owner-only (err u100))
```

Clear error codes make debugging easier and provide better user feedback.

### 5. Transfer Function

```clarity
(define-public (transfer (token-id uint) (sender principal) (recipient principal))
    (begin
        (asserts! (is-eq sender tx-sender) err-not-token-owner)
        (match (nft-transfer? my-nft token-id sender recipient)
            success (ok success)
            error (err error)
        )
    )
)
```

**Security Check:** Only the sender can initiate transfers of their own tokens.

### 6. Mint Function

```clarity
(define-public (mint (recipient principal))
    (let ((new-token-id (+ (var-get last-token-id) u1)))
        (asserts! (is-eq tx-sender contract-owner) err-owner-only)
        (match (nft-mint? my-nft new-token-id recipient)
            success (begin
                (var-set last-token-id new-token-id)
                (ok new-token-id)
            )
            error (err error)
        )
    )
)
```

**Key Pattern:** Increment ID → Check permissions → Mint → Update storage

---

## 🌐 Frontend Integration

### React Component with Stacks.js

```javascript
import { useEffect, useState } from 'react';
import { 
  openContractCall,
  showConnect,
  AppConfig,
  UserSession
} from '@stacks/connect';
import { 
  PostConditionMode,
  uintCV,
  standardPrincipalCV
} from '@stacks/transactions';

function NFTMinter() {
  const [userSession] = useState(() => 
    new UserSession({ appConfig: new AppConfig(['store_write']) })
  );
  const [userData, setUserData] = useState(null);

  // Check if wallet is connected
  useEffect(() => {
    if (userSession.isUserSignedIn()) {
      setUserData(userSession.loadUserData());
    }
  }, []);

  // Connect wallet
  const connectWallet = () => {
    showConnect({
      appDetails: {
        name: 'My NFT App',
        icon: window.location.origin + '/logo.png',
      },
      userSession,
      onFinish: () => {
        setUserData(userSession.loadUserData());
      },
    });
  };

  // Mint NFT function
  const mintNFT = async (recipientAddress) => {
    const functionArgs = [
      standardPrincipalCV(recipientAddress)
    ];

    const options = {
      contractAddress: 'ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM',
      contractName: 'my-nft',
      functionName: 'mint',
      functionArgs: functionArgs,
      network: 'testnet', // or 'mainnet'
      postConditionMode: PostConditionMode.Deny,
      onFinish: (data) => {
        console.log('Transaction ID:', data.txId);
        alert(`NFT minted! TX: ${data.txId}`);
      },
    };

    await openContractCall(options);
  };

  // Transfer NFT function
  const transferNFT = async (tokenId, recipient) => {
    const senderAddress = userData.profile.stxAddress.testnet;
    
    const functionArgs = [
      uintCV(tokenId),
      standardPrincipalCV(senderAddress),
      standardPrincipalCV(recipient)
    ];

    const options = {
      contractAddress: 'ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM',
      contractName: 'my-nft',
      functionName: 'transfer',
      functionArgs: functionArgs,
      network: 'testnet',
      postConditionMode: PostConditionMode.Deny,
      onFinish: (data) => {
        console.log('Transfer TX:', data.txId);
        alert(`NFT transferred! TX: ${data.txId}`);
      },
    };

    await openContractCall(options);
  };

  return (
    <div>
      {!userData ? (
        <button onClick={connectWallet}>
          Connect Wallet
        </button>
      ) : (
        <div>
          <h2>Connected: {userData.profile.stxAddress.testnet}</h2>
          <button onClick={() => mintNFT(userData.profile.stxAddress.testnet)}>
            Mint NFT to Yourself
          </button>
        </div>
      )}
    </div>
  );
}

export default NFTMinter;
```

### Reading NFT Data

```javascript
import { callReadOnlyFunction, cvToValue } from '@stacks/transactions';

async function getNFTOwner(tokenId) {
  const options = {
    contractAddress: 'ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM',
    contractName: 'my-nft',
    functionName: 'get-owner',
    functionArgs: [uintCV(tokenId)],
    network: 'testnet',
    senderAddress: 'ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM',
  };

  const result = await callReadOnlyFunction(options);
  const owner = cvToValue(result);
  return owner;
}

async function getLastTokenId() {
  const options = {
    contractAddress: 'ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM',
    contractName: 'my-nft',
    functionName: 'get-last-token-id',
    functionArgs: [],
    network: 'testnet',
    senderAddress: 'ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM',
  };

  const result = await callReadOnlyFunction(options);
  return cvToValue(result);
}
```

---

## 🧪 Testing with Clarinet

### Test File: `tests/nft_test.ts`

```typescript
import { Clarinet, Tx, Chain, Account, types } from 'https://deno.land/x/clarinet@v1.0.0/index.ts';
import { assertEquals } from 'https://deno.land/std@0.90.0/testing/asserts.ts';

Clarinet.test({
  name: "Can mint new NFT",
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
    
    // Check that mint succeeded
    block.receipts[0].result.expectOk().expectUint(1);
    
    // Verify owner
    let ownerCheck = chain.callReadOnlyFn(
      'my-nft',
      'get-owner',
      [types.uint(1)],
      deployer.address
    );
    
    ownerCheck.result
      .expectOk()
      .expectSome()
      .expectPrincipal(recipient.address);
  }
});

Clarinet.test({
  name: "Can transfer NFT",
  async fn(chain: Chain, accounts: Map<string, Account>) {
    const deployer = accounts.get('deployer')!;
    const wallet1 = accounts.get('wallet_1')!;
    const wallet2 = accounts.get('wallet_2')!;
    
    // Mint NFT
    let block = chain.mineBlock([
      Tx.contractCall(
        'my-nft',
        'mint',
        [types.principal(wallet1.address)],
        deployer.address
      )
    ]);
    
    // Transfer NFT
    block = chain.mineBlock([
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
    let ownerCheck = chain.callReadOnlyFn(
      'my-nft',
      'get-owner',
      [types.uint(1)],
      deployer.address
    );
    
    ownerCheck.result
      .expectOk()
      .expectSome()
      .expectPrincipal(wallet2.address);
  }
});

Clarinet.test({
  name: "Non-owner cannot transfer",
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
    
    // Should fail
    block.receipts[0].result.expectErr().expectUint(101);
  }
});
```

---

## ⚠️ Common Pitfalls

### 1. **Not Checking Ownership**

❌ **Wrong:**
```clarity
(define-public (transfer (token-id uint) (sender principal) (recipient principal))
    (nft-transfer? my-nft token-id sender recipient)
)
```

✅ **Correct:**
```clarity
(define-public (transfer (token-id uint) (sender principal) (recipient principal))
    (begin
        (asserts! (is-eq sender tx-sender) err-not-token-owner)
        (nft-transfer? my-nft token-id sender recipient)
    )
)
```

### 2. **Not Incrementing Token IDs**

❌ **Wrong:**
```clarity
(nft-mint? my-nft u1 recipient)  ;; Always mints ID 1
```

✅ **Correct:**
```clarity
(let ((new-id (+ (var-get last-token-id) u1)))
    (nft-mint? my-nft new-id recipient)
    (var-set last-token-id new-id)
)
```

### 3. **Missing Error Handling**

❌ **Wrong:**
```clarity
(nft-mint? my-nft token-id recipient)
```

✅ **Correct:**
```clarity
(match (nft-mint? my-nft token-id recipient)
    success (ok success)
    error (err error)
)
```

---

## 🚀 Deployment Checklist

### Pre-Deployment

- [ ] All tests passing
- [ ] Error codes documented
- [ ] Metadata URIs tested
- [ ] Contract reviewed
- [ ] Testnet deployment successful

### Deployment Steps

1. **Deploy to Testnet**
```bash
clarinet deploy --testnet
```

2. **Verify Contract**
   - Test all functions
   - Check metadata URIs
   - Verify ownership transfers

3. **Deploy to Mainnet**
```bash
clarinet deploy --mainnet
```

### Post-Deployment

- [ ] Contract address recorded
- [ ] Explorer verified
- [ ] Frontend updated with contract address
- [ ] Metadata hosting confirmed

---

## 📚 Additional Resources

- **SIP-009 Specification**: Full trait definition
- **Example Projects**: 
  - lunarcrush/NKMT_1
  - friedger/clarity-smart-contracts
- **Tools**: Clarinet, Stacks Explorer, Hiro Platform

---

## 🎓 Next Steps

1. **Add Metadata**: Create JSON metadata files
2. **Build Marketplace**: Allow buying/selling
3. **Add Royalties**: Implement creator fees
4. **Frontend UI**: Build minting interface

**Related Guides:**
- → NFT Metadata Standards
- → Building an NFT Marketplace
- → Advanced NFT Features