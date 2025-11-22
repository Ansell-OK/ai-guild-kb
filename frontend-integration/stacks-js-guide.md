# Stacks.js - Complete Library Guide

## 📖 Overview

Stacks.js is the official JavaScript library for interacting with the Stacks blockchain. It provides everything needed to build full-featured Web3 applications, from wallet connections to contract interactions.

### What You'll Learn
- Stacks.js architecture and packages
- Authentication and wallet connections
- Reading blockchain data
- Calling smart contracts
- Transaction construction
- Post-conditions
- Network configuration

### Prerequisites
- JavaScript/TypeScript knowledge
- Understanding of async/await
- Basic React knowledge

---

## 📦 Stacks.js Packages

### Core Packages

```bash
# Essential packages for most dApps
npm install @stacks/connect @stacks/transactions @stacks/network

# Additional packages
npm install @stacks/auth          # Authentication
npm install @stacks/storage       # Gaia storage
npm install @stacks/encryption    # Data encryption
npm install @stacks/stacking      # Stacking operations
npm install @stacks/blockchain-api-client  # API client
```

### Package Overview

| Package | Purpose |
|---------|---------|
| `@stacks/connect` | Wallet connection and transactions |
| `@stacks/transactions` | Transaction construction and signing |
| `@stacks/network` | Network configuration |
| `@stacks/auth` | User authentication |
| `@stacks/storage` | Decentralized storage |

---

## 🔐 Authentication & User Sessions

### Basic Setup

```javascript
import { AppConfig, UserSession, showConnect } from '@stacks/connect';

// Configure your app
const appConfig = new AppConfig(
  ['store_write', 'publish_data'],  // Scopes
  'https://myapp.com'                // App domain
);

// Create user session
const userSession = new UserSession({ appConfig });

// Check if user is signed in
if (userSession.isSignInPending()) {
  userSession.handlePendingSignIn().then((userData) => {
    console.log('User signed in:', userData);
  });
} else if (userSession.isUserSignedIn()) {
  const userData = userSession.loadUserData();
  console.log('Already signed in:', userData);
}
```

### Connect Wallet

```javascript
// Show connect dialog
const connectWallet = () => {
  showConnect({
    appDetails: {
      name: 'My DApp',
      icon: 'https://myapp.com/logo.png',
    },
    redirectTo: '/',
    onFinish: () => {
      const userData = userSession.loadUserData();
      console.log('Connected:', userData);
    },
    onCancel: () => {
      console.log('User cancelled');
    },
    userSession,
  });
};
```

### User Data Structure

```javascript
const userData = userSession.loadUserData();

console.log(userData.profile);
// {
//   stxAddress: {
//     mainnet: 'SP...',
//     testnet: 'ST...'
//   },
//   btcAddress: {
//     p2wpkh: {
//       mainnet: 'bc1...',
//       testnet: 'tb1...'
//     }
//   }
// }

// Get user's STX address
const stxAddress = userData.profile.stxAddress.testnet;
```

---

## 🌐 Network Configuration

### Network Types

```javascript
import { 
  StacksMainnet, 
  StacksTestnet,
  StacksMocknet 
} from '@stacks/network';

// Mainnet
const mainnet = new StacksMainnet();
console.log(mainnet.coreApiUrl);  // https://api.mainnet.hiro.so

// Testnet
const testnet = new StacksTestnet();
console.log(testnet.coreApiUrl);  // https://api.testnet.hiro.so

// Mocknet (local development)
const mocknet = new StacksMocknet();
```

### Custom Network

```javascript
import { StacksNetwork } from '@stacks/network';

const customNetwork = new StacksNetwork({
  url: 'https://my-custom-node.com',
  chainId: ChainID.Testnet
});
```

---

## 📖 Reading Blockchain Data

### Get Account Balance

```javascript
import { 
  AccountsApi, 
  Configuration 
} from '@stacks/blockchain-api-client';
import { StacksTestnet } from '@stacks/network';

async function getBalance(address) {
  const network = new StacksTestnet();
  
  const config = new Configuration({
    basePath: network.coreApiUrl,
  });
  
  const accountsApi = new AccountsApi(config);
  
  try {
    const balance = await accountsApi.getAccountBalance({
      principal: address,
    });
    
    return {
      stx: balance.stx.balance,
      locked: balance.stx.locked,
      fungibleTokens: balance.fungible_tokens,
      nonFungibleTokens: balance.non_fungible_tokens
    };
  } catch (error) {
    console.error('Error fetching balance:', error);
    throw error;
  }
}

// Usage
const balance = await getBalance('ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM');
console.log(`Balance: ${balance.stx / 1000000} STX`);
```

