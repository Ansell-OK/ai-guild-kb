# Post-Conditions - Complete Security Guide

## 📖 Overview

Post-conditions are a unique Clarity feature that allows users to specify constraints on what a transaction can do. They act as a safety net, protecting users from malicious contracts and unexpected behavior.

### What You'll Learn
- What post-conditions are and why they matter
- Types of post-conditions
- How to create post-conditions
- Frontend integration
- Best practices for user protection

### Prerequisites
- Understanding of Clarity transactions
- Frontend wallet integration
- Token standards (SIP-010, SIP-009)

---

## 🎯 What Are Post-Conditions?

### The Problem

Without post-conditions, a user calling a contract function has no guarantees about what the contract will actually do:

```clarity
;; Malicious contract
(define-public (swap-tokens (amount uint))
    (begin
        ;; User thinks they're getting fair swap
        ;; But contract steals their tokens!
        (try! (contract-call? .token transfer amount tx-sender contract-owner))
        (ok u0)  ;; Give them nothing
    )
)
```

### The Solution: Post-Conditions

Post-conditions let users specify exactly what they're willing to allow:

```javascript
// User can specify: "I will send exactly X tokens and receive at least Y tokens"
postConditions: [
  makeStandardFungiblePostCondition(
    senderAddress,
    FungibleConditionCode.Equal,
    amountToSend,
    tokenAsset
  ),
  makeStandardFungiblePostCondition(
    senderAddress,
    FungibleConditionCode.GreaterEqual,
    minAmountToReceive,
    otherTokenAsset
  )
]
```

**If the contract tries to transfer more or give less, the entire transaction fails.**

---

## 📝 Types of Post-Conditions

### 1. STX Post-Conditions

Protect native STX transfers.

```javascript
import { 
  makeStandardSTXPostCondition,
  FungibleConditionCode
} from '@stacks/transactions';

// User sends exactly 1 STX
const exactSTX = makeStandardSTXPostCondition(
  senderAddress,
  FungibleConditionCode.Equal,
  1000000  // 1 STX in microSTX
);

// User sends at most 1 STX
const maxSTX = makeStandardSTXPostCondition(
  senderAddress,
  FungibleConditionCode.LessEqual,
  1000000
);

// User sends at least 1 STX
const minSTX = makeStandardSTXPostCondition(
  senderAddress,
  FungibleConditionCode.GreaterEqual,
  1000000
);
```

### 2. Fungible Token Post-Conditions

Protect SIP-010 token transfers.

```javascript
import { makeStandardFungiblePostCondition } from '@stacks/transactions';

// Token asset format: contractAddress.contractName::assetName
const tokenAsset = 'ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM.my-token::my-token';

// User transfers exactly 100 tokens
const exactTokens = makeStandardFungiblePostCondition(
  senderAddress,
  FungibleConditionCode.Equal,
  100000000,  // 100 tokens with 6 decimals
  tokenAsset
);

// User receives at least 50 tokens
const minReceive = makeStandardFungiblePostCondition(
  recipientAddress,
  FungibleConditionCode.GreaterEqual,
  50000000,
  tokenAsset
);
```

### 3. Non-Fungible Token (NFT) Post-Conditions

Protect NFT transfers.

```javascript
import {
  makeStandardNonFungiblePostCondition,
  NonFungibleConditionCode,
  createAssetInfo,
  uintCV
} from '@stacks/transactions';

// Create asset info
const nftAsset = createAssetInfo(
  'ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM',  // Contract address
  'my-nft',  // Contract name
  'nft-asset'  // Asset name from contract
);

// User sends specific NFT
const sendsNFT = makeStandardNonFungiblePostCondition(
  senderAddress,
  NonFungibleConditionCode.Sends,
  nftAsset,
  uintCV(tokenId)  // Specific token ID
);

// User does NOT send NFT
const doesNotSendNFT = makeStandardNonFungiblePostCondition(
  senderAddress,
  NonFungibleConditionCode.DoesNotSend,
  nftAsset,
  uintCV(tokenId)
);

// User owns NFT (doesn't transfer it)
const ownsNFT = makeStandardNonFungiblePostCondition(
  senderAddress,
  NonFungibleConditionCode.Owns,
  nftAsset,
  uintCV(tokenId)
);
```

---

## 🔧 Fungible Condition Codes

```javascript
FungibleConditionCode.Equal          // Exactly this amount
FungibleConditionCode.Greater        // More than this amount
FungibleConditionCode.GreaterEqual   // At least this amount
FungibleConditionCode.Less           // Less than this amount
FungibleConditionCode.LessEqual      // At most this amount
```

