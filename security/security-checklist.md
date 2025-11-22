# Security Audit Checklist - Pre-Deployment Guide

## 📖 Overview

This comprehensive checklist ensures your smart contract is secure before mainnet deployment. Use this as your final review before going live with real user funds.

### Severity Levels

🔴 **Critical** - Must fix before deployment  
🟡 **High** - Should fix before deployment  
🟢 **Medium** - Consider fixing  
🔵 **Low** - Nice to have

---

## 🔴 Critical Security Checks

### Access Control

- [ ] **Owner Authorization**: All admin functions require owner check
  ```clarity
  (asserts! (is-eq tx-sender contract-owner) err-owner-only)
  ```

- [ ] **No Unauthorized Minting**: Token minting requires proper authorization
  ```clarity
  // Check all mint functions have access control
  ```

- [ ] **Protected State Changes**: Critical variables can only be changed by authorized parties
  ```clarity
  // Check fee settings, threshold changes, etc.
  ```

- [ ] **No Open Admin Functions**: No function allows anyone to perform admin actions

### Fund Security

- [ ] **STX Transfer Validation**: All STX transfers have proper checks
  ```clarity
  (asserts! (> amount u0) err-invalid-amount)
  (try! (stx-transfer? amount sender recipient))
  ```

- [ ] **Token Transfer Protection**: All token transfers validate sender/recipient
  ```clarity
  (asserts! (is-eq sender tx-sender) err-not-authorized)
  ```

- [ ] **Balance Checks**: Always verify sufficient balance before transfers
  ```clarity
  (asserts! (>= sender-balance amount) err-insufficient-balance)
  ```

- [ ] **No Reentrancy Vectors**: State changes before external calls

### Arithmetic Safety

- [ ] **Subtraction Checks**: All subtractions check for underflow
  ```clarity
  (asserts! (>= a b) err-underflow)
  (- a b)
  ```

- [ ] **Division by Zero**: All divisions check denominator is non-zero
  ```clarity
  (asserts! (> denominator u0) err-division-by-zero)
  ```

- [ ] **Overflow Protection**: Large calculations validated

---

## 🟡 High Priority Checks

### Input Validation

- [ ] **Amount Validation**: All amount parameters validated
  ```clarity
  (asserts! (> amount u0) err-invalid-amount)
  (asserts! (>= amount min-amount) err-below-minimum)
  (asserts! (<= amount max-amount) err-above-maximum)
  ```

- [ ] **Principal Validation**: Address parameters checked
  ```clarity
  (asserts! (not (is-eq recipient tx-sender)) err-self-transfer)
  (asserts! (is-standard recipient) err-invalid-address)
  ```

- [ ] **Parameter Bounds**: All parameters have reasonable limits
  ```clarity
  (asserts! (<= new-threshold owner-count) err-invalid-threshold)
  ```

- [ ] **Zero Checks**: Functions reject zero where inappropriate

### Error Handling

- [ ] **Unique Error Codes**: Each error has unique code
  ```clarity
  (define-constant err-owner-only (err u100))
  (define-constant err-not-found (err u101))
  // etc...
  ```

- [ ] **All Errors Documented**: Error codes explained in comments

- [ ] **Proper Error Propagation**: Use `try!` for nested calls
  ```clarity
  (try! (operation-one))
  (try! (operation-two))
  ```

- [ ] **No unwrap-panic in Production**: Replace with proper error handling
  ```clarity
  // Bad
  (unwrap-panic (map-get? data key))
  
  // Good
  (unwrap! (map-get? data key) err-not-found)
  ```

### State Management

- [ ] **Initialization Protected**: Can't be called twice
  ```clarity
  (asserts! (not (var-get initialized)) err-already-initialized)
  ```

- [ ] **State Consistency**: Related state variables updated together

- [ ] **No Race Conditions**: Proper ordering of state changes

---

## 🟢 Medium Priority Checks

### Code Quality

- [ ] **Function Size**: Functions are focused and not too large (< 50 lines)

- [ ] **Clear Naming**: Variables and functions have descriptive names
  ```clarity
  // Good
  (define-data-var total-staked uint u0)
  
  // Bad
  (define-data-var x uint u0)
  ```

- [ ] **Comments**: Complex logic is explained

- [ ] **Constants Used**: Magic numbers replaced with named constants
  ```clarity
  (define-constant basis-points u10000)
  (define-constant fee-percentage u250)  // 2.5%
  ```

### Event Logging

