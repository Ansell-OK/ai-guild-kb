# Deploying to Stacks Testnet

## 📖 Overview

This guide covers deploying Clarity smart contracts to the Stacks testnet using Clarinet CLI and manual methods. Learn how to prepare, deploy, and verify your contracts on testnet.

### What You'll Learn
- Preparing contracts for deployment
- Using Clarinet CLI for deployment
- Manual deployment via Explorer
- Managing deployment costs
- Verifying deployed contracts

### Prerequisites
- Clarinet CLI installed
- Contracts tested locally
- Testnet STX for deployment fees

---

## 🚀 Method 1: Clarinet CLI Deployment (Recommended)

### Step 1: Configure Deployment Wallet

Check your `settings/Testnet.toml` file has a valid mnemonic:

```toml
[accounts.deployer]
mnemonic = "board list obtain sugar hour worth raven scout denial thunder horse logic fury scorpion fold genuine phrase wealth news aim below celery when cabin"
balance = 100_000_000_000
```

**Requirements**:
- Must be 12, 15, 18, 21, or 24 words
- Keep this secure - don't commit to public repos

---

### Step 2: Generate Deployment Plan

```bash
clarinet deployments generate --testnet --low-cost
```

**Cost Strategy Options**:
- `--low-cost` - Slower but cheaper (good for testnet)
- `--medium-cost` - Balanced
- `--high-cost` - Faster confirmation

**Output**: Creates `deployments/default.testnet-plan.yaml`

**Example deployment plan**:
```yaml
---
id: 0
name: Testnet deployment
network: testnet
plan:
  batches:
    - id: 0
      transactions:
        - contract-publish:
            contract-name: sip009-nft-trait
            expected-sender: ST2NEB84ASENDXKYGJPQW86YXQCEFEX2ZQPG87ND
            cost: 105634
            path: "contracts\\sip009-nft-trait.clar"
        - contract-publish:
            contract-name: nft-minter
            expected-sender: ST2NEB84ASENDXKYGJPQW86YXQCEFEX2ZQPG87ND
            cost: 105806
            path: "contracts\\nft-minter.clar"
            depends_on: ['sip009-nft-trait']
```

**Important**: Note the `expected-sender` address - this is your deployer address.

---

### Step 3: Fund Deployer Address

1. **Get testnet STX** from faucet:
   - Visit: https://explorer.hiro.so/sandbox/faucet?chain=testnet
   - Enter your deployer address (from deployment plan)
   - Request testnet STX
   - Wait ~5-10 minutes for confirmation

2. **Verify balance**:
   ```
   https://explorer.hiro.so/address/YOUR_ADDRESS?chain=testnet
   ```

**How much STX needed**?
- Simple contracts: ~0.1 STX each
- Complex contracts: ~0.5 STX each
- Get 2-5 STX to be safe

---

### Step 4: Deploy Contracts

```bash
clarinet deployments apply --testnet
```

**What happens**:
1. Shows deployment summary and total cost
2. Asks for confirmation: `Continue [Y/n]?`
3. Broadcasts transactions to testnet
4. Waits for confirmation
5. Shows success message with transaction IDs

**Example output**:
```
Total cost: 0.117040 STX
Duration: 1 blocks

Continue [Y/n]? Y

✔ Transactions successfully confirmed on Testnet
```

---

### Step 5: Verify Deployment

**Check on Explorer**:
```
https://explorer.hiro.so/address/YOUR_DEPLOYER_ADDRESS?chain=testnet
```

Look for:
- ✅ Contract deployment transactions
- ✅ "Success" status
- ✅ Contract source code visible

**Get Contract Address**:
Your contracts are deployed at:
```
DEPLOYER_ADDRESS.contract-name
```

Example:
```
ST2NEB84ASENDXKYGJPQW86YXQCEFEX2ZQPG87ND.simple-counter
ST2NEB84ASENDXKYGJPQW86YXQCEFEX2ZQPG87ND.nft-minter
```

---

## 🌐 Method 2: Manual Deployment via Explorer

For cases where CLI deployment doesn't work or you prefer manual control.

### Step 1: Access Deploy Sandbox

Visit: https://explorer.hiro.so/sandbox/deploy?chain=testnet

### Step 2: Connect Wallet

1. Click "Connect Wallet"
2. Choose your wallet (Xverse/Hiro Wallet)
3. Switch to **Testnet** mode
4. Approve connection

### Step 3: Deploy Contracts

**Important**: Deploy in correct order (dependencies first)!

#### Example Order:
1. Deploy trait contracts first (e.g., `sip009-nft-trait`)
2. Then contracts that depend on them (e.g., `nft-minter`)

#### For Each Contract:

**a) Enter Contract Details**:
```
Contract name: simple-counter
Clarity version: 3
```

**b) Paste Contract Code**:
Copy entire contents of `contracts/simple-counter.clar`

**c) Deploy**:
1. Review gas fee estimate
2. Click "Deploy Contract"
3. Approve in wallet
4. Wait for confirmation (~1-2 minutes)

**d) Save Address**:
Note the deployed address for your frontend configuration.

---

### Step 4: Update Frontend Config

After deployment, update your frontend with deployed addresses:

**File**: `lib/contracts.ts`

```typescript
export const CONTRACTS = {
  SIMPLE_COUNTER: {
    address: "ST2NEB84ASENDXKYGJPQW86YXQCEFEX2ZQPG87ND", // Your address
    name: "simple-counter",
  },
  NFT_MINTER: {
    address: "ST2NEB84ASENDXKYGJPQW86YXQCEFEX2ZQPG87ND",
    name: "nft-minter",
  },
};
```

