# Escrow System Implementation Guide

## 📖 Overview

An escrow system holds assets (STX or tokens) in a smart contract until specific conditions are met. This enables trustless peer-to-peer transactions, protecting both buyers and sellers.

### What You'll Learn
- Escrow contract architecture  
- Creating and funding escrows
- Release and refund mechanisms
- Dispute resolution
- Time-locked releases
- Multi-party escrows

### Prerequisites
- Basic Clarity syntax
- Understanding of principals
- Familiarity with maps and variables

---

## 🎯 Key Concepts

### Escrow Lifecycle

1. **Creation**: Buyer creates escrow with terms
2. **Funding**: Buyer deposits funds
3. **Fulfillment**: Seller delivers goods/services
4. **Release**: Funds released to seller
5. **OR Refund**: Funds returned to buyer

### Escrow Types

**Simple Escrow**
- Two parties (buyer, seller)
- One-time release

**Time-Locked Escrow**
- Automatic release after deadline
- Prevents indefinite holds

**Multi-Party Escrow**
- Multiple beneficiaries
- Percentage-based splits

**Milestone Escrow**
- Phased releases
- Project-based payments

---

## 💻 Basic Escrow Contract

### Complete Implementation

```clarity
;; Simple Escrow Contract
;; Holds STX until release conditions are met

;; Constants
(define-constant contract-owner tx-sender)
(define-constant err-not-authorized (err u100))
(define-constant err-escrow-not-found (err u101))
(define-constant err-already-funded (err u102))
(define-constant err-not-funded (err u103))
(define-constant err-escrow-expired (err u104))
(define-constant err-escrow-not-expired (err u105))
(define-constant err-already-released (err u106))
(define-constant err-insufficient-funds (err u107))

;; Data variables
(define-data-var escrow-nonce uint u0)

;; Escrow data structure
(define-map escrows
    uint ;; escrow-id
    {
        buyer: principal,
        seller: principal,
        amount: uint,
        funded: bool,
        released: bool,
        expiry-block: uint,
        created-at: uint
    }
)

;; -----------------------------------------------------------
;; Escrow Creation & Funding
;; -----------------------------------------------------------

;; Create a new escrow
(define-public (create-escrow
    (seller principal)
    (amount uint)
    (expiry-blocks uint)
)
    (let
        (
            (escrow-id (+ (var-get escrow-nonce) u1))
            (expiry (+ block-height expiry-blocks))
        )
        ;; Validate amount
        (asserts! (> amount u0) err-insufficient-funds)
        
        ;; Create escrow
        (map-set escrows escrow-id {
            buyer: tx-sender,
            seller: seller,
            amount: amount,
            funded: false,
            released: false,
            expiry-block: expiry,
            created-at: block-height
        })
        
        ;; Increment nonce
        (var-set escrow-nonce escrow-id)
        
        (print {
            event: "escrow-created",
            escrow-id: escrow-id,
            buyer: tx-sender,
            seller: seller,
            amount: amount,
            expiry: expiry
        })
        
        (ok escrow-id)
    )
)

;; Fund an escrow (buyer only)
(define-public (fund-escrow (escrow-id uint))
    (let
        (
            (escrow (unwrap! (map-get? escrows escrow-id) err-escrow-not-found))
        )
        ;; Only buyer can fund
        (asserts! (is-eq tx-sender (get buyer escrow)) err-not-authorized)
        
        ;; Check not already funded
        (asserts! (not (get funded escrow)) err-already-funded)
        
        ;; Check not expired
        (asserts! (< block-height (get expiry-block escrow)) err-escrow-expired)
        
        ;; Transfer STX to contract
        (try! (stx-transfer? (get amount escrow) tx-sender (as-contract tx-sender)))
        
        ;; Mark as funded
        (map-set escrows escrow-id
            (merge escrow { funded: true })
        )
        
        (print {
            event: "escrow-funded",
            escrow-id: escrow-id,
            amount: (get amount escrow)
        })
        
        (ok true)
    )
)

;; -----------------------------------------------------------
;; Release & Refund Functions
;; -----------------------------------------------------------

;; Release funds to seller (buyer only)
(define-public (release-escrow (escrow-id uint))
    (let
        (
            (escrow (unwrap! (map-get? escrows escrow-id) err-escrow-not-found))
        )
        ;; Only buyer can release
        (asserts! (is-eq tx-sender (get buyer escrow)) err-not-authorized)
        
        ;; Must be funded
        (asserts! (get funded escrow) err-not-funded)
        
        ;; Not already released
        (asserts! (not (get released escrow)) err-already-released)
        
        ;; Transfer funds to seller
        (try! (as-contract (stx-transfer? 
            (get amount escrow)
            tx-sender
            (get seller escrow)
        )))
        
        ;; Mark as released
        (map-set escrows escrow-id
            (merge escrow { released: true })
        )
        
        (print {
            event: "escrow-released",
            escrow-id: escrow-id,
            seller: (get seller escrow),
            amount: (get amount escrow)
        })
        
        (ok true)
    )
)

;; Refund to buyer (buyer can refund after expiry, or seller can initiate refund)
(define-public (refund-escrow (escrow-id uint))
    (let
        (
            (escrow (unwrap! (map-get? escrows escrow-id) err-escrow-not-found))
            (is-buyer (is-eq tx-sender (get buyer escrow)))
            (is-seller (is-eq tx-sender (get seller escrow)))
        )
        ;; Must be funded
        (asserts! (get funded escrow) err-not-funded)
        
        ;; Not already released
        (asserts! (not (get released escrow)) err-already-released)
        
        ;; Authorization logic:
        ;; - Buyer can refund after expiry
        ;; - Seller can initiate refund anytime (goodwill gesture)
        (asserts! 
            (or 
                (and is-buyer (>= block-height (get expiry-block escrow)))
                is-seller
            )
            err-not-authorized
        )
        
        ;; Return funds to buyer
        (try! (as-contract (stx-transfer? 
            (get amount escrow)
            tx-sender
            (get buyer escrow)
        )))
        
        ;; Mark as released (to prevent double-spending)
        (map-set escrows escrow-id
            (merge escrow { released: true })
        )
        
        (print {
            event: "escrow-refunded",
            escrow-id: escrow-id,
            buyer: (get buyer escrow),
            amount: (get amount escrow)
        })
        
        (ok true)
    )
)

;; -----------------------------------------------------------
;; Read-Only Functions
;; -----------------------------------------------------------

;; Get escrow details
(define-read-only (get-escrow (escrow-id uint))
    (ok (map-get? escrows escrow-id))
)

;; Check if escrow is active
(define-read-only (is-active (escrow-id uint))
    (match (map-get? escrows escrow-id)
        escrow (ok (and 
            (get funded escrow)
            (not (get released escrow))
            (< block-height (get expiry-block escrow))
        ))
        (err err-escrow-not-found)
    )
)

;; Get current escrow nonce
(define-read-only (get-escrow-nonce)
    (ok (var-get escrow-nonce))
)
```