---

## 💻 Complete Examples

### Example 1: Simple Token Transfer

```javascript
import { openContractCall, makeStandardFungiblePostCondition } from '@stacks/connect';

async function transferTokens(recipient, amount) {
  const senderAddress = userSession.loadUserData().profile.stxAddress.testnet;
  const amountRaw = amount * 1000000;  // Convert to raw units
  
  const tokenAsset = 'ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM.my-token::my-token';
  
  // Post-condition: User sends exactly this amount
  const postCondition = makeStandardFungiblePostCondition(
    senderAddress,
    FungibleConditionCode.Equal,
    amountRaw,
    tokenAsset
  );
  
  const options = {
    network: new StacksTestnet(),
    contractAddress: 'ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM',
    contractName: 'my-token',
    functionName: 'transfer',
    functionArgs: [
      uintCV(amountRaw),
      principalCV(senderAddress),
      principalCV(recipient),
      noneCV()
    ],
    postConditions: [postCondition],
    postConditionMode: PostConditionMode.Deny,  // Strict mode
    onFinish: (data) => {
      console.log('Transfer complete:', data.txId);
    }
  };
  
  await openContractCall(options);
}
```

### Example 2: Token Swap with Slippage Protection

```javascript
async function swapTokens(amountIn, minAmountOut) {
  const senderAddress = userSession.loadUserData().profile.stxAddress.testnet;
  
  const tokenA = 'ST1...my-token-a::token-a';
  const tokenB = 'ST1...my-token-b::token-b';
  
  // Post-conditions:
  // 1. User sends exactly amountIn of token A
  const sendCondition = makeStandardFungiblePostCondition(
    senderAddress,
    FungibleConditionCode.Equal,
    amountIn,
    tokenA
  );
  
  // 2. User receives at least minAmountOut of token B
  const receiveCondition = makeStandardFungiblePostCondition(
    senderAddress,
    FungibleConditionCode.GreaterEqual,
    minAmountOut,
    tokenB
  );
  
  const options = {
    network: new StacksTestnet(),
    contractAddress: 'ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM',
    contractName: 'dex-router',
    functionName: 'swap-tokens',
    functionArgs: [
      uintCV(amountIn),
      uintCV(minAmountOut),
      // ... other args
    ],
    postConditions: [sendCondition, receiveCondition],
    postConditionMode: PostConditionMode.Deny,
    onFinish: (data) => {
      console.log('Swap complete:', data.txId);
    }
  };
  
  await openContractCall(options);
}
```

### Example 3: NFT Purchase

```javascript
async function purchaseNFT(nftId, price) {
  const buyerAddress = userSession.loadUserData().profile.stxAddress.testnet;
  const sellerAddress = 'ST2...';  // Get from marketplace listing
  
  const nftAsset = createAssetInfo(
    'ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM',
    'my-nft',
    'nft-asset'
  );
  
  // Post-conditions:
  // 1. Buyer sends exact STX amount
  const stxCondition = makeStandardSTXPostCondition(
    buyerAddress,
    FungibleConditionCode.Equal,
    price
  );
  
  // 2. Seller sends the NFT
  const nftCondition = makeStandardNonFungiblePostCondition(
    sellerAddress,
    NonFungibleConditionCode.Sends,
    nftAsset,
    uintCV(nftId)
  );
  
  const options = {
    network: new StacksTestnet(),
    contractAddress: 'ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM',
    contractName: 'nft-marketplace',
    functionName: 'purchase',
    functionArgs: [uintCV(nftId)],
    postConditions: [stxCondition, nftCondition],
    postConditionMode: PostConditionMode.Deny,
    onFinish: (data) => {
      console.log('NFT purchased:', data.txId);
    }
  };
  
  await openContractCall(options);
}
```

### Example 4: Staking with Rewards

```javascript
async function stakeTokens(amount) {
  const stakerAddress = userSession.loadUserData().profile.stxAddress.testnet;
  const contractAddress = 'ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM.staking-pool';
  
  const tokenAsset = 'ST1...staking-token::token';
  
  // Post-condition: User sends exactly stake amount
  const stakeCondition = makeStandardFungiblePostCondition(
    stakerAddress,
    FungibleConditionCode.Equal,
    amount,
    tokenAsset
  );
  
  const options = {
    network: new StacksTestnet(),
    contractAddress: 'ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM',
    contractName: 'staking-pool',
    functionName: 'stake',
    functionArgs: [
      principalCV('ST1...staking-token'),
      uintCV(amount)
    ],
    postConditions: [stakeCondition],
    postConditionMode: PostConditionMode.Deny,
    onFinish: (data) => {
      console.log('Tokens staked:', data.txId);
    }
  };
  
  await openContractCall(options);
}
```

