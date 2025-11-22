# React Integration - Complete Patterns Guide

## 📖 Overview

This guide covers React-specific patterns for building Web3 applications with Stacks.js, including hooks, context providers, state management, and component patterns.

### What You'll Learn
- React hooks for Stacks
- Context providers for wallet state
- Component patterns
- State management
- Error handling
- Real-time updates
- Performance optimization

### Prerequisites
- React fundamentals (hooks, context, effects)
- Stacks.js basics
- TypeScript (recommended)

---

## 🎣 Custom Hooks

### useWallet Hook

```typescript
// hooks/useWallet.ts
import { useState, useEffect, createContext, useContext } from 'react';
import { AppConfig, UserSession, showConnect } from '@stacks/connect';
import { StacksTestnet } from '@stacks/network';

interface WalletContextType {
  userSession: UserSession;
  userData: any | null;
  address: string | null;
  isConnected: boolean;
  network: StacksTestnet;
  connectWallet: () => void;
  disconnectWallet: () => void;
}

const WalletContext = createContext<WalletContextType | undefined>(undefined);

export function WalletProvider({ children }: { children: React.ReactNode }) {
  const [userSession] = useState(() => {
    const appConfig = new AppConfig(['store_write', 'publish_data']);
    return new UserSession({ appConfig });
  });
  
  const [userData, setUserData] = useState<any | null>(null);
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
    address: userData?.profile?.stxAddress?.testnet || null,
    isConnected: !!userData,
    network,
    connectWallet,
    disconnectWallet,
  };
  
  return (
    <WalletContext.Provider value={value}>
      {children}
    </WalletContext.Provider>
  );
}

export function useWallet() {
  const context = useContext(WalletContext);
  if (context === undefined) {
    throw new Error('useWallet must be used within a WalletProvider');
  }
  return context;
}
```

### useContractCall Hook

```typescript
// hooks/useContractCall.ts
import { useState } from 'react';
import { openContractCall } from '@stacks/connect';
import { useWallet } from './useWallet';
import { PostConditionMode, ClarityValue } from '@stacks/transactions';

interface ContractCallOptions {
  contractAddress: string;
  contractName: string;
  functionName: string;
  functionArgs: ClarityValue[];
  postConditions?: any[];
}

export function useContractCall() {
  const { network } = useWallet();
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<Error | null>(null);
  const [txId, setTxId] = useState<string | null>(null);
  
  const call = async (options: ContractCallOptions) => {
    setLoading(true);
    setError(null);
    setTxId(null);
    
    try {
      await openContractCall({
        network,
        contractAddress: options.contractAddress,
        contractName: options.contractName,
        functionName: options.functionName,
        functionArgs: options.functionArgs,
        postConditions: options.postConditions || [],
        postConditionMode: PostConditionMode.Deny,
        onFinish: (data) => {
          setTxId(data.txId);
          setLoading(false);
        },
        onCancel: () => {
          setLoading(false);
        },
      });
    } catch (err) {
      setError(err as Error);
      setLoading(false);
    }
  };
  
  return { call, loading, error, txId };
}
```

### useReadContract Hook

```typescript
// hooks/useReadContract.ts
import { useState, useEffect } from 'react';
import { callReadOnlyFunction, cvToValue, ClarityValue } from '@stacks/transactions';
import { useWallet } from './useWallet';

interface ReadContractOptions {
  contractAddress: string;
  contractName: string;
  functionName: string;
  functionArgs?: ClarityValue[];
  enabled?: boolean;
}

export function useReadContract(options: ReadContractOptions) {
  const { network, address } = useWallet();
  const [data, setData] = useState<any>(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<Error | null>(null);
  
  const fetch = async () => {
    if (!options.enabled && options.enabled !== undefined) return;
    
    setLoading(true);
    setError(null);
    
    try {
      const result = await callReadOnlyFunction({
        contractAddress: options.contractAddress,
        contractName: options.contractName,
        functionName: options.functionName,
        functionArgs: options.functionArgs || [],
        network,
        senderAddress: address || options.contractAddress,
      });
      
      const value = cvToValue(result);
      setData(value);
    } catch (err) {
      setError(err as Error);
    } finally {
      setLoading(false);
    }
  };
  
  useEffect(() => {
    fetch();
  }, [
    options.contractAddress,
    options.contractName,
    options.functionName,
    JSON.stringify(options.functionArgs),
    options.enabled,
  ]);
  
  return { data, loading, error, refetch: fetch };
}
```