---

## 🔧 Advanced Features

### 1. Milestone-Based Escrow

```clarity
;; Milestone escrow for project-based payments
(define-map milestone-escrows
    uint ;; escrow-id
    {
        buyer: principal,
        seller: principal,
        total-amount: uint,
        milestones: (list 10 {
            amount: uint,
            released: bool,
            description: (string-ascii 256)
        }),
        funded: bool,
        expiry-block: uint
    }
)

;; Release specific milestone
(define-public (release-milestone (escrow-id uint) (milestone-index uint))
    (let
        (
            (escrow (unwrap! (map-get? milestone-escrows escrow-id) err-escrow-not-found))
            (milestones (get milestones escrow))
            (milestone (unwrap! (element-at milestones milestone-index) err-escrow-not-found))
        )
        ;; Only buyer can release
        (asserts! (is-eq tx-sender (get buyer escrow)) err-not-authorized)
        
        ;; Check not already released
        (asserts! (not (get released milestone)) err-already-released)
        
        ;; Transfer milestone amount
        (try! (as-contract (stx-transfer? 
            (get amount milestone)
            tx-sender
            (get seller escrow)
        )))
        
        ;; Update milestone status
        ;; (Implementation would update the milestone list)
        
        (ok true)
    )
)
```

### 2. Multi-Party Escrow

