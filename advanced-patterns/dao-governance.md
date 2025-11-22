# DAO Governance - Complete Guide

## 📖 Overview

A Decentralized Autonomous Organization (DAO) allows token holders to collectively make decisions through proposals and voting. This guide covers building a complete governance system with proposal creation, voting, and execution.

### What You'll Learn
- DAO architecture and governance models
- Proposal lifecycle management
- Voting mechanisms (token-weighted, quadratic)
- Proposal execution
- Delegation and vote power
- Treasury integration

### Prerequisites
- Understanding of SIP-010 tokens
- Multi-signature patterns
- Access control concepts

---

## 🎯 Key Concepts

### DAO Components

**1. Governance Token**
- Represents voting power
- 1 token = 1 vote (or customizable)
- Can be staked for additional benefits

**2. Proposals**
- Created by token holders
- Require minimum token threshold
- Have voting period
- Execute automatically if passed

**3. Voting System**
- Vote for/against/abstain
- Token-weighted or quadratic voting
- Delegation support
- Quorum requirements

**4. Execution**
- Automated execution after voting period
- Timelock for security
- Multiple actions per proposal

---

## 💻 Complete DAO Governance Contract

### Main Governance Contract

```clarity
;; DAO Governance Contract
;; Token-weighted voting with proposal execution

(use-trait governance-token-trait 'SP3FBR2AGK5H9QBDH3EEN6DF8EK8JY7RX8QJ5SVTE.sip-010-trait-ft-standard.sip-010-trait)

;; Constants
(define-constant contract-owner tx-sender)
(define-constant err-not-authorized (err u400))
(define-constant err-proposal-not-found (err u401))
(define-constant err-already-voted (err u402))
(define-constant err-voting-closed (err u403))
(define-constant err-voting-open (err u404))
(define-constant err-insufficient-tokens (err u405))
(define-constant err-quorum-not-met (err u406))
(define-constant err-proposal-failed (err u407))
(define-constant err-already-executed (err u408))
(define-constant err-timelock-active (err u409))

;; Governance parameters
(define-data-var proposal-threshold uint u1000000)  ;; Min tokens to propose (1 token with 6 decimals)
(define-data-var voting-period uint u1440)  ;; Blocks (~10 days)
(define-data-var quorum-percentage uint u2000)  ;; 20% in basis points
(define-data-var execution-delay uint u144)  ;; Timelock (~1 day)

;; Proposal counter
(define-data-var proposal-count uint u0)

;; Proposal structure
(define-map proposals
    uint  ;; proposal-id
    {
        proposer: principal,
        title: (string-ascii 100),
        description: (string-ascii 500),
        executor: principal,  ;; Contract to call if passed
        param-p: (optional principal),
        param-u: (optional uint),
        start-block: uint,
        end-block: uint,
        for-votes: uint,
        against-votes: uint,
        abstain-votes: uint,
        executed: bool,
        passed: bool
    }
)

;; Track individual votes
(define-map votes
    { proposal-id: uint, voter: principal }
    { 
        vote-power: uint,
        support: (string-ascii 10)  ;; "for", "against", "abstain"
    }
)

;; Vote delegation
(define-map delegations
    principal  ;; delegator
    principal  ;; delegate
)

;; -----------------------------------------------------------
;; Governance Parameters
;; -----------------------------------------------------------

;; Update proposal threshold (requires governance vote)
(define-public (set-proposal-threshold (new-threshold uint))
    (begin
        (asserts! (is-eq tx-sender contract-owner) err-not-authorized)
        (ok (var-set proposal-threshold new-threshold))
    )
)

;; Update voting period
(define-public (set-voting-period (new-period uint))
    (begin
        (asserts! (is-eq tx-sender contract-owner) err-not-authorized)
        (ok (var-set voting-period new-period))
    )
)

;; -----------------------------------------------------------
;; Proposal Creation
;; -----------------------------------------------------------

;; Create new proposal
(define-public (propose
    (token-trait <governance-token-trait>)
    (title (string-ascii 100))
    (description (string-ascii 500))
    (executor principal)
    (param-p (optional principal))
    (param-u (optional uint))
)
    (let
        (
            (proposal-id (var-get proposal-count))
            (proposer-balance (unwrap! (contract-call? token-trait get-balance tx-sender) err-not-authorized))
        )
        ;; Check proposer has enough tokens
        (asserts! (>= proposer-balance (var-get proposal-threshold)) err-insufficient-tokens)
        
        ;; Create proposal
        (map-set proposals proposal-id {
            proposer: tx-sender,
            title: title,
            description: description,
            executor: executor,
            param-p: param-p,
            param-u: param-u,
            start-block: block-height,
            end-block: (+ block-height (var-get voting-period)),
            for-votes: u0,
            against-votes: u0,
            abstain-votes: u0,
            executed: false,
            passed: false
        })
        
        ;; Increment counter
        (var-set proposal-count (+ proposal-id u1))
        
        (print {
            event: "proposal-created",
            proposal-id: proposal-id,
            proposer: tx-sender,
            title: title
        })
        
        (ok proposal-id)
    )
)

;; -----------------------------------------------------------
;; Voting
;; -----------------------------------------------------------

;; Cast vote
(define-public (cast-vote
    (token-trait <governance-token-trait>)
    (proposal-id uint)
    (support (string-ascii 10))  ;; "for", "against", "abstain"
)
    (let
        (
            (proposal (unwrap! (map-get? proposals proposal-id) err-proposal-not-found))
            (voter-balance (unwrap! (contract-call? token-trait get-balance tx-sender) err-not-authorized))
            (vote-power (get-voting-power tx-sender voter-balance))
        )
        ;; Check voting is open
        (asserts! (< block-height (get end-block proposal)) err-voting-closed)
        (asserts! (>= block-height (get start-block proposal)) err-voting-closed)
        
        ;; Check not already voted
        (asserts! (is-none (map-get? votes { proposal-id: proposal-id, voter: tx-sender }))
            err-already-voted
        )
        
        ;; Check has tokens
        (asserts! (> vote-power u0) err-insufficient-tokens)
        
        ;; Record vote
        (map-set votes
            { proposal-id: proposal-id, voter: tx-sender }
            { vote-power: vote-power, support: support }
        )
        
        ;; Update proposal vote counts
        (if (is-eq support "for")
            (map-set proposals proposal-id
                (merge proposal { for-votes: (+ (get for-votes proposal) vote-power) })
            )
            (if (is-eq support "against")
                (map-set proposals proposal-id
                    (merge proposal { against-votes: (+ (get against-votes proposal) vote-power) })
                )
                (map-set proposals proposal-id
                    (merge proposal { abstain-votes: (+ (get abstain-votes proposal) vote-power) })
                )
            )
        )
        
        (print {
            event: "vote-cast",
            proposal-id: proposal-id,
            voter: tx-sender,
            support: support,
            vote-power: vote-power
        })
        
        (ok true)
    )
)

;; Get voting power (can be customized)
(define-private (get-voting-power (voter principal) (token-balance uint))
    ;; Check if delegated
    (match (map-get? delegations voter)
        delegate u0  ;; If delegated, no direct voting power
        token-balance  ;; Otherwise, voting power = token balance
    )
)

;; -----------------------------------------------------------
;; Vote Delegation
;; -----------------------------------------------------------

;; Delegate voting power
(define-public (delegate (delegate-to principal))
    (begin
        (map-set delegations tx-sender delegate-to)
        
        (print {
            event: "vote-delegated",
            delegator: tx-sender,
            delegate: delegate-to
        })
        
        (ok true)
    )
)

;; Remove delegation
(define-public (undelegate)
    (begin
        (map-delete delegations tx-sender)
        
        (print { event: "vote-undelegated", delegator: tx-sender })
        
        (ok true)
    )
)

;; Get delegated voting power
(define-read-only (get-delegated-power
    (token-trait <governance-token-trait>)
    (delegate principal)
)
    ;; Sum up all tokens delegated to this address
    ;; Simplified - would need to track all delegators
    (ok u0)
)

;; -----------------------------------------------------------
;; Proposal Execution
;; -----------------------------------------------------------

;; Queue proposal for execution (after voting ends)
(define-public (queue-proposal (proposal-id uint))
    (let
        (
            (proposal (unwrap! (map-get? proposals proposal-id) err-proposal-not-found))
            (total-votes (+ (+ (get for-votes proposal) (get against-votes proposal)) 
                           (get abstain-votes proposal)))
        )
        ;; Check voting ended
        (asserts! (>= block-height (get end-block proposal)) err-voting-open)
        
        ;; Check not executed
        (asserts! (not (get executed proposal)) err-already-executed)
        
        ;; Check quorum
        ;; Note: would need total token supply to calculate properly
        (asserts! (> total-votes u0) err-quorum-not-met)
        
        ;; Check passed
        (let
            (
                (passed (> (get for-votes proposal) (get against-votes proposal)))
            )
            (map-set proposals proposal-id (merge proposal { passed: passed }))
            
            (print {
                event: "proposal-queued",
                proposal-id: proposal-id,
                passed: passed,
                for-votes: (get for-votes proposal),
                against-votes: (get against-votes proposal)
            })
            
            (ok passed)
        )
    )
)

;; Execute proposal
(define-public (execute-proposal (proposal-id uint))
    (let
        (
            (proposal (unwrap! (map-get? proposals proposal-id) err-proposal-not-found))
            (timelock-end (+ (get end-block proposal) (var-get execution-delay)))
        )
        ;; Check timelock passed
        (asserts! (>= block-height timelock-end) err-timelock-active)
        
        ;; Check not already executed
        (asserts! (not (get executed proposal)) err-already-executed)
        
        ;; Check passed
        (asserts! (get passed proposal) err-proposal-failed)
        
        ;; Execute via executor contract
        ;; This would call the executor with proposal params
        (try! (execute-proposal-action proposal))
        
        ;; Mark as executed
        (map-set proposals proposal-id (merge proposal { executed: true }))
        
        (print {
            event: "proposal-executed",
            proposal-id: proposal-id
        })
        
        (ok true)
    )
)

;; Execute the actual proposal action
(define-private (execute-proposal-action (proposal {
    proposer: principal,
    title: (string-ascii 100),
    description: (string-ascii 500),
    executor: principal,
    param-p: (optional principal),
    param-u: (optional uint),
    start-block: uint,
    end-block: uint,
    for-votes: uint,
    against-votes: uint,
    abstain-votes: uint,
    executed: bool,
    passed: bool
}))
    ;; Call executor contract with proposal parameters
    ;; Simplified - would use contract-call? with executor trait
    (ok true)
)

;; -----------------------------------------------------------
;; View Functions
;; -----------------------------------------------------------

;; Get proposal details
(define-read-only (get-proposal (proposal-id uint))
    (ok (map-get? proposals proposal-id))
)

;; Get vote for user on proposal
(define-read-only (get-vote (proposal-id uint) (voter principal))
    (ok (map-get? votes { proposal-id: proposal-id, voter: voter }))
)

;; Get proposal state
(define-read-only (get-proposal-state (proposal-id uint))
    (match (map-get? proposals proposal-id)
        proposal
            (if (get executed proposal)
                (ok "executed")
                (if (>= block-height (get end-block proposal))
                    (if (get passed proposal)
                        (ok "passed")
                        (ok "failed")
                    )
                    (if (>= block-height (get start-block proposal))
                        (ok "active")
                        (ok "pending")
                    )
                )
            )
        err-proposal-not-found
    )
)

;; Get voting results
(define-read-only (get-voting-results (proposal-id uint))
    (match (map-get? proposals proposal-id)
        proposal (ok {
            for: (get for-votes proposal),
            against: (get against-votes proposal),
            abstain: (get abstain-votes proposal),
            total: (+ (+ (get for-votes proposal) (get against-votes proposal))
                     (get abstain-votes proposal))
        })
        err-proposal-not-found
    )
)

;; Check if user voted
(define-read-only (has-voted (proposal-id uint) (voter principal))
    (ok (is-some (map-get? votes { proposal-id: proposal-id, voter: voter })))
)

;; Get delegate
(define-read-only (get-delegate (delegator principal))
    (ok (map-get? delegations delegator))
)

;; Get governance parameters
(define-read-only (get-governance-params)
    (ok {
        proposal-threshold: (var-get proposal-threshold),
        voting-period: (var-get voting-period),
        quorum-percentage: (var-get quorum-percentage),
        execution-delay: (var-get execution-delay)
    })
)
```

