# Wallet Connection & Integration Guide

## 📖 Overview

Connecting to Stacks wallets is the foundation of any Web3 application. This guide covers everything you need to integrate wallet functionality into your dApp, from basic connection to advanced transaction handling.

### What You'll Learn
- Connecting to Leather and Xverse wallets
- Managing wallet state and sessions
- Reading wallet data
- Signing transactions
- Handling post-conditions
- Multi-wallet support

### Prerequisites
- Basic React knowledge
- Understanding of async/await
- Familiarity with wallet concepts

---

## 🎯 Key Concepts

### Wallet Providers

**Leather (formerly Hiro Wallet)**
- Browser extension wallet
- Most popular Stacks wallet
- Supports STX, BTC, and tokens

**Xverse**
- Mobile and browser wallet
- Bitcoin and Stacks support
- Ordinals and NFT focused

### Stacks Connect

`@stacks/connect` is the official library for wallet integration. It provides:
- Unified interface for all wallets
- Transaction signing
- Message signing
- Authentication

### User Sessions

User sessions maintain wallet connection state across page refreshes and provide access to user data.

---

## 💻 Basic Wallet Connection

### Installation

```bash
npm install @stacks/connect @stacks/transactions @stacks/network
```

### Complete Connection Component

```javascript
import React, { useState, useEffect } from 'react';
import {
  AppConfig,
  UserSession,
  showConnect,
  openContractCall
} from '@stacks/connect';
import { StacksTestnet, StacksMainnet } from '@stacks/network';

function WalletConnect() {
  // Initialize app configuration
  const appConfig = new AppConfig(['store_write', 'publish_data']);
  const userSession = new UserSession({ appConfig });

  // State management
  const [userData, setUserData] = useState(null);
  const [isConnecting, setIsConnecting] = useState(false);

  // Check if user is already authenticated
  useEffect(() => {
    if (userSession.isSignInPending()) {
      userSession.handlePendingSignIn().then((userData) => {
        setUserData(userData);
      });
    } else if (userSession.isUserSignedIn()) {
      setUserData(userSession.loadUserData());
    }
  }, []);

  // Connect wallet function
  const connectWallet = () => {
    setIsConnecting(true);
    
    showConnect({
      appDetails: {
        name: 'My Stacks App',
        icon: window.location.origin + '/logo.png',
      },
      redirectTo: '/',
      onFinish: () => {
        const userData = userSession.loadUserData();
        setUserData(userData);
        setIsConnecting(false);
      },
      onCancel: () => {
        setIsConnecting(false);
      },
      userSession: userSession,
    });
  };

  // Disconnect wallet
  const disconnectWallet = () => {
    userSession.signUserOut();
    setUserData(null);
  };

  return (
    <div className="wallet-connect">
      {!userData ? (
        <button
          onClick={connectWallet}
          disabled={isConnecting}
          className="connect-button"
        >
          {isConnecting ? 'Connecting...' : 'Connect Wallet'}
        </button>
      ) : (
        <div className="wallet-info">
          <div className="user-data">
            <p>Connected: {userData.profile.stxAddress.mainnet}</p>
            <button onClick={disconnectWallet} className="disconnect-button">
              Disconnect
            </button>
          </div>
        </div>
      )}
    </div>
  );
}

export default WalletConnect;
```

---

## 🔍 Accessing Wallet Data

### User Profile Information

```javascript
function UserProfile({ userSession }) {
  const [userData, setUserData] = useState(null);

  useEffect(() => {
    if (userSession.isUserSignedIn()) {
      const data = userSession.loadUserData();
      setUserData(data);
    }
  }, [userSession]);

  if (!userData) return null;

  return (
    <div className="user-profile">
      {/* Stacks Address */}
      <div className="address-info">
        <h3>Stacks Addresses</h3>
        <p><strong>Mainnet:</strong> {userData.profile.stxAddress.mainnet}</p>
        <p><strong>Testnet:</strong> {userData.profile.stxAddress.testnet}</p>
      </div>

      {/* BNS Name (if available) */}
      {userData.username && (
        <div className="bns-info">
          <h3>BNS Name</h3>
          <p>{userData.username}</p>
        </div>
      )}

      {/* Profile Data */}
      <div className="profile-data">
        <h3>Profile</h3>
        <pre>{JSON.stringify(userData.profile, null, 2)}</pre>
      </div>
    </div>
  );
}
```

### Getting STX Balance

