# Multi-Signature Wallets - Complete Guide

## 📖 Overview

A multi-signature (multisig) wallet requires multiple parties to approve transactions before execution. This provides enhanced security for treasuries, DAOs, and shared funds by eliminating single points of failure.

### What You'll Learn
- Multisig wallet architecture
- Proposal and confirmation system
- Owner management
- Transaction execution
- Threshold configuration
- Executor pattern for upgradability

### Prerequisites
- Understanding of principals and access control
- Contract traits
- Transaction handling

---

## 🎯 Key Concepts

### Multisig Components

**1. Owners**
- List of authorized signers
- Add/remove owners requires approval
- Each owner has equal voting power

**2. Threshold**
- Minimum confirmations required
- Example: 3-of-5 means 3 out of 5 owners must approve
- Configurable per wallet

**3. Transactions**
- Proposals submitted by owners
- Require threshold confirmations
- Execute once threshold met

**4. Executor Pattern**
- Separate contracts for different operations
- Add owner, remove owner, transfer funds
- Modular and upgradable

---

## 💻 Complete Multisig Wallet Contract

### Main Safe Contract

```clarity
;; Multi-Signature Wallet (Safe)
;; Based on Trust Machines MultiSafe pattern

;; Traits
(define-trait executor-trait
    (
        (execute (principal <safe-trait>) (response bool uint))
    )
)

(define-trait safe-trait
    (
        (get-owners () (response (list 20 principal) uint))
        (get-threshold () (response uint uint))
    )
)

;; Constants
(define-constant err-owner-only (err u100))
(define-constant err-already-owner (err u101))
(define-constant err-not-owner (err u102))
(define-constant err-tx-not-found (err u103))
(define-constant err-already-confirmed (err u104))
(define-constant err-threshold-not-met (err u105))
(define-constant err-tx-already-executed (err u106))
(define-constant err-invalid-threshold (err u107))
(define-constant err-too-many-owners (err u108))

;; Maximum owners
(define-constant max-owners u20)

;; Data variables
(define-data-var owners (list 20 principal) (list))
(define-data-var threshold uint u0)
(define-data-var nonce uint u0)

;; Transaction data
(define-map transactions
    uint  ;; tx-id
    {
        executor: principal,
        param-p: (optional principal),
        param-u: (optional uint),
        param-b: (optional (buff 20)),
        confirmations: uint,
        executed: bool,
        proposer: principal
    }
)

;; Track confirmations
(define-map confirmations
    { tx-id: uint, owner: principal }
    bool
)

;; -----------------------------------------------------------
;; Initialization
;; -----------------------------------------------------------

;; Initialize safe with owners and threshold
(define-public (init (initial-owners (list 20 principal)) (initial-threshold uint))
    (begin
        ;; Can only init once
        (asserts! (is-eq (len (var-get owners)) u0) (err u200))
        
        ;; Validate threshold
        (asserts! (> initial-threshold u0) err-invalid-threshold)
        (asserts! (<= initial-threshold (len initial-owners)) err-invalid-threshold)
        
        ;; Set owners and threshold
        (var-set owners initial-owners)
        (var-set threshold initial-threshold)
        
        (ok true)
    )
)

;; -----------------------------------------------------------
;; Owner Management
;; -----------------------------------------------------------

;; Check if address is owner
(define-read-only (is-owner (address principal))
    (is-some (index-of (var-get owners) address))
)

;; Get all owners
(define-read-only (get-owners)
    (ok (var-get owners))
)

;; Get threshold
(define-read-only (get-threshold)
    (ok (var-get threshold))
)

;; Internal: Add owner (called by executor)
(define-public (add-owner-internal (new-owner principal))
    (let
        (
            (current-owners (var-get owners))
        )
        ;; Check not already owner
        (asserts! (not (is-owner new-owner)) err-already-owner)
        
        ;; Check max owners
        (asserts! (< (len current-owners) max-owners) err-too-many-owners)
        
        ;; Add owner
        (var-set owners (unwrap! (as-max-len? (append current-owners new-owner) u20) err-too-many-owners))
        
        (print { event: "owner-added", owner: new-owner })
        (ok true)
    )
)

;; Internal: Remove owner (called by executor)
(define-public (remove-owner-internal (owner-to-remove principal))
    (let
        (
            (current-owners (var-get owners))
            (current-threshold (var-get threshold))
        )
        ;; Check is owner
        (asserts! (is-owner owner-to-remove) err-not-owner)
        
        ;; Remove owner
        (var-set owners (filter is-not-removed current-owners))
        
        ;; Adjust threshold if needed
        (if (> current-threshold (len (var-get owners)))
            (var-set threshold (len (var-get owners)))
            true
        )
        
        (print { event: "owner-removed", owner: owner-to-remove })
        (ok true)
    )
)

;; Helper for filtering
(define-private (is-not-removed (owner principal))
    true  ;; Simplified - would check against removal target
)

;; Internal: Set threshold (called by executor)
(define-public (set-threshold-internal (new-threshold uint))
    (let
        (
            (owner-count (len (var-get owners)))
        )
        ;; Validate threshold
        (asserts! (> new-threshold u0) err-invalid-threshold)
        (asserts! (<= new-threshold owner-count) err-invalid-threshold)
        
        (var-set threshold new-threshold)
        
        (print { event: "threshold-changed", threshold: new-threshold })
        (ok true)
    )
)

;; -----------------------------------------------------------
;; Transaction Management
;; -----------------------------------------------------------

;; Submit new transaction
(define-public (submit
    (executor <executor-trait>)
    (param-p (optional principal))
    (param-u (optional uint))
    (param-b (optional (buff 20)))
)
    (let
        (
            (tx-id (var-get nonce))
        )
        ;; Only owners can submit
        (asserts! (is-owner tx-sender) err-owner-only)
        
        ;; Create transaction
        (map-set transactions tx-id {
            executor: (contract-of executor),
            param-p: param-p,
            param-u: param-u,
            param-b: param-b,
            confirmations: u1,  ;; Proposer auto-confirms
            executed: false,
            proposer: tx-sender
        })
        
        ;; Record proposer's confirmation
        (map-set confirmations { tx-id: tx-id, owner: tx-sender } true)
        
        ;; Increment nonce
        (var-set nonce (+ tx-id u1))
        
        (print {
            event: "transaction-submitted",
            tx-id: tx-id,
            executor: (contract-of executor),
            proposer: tx-sender
        })
        
        (ok tx-id)
    )
)

;; Confirm transaction
(define-public (confirm (tx-id uint) (executor <executor-trait>))
    (let
        (
            (tx (unwrap! (map-get? transactions tx-id) err-tx-not-found))
            (current-threshold (var-get threshold))
        )
        ;; Only owners can confirm
        (asserts! (is-owner tx-sender) err-owner-only)
        
        ;; Check not already executed
        (asserts! (not (get executed tx)) err-tx-already-executed)
        
        ;; Check not already confirmed by this owner
        (asserts! (is-none (map-get? confirmations { tx-id: tx-id, owner: tx-sender })) 
            err-already-confirmed
        )
        
        ;; Verify executor matches
        (asserts! (is-eq (contract-of executor) (get executor tx)) (err u109))
        
        ;; Record confirmation
        (map-set confirmations { tx-id: tx-id, owner: tx-sender } true)
        
        ;; Update confirmation count
        (let
            (
                (new-confirmations (+ (get confirmations tx) u1))
            )
            (map-set transactions tx-id (merge tx { confirmations: new-confirmations }))
            
            (print {
                event: "transaction-confirmed",
                tx-id: tx-id,
                confirmer: tx-sender,
                confirmations: new-confirmations,
                threshold: current-threshold
            })
            
            ;; Execute if threshold met
            (if (>= new-confirmations current-threshold)
                (execute-transaction tx-id executor)
                (ok true)
            )
        )
    )
)

;; Execute transaction
(define-private (execute-transaction (tx-id uint) (executor <executor-trait>))
    (let
        (
            (tx (unwrap! (map-get? transactions tx-id) err-tx-not-found))
        )
        ;; Mark as executed
        (map-set transactions tx-id (merge tx { executed: true }))
        
        ;; Execute via executor contract
        (match (contract-call? executor execute (as-contract tx-sender))
            success
                (begin
                    (print {
                        event: "transaction-executed",
                        tx-id: tx-id,
                        success: true
                    })
                    (ok true)
                )
            error
                (begin
                    (print {
                        event: "transaction-execution-failed",
                        tx-id: tx-id,
                        error: error
                    })
                    (err error)
                )
        )
    )
)

;; Revoke confirmation (before execution)
(define-public (revoke (tx-id uint))
    (let
        (
            (tx (unwrap! (map-get? transactions tx-id) err-tx-not-found))
        )
        ;; Only owners can revoke
        (asserts! (is-owner tx-sender) err-owner-only)
        
        ;; Check not executed
        (asserts! (not (get executed tx)) err-tx-already-executed)
        
        ;; Check owner has confirmed
        (asserts! (is-some (map-get? confirmations { tx-id: tx-id, owner: tx-sender }))
            (err u110)
        )
        
        ;; Remove confirmation
        (map-delete confirmations { tx-id: tx-id, owner: tx-sender })
        
        ;; Update count
        (map-set transactions tx-id 
            (merge tx { confirmations: (- (get confirmations tx) u1) })
        )
        
        (print { event: "confirmation-revoked", tx-id: tx-id, owner: tx-sender })
        (ok true)
    )
)

;; -----------------------------------------------------------
;; View Functions
;; -----------------------------------------------------------

;; Get transaction details
(define-read-only (get-transaction (tx-id uint))
    (ok (map-get? transactions tx-id))
)

;; Check if owner confirmed
(define-read-only (has-confirmed (tx-id uint) (owner principal))
    (ok (default-to false (map-get? confirmations { tx-id: tx-id, owner: owner })))
)

;; Get next transaction ID
(define-read-only (get-nonce)
    (ok (var-get nonce))
)

;; Get confirmation count
(define-read-only (get-confirmations (tx-id uint))
    (match (map-get? transactions tx-id)
        tx (ok (get confirmations tx))
        err-tx-not-found
    )
)
```