### useBalance Hook

```typescript
// hooks/useBalance.ts
import { useState, useEffect } from 'react';
import { 
  AccountsApi, 
  Configuration 
} from '@stacks/blockchain-api-client';
import { useWallet } from './useWallet';

export function useBalance() {
  const { address, network } = useWallet();
  const [balance, setBalance] = useState<number>(0);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<Error | null>(null);
  
  const fetchBalance = async () => {
    if (!address) return;
    
    setLoading(true);
    setError(null);
    
    try {
      const config = new Configuration({
        basePath: network.coreApiUrl,
      });
      
      const accountsApi = new AccountsApi(config);
      const balanceData = await accountsApi.getAccountBalance({
        principal: address,
      });
      
      setBalance(parseInt(balanceData.stx.balance));
    } catch (err) {
      setError(err as Error);
    } finally {
      setLoading(false);
    }
  };
  
  useEffect(() => {
    fetchBalance();
    
    // Poll every 30 seconds
    const interval = setInterval(fetchBalance, 30000);
    return () => clearInterval(interval);
  }, [address]);
  
  return { balance, loading, error, refetch: fetchBalance };
}
```

### useTransaction Hook

```typescript
// hooks/useTransaction.ts
import { useState, useEffect } from 'react';
import { TransactionsApi, Configuration } from '@stacks/blockchain-api-client';
import { useWallet } from './useWallet';

export function useTransaction(txId: string | null) {
  const { network } = useWallet();
  const [transaction, setTransaction] = useState<any>(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<Error | null>(null);
  
  useEffect(() => {
    if (!txId) return;
    
    const fetchTransaction = async () => {
      setLoading(true);
      setError(null);
      
      try {
        const config = new Configuration({
          basePath: network.coreApiUrl,
        });
        
        const txApi = new TransactionsApi(config);
        const tx = await txApi.getTransactionById({ txId });
        setTransaction(tx);
      } catch (err) {
        setError(err as Error);
      } finally {
        setLoading(false);
      }
    };
    
    fetchTransaction();
    
    // Poll until confirmed
    const interval = setInterval(() => {
      if (transaction?.tx_status === 'success') {
        clearInterval(interval);
      } else {
        fetchTransaction();
      }
    }, 5000);
    
    return () => clearInterval(interval);
  }, [txId]);
  
  return { transaction, loading, error };
}
```

---

## 🧩 Component Patterns

### Connect Button Component

```typescript
// components/ConnectButton.tsx
import { useWallet } from '../hooks/useWallet';

export function ConnectButton() {
  const { isConnected, address, connectWallet, disconnectWallet } = useWallet();
  
  if (isConnected) {
    return (
      <div className="flex items-center gap-2">
        <span className="text-sm">
          {address?.slice(0, 6)}...{address?.slice(-4)}
        </span>
        <button
          onClick={disconnectWallet}
          className="px-4 py-2 bg-red-500 text-white rounded"
        >
          Disconnect
        </button>
      </div>
    );
  }
  
  return (
    <button
      onClick={connectWallet}
      className="px-4 py-2 bg-blue-500 text-white rounded"
    >
      Connect Wallet
    </button>
  );
}
```

### Balance Display Component

```typescript
// components/BalanceDisplay.tsx
import { useBalance } from '../hooks/useBalance';

export function BalanceDisplay() {
  const { balance, loading, error } = useBalance();
  
  if (loading) {
    return <div>Loading balance...</div>;
  }
  
  if (error) {
    return <div>Error loading balance</div>;
  }
  
  return (
    <div className="balance-display">
      <span className="label">Balance:</span>
      <span className="amount">{(balance / 1000000).toFixed(6)} STX</span>
    </div>
  );
}
```

### Token Transfer Component