- [ ] **Critical Events Logged**: Transfers, admin actions logged
  ```clarity
  (print {
      event: "transfer",
      from: sender,
      to: recipient,
      amount: amount
  })
  ```

- [ ] **Sufficient Event Data**: Events include all relevant information

- [ ] **Consistent Format**: Events use consistent structure

### Testing

- [ ] **>90% Code Coverage**: Most code paths tested

- [ ] **Happy Path Tests**: Normal operations tested

- [ ] **Error Path Tests**: All errors tested
  ```typescript
  Clarinet.test({
      name: "Fails with insufficient balance",
      // Test error condition
  });
  ```

- [ ] **Edge Cases Tested**: Boundary conditions covered
  - Zero amounts
  - Maximum values
  - Empty states

- [ ] **Integration Tests**: Multi-contract interactions tested

---

## 🔵 Low Priority / Best Practices

### Documentation

- [ ] **README**: Contract purpose and usage explained

- [ ] **Function Documentation**: Each public function documented
  ```clarity
  ;; Transfer tokens from sender to recipient
  ;; @param amount - Amount to transfer (in raw units)
  ;; @param recipient - Address to receive tokens
  ;; @returns (response bool uint)
  (define-public (transfer ...)
  ```

- [ ] **Deployment Guide**: Instructions for deployment

- [ ] **Example Usage**: Code examples provided

### Gas Optimization

- [ ] **Efficient Loops**: No unbounded loops

- [ ] **Storage Optimization**: Maps preferred over large lists

- [ ] **Read-Only Functions**: Query functions are read-only

### User Experience

- [ ] **Clear Error Messages**: Errors are understandable

- [ ] **Sensible Defaults**: Default values are reasonable

- [ ] **Frontend Integration**: Post-conditions examples provided

---

## 📋 Contract-Specific Checklists

### Token Contracts

- [ ] **SIP-010 Compliant**: Implements full trait
- [ ] **Transfer Function Protected**: Only sender can initiate
- [ ] **Mint Function Protected**: Only authorized addresses
- [ ] **Total Supply Tracked**: Supply calculations correct
- [ ] **Decimals Consistent**: Decimal places used correctly

### NFT Contracts

- [ ] **SIP-009 Compliant**: Implements full trait
- [ ] **Unique Token IDs**: IDs increment properly
- [ ] **Owner Tracking**: Ownership correctly maintained
- [ ] **Transfer Protection**: Only owner can transfer
- [ ] **Metadata URIs**: URI generation works correctly

### DEX/AMM Contracts

- [ ] **Slippage Protection**: Minimum output enforced
- [ ] **Price Impact Calculated**: Large trades limited
- [ ] **Liquidity Checks**: Sufficient liquidity verified
- [ ] **Fee Calculation**: Fees calculated correctly
- [ ] **K Invariant**: Constant product maintained
  ```clarity
  (asserts! (>= (* new-reserve-x new-reserve-y) k) err-invariant)
  ```

### Governance Contracts

- [ ] **Voting Period Enforced**: Can't vote after deadline
- [ ] **One Vote Per Address**: Double voting prevented
- [ ] **Quorum Checked**: Minimum participation required
- [ ] **Execution Delay**: Timelock for execution
- [ ] **Proposal Validation**: Proposals properly validated

### Multisig Contracts

- [ ] **Threshold Validation**: Threshold reasonable (1 < t <= n)
- [ ] **Owner Management**: Add/remove owners protected
- [ ] **Confirmation Tracking**: Confirmations counted correctly
- [ ] **Execution Protection**: Only executes with enough confirmations
- [ ] **Transaction Uniqueness**: Can't execute twice

---

## 🧪 Testing Checklist

### Unit Tests

- [ ] Test each function individually
- [ ] Test with valid inputs
- [ ] Test with invalid inputs
- [ ] Test boundary conditions
- [ ] Test error conditions

### Integration Tests

- [ ] Test contract interactions
- [ ] Test complex workflows
- [ ] Test state transitions
- [ ] Test event emissions

### Security Tests

- [ ] Test unauthorized access attempts
- [ ] Test overflow/underflow scenarios
- [ ] Test reentrancy (if applicable)
- [ ] Test front-running scenarios
- [ ] Test DoS vectors

### Example Test Template