---

## 🔧 Executor Contracts

### Add Owner Executor

```clarity
;; Executor: Add Owner
;; Allows multisig to add new owners

(impl-trait .safe.executor-trait)
(use-trait safe-trait .safe.safe-trait)

(define-constant err-execution-failed (err u200))

;; Execute: Add owner to safe
(define-public (execute (safe <safe-trait>) (new-owner principal))
    (begin
        ;; Call safe's internal add-owner function
        (try! (contract-call? safe add-owner-internal new-owner))
        
        (print {
            event: "add-owner-executed",
            safe: (contract-of safe),
            new-owner: new-owner
        })
        
        (ok true)
    )
)
```

### Remove Owner Executor

```clarity
;; Executor: Remove Owner

(impl-trait .safe.executor-trait)
(use-trait safe-trait .safe.safe-trait)

(define-public (execute (safe <safe-trait>) (owner-to-remove principal))
    (begin
        (try! (contract-call? safe remove-owner-internal owner-to-remove))
        
        (print {
            event: "remove-owner-executed",
            safe: (contract-of safe),
            removed-owner: owner-to-remove
        })
        
        (ok true)
    )
)
```

### Transfer STX Executor

```clarity
;; Executor: Transfer STX

(impl-trait .safe.executor-trait)
(use-trait safe-trait .safe.safe-trait)

(define-public (execute 
    (safe <safe-trait>)
    (recipient principal)
    (amount uint)
)
    (begin
        ;; Safe must call this as contract principal
        (try! (as-contract (stx-transfer? amount tx-sender recipient)))
        
        (print {
            event: "stx-transferred",
            safe: (contract-of safe),
            recipient: recipient,
            amount: amount
        })
        
        (ok true)
    )
)
```

