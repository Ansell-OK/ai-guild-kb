# Advanced Contract Calls - Complete Guide

## 📖 Overview

This guide covers advanced patterns for interacting with Clarity smart contracts, including complex data structures, batch operations, transaction monitoring, and error handling strategies.

### What You'll Learn
- Complex Clarity value construction
- Multi-contract interactions
- Transaction batching
- Advanced post-conditions
- Error recovery strategies
- Transaction monitoring
- Gas optimization

---

## 🔨 Complex Clarity Values

### Working with Tuples

```typescript
import { tupleCV, uintCV, stringAsciiCV, boolCV, principalCV } from '@stacks/transactions';

// Create complex tuple
const userProfile = tupleCV({
  'user-id': uintCV(123),
  'username': stringAsciiCV('alice'),
  'email': stringAsciiCV('alice@example.com'),
  'active': boolCV(true),
  'registered-at': uintCV(Date.now())
});

// Nested tuple
const orderData = tupleCV({
  'order-id': uintCV(456),
  'buyer': principalCV('ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM'),
  'items': listCV([
    tupleCV({
      'item-id': uintCV(1),
      'quantity': uintCV(2),
      'price': uintCV(100000)
    }),
    tupleCV({
      'item-id': uintCV(2),
      'quantity': uintCV(1),
      'price': uintCV(50000)
    })
  ]),
  'total': uintCV(250000)
});
```

### Working with Lists

```typescript
import { listCV, uintCV, principalCV } from '@stacks/transactions';

// List of numbers
const tokenIds = listCV([
  uintCV(1),
  uintCV(2),
  uintCV(3)
]);

// List of addresses
const recipients = listCV([
  principalCV('ST1...'),
  principalCV('ST2...'),
  principalCV('ST3...')
]);

// List of tuples
const transactions = listCV([
  tupleCV({
    'recipient': principalCV('ST1...'),
    'amount': uintCV(1000000)
  }),
  tupleCV({
    'recipient': principalCV('ST2...'),
    'amount': uintCV(2000000)
  })
]);
```

### Working with Optionals

```typescript
import { someCV, noneCV, uintCV, bufferCV } from '@stacks/transactions';

// Optional with value
const memo = someCV(bufferCV(Buffer.from('Payment for services')));

// Optional without value
const noMemo = noneCV();

// Conditional optional
const getMemo = (text: string | null) => {
  return text ? someCV(bufferCV(Buffer.from(text))) : noneCV();
};
```

---

## 🔄 Multi-Contract Interactions

### Sequential Contract Calls

```typescript
async function setupLiquidityPool(tokenA: string, tokenB: string, amountA: number, amountB: number) {
  // Step 1: Approve token A
  await openContractCall({
    contractAddress: tokenA.split('.')[0],
    contractName: tokenA.split('.')[1],
    functionName: 'approve',
    functionArgs: [
      principalCV('ST1...liquidity-pool'),
      uintCV(amountA * 1000000)
    ],
    postConditionMode: PostConditionMode.Deny,
    onFinish: async (data1) => {
      console.log('Token A approved:', data1.txId);
      
      // Step 2: Approve token B
      await openContractCall({
        contractAddress: tokenB.split('.')[0],
        contractName: tokenB.split('.')[1],
        functionName: 'approve',
        functionArgs: [
          principalCV('ST1...liquidity-pool'),
          uintCV(amountB * 1000000)
        ],
        postConditionMode: PostConditionMode.Deny,
        onFinish: async (data2) => {
          console.log('Token B approved:', data2.txId);
          
          // Step 3: Add liquidity
          await openContractCall({
            contractAddress: 'ST1...',
            contractName: 'liquidity-pool',
            functionName: 'add-liquidity',
            functionArgs: [
              principalCV(tokenA),
              principalCV(tokenB),
              uintCV(amountA * 1000000),
              uintCV(amountB * 1000000)
            ],
            postConditionMode: PostConditionMode.Deny,
            onFinish: (data3) => {
              console.log('Liquidity added:', data3.txId);
            }
          });
        }
      });
    }
  });
}
```

### Parallel Data Fetching