```clarity
;; Escrow with multiple beneficiaries
(define-map multi-party-escrows
    uint
    {
        buyer: principal,
        total-amount: uint,
        beneficiaries: (list 5 {
            recipient: principal,
            percentage: uint, ;; Basis points (e.g., 2500 = 25%)
            released: bool
        }),
        funded: bool,
        released: bool,
        expiry-block: uint
    }
)

;; Release funds to all beneficiaries
(define-public (release-multi-party (escrow-id uint))
    (let
        (
            (escrow (unwrap! (map-get? multi-party-escrows escrow-id) err-escrow-not-found))
        )
        ;; Only buyer can release
        (asserts! (is-eq tx-sender (get buyer escrow)) err-not-authorized)
        
        ;; Check funded and not released
        (asserts! (get funded escrow) err-not-funded)
        (asserts! (not (get released escrow)) err-already-released)
        
        ;; Transfer to each beneficiary
        (try! (distribute-funds escrow-id))
        
        ;; Mark as released
        (map-set multi-party-escrows escrow-id
            (merge escrow { released: true })
        )
        
        (ok true)
    )
)

;; Helper function to distribute funds
(define-private (distribute-funds (escrow-id uint))
    (let
        (
            (escrow (unwrap! (map-get? multi-party-escrows escrow-id) err-escrow-not-found))
            (total (get total-amount escrow))
        )
        ;; Would iterate through beneficiaries and transfer percentages
        ;; Implementation would use fold or similar
        (ok true)
    )
)
```

### 3. Arbitrated Escrow

```clarity
;; Escrow with arbiter for dispute resolution
(define-map arbitrated-escrows
    uint
    {
        buyer: principal,
        seller: principal,
        arbiter: principal,
        amount: uint,
        funded: bool,
        released: bool,
        disputed: bool,
        expiry-block: uint
    }
)

;; Raise dispute
(define-public (raise-dispute (escrow-id uint))
    (let
        (
            (escrow (unwrap! (map-get? arbitrated-escrows escrow-id) err-escrow-not-found))
        )
        ;; Only buyer or seller can raise dispute
        (asserts! (or 
            (is-eq tx-sender (get buyer escrow))
            (is-eq tx-sender (get seller escrow))
        ) err-not-authorized)
        
        ;; Must be funded and not released
        (asserts! (get funded escrow) err-not-funded)
        (asserts! (not (get released escrow)) err-already-released)
        
        ;; Mark as disputed
        (map-set arbitrated-escrows escrow-id
            (merge escrow { disputed: true })
        )
        
        (ok true)
    )
)

;; Resolve dispute (arbiter only)
(define-public (resolve-dispute 
    (escrow-id uint)
    (release-to-seller bool)
)
    (let
        (
            (escrow (unwrap! (map-get? arbitrated-escrows escrow-id) err-escrow-not-found))
            (recipient (if release-to-seller 
                (get seller escrow)
                (get buyer escrow)
            ))
        )
        ;; Only arbiter can resolve
        (asserts! (is-eq tx-sender (get arbiter escrow)) err-not-authorized)
        
        ;; Must be disputed
        (asserts! (get disputed escrow) err-not-authorized)
        
        ;; Transfer funds
        (try! (as-contract (stx-transfer? 
            (get amount escrow)
            tx-sender
            recipient
        )))
        
        ;; Mark as released
        (map-set arbitrated-escrows escrow-id
            (merge escrow { released: true, disputed: false })
        )
        
        (print {
            event: "dispute-resolved",
            escrow-id: escrow-id,
            winner: recipient,
            amount: (get amount escrow)
        })
        
        (ok true)
    )
)
```

---

## 🌐 Frontend Integration

### Complete Escrow UI