```javascript
import { StacksTestnet } from '@stacks/network';
import { AccountsApi, Configuration } from '@stacks/blockchain-api-client';

async function getSTXBalance(address) {
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
      unlock_height: balance.stx.unlock_height,
    };
  } catch (error) {
    console.error('Error fetching balance:', error);
    return null;
  }
}

// Usage in component
function BalanceDisplay({ address }) {
  const [balance, setBalance] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    if (address) {
      getSTXBalance(address).then((data) => {
        setBalance(data);
        setLoading(false);
      });
    }
  }, [address]);

  if (loading) return <div>Loading balance...</div>;
  if (!balance) return <div>Unable to load balance</div>;

  // Convert microSTX to STX (1 STX = 1,000,000 microSTX)
  const stxBalance = (balance.stx / 1_000_000).toFixed(6);
  const lockedBalance = (balance.locked / 1_000_000).toFixed(6);

  return (
    <div className="balance-display">
      <p><strong>Available STX:</strong> {stxBalance}</p>
      <p><strong>Locked STX:</strong> {lockedBalance}</p>
    </div>
  );
}
```

---

## 📝 Calling Smart Contracts

### Read-Only Function Calls

```javascript
import { 
  callReadOnlyFunction, 
  cvToValue,
  uintCV,
  standardPrincipalCV 
} from '@stacks/transactions';
import { StacksTestnet } from '@stacks/network';

async function readContractData(contractAddress, contractName, functionName, functionArgs = []) {
  const network = new StacksTestnet();
  
  const options = {
    contractAddress: contractAddress,
    contractName: contractName,
    functionName: functionName,
    functionArgs: functionArgs,
    network: network,
    senderAddress: contractAddress, // Can be any address for read-only
  };

  try {
    const result = await callReadOnlyFunction(options);
    return cvToValue(result);
  } catch (error) {
    console.error('Error calling read-only function:', error);
    throw error;
  }
}

// Example: Get NFT owner
async function getNFTOwner(tokenId) {
  const owner = await readContractData(
    'ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM',
    'my-nft',
    'get-owner',
    [uintCV(tokenId)]
  );
  
  return owner;
}

// Example: Get token balance
async function getTokenBalance(userAddress) {
  const balance = await readContractData(
    'ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM',
    'my-token',
    'get-balance',
    [standardPrincipalCV(userAddress)]
  );
  
  return balance;
}
```

### Write Function Calls (Transactions)

```javascript
import { 
  openContractCall,
  PostConditionMode 
} from '@stacks/connect';
import {
  uintCV,
  standardPrincipalCV,
  makeStandardSTXPostCondition,
  FungibleConditionCode
} from '@stacks/transactions';
import { StacksTestnet } from '@stacks/network';

async function callContractFunction(
  contractAddress,
  contractName,
  functionName,
  functionArgs,
  postConditions = []
) {
  const network = new StacksTestnet();

  const options = {
    network,
    contractAddress,
    contractName,
    functionName,
    functionArgs,
    postConditions,
    postConditionMode: PostConditionMode.Deny,
    onFinish: (data) => {
      console.log('Transaction ID:', data.txId);
      console.log('Transaction:', data);
      return data;
    },
    onCancel: () => {
      console.log('Transaction cancelled');
    },
  };

  await openContractCall(options);
}

// Example: Mint NFT
async function mintNFT(recipientAddress) {
  await callContractFunction(
    'ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM',
    'my-nft',
    'mint',
    [standardPrincipalCV(recipientAddress)],
    [] // No post conditions for minting
  );
}

// Example: Transfer tokens with post condition
async function transferTokens(amount, senderAddress, recipientAddress) {
  // Create post condition to ensure exact amount is transferred
  const postCondition = makeStandardFungiblePostCondition(
    senderAddress,
    FungibleConditionCode.Equal,
    amount,
    'ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM.my-token::my-token'
  );

  await callContractFunction(
    'ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM',
    'my-token',
    'transfer',
    [
      uintCV(amount),
      standardPrincipalCV(senderAddress),
      standardPrincipalCV(recipientAddress),
      noneCV()
    ],
    [postCondition]
  );
}
```

---

## 🔐 Post Conditions

Post conditions are safety checks that protect users from malicious contracts.

### Types of Post Conditions

**1. STX Post Conditions**

```javascript
import { 
  makeStandardSTXPostCondition,
  FungibleConditionCode 
} from '@stacks/transactions';

// User sends exactly 1 STX
const exactSTXCondition = makeStandardSTXPostCondition(
  senderAddress,
  FungibleConditionCode.Equal,
  1000000 // 1 STX in microSTX
);

// User sends at most 1 STX
const maxSTXCondition = makeStandardSTXPostCondition(
  senderAddress,
  FungibleConditionCode.LessEqual,
  1000000
);

// User sends at least 1 STX
const minSTXCondition = makeStandardSTXPostCondition(
  senderAddress,
  FungibleConditionCode.GreaterEqual,
  1000000
);
```