```typescript
// 1. Setup test
Clarinet.test({
    name: "Feature works correctly",
    async fn(chain: Chain, accounts: Map<string, Account>) {
        // 2. Prepare test data
        const user = accounts.get('wallet_1')!;
        
        // 3. Execute operation
        let block = chain.mineBlock([
            Tx.contractCall(...)
        ]);
        
        // 4. Verify result
        block.receipts[0].result.expectOk();
        
        // 5. Check state changes
        let balance = chain.callReadOnlyFn(...);
        balance.result.expectUint(expectedValue);
    }
});
```

---

## 🚀 Deployment Checklist

### Pre-Deployment

- [ ] **All Tests Pass**: 100% test success rate
  ```bash
  clarinet test
  ```

- [ ] **Testnet Deployed**: Contract tested on testnet

- [ ] **Testnet Transactions**: Real transactions executed

- [ ] **Explorer Verified**: Contract readable on explorer

- [ ] **Security Review**: This checklist completed

- [ ] **Code Audit**: Professional audit if handling significant value

### Deployment Configuration

- [ ] **Correct Network**: Mainnet configuration set

- [ ] **Owner Address**: Correct owner address used

- [ ] **Initial Values**: All constants set correctly
  ```clarity
  (define-constant contract-owner tx-sender)
  (define-constant token-name "MyToken")
  ```

- [ ] **Gas Budget**: Sufficient STX for deployment

### Post-Deployment

- [ ] **Contract Verified**: Confirmed on Stacks Explorer

- [ ] **Source Published**: Source code available

- [ ] **Documentation Updated**: Deployment address documented

- [ ] **Frontend Updated**: Correct contract address in UI

- [ ] **Test Transaction**: Small test transaction successful

- [ ] **Monitoring Setup**: Watch for issues

---

## 📊 Severity Examples

### 🔴 Critical Issues

```clarity
// CRITICAL: Anyone can mint tokens
(define-public (mint (amount uint))
    (ft-mint? token amount tx-sender)  // ❌ No authorization!
)

// FIX:
(define-public (mint (amount uint))
    (begin
        (asserts! (is-eq tx-sender contract-owner) err-owner-only)
        (ft-mint? token amount tx-sender)
    )
)
```

### 🟡 High Priority Issues

```clarity
// HIGH: No balance check before transfer
(define-public (transfer (amount uint))
    (begin
        (set-balance tx-sender (- (get-balance tx-sender) amount))  // ❌ Could underflow
        (ok true)
    )
)

// FIX:
(define-public (transfer (amount uint))
    (let ((balance (get-balance tx-sender)))
        (asserts! (>= balance amount) err-insufficient-balance)
        (set-balance tx-sender (- balance amount))
        (ok true)
    )
)
```

### 🟢 Medium Priority Issues

```clarity
// MEDIUM: No event logging
(define-public (transfer (amount uint))
    (begin
        (execute-transfer amount)
        (ok true)  // ❌ No event
    )
)

// FIX:
(define-public (transfer (amount uint))
    (begin
        (execute-transfer amount)
        (print { event: "transfer", amount: amount })  // ✅ Event logged
        (ok true)
    )
)
```

---

## 🎯 Final Verification

### Before Mainnet Deployment

**I confirm that:**

- [ ] All critical (🔴) issues are resolved
- [ ] All high priority (🟡) issues are resolved  
- [ ] Security-sensitive functions have proper access control
- [ ] All user funds are protected by proper validation
- [ ] Error handling is comprehensive
- [ ] Tests cover >90% of code
- [ ] Code has been reviewed by at least one other developer
- [ ] Testnet deployment was successful
- [ ] This contract is ready for mainnet deployment

**Reviewed by:** ________________  
**Date:** ________________  
**Contract Version:** ________________

---

## 📚 Additional Resources

### Security Guidelines
- Clarity Security Documentation
- Stacks Security Best Practices
- Common Vulnerability Database

### Audit Services
- Professional audit services for high-value contracts
- Community security reviews
- Bug bounty programs

### Tools
- Clarinet for testing
- Stacks Explorer for verification
- Post-condition helpers for frontend

---

## 🆘 If Issues Found

### Critical Issues
1. **Stop deployment immediately**
2. Fix the issue
3. Re-run full test suite
4. Complete checklist again
5. Consider professional audit

### High Priority Issues
1. Assess impact
2. Fix before mainnet
3. Test thoroughly
4. Review related code

### Medium/Low Issues
1. Document for future update
2. Consider fixing before mainnet
3. Can be addressed in v2

---

**Remember:** Smart contracts are immutable once deployed. Take time to ensure security before mainnet deployment.