---

## 🎨 Post-Condition Modes

### Deny Mode (Recommended)

```javascript
postConditionMode: PostConditionMode.Deny
```

**Behavior:** Transaction fails if ANY asset transfer occurs that isn't covered by a post-condition.

**Use when:** You want maximum security and know all assets that will be transferred.

### Allow Mode (Use Carefully)

```javascript
postConditionMode: PostConditionMode.Allow
```

**Behavior:** Transaction proceeds even if transfers occur that aren't in post-conditions.

**Use when:** 
- You can't predict all asset transfers
- Contract behavior is complex and varies
- You trust the contract completely

**⚠️ Warning:** This mode reduces user protection. Only use with trusted contracts.

---

## 🛡️ Best Practices

### 1. Always Use Deny Mode

```javascript
// Good
const options = {
  postConditions: [...],
  postConditionMode: PostConditionMode.Deny,
  ...
};

// Risky
const options = {
  postConditionMode: PostConditionMode.Allow,  // Avoid if possible
  ...
};
```

### 2. Specify Exact Amounts When Possible

```javascript
// Good: Exact amount
makeStandardFungiblePostCondition(
  address,
  FungibleConditionCode.Equal,
  amount,
  asset
);

// Less ideal: Range (use only when necessary)
makeStandardFungiblePostCondition(
  address,
  FungibleConditionCode.LessEqual,
  maxAmount,
  asset
);
```

### 3. Include All Asset Transfers

```javascript
// If contract transfers multiple assets, include all
const postConditions = [
  // STX transfer
  makeStandardSTXPostCondition(...),
  
  // Token A transfer
  makeStandardFungiblePostCondition(..., tokenA),
  
  // Token B transfer
  makeStandardFungiblePostCondition(..., tokenB),
];
```

### 4. Calculate Amounts Carefully

```javascript
// Account for decimals
const displayAmount = 10;  // 10 tokens
const decimals = 6;
const rawAmount = displayAmount * Math.pow(10, decimals);  // 10000000

// Account for fees
const swapAmount = 1000000;
const feePercent = 0.003;  // 0.3%
const fee = Math.floor(swapAmount * feePercent);
const amountAfterFee = swapAmount - fee;

// Use amountAfterFee in post-condition
```

### 5. Protect Against Front-Running

```javascript
// User specifies minimum acceptable output
const minAcceptableOutput = expectedOutput * 0.98;  // 2% slippage tolerance

const postCondition = makeStandardFungiblePostCondition(
  userAddress,
  FungibleConditionCode.GreaterEqual,  // At least this much
  minAcceptableOutput,
  outputToken
);
```

---

## 🧪 Testing Post-Conditions

### Test Success Case

```typescript
Clarinet.test({
    name: "Transfer succeeds with correct post-condition",
    async fn(chain: Chain, accounts: Map<string, Account>) {
        const sender = accounts.get('wallet_1')!;
        const recipient = accounts.get('wallet_2')!;
        
        let block = chain.mineBlock([
            Tx.contractCall(
                'my-token',
                'transfer',
                [
                    types.uint(1000000),
                    types.principal(sender.address),
                    types.principal(recipient.address),
                    types.none()
                ],
                sender.address
            )
        ]);
        
        // With correct post-condition, transfer succeeds
        block.receipts[0].result.expectOk();
    }
});
```

### Test Failure Case

```typescript
Clarinet.test({
    name: "Transfer fails with incorrect post-condition",
    async fn(chain: Chain, accounts: Map<string, Account>) {
        // Test that transaction fails if contract tries to transfer
        // more than specified in post-condition
    }
});
```

---

## 🎯 Common Patterns

### Pattern 1: Token Swap Protection

```javascript
function createSwapPostConditions(
  userAddress,
  amountIn,
  minAmountOut,
  tokenIn,
  tokenOut
) {
  return [
    // User sends exact input
    makeStandardFungiblePostCondition(
      userAddress,
      FungibleConditionCode.Equal,
      amountIn,
      tokenIn
    ),
    // User receives at least minimum output
    makeStandardFungiblePostCondition(
      userAddress,
      FungibleConditionCode.GreaterEqual,
      minAmountOut,
      tokenOut
    )
  ];
}
```

### Pattern 2: NFT Trade Protection