```typescript
async function fetchAllUserData(address: string) {
  const network = new StacksTestnet();
  
  // Fetch multiple contract states in parallel
  const [tokenBalance, nftBalance, stakedAmount, votingPower] = await Promise.all([
    callReadOnlyFunction({
      contractAddress: 'ST1...',
      contractName: 'token',
      functionName: 'get-balance',
      functionArgs: [principalCV(address)],
      network,
      senderAddress: address
    }),
    callReadOnlyFunction({
      contractAddress: 'ST1...',
      contractName: 'nft',
      functionName: 'get-balance',
      functionArgs: [principalCV(address)],
      network,
      senderAddress: address
    }),
    callReadOnlyFunction({
      contractAddress: 'ST1...',
      contractName: 'staking',
      functionName: 'get-stake',
      functionArgs: [principalCV(address)],
      network,
      senderAddress: address
    }),
    callReadOnlyFunction({
      contractAddress: 'ST1...',
      contractName: 'governance',
      functionName: 'get-voting-power',
      functionArgs: [principalCV(address)],
      network,
      senderAddress: address
    })
  ]);
  
  return {
    tokenBalance: cvToValue(tokenBalance),
    nftBalance: cvToValue(nftBalance),
    stakedAmount: cvToValue(stakedAmount),
    votingPower: cvToValue(votingPower)
  };
}
```

---

## 📦 Batch Operations

### Batch Token Transfers

```typescript
interface Transfer {
  recipient: string;
  amount: number;
}

async function batchTransfer(transfers: Transfer[]) {
  const network = new StacksTestnet();
  const userAddress = getUserAddress();
  
  // Calculate total amount for post-condition
  const totalAmount = transfers.reduce((sum, t) => sum + t.amount, 0);
  
  // Create list of transfers
  const transferList = listCV(
    transfers.map(t => 
      tupleCV({
        'recipient': principalCV(t.recipient),
        'amount': uintCV(t.amount * 1000000)
      })
    )
  );
  
  // Create post-condition for total
  const postCondition = makeStandardFungiblePostCondition(
    userAddress,
    FungibleConditionCode.Equal,
    totalAmount * 1000000,
    'ST1...token::token'
  );
  
  await openContractCall({
    network,
    contractAddress: 'ST1...',
    contractName: 'batch-transfer',
    functionName: 'batch-transfer',
    functionArgs: [transferList],
    postConditions: [postCondition],
    postConditionMode: PostConditionMode.Deny,
    onFinish: (data) => {
      console.log('Batch transfer complete:', data.txId);
    }
  });
}

// Usage
await batchTransfer([
  { recipient: 'ST1...', amount: 10 },
  { recipient: 'ST2...', amount: 20 },
  { recipient: 'ST3...', amount: 30 }
]);
```

---

## 🛡️ Advanced Post-Conditions

### Dynamic Post-Condition Creation

```typescript
function createSwapPostConditions(
  userAddress: string,
  tokenIn: string,
  tokenOut: string,
  amountIn: number,
  minAmountOut: number
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

// Usage in swap
async function executeSwap(tokenIn: string, tokenOut: string, amountIn: number, slippage: number) {
  const userAddress = getUserAddress();
  
  // Calculate minimum output with slippage
  const expectedOutput = await calculateSwapOutput(tokenIn, tokenOut, amountIn);
  const minAmountOut = Math.floor(expectedOutput * (1 - slippage));
  
  const postConditions = createSwapPostConditions(
    userAddress,
    tokenIn,
    tokenOut,
    amountIn,
    minAmountOut
  );
  
  await openContractCall({
    // ... swap configuration
    postConditions,
    postConditionMode: PostConditionMode.Deny
  });
}
```

### Multi-Asset Post-Conditions

```typescript
async function complexTrade(params: {
  sendSTX: number;
  sendTokenA: number;
  receiveTokenB: number;
  receiveNFT: number;
}) {
  const userAddress = getUserAddress();
  const sellerAddress = 'ST2...';
  
  const postConditions = [
    // Buyer sends STX
    makeStandardSTXPostCondition(
      userAddress,
      FungibleConditionCode.Equal,
      params.sendSTX
    ),
    // Buyer sends Token A
    makeStandardFungiblePostCondition(
      userAddress,
      FungibleConditionCode.Equal,
      params.sendTokenA,
      'ST1...token-a::token-a'
    ),
    // Buyer receives Token B
    makeStandardFungiblePostCondition(
      userAddress,
      FungibleConditionCode.GreaterEqual,
      params.receiveTokenB,
      'ST1...token-b::token-b'
    ),
    // Seller sends NFT
    makeStandardNonFungiblePostCondition(
      sellerAddress,
      NonFungibleConditionCode.Sends,
      createAssetInfo('ST1...', 'nft', 'nft-asset'),
      uintCV(params.receiveNFT)
    )
  ];
  
  await openContractCall({
    // ... trade configuration
    postConditions,
    postConditionMode: PostConditionMode.Deny
  });
}
```

---

## 📡 Transaction Monitoring

### Advanced Transaction Watcher

