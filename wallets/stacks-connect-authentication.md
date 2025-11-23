# Stacks Wallet Authentication with @stacks/connect

## 📖 Overview

Modern Stacks applications use `@stacks/connect` to authenticate users with their Stacks wallets (Xverse, Hiro Wallet/Leather, Asigna). This guide shows you how to implement proper wallet authentication with session persistence and BNS name resolution.

### What You'll Learn
- Setting up @stacks/connect
- Creating a custom authentication hook
- Implementing connect/disconnect flows
- Session persistence with localStorage
- BNS name resolution
- Supporting multiple wallets

### Prerequisites
- React or Next.js application
- Basic understanding of React hooks
- Node.js and npm installed

---

## 🚀 Installation

Install the required Stacks packages:

```bash
npm install @stacks/auth @stacks/connect @stacks/profile @stacks/transactions @stacks/blockchain-api-client
```

**Package purposes:**
- `@stacks/auth` - Authentication utilities
- `@stacks/connect` - Wallet connection management
- `@stacks/profile` - User profile data
- `@stacks/transactions` - Transaction building
- `@stacks/blockchain-api-client` - API interactions

---

## 🔧 Custom Authentication Hook

Create a custom hook to manage wallet authentication state:

**File**: `hooks/useStacksAuth.ts`

```typescript
"use client";

import { useState, useEffect } from "react";
import { connect, disconnect, isConnected, getLocalStorage } from "@stacks/connect";

interface UserData {
  addresses: {
    stx: Array<{ address: string }>;
  };
}

export function useStacksAuth() {
  const [isAuthenticated, setIsAuthenticated] = useState(false);
  const [userData, setUserData] = useState<UserData | null>(null);
  const [bnsName, setBnsName] = useState<string | null>(null);
  const [address, setAddress] = useState<string | null>(null);

  useEffect(() => {
    // Check for existing connection on mount
    const checkConnection = async () => {
      if (isConnected()) {
        const localData = getLocalStorage();
        if (localData?.addresses?.stx?.[0]?.address) {
          const stxAddress = localData.addresses.stx[0].address;
          setUserData(localData);
          setAddress(stxAddress);
          setIsAuthenticated(true);
          
          // Fetch BNS name
          try {
            const response = await fetch(
              `https://api.bnsv2.com/names/address/${stxAddress}/valid`
            );
            const data = await response.json();
            if (data?.names?.length > 0) {
              setBnsName(data.names[0]);
            }
          } catch (error) {
            console.log("No BNS name found");
          }
        }
      }
    };
    
    checkConnection();
  }, []);

  const signIn = async () => {
    try {
      await connect({
        onFinish: async (localData: any) => {
          if (localData?.addresses?.stx?.[0]?.address) {
            const stxAddress = localData.addresses.stx[0].address;
            setUserData(localData);
            setAddress(stxAddress);
            setIsAuthenticated(true);
            
            // Fetch BNS name
            try {
              const response = await fetch(
                `https://api.bnsv2.com/names/address/${stxAddress}/valid`
              );
              const data = await response.json();
              if (data?.names?.length > 0) {
                setBnsName(data.names[0]);
              }
            } catch (error) {
              console.log("No BNS name found");
            }
          }
        },
        onCancel: () => {
          console.log("Connection cancelled");
        },
      });
    } catch (error) {
      console.error("Failed to connect:", error);
    }
  };

  const signOut = () => {
    disconnect();
    setIsAuthenticated(false);
    setUserData(null);
    setBnsName(null);
    setAddress(null);
  };

  return {
    isAuthenticated,
    userData,
    bnsName,
    address,
    signIn,
    signOut,
  };
}
```

---

## 💻 Using the Hook in Components

### Wallet Connect Button

```typescript
"use client";

import { useStacksAuth } from "@/hooks/useStacksAuth";
import { Wallet, LogOut } from "lucide-react";

export function WalletConnect() {
  const { isAuthenticated, address, bnsName, signIn, signOut } = useStacksAuth();

  if (!isAuthenticated) {
    return (
      <button onClick={signIn} className="btn-primary">
        <Wallet className="h-5 w-5" />
        Connect Wallet
      </button>
    );
  }

  const displayName = bnsName || `${address?.slice(0, 6)}...${address?.slice(-4)}`;

  return (
    <div className="flex items-center gap-3">
      <div className="wallet-address">
        {displayName}
      </div>
      <button onClick={signOut} className="btn-danger">
        <LogOut className="h-5 w-5" />
      </button>
    </div>
  );
}
```

### Protected Route/Component

```typescript
"use client";