**2. Fungible Token Post Conditions**

```javascript
import { makeStandardFungiblePostCondition } from '@stacks/transactions';

// User transfers exactly 100 tokens
const tokenCondition = makeStandardFungiblePostCondition(
  senderAddress,
  FungibleConditionCode.Equal,
  100000000, // 100 tokens with 6 decimals
  `${contractAddress}.${contractName}::${tokenName}`
);
```

**3. NFT Post Conditions**

```javascript
import { 
  makeStandardNonFungiblePostCondition,
  NonFungibleConditionCode,
  createAssetInfo
} from '@stacks/transactions';

// User sends specific NFT
const nftCondition = makeStandardNonFungiblePostCondition(
  senderAddress,
  NonFungibleConditionCode.Sends,
  createAssetInfo(contractAddress, contractName, assetName),
  uintCV(tokenId)
);

// User does not send NFT
const nftNotSentCondition = makeStandardNonFungiblePostCondition(
  senderAddress,
  NonFungibleConditionCode.DoesNotSend,
  createAssetInfo(contractAddress, contractName, assetName),
  uintCV(tokenId)
);
```

### Complete Example with Post Conditions

```javascript
function NFTPurchase({ nftId, price, sellerAddress, userAddress }) {
  const purchaseNFT = async () => {
    // Post conditions:
    // 1. Buyer sends exact STX amount
    // 2. Buyer receives the NFT
    
    const stxPostCondition = makeStandardSTXPostCondition(
      userAddress,
      FungibleConditionCode.Equal,
      price
    );

    const nftPostCondition = makeStandardNonFungiblePostCondition(
      sellerAddress,
      NonFungibleConditionCode.Sends,
      createAssetInfo(
        'ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM',
        'nft-marketplace',
        'nft-asset'
      ),
      uintCV(nftId)
    );

    const options = {
      network: new StacksTestnet(),
      contractAddress: 'ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM',
      contractName: 'nft-marketplace',
      functionName: 'purchase',
      functionArgs: [uintCV(nftId)],
      postConditions: [stxPostCondition, nftPostCondition],
      postConditionMode: PostConditionMode.Deny,
      onFinish: (data) => {
        alert(`Purchase successful! TX: ${data.txId}`);
      },
    };

    await openContractCall(options);
  };

  return (
    <button onClick={purchaseNFT}>
      Purchase NFT #{nftId} for {price / 1000000} STX
    </button>
  );
}
```

---

## 🌐 Complete DApp Example

### Full-Featured Wallet Integration

```javascript
import React, { useState, useEffect, createContext, useContext } from 'react';
import {
  AppConfig,
  UserSession,
  showConnect,
} from '@stacks/connect';
import { StacksTestnet } from '@stacks/network';

// Create Wallet Context
const WalletContext = createContext();

export function WalletProvider({ children }) {
  const [userSession] = useState(() => 
    new UserSession({ 
      appConfig: new AppConfig(['store_write', 'publish_data']) 
    })
  );
  const [userData, setUserData] = useState(null);
  const [network] = useState(new StacksTestnet());

  useEffect(() => {
    if (userSession.isSignInPending()) {
      userSession.handlePendingSignIn().then((data) => {
        setUserData(data);
      });
    } else if (userSession.isUserSignedIn()) {
      setUserData(userSession.loadUserData());
    }
  }, [userSession]);

  const connectWallet = () => {
    showConnect({
      appDetails: {
        name: 'My DApp',
        icon: window.location.origin + '/logo.png',
      },
      redirectTo: '/',
      onFinish: () => {
        setUserData(userSession.loadUserData());
      },
      userSession,
    });
  };

  const disconnectWallet = () => {
    userSession.signUserOut();
    setUserData(null);
  };

  const value = {
    userSession,
    userData,
    network,
    connectWallet,
    disconnectWallet,
    isConnected: !!userData,
    address: userData?.profile?.stxAddress?.testnet,
  };

  return (
    <WalletContext.Provider value={value}>
      {children}
    </WalletContext.Provider>
  );
}

// Custom hook to use wallet context
export function useWallet() {
  const context = useContext(WalletContext);
  if (!context) {
    throw new Error('useWallet must be used within WalletProvider');
  }
  return context;
}

// Usage in App
function App() {
  return (
    <WalletProvider>
      <Header />
      <MainContent />
    </WalletProvider>
  );
}

function Header() {
  const { isConnected, address, connectWallet, disconnectWallet } = useWallet();

  return (
    <header className="app-header">
      <h1>My DApp</h1>
      {!isConnected ? (
        <button onClick={connectWallet} className="connect-btn">
          Connect Wallet
        </button>
      ) : (
        <div className="wallet-info">
          <span className="address">
            {address.slice(0, 6)}...{address.slice(-4)}
          </span>
          <button onClick={disconnectWallet} className="disconnect-btn">
            Disconnect
          </button>
        </div>
      )}
    </header>
  );
}

function MainContent() {
  const { isConnected, address } = useWallet();

  if (!isConnected) {
    return (
      <div className="connect-prompt">
        <h2>Connect your wallet to get started</h2>
      </div>
    );
  }

  return (
    <div className="main-content">
      <h2>Welcome, {address}!</h2>
      <BalanceDisplay address={address} />
      <ContractInteraction />
    </div>
  );
}
```

