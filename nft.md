# NFT Minting Guide for Sui Blockchain

## 1. MintNFTForm.tsx — The NFT Minting Form

### Purpose
Allows users to input NFT details (name, description, image URL), encode them properly, and call the Sui smart contract's mint function.

### Key Imports

```typescript
import { useCurrentAccount, useSignAndExecuteTransaction } from '@mysten/dapp-kit';
import { Transaction } from '@mysten/sui/transactions';
import { useState } from 'react';
```

- `useCurrentAccount()` → gives us the connected wallet's address
- `useSignAndExecuteTransaction()` → allows us to sign and send a transaction to Sui blockchain
- `Transaction` → used to construct and encode a Move call

### The Core Logic: Mint Function

When the user fills the form and clicks Mint, this happens:

#### 1. Check Wallet Connection

```typescript
if (!currentAccount) {
  alert('Please connect your wallet first');
  return;
}
```

#### 2. Encode Strings into BCS

The Sui blockchain uses a binary format called BCS (Binary Canonical Serialization). We manually encode our strings because the SDK sometimes has issues with string encoding.

```typescript
const encodeToBCS = (text: string): Uint8Array => {
  const textEncoder = new TextEncoder();
  const stringBytes = textEncoder.encode(text);
  const length = stringBytes.length;
  // manually add ULEB128 prefix
  return new Uint8Array([length, ...stringBytes]);
};
```

#### 3. Build the Transaction

```typescript
const tx = new Transaction();
tx.moveCall({
  target: `${packageId}::nft::mint_to_sender`,
  arguments: [
    tx.pure(nameBytes),
    tx.pure(descBytes),
    tx.pure(urlBytes),
  ],
});
```

This calls your smart contract function `mint_to_sender` in the module `nft` inside your deployed package. Each argument is encoded as pure BCS bytes.

#### 4. Sign and Send the Transaction

```typescript
signAndExecuteTransaction(
  { transaction: tx },
  {
    onSuccess: async (result) => {
      await onMintSuccess(result.digest);
      await onFetchNFTs();
    },
    onError: (error) => alert('Transaction failed: ' + error.message),
  }
);
```

### Summary

The `MintNFTForm` collects data → encodes it → calls the Move function → signs and executes → triggers a success callback.

---

## 2. UserNFTList.tsx — Display NFTs Owned by the User

### Purpose
Show all NFTs currently owned by the connected wallet.

### Key Imports

```typescript
import { useCurrentAccount } from '@mysten/dapp-kit';
```

### Component Props

```typescript
interface UserNFTListProps {
  nfts: any[];
  loading: boolean;
  onRefresh: () => Promise<void>;
}
```

- `nfts` → the NFTs fetched from the blockchain
- `onRefresh()` → allows the user to reload NFT data after minting

### Example Rendering

```tsx
{loading ? (
  <p>Loading NFTs...</p>
) : (
  nfts.map((nft) => (
    <div key={nft.data?.objectId}>
      <h3>{nft.data?.display?.data?.name}</h3>
      <img src={nft.data?.display?.data?.image_url} alt="nft" />
    </div>
  ))
)}
```

### Summary

`UserNFTList` is responsible for visually showing the NFTs belonging to the wallet address.

---

## 3. NFTMintedEvents.tsx — Track Minted Events from Blockchain

### Purpose
Fetch and display all `NFTMinted` events emitted by the contract — both historical and recent.

### Event Structure

```typescript
interface NFTMintedEvent {
  objectId: string;
  creator: string;
  name: string;
  timestamp: string;
  txDigest: string;
}
```

### Example Rendering

```tsx
{events.map((event) => (
  <div key={event.txDigest}>
    <p><b>{event.name}</b> minted by {event.creator}</p>
    <small>{event.timestamp}</small>
  </div>
))}
```

### Summary

Displays blockchain-level history of NFT minting — similar to an on-chain "activity log".

---

## 4. NFTPage.tsx — The Parent Page (Combining All Components)

### Purpose
This is the main page that coordinates all 3 components.

It:
- Handles blockchain queries (events + owned NFTs)
- Passes data and handlers to each child component
- Manages the global state (loading, NFT data, events)

### 1. Connecting to Sui

```typescript
const currentAccount = useCurrentAccount();
const suiClient = useSuiClient();
```

`useSuiClient()` gives us access to RPC calls like `getOwnedObjects` and `queryEvents`.

### 2. Querying Events

Fetches all `NFTMinted` events from your contract:

```typescript
const eventType = `${PACKAGE_ID}::nft::NFTMinted`;

const response = await suiClient.queryEvents({
  query: { MoveEventType: eventType },
  order: 'descending',
  limit: 50,
});
```

Each event is parsed and saved into `mintedEvents`.

### 3. Fetching User NFTs

Gets NFTs owned by the connected wallet:

```typescript
const objects = await suiClient.getOwnedObjects({
  owner: currentAccount.address,
  filter: { StructType: `${PACKAGE_ID}::nft::NFT` },
  options: { showContent: true, showDisplay: true },
});
setMintedNFTs(objects.data);
```

### 4. Handling Mint Success

After minting, parse the transaction for `NFTMinted` events and update the list:

```typescript
const handleMintSuccess = async (digest: string) => {
  const events = await parseNFTMintedEvents(digest);
  setMintedEvents(prev => [...events, ...prev]);
  alert(`NFT Minted! Object ID: ${events[0].objectId}`);
};
```

### 5. Rendering the Page

```tsx
return (
  <div className="p-6">
    <ConnectButton />
    <MintNFTForm 
      packageId={PACKAGE_ID}
      onMintSuccess={handleMintSuccess}
      onFetchNFTs={fetchUserNFTs}
    />

    <UserNFTList 
      nfts={mintedNFTs}
      loading={loading}
      onRefresh={fetchUserNFTs}
    />

    <NFTMintedEvents 
      events={mintedEvents}
      loading={loading}
      onRefresh={handleRefreshEvents}
    />
  </div>
);
```

### Summary

The `NFTPage` acts as the controller — fetching blockchain data, updating UI state, and coordinating all components.