```typescript
class TransactionWatcher {
  private network: StacksTestnet;
  private txApi: TransactionsApi;
  private intervals: Map<string, NodeJS.Timeout>;
  
  constructor() {
    this.network = new StacksTestnet();
    const config = new Configuration({
      basePath: this.network.coreApiUrl
    });
    this.txApi = new TransactionsApi(config);
    this.intervals = new Map();
  }
  
  async waitForConfirmation(
    txId: string,
    onUpdate?: (status: string) => void,
    timeout = 300000 // 5 minutes
  ): Promise<any> {
    const startTime = Date.now();
    
    return new Promise((resolve, reject) => {
      const check = async () => {
        try {
          const tx = await this.txApi.getTransactionById({ txId });
          
          onUpdate?.(tx.tx_status);
          
          if (tx.tx_status === 'success') {
            this.stopWatching(txId);
            resolve(tx);
          } else if (tx.tx_status === 'abort_by_response' || tx.tx_status === 'abort_by_post_condition') {
            this.stopWatching(txId);
            reject(new Error(`Transaction failed: ${tx.tx_status}`));
          } else if (Date.now() - startTime > timeout) {
            this.stopWatching(txId);
            reject(new Error('Transaction confirmation timeout'));
          }
        } catch (error: any) {
          if (error.status !== 404) {
            this.stopWatching(txId);
            reject(error);
          }
          // 404 means transaction not yet in mempool, continue waiting
        }
      };
      
      // Check immediately
      check();
      
      // Then check every 5 seconds
      const interval = setInterval(check, 5000);
      this.intervals.set(txId, interval);
    });
  }
  
  stopWatching(txId: string) {
    const interval = this.intervals.get(txId);
    if (interval) {
      clearInterval(interval);
      this.intervals.delete(txId);
    }
  }
  
  stopAll() {
    this.intervals.forEach(interval => clearInterval(interval));
    this.intervals.clear();
  }
}

// Usage
const watcher = new TransactionWatcher();

try {
  const tx = await watcher.waitForConfirmation(
    txId,
    (status) => {
      console.log('Transaction status:', status);
      // Update UI
    }
  );
  console.log('Transaction confirmed!', tx);
} catch (error) {
  console.error('Transaction failed:', error);
}
```

### React Hook for Transaction Monitoring

```typescript
function useTransactionMonitor(txId: string | null) {
  const [status, setStatus] = useState<string>('pending');
  const [transaction, setTransaction] = useState<any>(null);
  const [error, setError] = useState<Error | null>(null);
  
  useEffect(() => {
    if (!txId) return;
    
    const watcher = new TransactionWatcher();
    
    watcher.waitForConfirmation(
      txId,
      (newStatus) => setStatus(newStatus)
    ).then((tx) => {
      setTransaction(tx);
      setStatus('confirmed');
    }).catch((err) => {
      setError(err);
      setStatus('failed');
    });
    
    return () => watcher.stopWatching(txId);
  }, [txId]);
  
  return { status, transaction, error };
}

// Usage in component
function TransactionStatus({ txId }: { txId: string }) {
  const { status, transaction, error } = useTransactionMonitor(txId);
  
  if (error) {
    return <div className="error">Error: {error.message}</div>;
  }
  
  return (
    <div className="tx-status">
      <span>Status: {status}</span>
      {status === 'confirmed' && (
        <a href={`https://explorer.stacks.co/txid/${txId}`}>
          View on Explorer
        </a>
      )}
    </div>
  );
}
```

---

## 🚨 Error Handling & Recovery

### Comprehensive Error Handler

```typescript
class ContractCallError extends Error {
  constructor(
    message: string,
    public code: string,
    public details?: any
  ) {
    super(message);
    this.name = 'ContractCallError';
  }
}

async function safeContractCall(options: any): Promise<any> {
  try {
    return await new Promise((resolve, reject) => {
      openContractCall({
        ...options,
        onFinish: (data) => resolve(data),
        onCancel: () => reject(new ContractCallError(
          'User cancelled transaction',
          'USER_CANCELLED'
        ))
      });
    });
  } catch (error: any) {
    // Parse and categorize error
    if (error.message?.includes('Insufficient balance')) {
      throw new ContractCallError(
        'Insufficient balance to complete transaction',
        'INSUFFICIENT_BALANCE',
        error
      );
    } else if (error.message?.includes('Post-condition')) {
      throw new ContractCallError(
        'Transaction would violate post-conditions',
        'POST_CONDITION_FAILED',
        error
      );
    } else if (error.code === 'USER_CANCELLED') {
      throw error;
    } else {
      throw new ContractCallError(
        'Transaction failed',
        'UNKNOWN_ERROR',
        error
      );
    }
  }
}