### Read-Only Contract Calls

```javascript
import { 
  callReadOnlyFunction,
  cvToValue,
  uintCV,
  standardPrincipalCV
} from '@stacks/transactions';
import { StacksTestnet } from '@stacks/network';

async function getTokenBalance(tokenContract, userAddress) {
  const network = new StacksTestnet();
  
  const options = {
    contractAddress: 'ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM',
    contractName: 'my-token',
    functionName: 'get-balance',
    functionArgs: [standardPrincipalCV(userAddress)],
    network: network,
    senderAddress: userAddress,
  };
  
  try {
    const result = await callReadOnlyFunction(options);
    const value = cvToValue(result);
    return value.value;
  } catch (error) {
    console.error('Error reading contract:', error);
    throw error;
  }
}

// Usage
const balance = await getTokenBalance(
  'my-token',
  'ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM'
);
```

### Get Transaction Info

```javascript
import { TransactionsApi } from '@stacks/blockchain-api-client';

async function getTransaction(txId) {
  const network = new StacksTestnet();
  const config = new Configuration({
    basePath: network.coreApiUrl,
  });
  
  const txApi = new TransactionsApi(config);
  
  const tx = await txApi.getTransactionById({ txId });
  return tx;
}

// Usage
const tx = await getTransaction('0x1234...');
console.log('Status:', tx.tx_status);
console.log('Type:', tx.tx_type);
```

---

## 🔨 Transaction Construction

### Clarity Value (CV) Types

```javascript
import {
  uintCV,
  intCV,
  boolCV,
  stringAsciiCV,
  stringUtf8CV,
  standardPrincipalCV,
  contractPrincipalCV,
  bufferCV,
  listCV,
  tupleCV,
  someCV,
  noneCV
} from '@stacks/transactions';

// Basic types
const uint = uintCV(100);
const int = intCV(-50);
const bool = boolCV(true);

// Strings
const ascii = stringAsciiCV('Hello');
const utf8 = stringUtf8CV('Hello 👋');

// Principals
const standardAddr = standardPrincipalCV('ST1...');
const contractAddr = contractPrincipalCV('ST1...', 'my-contract');

// Buffer
const buffer = bufferCV(Buffer.from('data'));

// List
const list = listCV([
  uintCV(1),
  uintCV(2),
  uintCV(3)
]);

// Tuple
const tuple = tupleCV({
  'name': stringAsciiCV('Alice'),
  'age': uintCV(30),
  'active': boolCV(true)
});

// Optional
const some = someCV(uintCV(100));
const none = noneCV();
```

### Convert CV to JavaScript

```javascript
import { cvToValue, cvToJSON } from '@stacks/transactions';

// Example: Reading contract response
const result = await callReadOnlyFunction(options);

// Convert to JavaScript value
const jsValue = cvToValue(result);
console.log(jsValue);

// Convert to JSON
const jsonValue = cvToJSON(result);
console.log(jsonValue);
```

---

## 📝 Contract Calls

### Simple Contract Call

```javascript
import { openContractCall } from '@stacks/connect';
import { 
  uintCV, 
  standardPrincipalCV,
  PostConditionMode 
} from '@stacks/transactions';

async function transferTokens(recipient, amount) {
  const options = {
    network: new StacksTestnet(),
    contractAddress: 'ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM',
    contractName: 'my-token',
    functionName: 'transfer',
    functionArgs: [
      uintCV(amount * 1000000),  // Convert to raw amount
      standardPrincipalCV(userAddress),
      standardPrincipalCV(recipient),
      noneCV()  // Optional memo
    ],
    postConditionMode: PostConditionMode.Deny,
    postConditions: [
      // Add post-conditions for safety
    ],
    onFinish: (data) => {
      console.log('Transaction ID:', data.txId);
      console.log('Transaction:', data);
    },
    onCancel: () => {
      console.log('User cancelled');
    },
  };
  
  await openContractCall(options);
}
```

### Contract Call with Post-Conditions

