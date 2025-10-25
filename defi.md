# DeFi Pool Guide for Sui Blockchain

## Step 1: Setting Up — Client Components and Wallet Connection

### Concept

In Next.js 13+ (or 15), components that use browser-only features (like connecting a wallet) must run on the client side — hence the `'use client'` directive.

We also import Sui Dapp Kit hooks to interact with the blockchain.

### Code Implementation

```typescript
'use client';

import { useState, useEffect } from 'react';
import { 
  useCurrentAccount, 
  useSignAndExecuteTransaction,
  useSuiClient 
} from '@mysten/dapp-kit';
import { Transaction } from '@mysten/sui/transactions';
import { ConnectButton } from '@mysten/dapp-kit';
import Link from 'next/link';
```

### Hook Explanations

- `useCurrentAccount()` → gets the connected wallet account
- `useSignAndExecuteTransaction()` → allows you to sign and send blockchain transactions
- `useSuiClient()` → lets you query blockchain data
- `Transaction` → constructs Sui blockchain transactions in JavaScript

### Wallet Connection

When we click Connect Wallet, we use:

```typescript
const currentAccount = useCurrentAccount();
```

And add the wallet button:

```tsx
<ConnectButton />
```

This automatically integrates with Sui wallets (like Sui Wallet or Ethos).

---

## Step 2: Smart Contract Setup

Here, we already deployed a simple Move smart contract for a DeFi pool. We only need to know the package ID (the deployed contract) and the pool object ID.

```typescript
const PACKAGE_ID = '0xff458614c6a15f53e710e9a93ff2437a8d4afd724f527a9740233dad77759ed5';
const POOL_ID = '0x8257dacc05b5c72cfe5725c73840dfe984fb826f4b1327cc10a1262e32611af6';
```

### Key Concepts

- **Package ID** → like the "smart contract address"
- **POOL_ID** → an object on-chain that stores all the deposits and borrows

---

## Step 3: Fetch Blockchain Data (useSuiClient)

We use the Sui client to read blockchain data like pool balance or user's SUI balance.

### Implementation

```typescript
const fetchPoolData = async () => {
  if (!currentAccount) return;

  try {
    // Get pool object
    const poolObject = await suiClient.getObject({
      id: POOL_ID,
      options: { showContent: true },
    });

    // Extract balance
    if (poolObject.data?.content?.dataType === 'moveObject') {
      const fields = poolObject.data.content.fields as any;
      const balance = fields.deposits || '0';
      setPoolBalance((Number(balance) / 1_000_000_000).toFixed(4));
    }

    // Get user's wallet balance
    const coins = await suiClient.getBalance({
      owner: currentAccount.address,
      coinType: '0x2::sui::SUI',
    });
    setUserBalance((Number(coins.totalBalance) / 1_000_000_000).toFixed(4));
  } catch (error) {
    console.error('Error fetching pool data:', error);
  }
};
```

### Important Note

Sui balances are stored in **MIST** (1 SUI = 1,000,000,000 MIST). Always convert to human-readable values before displaying.

---

## Step 4: Deposit Function (Writing to the Blockchain)

Let's see how we send a transaction — here we deposit SUI into the DeFi pool.

### Implementation

```typescript
const handleDeposit = async () => {
  if (!currentAccount) return alert('Connect wallet first!');
  if (!depositAmount || Number(depositAmount) <= 0) return alert('Invalid amount');

  try {
    setLoading(true);
    const tx = new Transaction();

    const amountInMist = Math.floor(Number(depositAmount) * 1_000_000_000);
    const [coin] = tx.splitCoins(tx.gas, [amountInMist]);

    tx.moveCall({
      target: `${PACKAGE_ID}::defi::deposit`,
      arguments: [
        tx.object(POOL_ID),
        coin,
      ],
    });

    signAndExecuteTransaction({ transaction: tx }, {
      onSuccess: async (result) => {
        console.log('Deposit successful:', result);
        alert(`Deposit successful! Tx: ${result.digest}`);
        setDepositAmount('');
        await fetchPoolData();
      },
      onError: (error) => alert('Deposit failed: ' + error.message),
    });
  } finally {
    setLoading(false);
  }
};
```

### Line-by-Line Explanation

1. `new Transaction()` → creates a blockchain transaction
2. `splitCoins(tx.gas, [amount])` → splits the gas coin into the amount to send
3. `tx.moveCall(...)` → calls the Move function `defi::deposit`
4. `signAndExecuteTransaction(...)` → signs with wallet and sends to the blockchain
5. On success, we update UI and fetch new balances

---

## Step 5: Borrow and Repay

These are similar write operations calling different Move functions.

### Borrow Function

```typescript
tx.moveCall({
  target: `${PACKAGE_ID}::defi::borrow`,
  arguments: [tx.object(POOL_ID), tx.pure.u64(amountInMist)],
});
```

### Repay Function

```typescript
const [coin] = tx.splitCoins(tx.gas, [amountInMist]);
tx.moveCall({
  target: `${PACKAGE_ID}::defi::repay`,
  arguments: [tx.object(POOL_ID), coin],
});
```

### Key Difference

**Deposit** and **Repay** send coins to the pool, while **Borrow** transfers coins from the pool to the user.

---

## Step 6: Frontend UI Flow

Build a simple UI to interact with the DeFi pool:

```tsx
<div className="p-4">
  <ConnectButton />
  <h2>Pool Balance: {poolBalance} SUI</h2>
  <h2>Your Balance: {userBalance} SUI</h2>

  <input
    placeholder="Amount to deposit"
    value={depositAmount}
    onChange={(e) => setDepositAmount(e.target.value)}
  />
  <button onClick={handleDeposit}>Deposit</button>
</div>
```

You can then extend this pattern for Borrow and Repay functionality.