---

## 🌐 Multisig Frontend Interface

```javascript
import React, { useState, useEffect } from 'react';
import { useWallet } from './WalletContext';
import {
  openContractCall,
  uintCV,
  principalCV,
  someCV,
  noneCV,
  listCV
} from '@stacks/transactions';

function MultisigDashboard() {
  const { address, isConnected } = useWallet();
  const [safeInfo, setSafeInfo] = useState(null);
  const [pendingTxs, setPendingTxs] = useState([]);
  const [isOwner, setIsOwner] = useState(false);

  const SAFE_CONTRACT = 'ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM.safe';

  useEffect(() => {
    if (isConnected) {
      loadSafeInfo();
      loadPendingTransactions();
    }
  }, [isConnected, address]);

  return (
    <div className="multisig-dashboard">
      <h1>MultiSig Wallet</h1>

      {safeInfo && (
        <SafeInfo info={safeInfo} />
      )}

      {isOwner && (
        <>
          <CreateProposal safeAddress={SAFE_CONTRACT} />
          <PendingTransactions 
            transactions={pendingTxs}
            address={address}
            safeAddress={SAFE_CONTRACT}
          />
        </>
      )}
    </div>
  );
}

function SafeInfo({ info }) {
  return (
    <div className="safe-info">
      <h2>Safe Information</h2>
      <div className="info-grid">
        <div className="info-item">
          <label>Owners:</label>
          <span>{info.owners.length}</span>
        </div>
        <div className="info-item">
          <label>Threshold:</label>
          <span>{info.threshold} of {info.owners.length}</span>
        </div>
        <div className="info-item">
          <label>Balance:</label>
          <span>{info.balance} STX</span>
        </div>
      </div>

      <div className="owners-list">
        <h3>Owners</h3>
        {info.owners.map((owner, idx) => (
          <div key={idx} className="owner">
            {owner.slice(0, 8)}...{owner.slice(-6)}
          </div>
        ))}
      </div>
    </div>
  );
}

function CreateProposal({ safeAddress }) {
  const [proposalType, setProposalType] = useState('transfer');
  const [recipient, setRecipient] = useState('');
  const [amount, setAmount] = useState('');
  const [newOwner, setNewOwner] = useState('');

  const submitProposal = async () => {
    let executorContract, params;

    if (proposalType === 'transfer') {
      executorContract = 'transfer-stx';
      params = {
        'param-p': someCV(principalCV(recipient)),
        'param-u': someCV(uintCV(parseFloat(amount) * 1000000)),
        'param-b': noneCV()
      };
    } else if (proposalType === 'add-owner') {
      executorContract = 'add-owner';
      params = {
        'param-p': someCV(principalCV(newOwner)),
        'param-u': noneCV(),
        'param-b': noneCV()
      };
    }

    const options = {
      network: new StacksTestnet(),
      contractAddress: 'ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM',
      contractName: 'safe',
      functionName: 'submit',
      functionArgs: [
        principalCV(`ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM.${executorContract}`),
        params['param-p'],
        params['param-u'],
        params['param-b']
      ],
      postConditionMode: PostConditionMode.Deny,
      onFinish: (data) => {
        alert(`Proposal submitted! TX: ${data.txId}`);
      },
    };

    await openContractCall(options);
  };

  return (
    <div className="create-proposal">
      <h2>Create Proposal</h2>

      <select value={proposalType} onChange={(e) => setProposalType(e.target.value)}>
        <option value="transfer">Transfer STX</option>
        <option value="add-owner">Add Owner</option>
        <option value="remove-owner">Remove Owner</option>
      </select>

      {proposalType === 'transfer' && (
        <>
          <input
            type="text"
            placeholder="Recipient address"
            value={recipient}
            onChange={(e) => setRecipient(e.target.value)}
          />
          <input
            type="number"
            placeholder="Amount (STX)"
            value={amount}
            onChange={(e) => setAmount(e.target.value)}
          />
        </>
      )}

      {proposalType === 'add-owner' && (
        <input
          type="text"
          placeholder="New owner address"
          value={newOwner}
          onChange={(e) => setNewOwner(e.target.value)}
        />
      )}

      <button onClick={submitProposal}>Submit Proposal</button>
    </div>
  );
}

function PendingTransactions({ transactions, address, safeAddress }) {
  const confirmTransaction = async (txId) => {
    const options = {
      network: new StacksTestnet(),
      contractAddress: 'ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM',
      contractName: 'safe',
      functionName: 'confirm',
      functionArgs: [
        uintCV(txId),
        principalCV('ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM.transfer-stx')
      ],
      postConditionMode: PostConditionMode.Deny,
      onFinish: (data) => {
        alert(`Transaction confirmed! TX: ${data.txId}`);
      },
    };

    await openContractCall(options);
  };

  return (
    <div className="pending-transactions">
      <h2>Pending Proposals</h2>

      {transactions.length === 0 ? (
        <p>No pending proposals</p>
      ) : (
        <table>
          <thead>
            <tr>
              <th>ID</th>
              <th>Type</th>
              <th>Details</th>
              <th>Confirmations</th>
              <th>Action</th>
            </tr>
          </thead>
          <tbody>
            {transactions.map((tx) => (
              <tr key={tx.id}>
                <td>{tx.id}</td>
                <td>{tx.type}</td>
                <td>{tx.details}</td>
                <td>
                  {tx.confirmations}/{tx.threshold}
                  <ProgressBar 
                    current={tx.confirmations} 
                    total={tx.threshold}
                  />
                </td>
                <td>
                  {!tx.confirmedByUser ? (
                    <button onClick={() => confirmTransaction(tx.id)}>
                      Confirm
                    </button>
                  ) : (
                    <span className="confirmed">✓ Confirmed</span>
                  )}
                </td>
              </tr>
            ))}
          </tbody>
        </table>
      )}
    </div>
  );
}

function ProgressBar({ current, total }) {
  const percentage = (current / total) * 100;
  
  return (
    <div className="progress-bar">
      <div 
        className="progress-fill" 
        style={{ width: `${percentage}%` }}
      />
    </div>
  );
}

export default MultisigDashboard;
```