```javascript
import { 
  makeStandardFungiblePostCondition,
  FungibleConditionCode
} from '@stacks/transactions';

async function safeTransfer(recipient, amount) {
  const amountRaw = amount * 1000000;
  
  // Create post-condition
  const postCondition = makeStandardFungiblePostCondition(
    userAddress,
    FungibleConditionCode.Equal,
    amountRaw,
    'ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM.my-token::my-token'
  );
  
  const options = {
    network: new StacksTestnet(),
    contractAddress: 'ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM',
    contractName: 'my-token',
    functionName: 'transfer',
    functionArgs: [
      uintCV(amountRaw),
      standardPrincipalCV(userAddress),
      standardPrincipalCV(recipient),
      noneCV()
    ],
    postConditions: [postCondition],
    postConditionMode: PostConditionMode.Deny,
    onFinish: (data) => {
      console.log('Safe transfer complete:', data.txId);
    }
  };
  
  await openContractCall(options);
}
```

---

## 💰 STX Transfers

### Simple STX Transfer

```javascript
import { openSTXTransfer } from '@stacks/connect';
import { 
  makeStandardSTXPostCondition,
  FungibleConditionCode
} from '@stacks/transactions';

async function sendSTX(recipient, amount) {
  const amountMicroSTX = amount * 1000000;
  
  // Create post-condition
  const postCondition = makeStandardSTXPostCondition(
    userAddress,
    FungibleConditionCode.Equal,
    amountMicroSTX
  );
  
  const options = {
    network: new StacksTestnet(),
    recipient: recipient,
    amount: amountMicroSTX,
    memo: 'Payment',
    postConditions: [postCondition],
    postConditionMode: PostConditionMode.Deny,
    onFinish: (data) => {
      console.log('STX sent:', data.txId);
    }
  };
  
  await openSTXTransfer(options);
}
```

---

## 🔍 Advanced Features

### Batch Transactions

```javascript
// Multiple operations in one transaction
async function batchOperations() {
  const operations = [
    {
      contractAddress: 'ST1...',
      contractName: 'token-a',
      functionName: 'transfer',
      functionArgs: [/* args */]
    },
    {
      contractAddress: 'ST1...',
      contractName: 'token-b',
      functionName: 'transfer',
      functionArgs: [/* args */]
    }
  ];
  
  // Execute each operation
  for (const op of operations) {
    await openContractCall({
      ...op,
      network: new StacksTestnet(),
      postConditionMode: PostConditionMode.Deny
    });
  }
}
```

### Transaction Monitoring

```javascript
async function waitForTransaction(txId) {
  const network = new StacksTestnet();
  const config = new Configuration({
    basePath: network.coreApiUrl,
  });
  
  const txApi = new TransactionsApi(config);
  
  let attempts = 0;
  const maxAttempts = 30;
  
  while (attempts < maxAttempts) {
    try {
      const tx = await txApi.getTransactionById({ txId });
      
      if (tx.tx_status === 'success') {
        console.log('Transaction confirmed!');
        return tx;
      } else if (tx.tx_status === 'abort_by_response' || tx.tx_status === 'abort_by_post_condition') {
        console.error('Transaction failed:', tx.tx_status);
        throw new Error(`Transaction failed: ${tx.tx_status}`);
      }
      
      // Wait 5 seconds before checking again
      await new Promise(resolve => setTimeout(resolve, 5000));
      attempts++;
    } catch (error) {
      if (error.status === 404) {
        // Transaction not found yet, wait and retry
        await new Promise(resolve => setTimeout(resolve, 5000));
        attempts++;
      } else {
        throw error;
      }
    }
  }
  
  throw new Error('Transaction confirmation timeout');
}

// Usage
const txId = '0x1234...';
const confirmedTx = await waitForTransaction(txId);
```

### Message Signing

```javascript
import { openSignatureRequestPopup } from '@stacks/connect';

async function signMessage(message) {
  await openSignatureRequestPopup({
    message: message,
    network: new StacksTestnet(),
    onFinish: (data) => {
      console.log('Signature:', data.signature);
      console.log('Public key:', data.publicKey);
    }
  });
}
```

---

## 🎨 Helper Utilities

### Address Validation

```javascript
import { validateStacksAddress } from '@stacks/transactions';

function isValidAddress(address) {
  try {
    return validateStacksAddress(address);
  } catch (error) {
    return false;
  }
}

// Usage
console.log(isValidAddress('ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM'));  // true
console.log(isValidAddress('invalid'));  // false
```

### Amount Formatting

```javascript
// Convert microSTX to STX
function microToSTX(micro) {
  return micro / 1000000;
}

// Convert STX to microSTX
function stxToMicro(stx) {
  return stx * 1000000;
}

// Format for display
function formatSTX(micro) {
  return `${microToSTX(micro).toFixed(6)} STX`;
}

// Usage
console.log(formatSTX(1500000));  // "1.500000 STX"
```

### Short Address Display