```typescript
// components/TokenTransfer.tsx
import { useState } from 'react';
import { useContractCall } from '../hooks/useContractCall';
import { useWallet } from '../hooks/useWallet';
import {
  uintCV,
  standardPrincipalCV,
  noneCV,
  makeStandardFungiblePostCondition,
  FungibleConditionCode
} from '@stacks/transactions';

export function TokenTransfer() {
  const { address } = useWallet();
  const { call, loading, txId } = useContractCall();
  const [recipient, setRecipient] = useState('');
  const [amount, setAmount] = useState('');
  
  const handleTransfer = async () => {
    if (!address) return;
    
    const amountRaw = parseFloat(amount) * 1000000;
    
    const postCondition = makeStandardFungiblePostCondition(
      address,
      FungibleConditionCode.Equal,
      amountRaw,
      'ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM.my-token::my-token'
    );
    
    await call({
      contractAddress: 'ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM',
      contractName: 'my-token',
      functionName: 'transfer',
      functionArgs: [
        uintCV(amountRaw),
        standardPrincipalCV(address),
        standardPrincipalCV(recipient),
        noneCV()
      ],
      postConditions: [postCondition]
    });
  };
  
  return (
    <div className="token-transfer">
      <h2>Transfer Tokens</h2>
      
      <input
        type="text"
        placeholder="Recipient address"
        value={recipient}
        onChange={(e) => setRecipient(e.target.value)}
      />
      
      <input
        type="number"
        placeholder="Amount"
        value={amount}
        onChange={(e) => setAmount(e.target.value)}
      />
      
      <button
        onClick={handleTransfer}
        disabled={loading || !recipient || !amount}
      >
        {loading ? 'Transferring...' : 'Transfer'}
      </button>
      
      {txId && (
        <div className="success">
          Transaction ID: {txId}
        </div>
      )}
    </div>
  );
}
```

### Contract Data Display

```typescript
// components/ContractData.tsx
import { useReadContract } from '../hooks/useReadContract';
import { useWallet } from '../hooks/useWallet';
import { standardPrincipalCV } from '@stacks/transactions';

export function ContractData() {
  const { address } = useWallet();
  
  const { data, loading, error } = useReadContract({
    contractAddress: 'ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM',
    contractName: 'my-token',
    functionName: 'get-balance',
    functionArgs: address ? [standardPrincipalCV(address)] : undefined,
    enabled: !!address,
  });
  
  if (!address) {
    return <div>Connect wallet to view data</div>;
  }
  
  if (loading) {
    return <div>Loading...</div>;
  }
  
  if (error) {
    return <div>Error: {error.message}</div>;
  }
  
  return (
    <div className="contract-data">
      <h3>Token Balance</h3>
      <p>{data?.value ? (data.value / 1000000).toFixed(6) : '0'} Tokens</p>
    </div>
  );
}
```

---

## 🎨 Full Application Example

```typescript
// App.tsx
import { WalletProvider } from './hooks/useWallet';
import { ConnectButton } from './components/ConnectButton';
import { BalanceDisplay } from './components/BalanceDisplay';
import { TokenTransfer } from './components/TokenTransfer';
import { ContractData } from './components/ContractData';

function App() {
  return (
    <WalletProvider>
      <div className="app">
        <header>
          <h1>My DApp</h1>
          <ConnectButton />
        </header>
        
        <main>
          <BalanceDisplay />
          <ContractData />
          <TokenTransfer />
        </main>
      </div>
    </WalletProvider>
  );
}

export default App;
```

---

## 🔄 State Management Patterns

### Using Context for Global State