---

## 🧪 Testing Wallet Integration

### Manual Testing Checklist

- [ ] Connect wallet successfully
- [ ] Disconnect wallet
- [ ] Reconnect preserves session
- [ ] Read-only calls work
- [ ] Write calls open wallet
- [ ] Post conditions display correctly
- [ ] Transaction success/failure handled
- [ ] Network switching works
- [ ] Multiple accounts work

### Automated Testing

```javascript
// Mock wallet for testing
const mockUserSession = {
  isUserSignedIn: () => true,
  loadUserData: () => ({
    profile: {
      stxAddress: {
        mainnet: 'SP2...',
        testnet: 'ST2...',
      },
    },
  }),
  signUserOut: jest.fn(),
};

test('connects wallet successfully', () => {
  render(
    <WalletProvider userSession={mockUserSession}>
      <TestComponent />
    </WalletProvider>
  );
  
  expect(screen.getByText(/ST2.../)).toBeInTheDocument();
});
```

---

## ⚠️ Common Pitfalls

### 1. **Not Handling Pending Sign-In**

❌ **Wrong:**
```javascript
const userData = userSession.loadUserData();
```

✅ **Correct:**
```javascript
useEffect(() => {
  if (userSession.isSignInPending()) {
    userSession.handlePendingSignIn().then(setUserData);
  } else if (userSession.isUserSignedIn()) {
    setUserData(userSession.loadUserData());
  }
}, []);
```

### 2. **Using Wrong Network**

Make sure contract addresses and network match:
```javascript
// Testnet
const network = new StacksTestnet();
const address = userData.profile.stxAddress.testnet;

// Mainnet  
const network = new StacksMainnet();
const address = userData.profile.stxAddress.mainnet;
```

### 3. **Missing Post Conditions**

Always add post conditions for user protection:
```javascript
// Bad - no protection
await openContractCall({ ...options, postConditions: [] });

// Good - protected
await openContractCall({ ...options, postConditions: [stxCondition] });
```

### 4. **Not Converting STX Units**

```javascript
// Wrong - showing microSTX
<p>Balance: {balance}</p>

// Correct - showing STX
<p>Balance: {(balance / 1_000_000).toFixed(6)} STX</p>
```

---

## 🚀 Best Practices

1. **Use Context for Wallet State**
   - Avoid prop drilling
   - Centralized wallet management

2. **Handle All States**
   - Loading
   - Connected
   - Disconnected
   - Error

3. **Provide Clear Feedback**
   - Show transaction status
   - Display errors clearly
   - Confirm actions

4. **Test on Both Networks**
   - Testnet for development
   - Mainnet for production

5. **Implement Proper Error Handling**
   ```javascript
   try {
     await openContractCall(options);
   } catch (error) {
     if (error.code === 4001) {
       console.log('User rejected');
     } else {
       console.error('Transaction failed:', error);
     }
   }
   ```

---

## 📚 Additional Resources

- **Stacks.js Documentation**: https://stacks.js.org
- **Connect Documentation**: https://github.com/hirosystems/connect
- **Leather Wallet**: https://leather.io
- **Xverse Wallet**: https://xverse.app

---

## 🎓 Next Steps

1. **Add Transaction History**: Track user's past transactions
2. **Implement Message Signing**: For authentication
3. **Multi-Wallet Support**: Let users choose their wallet
4. **Network Switching**: Toggle between testnet/mainnet

**Related Guides:**
- → Making Contract Calls
- → Building an NFT Marketplace
- → Post Conditions Deep Dive