```javascript
function shortenAddress(address, chars = 4) {
  if (!address) return '';
  return `${address.slice(0, chars + 2)}...${address.slice(-chars)}`;
}

// Usage
console.log(shortenAddress('ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM'));
// "ST1PQH...ZGZGM"
```

---

## 🧪 Testing

### Mock UserSession

```javascript
// For testing without actual wallet
class MockUserSession {
  isUserSignedIn() {
    return true;
  }
  
  loadUserData() {
    return {
      profile: {
        stxAddress: {
          mainnet: 'SP2J6ZY48GV1EZ5V2V5RB9MP66SW86PYKKNRV9EJ7',
          testnet: 'ST2J6ZY48GV1EZ5V2V5RB9MP66SW86PYKKPVKG2CE'
        }
      }
    };
  }
}

// Use in tests
const mockSession = new MockUserSession();
```

---

## 📚 Complete Example

### Full Integration Example

```javascript
import { 
  AppConfig, 
  UserSession, 
  showConnect,
  openContractCall
} from '@stacks/connect';
import {
  uintCV,
  standardPrincipalCV,
  callReadOnlyFunction,
  cvToValue,
  makeStandardFungiblePostCondition,
  FungibleConditionCode,
  PostConditionMode
} from '@stacks/transactions';
import { StacksTestnet } from '@stacks/network';

class StacksClient {
  constructor() {
    this.appConfig = new AppConfig(['store_write', 'publish_data']);
    this.userSession = new UserSession({ appConfig: this.appConfig });
    this.network = new StacksTestnet();
  }
  
  // Connect wallet
  async connect() {
    return new Promise((resolve, reject) => {
      showConnect({
        appDetails: {
          name: 'My DApp',
          icon: window.location.origin + '/logo.png',
        },
        redirectTo: '/',
        onFinish: () => {
          const userData = this.userSession.loadUserData();
          resolve(userData);
        },
        onCancel: () => {
          reject(new Error('User cancelled'));
        },
        userSession: this.userSession,
      });
    });
  }
  
  // Get user address
  getUserAddress() {
    if (!this.userSession.isUserSignedIn()) {
      return null;
    }
    const userData = this.userSession.loadUserData();
    return userData.profile.stxAddress.testnet;
  }
  
  // Read contract
  async readContract(contractAddress, contractName, functionName, args = []) {
    const options = {
      contractAddress,
      contractName,
      functionName,
      functionArgs: args,
      network: this.network,
      senderAddress: contractAddress,
    };
    
    const result = await callReadOnlyFunction(options);
    return cvToValue(result);
  }
  
  // Call contract
  async callContract(contractAddress, contractName, functionName, args, postConditions = []) {
    return new Promise((resolve, reject) => {
      openContractCall({
        network: this.network,
        contractAddress,
        contractName,
        functionName,
        functionArgs: args,
        postConditions,
        postConditionMode: PostConditionMode.Deny,
        onFinish: (data) => resolve(data),
        onCancel: () => reject(new Error('User cancelled')),
      });
    });
  }
  
  // Transfer tokens with post-condition
  async transferTokens(tokenContract, recipient, amount) {
    const userAddress = this.getUserAddress();
    const amountRaw = amount * 1000000;
    
    const postCondition = makeStandardFungiblePostCondition(
      userAddress,
      FungibleConditionCode.Equal,
      amountRaw,
      `${tokenContract}::token`
    );
    
    return this.callContract(
      tokenContract.split('.')[0],
      tokenContract.split('.')[1],
      'transfer',
      [
        uintCV(amountRaw),
        standardPrincipalCV(userAddress),
        standardPrincipalCV(recipient),
        noneCV()
      ],
      [postCondition]
    );
  }
}

// Usage
const client = new StacksClient();

// Connect
await client.connect();

// Read balance
const balance = await client.readContract(
  'ST1...',
  'my-token',
  'get-balance',
  [standardPrincipalCV(client.getUserAddress())]
);

// Transfer tokens
await client.transferTokens(
  'ST1...my-token',
  'ST2...',
  10
);
```

---

## 🎓 Best Practices

1. **Always use post-conditions** for user protection
2. **Handle errors gracefully** with try/catch
3. **Validate inputs** before sending transactions
4. **Use TypeScript** for better type safety
5. **Cache network responses** to reduce API calls
6. **Test on testnet** before mainnet
7. **Monitor transactions** for user feedback

---

## 📚 Next Steps

**Related Guides:**
- → React Integration Patterns
- → Advanced Contract Calls
- → Error Handling in Frontend