```typescript
// contexts/AppContext.tsx
import { createContext, useContext, useState, useEffect } from 'react';
import { useWallet } from '../hooks/useWallet';

interface AppState {
  tokenBalance: number;
  nftCount: number;
  stakedAmount: number;
}

interface AppContextType {
  state: AppState;
  refreshData: () => Promise<void>;
  loading: boolean;
}

const AppContext = createContext<AppContextType | undefined>(undefined);

export function AppProvider({ children }: { children: React.ReactNode }) {
  const { address } = useWallet();
  const [state, setState] = useState<AppState>({
    tokenBalance: 0,
    nftCount: 0,
    stakedAmount: 0,
  });
  const [loading, setLoading] = useState(false);
  
  const refreshData = async () => {
    if (!address) return;
    
    setLoading(true);
    try {
      // Fetch all data
      const [tokenBalance, nftCount, stakedAmount] = await Promise.all([
        fetchTokenBalance(address),
        fetchNFTCount(address),
        fetchStakedAmount(address),
      ]);
      
      setState({ tokenBalance, nftCount, stakedAmount });
    } catch (error) {
      console.error('Error refreshing data:', error);
    } finally {
      setLoading(false);
    }
  };
  
  useEffect(() => {
    refreshData();
  }, [address]);
  
  return (
    <AppContext.Provider value={{ state, refreshData, loading }}>
      {children}
    </AppContext.Provider>
  );
}

export function useApp() {
  const context = useContext(AppContext);
  if (!context) {
    throw new Error('useApp must be used within AppProvider');
  }
  return context;
}
```

---

## ⚡ Performance Optimization

### Memoization

```typescript
import { useMemo } from 'react';

function ExpensiveComponent({ data }: { data: any[] }) {
  // Memoize expensive calculations
  const processedData = useMemo(() => {
    return data.map(item => {
      // Expensive processing
      return processItem(item);
    });
  }, [data]);
  
  return <div>{/* Render processed data */}</div>;
}
```

### Debounced Input

```typescript
import { useState, useEffect } from 'react';

function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState<T>(value);
  
  useEffect(() => {
    const handler = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);
    
    return () => {
      clearTimeout(handler);
    };
  }, [value, delay]);
  
  return debouncedValue;
}

// Usage
function SearchComponent() {
  const [searchTerm, setSearchTerm] = useState('');
  const debouncedSearch = useDebounce(searchTerm, 500);
  
  useEffect(() => {
    if (debouncedSearch) {
      // Perform search
      searchContracts(debouncedSearch);
    }
  }, [debouncedSearch]);
  
  return (
    <input
      value={searchTerm}
      onChange={(e) => setSearchTerm(e.target.value)}
      placeholder="Search..."
    />
  );
}
```

---

## 🐛 Error Handling

### Error Boundary Component

```typescript
// components/ErrorBoundary.tsx
import { Component, ReactNode } from 'react';

interface Props {
  children: ReactNode;
  fallback?: ReactNode;
}

interface State {
  hasError: boolean;
  error: Error | null;
}

export class ErrorBoundary extends Component<Props, State> {
  constructor(props: Props) {
    super(props);
    this.state = { hasError: false, error: null };
  }
  
  static getDerivedStateFromError(error: Error) {
    return { hasError: true, error };
  }
  
  componentDidCatch(error: Error, errorInfo: any) {
    console.error('Error caught by boundary:', error, errorInfo);
  }
  
  render() {
    if (this.state.hasError) {
      return this.props.fallback || (
        <div className="error-boundary">
          <h2>Something went wrong</h2>
          <p>{this.state.error?.message}</p>
          <button onClick={() => this.setState({ hasError: false, error: null })}>
            Try again
          </button>
        </div>
      );
    }
    
    return this.props.children;
  }
}
```

### Toast Notifications

```typescript
// hooks/useToast.ts
import { useState, useCallback } from 'react';

interface Toast {
  id: string;
  message: string;
  type: 'success' | 'error' | 'info';
}

export function useToast() {
  const [toasts, setToasts] = useState<Toast[]>([]);
  
  const showToast = useCallback((message: string, type: Toast['type'] = 'info') => {
    const id = Math.random().toString(36);
    setToasts(prev => [...prev, { id, message, type }]);
    
    setTimeout(() => {
      setToasts(prev => prev.filter(t => t.id !== id));
    }, 5000);
  }, []);
  
  const removeToast = useCallback((id: string) => {
    setToasts(prev => prev.filter(t => t.id !== id));
  }, []);
  
  return { toasts, showToast, removeToast };
}
```

---

## 📚 Next Steps

**Related Guides:**
- → Advanced Contract Calls
- → Stacks.js Complete Guide
- → State Management Best Practices