```javascript
import React, { useState, useEffect } from 'react';
import { useWallet } from './WalletContext';
import {
  openContractCall,
  makeStandardSTXPostCondition,
  FungibleConditionCode,
  uintCV,
  standardPrincipalCV
} from '@stacks/transactions';
import { StacksTestnet } from '@stacks/network';

function EscrowDashboard() {
  const { address, isConnected } = useWallet();
  const [myEscrows, setMyEscrows] = useState([]);
  const [showCreateModal, setShowCreateModal] = useState(false);

  const ESCROW_CONTRACT = 'ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM.escrow';

  return (
    <div className="escrow-dashboard">
      <h1>Escrow Dashboard</h1>

      {isConnected && (
        <>
          <button onClick={() => setShowCreateModal(true)}>
            Create New Escrow
          </button>

          {showCreateModal && (
            <CreateEscrowModal 
              onClose={() => setShowCreateModal(false)}
              address={address}
            />
          )}

          <div className="escrow-list">
            <h2>My Escrows</h2>
            {myEscrows.length === 0 ? (
              <p>No escrows found</p>
            ) : (
              myEscrows.map((escrow) => (
                <EscrowCard key={escrow.id} escrow={escrow} userAddress={address} />
              ))
            )}
          </div>
        </>
      )}
    </div>
  );
}

// Create Escrow Modal
function CreateEscrowModal({ onClose, address }) {
  const [seller, setSeller] = useState('');
  const [amount, setAmount] = useState('');
  const [expiryDays, setExpiryDays] = useState('7');

  const createEscrow = async () => {
    const amountInMicroSTX = parseFloat(amount) * 1000000;
    const expiryBlocks = parseInt(expiryDays) * 144; // ~144 blocks per day

    const options = {
      network: new StacksTestnet(),
      contractAddress: 'ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM',
      contractName: 'escrow',
      functionName: 'create-escrow',
      functionArgs: [
        standardPrincipalCV(seller),
        uintCV(amountInMicroSTX),
        uintCV(expiryBlocks)
      ],
      postConditionMode: PostConditionMode.Deny,
      onFinish: (data) => {
        alert(`Escrow created! TX: ${data.txId}`);
        onClose();
      },
    };

    await openContractCall(options);
  };

  return (
    <div className="modal">
      <div className="modal-content">
        <h2>Create Escrow</h2>

        <div className="form-group">
          <label>Seller Address</label>
          <input
            type="text"
            placeholder="ST1..."
            value={seller}
            onChange={(e) => setSeller(e.target.value)}
          />
        </div>

        <div className="form-group">
          <label>Amount (STX)</label>
          <input
            type="number"
            step="0.01"
            placeholder="10.0"
            value={amount}
            onChange={(e) => setAmount(e.target.value)}
          />
        </div>

        <div className="form-group">
          <label>Expiry (Days)</label>
          <input
            type="number"
            placeholder="7"
            value={expiryDays}
            onChange={(e) => setExpiryDays(e.target.value)}
          />
        </div>

        <div className="modal-actions">
          <button onClick={createEscrow} className="create-btn">
            Create Escrow
          </button>
          <button onClick={onClose} className="cancel-btn">
            Cancel
          </button>
        </div>
      </div>
    </div>
  );
}

// Escrow Card Component
function EscrowCard({ escrow, userAddress }) {
  const isBuyer = escrow.buyer === userAddress;
  const isSeller = escrow.seller === userAddress;

  const fundEscrow = async () => {
    const postCondition = makeStandardSTXPostCondition(
      userAddress,
      FungibleConditionCode.Equal,
      escrow.amount
    );

    const options = {
      network: new StacksTestnet(),
      contractAddress: 'ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM',
      contractName: 'escrow',
      functionName: 'fund-escrow',
      functionArgs: [uintCV(escrow.id)],
      postConditions: [postCondition],
      postConditionMode: PostConditionMode.Deny,
      onFinish: (data) => {
        alert(`Escrow funded! TX: ${data.txId}`);
      },
    };

    await openContractCall(options);
  };

  const releaseEscrow = async () => {
    const options = {
      network: new StacksTestnet(),
      contractAddress: 'ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM',
      contractName: 'escrow',
      functionName: 'release-escrow',
      functionArgs: [uintCV(escrow.id)],
      postConditionMode: PostConditionMode.Deny,
      onFinish: (data) => {
        alert(`Escrow released! TX: ${data.txId}`);
      },
    };

    await openContractCall(options);
  };

  const refundEscrow = async () => {
    const options = {
      network: new StacksTestnet(),
      contractAddress: 'ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM',
      contractName: 'escrow',
      functionName: 'refund-escrow',
      functionArgs: [uintCV(escrow.id)],
      postConditionMode: PostConditionMode.Deny,
      onFinish: (data) => {
        alert(`Escrow refunded! TX: ${data.txId}`);
      },
    };

    await openContractCall(options);
  };

  return (
    <div className="escrow-card">
      <div className="escrow-header">
        <h3>Escrow #{escrow.id}</h3>
        <span className={`status ${escrow.status}`}>{escrow.status}</span>
      </div>

      <div className="escrow-details">
        <p><strong>Amount:</strong> {(escrow.amount / 1000000).toFixed(2)} STX</p>
        <p><strong>Buyer:</strong> {escrow.buyer.slice(0, 8)}...</p>
        <p><strong>Seller:</strong> {escrow.seller.slice(0, 8)}...</p>
        <p><strong>Expiry:</strong> Block {escrow.expiryBlock}</p>
        <p><strong>Funded:</strong> {escrow.funded ? 'Yes' : 'No'}</p>
        <p><strong>Released:</strong> {escrow.released ? 'Yes' : 'No'}</p>
      </div>

      <div className="escrow-actions">
        {isBuyer && !escrow.funded && (
          <button onClick={fundEscrow} className="fund-btn">
            Fund Escrow
          </button>
        )}

        {isBuyer && escrow.funded && !escrow.released && (
          <button onClick={releaseEscrow} className="release-btn">
            Release to Seller
          </button>
        )}

        {isBuyer && escrow.funded && !escrow.released && (
          <button onClick={refundEscrow} className="refund-btn">
            Request Refund
          </button>
        )}

        {isSeller && escrow.funded && !escrow.released && (
          <button onClick={refundEscrow} className="cancel-btn">
            Cancel & Refund
          </button>
        )}
      </div>
    </div>
  );
}

export default EscrowDashboard;
```