// Usage with error handling
try {
  const result = await safeContractCall({
    contractAddress: 'ST1...',
    contractName: 'my-contract',
    functionName: 'my-function',
    functionArgs: [...]
  });
  console.log('Success:', result);
} catch (error) {
  if (error instanceof ContractCallError) {
    switch (error.code) {
      case 'USER_CANCELLED':
        console.log('User cancelled');
        break;
      case 'INSUFFICIENT_BALANCE':
        console.error('Not enough funds');
        break;
      case 'POST_CONDITION_FAILED':
        console.error('Safety check failed');
        break;
      default:
        console.error('Unknown error:', error.message);
    }
  }
}
```

### Retry Logic

```typescript
async function retryContractCall(
  callFn: () => Promise<any>,
  maxRetries = 3,
  delay = 1000
): Promise<any> {
  let lastError: Error;
  
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await callFn();
    } catch (error: any) {
      lastError = error;
      
      // Don't retry user cancellations
      if (error.code === 'USER_CANCELLED') {
        throw error;
      }
      
      if (i < maxRetries - 1) {
        console.log(`Retry ${i + 1}/${maxRetries} after ${delay}ms`);
        await new Promise(resolve => setTimeout(resolve, delay));
        delay *= 2; // Exponential backoff
      }
    }
  }
  
  throw lastError!;
}

// Usage
const result = await retryContractCall(
  () => safeContractCall({
    contractAddress: 'ST1...',
    contractName: 'my-contract',
    functionName: 'my-function',
    functionArgs: [...]
  }),
  3,  // Max 3 retries
  2000 // Initial delay 2 seconds
);
```

---

## ⚡ Gas Optimization

### Estimate Gas Before Calling

```typescript
async function estimateTransactionFee(options: any): Promise<number> {
  // Estimate based on transaction complexity
  const baseGas = 1000;
  const gasPerArg = 100;
  const gasPerPostCondition = 50;
  
  const totalGas = 
    baseGas +
    (options.functionArgs?.length || 0) * gasPerArg +
    (options.postConditions?.length || 0) * gasPerPostCondition;
  
  return totalGas;
}

// Show estimate before transaction
async function callWithGasEstimate(options: any) {
  const estimatedFee = await estimateTransactionFee(options);
  
  const confirmed = window.confirm(
    `This transaction will cost approximately ${estimatedFee} microSTX. Continue?`
  );
  
  if (confirmed) {
    await openContractCall(options);
  }
}
```

---

## 📚 Complete Advanced Example

```typescript
// Advanced DeFi interaction example
class DeFiClient {
  private network: StacksTestnet;
  private watcher: TransactionWatcher;
  
  constructor() {
    this.network = new StacksTestnet();
    this.watcher = new TransactionWatcher();
  }
  
  async swapAndStake(
    tokenIn: string,
    amountIn: number,
    minAmountOut: number,
    onProgress: (step: string) => void
  ) {
    try {
      // Step 1: Swap tokens
      onProgress('Swapping tokens...');
      const swapTxId = await this.swap(tokenIn, amountIn, minAmountOut);
      
      // Wait for swap confirmation
      await this.watcher.waitForConfirmation(swapTxId);
      onProgress('Swap confirmed');
      
      // Step 2: Stake received tokens
      onProgress('Staking tokens...');
      const stakeTxId = await this.stake(minAmountOut);
      
      // Wait for stake confirmation
      await this.watcher.waitForConfirmation(stakeTxId);
      onProgress('Stake confirmed');
      
      return { swapTxId, stakeTxId };
    } catch (error) {
      throw new ContractCallError(
        'Swap and stake failed',
        'OPERATION_FAILED',
        error
      );
    }
  }
  
  private async swap(tokenIn: string, amountIn: number, minAmountOut: number) {
    // Implementation
    return 'swap-tx-id';
  }
  
  private async stake(amount: number) {
    // Implementation
    return 'stake-tx-id';
  }
}
```

---

## 🎓 Best Practices

1. **Always use post-conditions** for user protection
2. **Handle all error cases** explicitly
3. **Monitor transactions** until confirmation
4. **Batch operations** when possible
5. **Use TypeScript** for type safety
6. **Test on testnet** thoroughly
7. **Implement retry logic** for network issues
8. **Provide user feedback** at every step

---

## 📚 Next Steps

**Related Guides:**
- → Stacks.js Complete Guide
- → React Integration Patterns
- → Post-Conditions Deep Dive