---

## 🧪 Testing Multisig

```typescript
import { Clarinet, Tx, Chain, Account, types } from 'https://deno.land/x/clarinet@v1.0.0/index.ts';

Clarinet.test({
    name: "Can initialize safe with owners",
    async fn(chain: Chain, accounts: Map<string, Account>) {
        const deployer = accounts.get('deployer')!;
        const wallet1 = accounts.get('wallet_1')!;
        const wallet2 = accounts.get('wallet_2')!;
        
        let block = chain.mineBlock([
            Tx.contractCall(
                'safe',
                'init',
                [
                    types.list([
                        types.principal(deployer.address),
                        types.principal(wallet1.address),
                        types.principal(wallet2.address)
                    ]),
                    types.uint(2)  // 2-of-3 threshold
                ],
                deployer.address
            )
        ]);
        
        block.receipts[0].result.expectOk();
        
        // Verify owners
        let owners = chain.callReadOnlyFn(
            'safe',
            'get-owners',
            [],
            deployer.address
        );
        
        let ownersList = owners.result.expectOk().expectList();
        assertEquals(ownersList.length, 3);
    },
});

Clarinet.test({
    name: "Can submit and confirm transaction",
    async fn(chain: Chain, accounts: Map<string, Account>) {
        const deployer = accounts.get('deployer')!;
        const wallet1 = accounts.get('wallet_1')!;
        const wallet2 = accounts.get('wallet_2')!;
        
        // Initialize
        chain.mineBlock([
            Tx.contractCall('safe', 'init',
                [types.list([...]), types.uint(2)],
                deployer.address
            )
        ]);
        
        // Submit transaction
        let block = chain.mineBlock([
            Tx.contractCall(
                'safe',
                'submit',
                [
                    types.principal('ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM.transfer-stx'),
                    types.some(types.principal(wallet2.address)),
                    types.some(types.uint(1000000)),
                    types.none()
                ],
                deployer.address
            )
        ]);
        
        let txId = block.receipts[0].result.expectOk().expectUint(0);
        
        // Confirm with second owner
        block = chain.mineBlock([
            Tx.contractCall(
                'safe',
                'confirm',
                [
                    types.uint(txId),
                    types.principal('ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM.transfer-stx')
                ],
                wallet1.address
            )
        ]);
        
        block.receipts[0].result.expectOk();
        
        // Check transaction executed
        let tx = chain.callReadOnlyFn(
            'safe',
            'get-transaction',
            [types.uint(txId)],
            deployer.address
        );
        
        let txData = tx.result.expectOk().expectSome().expectTuple();
        txData['executed'].expectBool(true);
    },
});
```

---

## ⚠️ Security Considerations

### 1. **Replay Protection**
Ensure transactions can only be executed once

### 2. **Threshold Validation**
Always validate threshold is between 1 and owner count

### 3. **Owner Management**
Be careful when removing owners - adjust threshold if needed

### 4. **Executor Pattern**
Only allow trusted executor contracts

---

## 🎓 Advanced Features

### Time-Locked Transactions
```clarity
(define-map transaction-timelock
    uint  ;; tx-id
    uint  ;; unlock-block
)

(define-public (execute-after-timelock (tx-id uint))
    (let
        (
            (unlock-block (unwrap! (map-get? transaction-timelock tx-id) err-not-found))
        )
        (asserts! (>= block-height unlock-block) err-timelock-active)
        (execute-transaction tx-id)
    )
)
```

---

## 📚 Next Steps

**Related Guides:**
- → DAO Governance
- → Treasury Management
- → Access Control Patterns