---

## 🧪 Testing

```typescript
Clarinet.test({
  name: "Can create and fund escrow",
  async fn(chain: Chain, accounts: Map<string, Account>) {
    const buyer = accounts.get('wallet_1')!;
    const seller = accounts.get('wallet_2')!;
    
    // Create escrow
    let block = chain.mineBlock([
      Tx.contractCall(
        'escrow',
        'create-escrow',
        [
          types.principal(seller.address),
          types.uint(1000000), // 1 STX
          types.uint(144) // 1 day expiry
        ],
        buyer.address
      )
    ]);
    
    const escrowId = block.receipts[0].result.expectOk().expectUint(1);
    
    // Fund escrow
    block = chain.mineBlock([
      Tx.contractCall(
        'escrow',
        'fund-escrow',
        [types.uint(escrowId)],
        buyer.address
      )
    ]);
    
    block.receipts[0].result.expectOk();
  }
});

Clarinet.test({
  name: "Buyer can release funds to seller",
  async fn(chain: Chain, accounts: Map<string, Account>) {
    const buyer = accounts.get('wallet_1')!;
    const seller = accounts.get('wallet_2')!;
    
    // Create and fund
    chain.mineBlock([
      Tx.contractCall('escrow', 'create-escrow',
        [types.principal(seller.address), types.uint(1000000), types.uint(144)],
        buyer.address
      )
    ]);
    
    chain.mineBlock([
      Tx.contractCall('escrow', 'fund-escrow', [types.uint(1)], buyer.address)
    ]);
    
    // Release
    let block = chain.mineBlock([
      Tx.contractCall('escrow', 'release-escrow', [types.uint(1)], buyer.address)
    ]);
    
    block.receipts[0].result.expectOk();
  }
});
```

---

## ⚠️ Security Considerations

1. **Reentrancy Protection**: Use checks-effects-interactions pattern
2. **Expiry Validation**: Always check block height
3. **Access Control**: Verify caller authorization
4. **Amount Validation**: Check for zero amounts
5. **State Management**: Prevent double-spending

---

## 🎓 Use Cases

- **Freelance Payments**: Milestone-based releases
- **P2P Trading**: Secure asset exchanges
- **Service Payments**: Payment on delivery
- **Rental Deposits**: Refundable security deposits
- **Crowdfunding**: Conditional fund releases

---

## 📚 Next Steps

**Related Guides:**
- → Multi-Signature Wallets
- → Payment Streaming
- → Dispute Resolution Systems