---

## ⚙️ Deployment Configuration

### Contract Dependencies

In `Clarinet.toml`, specify dependencies:

```toml
[contracts.sip009-nft-trait]
path = 'contracts/sip009-nft-trait.clar'
clarity_version = 3

[contracts.nft-minter]
path = 'contracts/nft-minter.clar'
depends_on = ['sip009-nft-trait']  # Deploy trait first!
clarity_version = 3
```

### Clarity Version

Always specify Clarity version:
```toml
clarity_version = 3  # Latest version
epoch = 'latest'
```

---

## 💰 Cost Management

### Estimating Costs

**Factors affecting cost**:
- Contract size (lines of code)
- Complexity (number of functions)
- Network congestion

**Typical costs (testnet)**:
- Simple contracts (< 100 lines): 0.05-0.1 STX
- Medium contracts (100-300 lines): 0.1-0.3 STX
- Complex contracts (> 300 lines): 0.3-0.5 STX

### Cost Strategies

```bash
# Low cost (1-2 blocks wait)
clarinet deployments generate --testnet --low-cost

# Medium cost (< 1 block)
clarinet deployments generate --testnet --medium-cost

# High cost (immediate)
clarinet deployments generate --testnet --high-cost
```

**Recommendation**: Use `--low-cost` for testnet, `--medium-cost` for mainnet.

---

## ⚠️ Common Issues

### Issue 1: Invalid Mnemonic

**Error**:
```
mnemonic has an invalid word count: 5
```

**Solution**: Ensure `settings/Testnet.toml` has valid 12 or 24-word mnemonic.

See: [Common Errors Guide](../fundamentals/common-errors.md#invalid-mnemonic)

---

### Issue 2: Insufficient Funds

**Error**:
```
insufficient funds
```

**Solution**:
1. Get more testnet STX from faucet
2. Wait for existing faucet request to complete
3. Verify balance on explorer

---

### Issue 3: Cost Strategy Required

**Error**:
```
cost strategy not specified
```

**Solution**: Always include cost flag:
```bash
clarinet deployments generate --testnet --low-cost
```

---

### Issue 4: Contract Deployment Order

**Error**:
```
trait sip009-nft-trait not found
```

**Solution**: Deploy dependencies first, or use `depends_on` in Clarinet.toml:

```toml
[contracts.nft-minter]
depends_on = ['sip009-nft-trait']
```

---

## 🔍 Verifying Deployment

### Check Contract on Explorer

```
https://explorer.hiro.so/txid/TRANSACTION_ID?chain=testnet
```

**What to verify**:
- ✅ Transaction status: "Success"
- ✅ Contract source code visible
- ✅ No error messages
- ✅ Block height confirmed

### Test Contract Calls

Use the explorer's "Call Function" feature:

1. Go to contract page
2. Select "Call Function"
3. Choose a read-only function
4. Click "Call"
5. Verify expected output

---

## 📋 Deployment Checklist

Before deploying:

- [ ] All tests passing locally (`npm test`)
- [ ] Contracts using LF line endings (Windows only)
- [ ] Dependencies specified in Clarinet.toml
- [ ] Error codes documented
- [ ] Contract checked (`clarinet check`)
- [ ] Testnet STX in deployer wallet (2-5 STX)
- [ ] Deployment plan reviewed
- [ ] Frontend config ready to update

After deploying:

- [ ] Verified contract on explorer
- [ ] Tested read-only functions
- [ ] Updated frontend with addresses
- [ ] Tested frontend contract interactions
- [ ] Documented deployment addresses

---

## 🎯 Best Practices

1. **Test Thoroughly Locally** - Fix all bugs before deploying
2. **Deploy to Testnet First** - Never deploy untested code to mainnet
3. **Keep Private Keys Secure** - Never commit mnemonics to git
4. **Document Addresses** - Save all deployed contract addresses
5. **Version Control** - Tag releases when deploying
6. **Use Low Cost on Testnet** - Save STX for testing
7. **Verify Immediately** - Check deployment succeeded before moving on

---

## 🚢 Deploying to Mainnet

**When ready for mainnet**:

```bash
# Generate mainnet deployment
clarinet deployments generate --mainnet --medium-cost

# ⚠️ REAL STX REQUIRED
clarinet deployments apply --mainnet
```

**Key differences**:
- Real STX required (not free!)
- Higher costs ($5-50 per contract)
- Permanent - can't delete
- Use `--medium-cost` or `--high-cost`

**Mainnet checklist**:
- [ ] Thoroughly tested on testnet
- [ ] Security audit completed (for high-value contracts)
- [ ] Emergency pause mechanism (if applicable)
- [ ] Sufficient STX in wallet
- [ ] Team approval
- [ ] Documentation complete

---

## 📚 Additional Resources

- **Testnet Faucet**: https://explorer.hiro.so/sandbox/faucet?chain=testnet
- **Testnet Explorer**: https://explorer.hiro.so/?chain=testnet
- **Deploy Sandbox**: https://explorer.hiro.so/sandbox/deploy?chain=testnet
- **Clarinet Docs**: https://docs.hiro.so/clarinet

---

## 🔗 Related Guides

- [Common Errors](../fundamentals/common-errors.md)
- [Wallet Authentication](../wallets/stacks-connect-authentication.md)
- [Contract Interaction](../frontend-integration/contract-calls.md)