import { useStacksAuth } from "@/hooks/useStacksAuth";
import { useRouter } from "next/navigation";
import { useEffect } from "react";

export function Dashboard() {
  const { isAuthenticated, address } = useStacksAuth();
  const router = useRouter();

  useEffect(() => {
    if (!isAuthenticated) {
      router.push("/");
    }
  }, [isAuthenticated, router]);

  if (!isAuthenticated) {
    return <div>Please connect your wallet...</div>;
  }

  return (
    <div>
      <h1>Welcome, {address}</h1>
      {/* Dashboard content */}
    </div>
  );
}
```

---

## 🌐 Supported Wallets

The `connect()` function automatically detects and supports:

1. **Xverse** - Mobile and browser extension
2. **Hiro Wallet (Leather)** - Browser extension (formerly Hiro Wallet)
3. **Asigna** - Multi-sig wallet

No additional configuration needed - users choose their wallet when connecting.

---

## 🔍 BNS Name Resolution

BNS (Bitcoin Name System) allows users to have human-readable names instead of addresses.

**API Endpoint**: `https://api.bnsv2.com/names/address/{address}/valid`

**Example Response**:
```json
{
  "names": ["alice.btc", "alice.id.stx"]
}
```

**Usage Pattern**:
```typescript
const fetchBNSName = async (address: string) => {
  try {
    const response = await fetch(
      `https://api.bnsv2.com/names/address/${address}/valid`
    );
    const data = await response.json();
    return data?.names?.[0] || null;
  } catch {
    return null;
  }
};
```

---

## 💾 Session Persistence

`@stacks/connect` automatically handles session persistence via `localStorage`.

**How it works**:
1. User connects wallet → data saved to localStorage
2. Page refresh → `isConnected()` returns `true`
3. `getLocalStorage()` retrieves saved data
4. No need to reconnect!

**Manual check**:
```typescript
import { isConnected, getLocalStorage } from "@stacks/connect";

if (isConnected()) {
  const data = getLocalStorage();
  console.log("Stored address:", data.addresses.stx[0].address);
}
```

---

## ⚠️ Common Issues

### "connect is not a function"

**Problem**: Using old API with `showConnect()`

```typescript
// ❌ Wrong - Old API
import { showConnect } from "@stacks/connect";
showConnect({ ... });

// ✅ Correct - New API
import { connect } from "@stacks/connect";
connect({ ... });
```

### "No wallet detected"

**Solution**: Users must have a Stacks wallet extension installed

**Friendly error handling**:
```typescript
const signIn = async () => {
  try {
    await connect({ ... });
  } catch (error) {
    alert("Please install a Stacks wallet (Xverse or Hiro Wallet)");
  }
};
```

### Session not persisting

**Check**: Make sure you're using `isConnected()` and `getLocalStorage()` on mount

```typescript
useEffect(() => {
  if (isConnected()) {
    const data = getLocalStorage();
    // Restore session
  }
}, []);
```

---

## 🎯 Best Practices

1. **Check connection on mount** - Restore sessions automatically
2. **Fetch BNS names** - Better UX than showing addresses
3. **Handle errors gracefully** - Provide clear user feedback
4. **Implement loading states** - Show when connecting
5. **Clear state on disconnect** - Clean up all user data
6. **Use TypeScript** - Type safety for user data

---

## 📱 Mobile Support

For mobile apps, consider using:
- **Xverse Mobile** - Deep linking support
- **Mobile Connect Kit** - `@stacks/connect-react` for React Native

**Deep linking example**:
```typescript
import { openContractCall } from "@stacks/connect";

// Mobile wallets will open automatically
await openContractCall({
  ...options,
  appDetails: {
    name: "My App",
    icon: window.location.origin + "/logo.png",
  }
});
```

---

## 🔗 Related Guides

- [Contract Calls](../frontend-integration/contract-calls.md)
- [Next.js Integration](../frontend-integration/nextjs-integration.md)
- [STX Transfers](stx-transfers.md)

---

## 📚 Additional Resources

- [Stacks.js Documentation](https://docs.hiro.so/stacks.js)
- [BNSv2 API Docs](https://docs.bns.xyz)
- [Xverse Wallet](https://www.xverse.app/)
- [Hiro Wallet](https://wallet.hiro.so/)