```javascript
function createNFTTradePostConditions(
  buyer,
  seller,
  price,
  nftAsset,
  tokenId
) {
  return [
    // Buyer sends exact payment
    makeStandardSTXPostCondition(
      buyer,
      FungibleConditionCode.Equal,
      price
    ),
    // Seller sends NFT
    makeStandardNonFungiblePostCondition(
      seller,
      NonFungibleConditionCode.Sends,
      nftAsset,
      uintCV(tokenId)
    )
  ];
}
```

### Pattern 3: Liquidity Provision

```javascript
function createLiquidityPostConditions(
  provider,
  amountA,
  amountB,
  tokenA,
  tokenB
) {
  return [
    // Provider sends token A
    makeStandardFungiblePostCondition(
      provider,
      FungibleConditionCode.Equal,
      amountA,
      tokenA
    ),
    // Provider sends token B
    makeStandardFungiblePostCondition(
      provider,
      FungibleConditionCode.Equal,
      amountB,
      tokenB
    )
  ];
}
```

---

## ⚠️ Common Mistakes

### 1. Wrong Amount Units

```javascript
// Wrong: Using display amount
const amount = 10;  // This is 0.00001 if token has 6 decimals!

// Correct: Convert to raw amount
const decimals = 6;
const amount = 10 * Math.pow(10, decimals);  // 10000000
```

### 2. Wrong Asset Format

```javascript
// Wrong
const asset = 'my-token';

// Correct
const asset = 'ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM.my-token::my-token';
//            ^contract-address        ^contract-name  ^asset-name
```

### 3. Missing Post-Conditions

```javascript
// Wrong: No protection
const options = {
  postConditions: [],  // Dangerous!
  postConditionMode: PostConditionMode.Allow
};

// Correct: Always protect user
const options = {
  postConditions: [appropriateCondition],
  postConditionMode: PostConditionMode.Deny
};
```

### 4. Using Allow Mode Unnecessarily

```javascript
// Wrong: Too permissive
postConditionMode: PostConditionMode.Allow

// Correct: Strict protection
postConditionMode: PostConditionMode.Deny
```

---

## 📚 Complete Example: DEX Interface

```javascript
function DEXSwapWithPostConditions() {
  const { userSession } = useWallet();
  
  const swap = async (amountIn, minAmountOut, tokenIn, tokenOut) => {
    const userAddress = userSession.loadUserData().profile.stxAddress.testnet;
    
    // Calculate amounts with decimals
    const amountInRaw = parseFloat(amountIn) * 1000000;
    const minAmountOutRaw = parseFloat(minAmountOut) * 1000000;
    
    // Create asset strings
    const tokenInAsset = `ST1...${tokenIn}::${tokenIn}`;
    const tokenOutAsset = `ST1...${tokenOut}::${tokenOut}`;
    
    // Create post-conditions
    const postConditions = [
      // User sends exact input
      makeStandardFungiblePostCondition(
        userAddress,
        FungibleConditionCode.Equal,
        amountInRaw,
        tokenInAsset
      ),
      // User receives at least minimum output
      makeStandardFungiblePostCondition(
        userAddress,
        FungibleConditionCode.GreaterEqual,
        minAmountOutRaw,
        tokenOutAsset
      )
    ];
    
    const options = {
      network: new StacksTestnet(),
      contractAddress: 'ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM',
      contractName: 'dex-router',
      functionName: 'swap-exact-tokens-for-tokens',
      functionArgs: [
        uintCV(amountInRaw),
        uintCV(minAmountOutRaw),
        listCV([
          principalCV(tokenInAsset.split('.')[0]),
          principalCV(tokenOutAsset.split('.')[0])
        ])
      ],
      postConditions: postConditions,
      postConditionMode: PostConditionMode.Deny,
      onFinish: (data) => {
        console.log('Swap successful:', data.txId);
      },
      onCancel: () => {
        console.log('User cancelled');
      }
    };
    
    await openContractCall(options);
  };
  
  return (
    // UI components
  );
}
```

---

## 🎓 Summary

**Post-Conditions:**
- ✅ Protect users from malicious contracts
- ✅ Specify exactly what assets can move
- ✅ Prevent front-running and manipulation
- ✅ Make transactions safer and more predictable

**Always:**
- Use `PostConditionMode.Deny`
- Include post-conditions for all asset transfers
- Test thoroughly
- Calculate amounts correctly

---

## 📚 Next Steps

**Related Guides:**
- → Security Best Practices
- → Common Vulnerabilities
- → Security Audit Checklist