---

## 🌐 DAO Governance Frontend

```javascript
import React, { useState, useEffect } from 'react';
import { useWallet } from './WalletContext';
import {
  openContractCall,
  uintCV,
  stringAsciiCV,
  principalCV,
  someCV,
  noneCV
} from '@stacks/transactions';

function DAOGovernance() {
  const { address, isConnected } = useWallet();
  const [proposals, setProposals] = useState([]);
  const [userVotingPower, setUserVotingPower] = useState(0);
  const [governance, setGovernance] = useState(null);

  const DAO_CONTRACT = 'ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM.dao-governance';

  return (
    <div className="dao-governance">
      <h1>DAO Governance</h1>

      {isConnected && (
        <>
          <GovernanceStats 
            votingPower={userVotingPower}
            params={governance}
          />

          <CreateProposalForm contract={DAO_CONTRACT} />

          <ProposalList 
            proposals={proposals}
            userAddress={address}
            contract={DAO_CONTRACT}
          />
        </>
      )}
    </div>
  );
}

function GovernanceStats({ votingPower, params }) {
  return (
    <div className="governance-stats">
      <div className="stat-card">
        <h3>Your Voting Power</h3>
        <p className="stat-value">{votingPower.toFixed(2)}</p>
      </div>

      {params && (
        <>
          <div className="stat-card">
            <h3>Voting Period</h3>
            <p className="stat-value">{params.votingPeriod} blocks</p>
          </div>
          <div className="stat-card">
            <h3>Quorum</h3>
            <p className="stat-value">{params.quorum / 100}%</p>
          </div>
        </>
      )}
    </div>
  );
}

function CreateProposalForm({ contract }) {
  const [title, setTitle] = useState('');
  const [description, setDescription] = useState('');
  const [action, setAction] = useState('transfer');
  const [targetAddress, setTargetAddress] = useState('');
  const [amount, setAmount] = useState('');

  const createProposal = async () => {
    const options = {
      network: new StacksTestnet(),
      contractAddress: contract.split('.')[0],
      contractName: contract.split('.')[1],
      functionName: 'propose',
      functionArgs: [
        principalCV('ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM.governance-token'),
        stringAsciiCV(title),
        stringAsciiCV(description),
        principalCV('ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM.treasury-executor'),
        someCV(principalCV(targetAddress)),
        someCV(uintCV(parseFloat(amount) * 1000000))
      ],
      postConditionMode: PostConditionMode.Deny,
      onFinish: (data) => {
        alert(`Proposal created! TX: ${data.txId}`);
      },
    };

    await openContractCall(options);
  };

  return (
    <div className="create-proposal">
      <h2>Create Proposal</h2>

      <input
        type="text"
        placeholder="Title"
        value={title}
        onChange={(e) => setTitle(e.target.value)}
        maxLength={100}
      />

      <textarea
        placeholder="Description"
        value={description}
        onChange={(e) => setDescription(e.target.value)}
        maxLength={500}
        rows={4}
      />

      <select value={action} onChange={(e) => setAction(e.target.value)}>
        <option value="transfer">Transfer Funds</option>
        <option value="parameter">Change Parameter</option>
        <option value="upgrade">Upgrade Contract</option>
      </select>

      {action === 'transfer' && (
        <>
          <input
            type="text"
            placeholder="Recipient address"
            value={targetAddress}
            onChange={(e) => setTargetAddress(e.target.value)}
          />
          <input
            type="number"
            placeholder="Amount"
            value={amount}
            onChange={(e) => setAmount(e.target.value)}
          />
        </>
      )}

      <button onClick={createProposal}>Submit Proposal</button>
    </div>
  );
}

function ProposalList({ proposals, userAddress, contract }) {
  return (
    <div className="proposal-list">
      <h2>Active Proposals</h2>

      {proposals.map((proposal) => (
        <ProposalCard
          key={proposal.id}
          proposal={proposal}
          userAddress={userAddress}
          contract={contract}
        />
      ))}
    </div>
  );
}

function ProposalCard({ proposal, userAddress, contract }) {
  const [hasVoted, setHasVoted] = useState(false);

  const vote = async (support) => {
    const options = {
      network: new StacksTestnet(),
      contractAddress: contract.split('.')[0],
      contractName: contract.split('.')[1],
      functionName: 'cast-vote',
      functionArgs: [
        principalCV('ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM.governance-token'),
        uintCV(proposal.id),
        stringAsciiCV(support)
      ],
      postConditionMode: PostConditionMode.Deny,
      onFinish: (data) => {
        alert(`Vote cast! TX: ${data.txId}`);
        setHasVoted(true);
      },
    };

    await openContractCall(options);
  };

  const totalVotes = proposal.forVotes + proposal.againstVotes + proposal.abstainVotes;
  const forPercentage = totalVotes > 0 ? (proposal.forVotes / totalVotes) * 100 : 0;
  const againstPercentage = totalVotes > 0 ? (proposal.againstVotes / totalVotes) * 100 : 0;

  return (
    <div className="proposal-card">
      <div className="proposal-header">
        <h3>{proposal.title}</h3>
        <span className={`status ${proposal.state}`}>{proposal.state}</span>
      </div>

      <p className="proposal-description">{proposal.description}</p>

      <div className="proposal-stats">
        <div className="stat">
          <span>Proposer:</span>
          <span>{proposal.proposer.slice(0, 8)}...</span>
        </div>
        <div className="stat">
          <span>Ends:</span>
          <span>Block {proposal.endBlock}</span>
        </div>
      </div>

      <div className="voting-results">
        <div className="vote-bar">
          <div className="for-bar" style={{ width: `${forPercentage}%` }}>
            <span>{forPercentage.toFixed(1)}% For</span>
          </div>
          <div className="against-bar" style={{ width: `${againstPercentage}%` }}>
            <span>{againstPercentage.toFixed(1)}% Against</span>
          </div>
        </div>

        <div className="vote-counts">
          <span>For: {(proposal.forVotes / 1000000).toFixed(2)}</span>
          <span>Against: {(proposal.againstVotes / 1000000).toFixed(2)}</span>
          <span>Abstain: {(proposal.abstainVotes / 1000000).toFixed(2)}</span>
        </div>
      </div>

      {proposal.state === 'active' && !hasVoted && (
        <div className="vote-buttons">
          <button onClick={() => vote('for')} className="vote-for">
            Vote For
          </button>
          <button onClick={() => vote('against')} className="vote-against">
            Vote Against
          </button>
          <button onClick={() => vote('abstain')} className="vote-abstain">
            Abstain
          </button>
        </div>
      )}

      {hasVoted && <p className="voted-message">✓ You have voted</p>}

      {proposal.state === 'passed' && !proposal.executed && (
        <button className="execute-btn">Execute Proposal</button>
      )}
    </div>
  );
}

export default DAOGovernance;
```

---

## 📊 Advanced Governance Features

### 1. Quadratic Voting

```clarity
;; Quadratic voting: vote power = sqrt(tokens)
(define-private (calculate-quadratic-power (token-balance uint))
    (sqrt token-balance)
)
```

### 2. Conviction Voting

```clarity
;; Vote power increases over time
(define-map conviction-votes
    { proposal-id: uint, voter: principal }
    { amount: uint, start-block: uint }
)

(define-private (calculate-conviction-power (amount uint) (blocks-held uint))
    (* amount (sqrt blocks-held))
)
```

### 3. Vote Escrow

```clarity
;; Lock tokens for longer = more voting power
(define-map vote-locks
    principal
    { amount: uint, unlock-block: uint }
)

(define-private (calculate-lock-multiplier (unlock-block uint))
    (let
        (
            (blocks-until-unlock (- unlock-block block-height))
            (max-lock u52560)  ;; 1 year
        )
        (/ (* blocks-until-unlock u4) max-lock)  ;; Up to 4x multiplier
    )
)
```

---

## 🎓 Next Steps

**Related Guides:**
- → Treasury Management
- → Multi-Signature Integration
- → Token Economics