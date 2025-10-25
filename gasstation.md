# Gasless Transactions on Sui with Shinami Gas Station

A complete 1-hour workshop kit: slides, script, code walk-through, and lab guide — in one markdown file.

---

## Table of Contents

1. [Workshop Goals](#workshop-goals)
2. [Audience & Prerequisites](#audience--prerequisites)
3. [Problem & Solution Overview](#problem--solution-overview)
4. [Architecture at a Glance](#architecture-at-a-glance)
5. [Project Structure](#project-structure)
6. [Setup & Installation](#setup--installation)
7. [Configuration](#configuration)
8. [Core Concepts](#core-concepts)
9. [Codebase Walk-through (File by File)](#codebase-walk-through-file-by-file)
10. [Backend API Routes](#backend-api-routes)
11. [Frontend Components](#frontend-components)
12. [Smart Contract Module (Move)](#smart-contract-module-move)
13. [End-to-End Flows](#end-to-end-flows)
14. [Live Demo Script](#live-demo-script)
15. [Common Pitfalls & Fixes](#common-pitfalls--fixes)
16. [Production Considerations](#production-considerations)
17. [Testing & Verification](#testing--verification)
18. [Workshop Timeline](#workshop-timeline)
19. [Presenter Notes](#presenter-notes)
20. [Quick Reference](#quick-reference)
21. [Next Steps & Resources](#next-steps--resources)

---

## Workshop Goals

- Understand why gas sponsorship removes major onboarding friction
- Learn how Shinami Gas Station works on Sui using two signatures
- Integrate a working gasless experience in a Next.js App Router project
- Deploy and test a simple Move module and call it with sponsored gas
- Leave with a repeatable pattern you can drop into real dApps

---

## Audience & Prerequisites

### Who this is for

Frontend and full-stack developers exploring Sui; teams that want gasless UX.

### You need

- Node.js 18+, npm, Git, a code editor
- A Sui wallet extension (Sui Wallet, Ethos, Suiet) on Testnet
- A Shinami account with Gas Station and Node keys (Testnet is fine)

---

## Problem & Solution Overview

### The Problem

Users must obtain SUI before they can even try your app:

- Install wallet → get tokens → configure network → then try the dApp
- This causes high drop-off during onboarding

### The Solution

A Gas Station sponsors gas fees on behalf of users:

- Users still sign the transaction content
- The sponsor pays the gas fee
- Users can try your app immediately

### The Core Idea: Two Signatures

- **User signature** → "I approve this transaction's content"
- **Sponsor signature** → "I will pay the gas for this transaction"
- The blockchain validates both, then executes

---

## Architecture at a Glance

```
Frontend (Browser)
• Connect wallet
• Collect params
• Ask backend to sponsor
• User signs txBytes
• Execute with both signatures
          │
          ▼
Backend (Next.js API Routes)
• Initialize Shinami clients
• Build gasless transaction
• Get sponsor signature from Shinami
• Return txBytes + sponsorSignature
          │
          ▼
Shinami Services
• Gas Station = sponsor signature + gas object
• Node Service = fast RPC
          │
          ▼
Sui Blockchain
• Validates user + sponsor signatures
• Executes transaction
```

---

## Project Structure

```
sui-gas-station-app/
├── app/
│   ├── api/
│   │   ├── buildSponsoredTx/route.ts       # Sponsor Move calls
│   │   ├── buildTransferTx/route.ts        # Sponsor SUI transfers
│   │   └── executeSponsoredTx/route.ts     # Optional backend execution
│   ├── layout.tsx                          # Root layout (wraps Providers)
│   └── page.tsx                            # Main demo page
├── components/
│   ├── Providers.tsx                       # Wallet + React Query providers
│   └── TransferForm.tsx                    # Gasless SUI transfer UI
├── lib/
│   ├── shinami-client.ts                   # Shinami clients init
│   └── types.ts                            # Request/response types
├── move-example/
│   └── sources/math.move                   # Demo Move module
├── .env.local                              # API keys and config
├── next.config.ts                          # Next.js config
└── package.json
```

---

## Setup & Installation

```bash
# 1) Create a Next.js App Router project (or use your existing app)
npx create-next-app@latest sui-gas-station-app
cd sui-gas-station-app

# 2) Install dependencies
npm install @mysten/sui @mysten/dapp-kit @shinami/clients @tanstack/react-query

# 3) Start dev server
npm run dev
```

---

## Configuration

### Environment Variables (.env.local)

```bash
# Shinami keys (get from your Shinami dashboard)
SHINAMI_GAS_STATION_ACCESS_KEY=us1_sui_testnet_xxxxx
SHINAMI_NODE_ACCESS_KEY=us1_sui_testnet_xxxxx

# Network selection
NEXT_PUBLIC_SUI_NETWORK=testnet

# Your Move package id (after publishing)
NEXT_PUBLIC_MOVE_PACKAGE_ID=0xYOUR_PACKAGE_ID
```

### Next.js Config (next.config.ts)

Externalize Shinami SDK on the server to avoid bundling errors.

```typescript
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  serverExternalPackages: ["@shinami/clients"],
  webpack: (config, { isServer }) => {
    if (isServer) {
      config.externals = [...(config.externals || []), "@shinami/clients"];
    }
    return config;
  },
};

export default nextConfig;
```

---

## Core Concepts

- **GaslessTransaction**: built with `buildGaslessTransaction()`; sponsor-ready
- **Never use `txb.gas` in gasless flows**; the gas object does not exist yet
- **Sender provides assets, sponsor pays gas only**
- Execute with two signatures in this order: `[userSignature, sponsorSignature]`
- Keep Shinami API keys strictly on the backend

---

## Codebase Walk-through (File by File)

### package.json — Key Dependencies

```json
{
  "@shinami/clients": "^0.9.7",
  "@mysten/sui": "^1.43.1",
  "@mysten/dapp-kit": "^0.19.6",
  "@tanstack/react-query": "^5.90.5",
  "next": "15.5.6",
  "react": "19.1.0"
}
```

- `@shinami/clients`: Gas Station SDK
- `@mysten/sui` + `@mysten/dapp-kit`: Sui client, wallet UI and hooks
- `@tanstack/react-query`: Required by dApp Kit for caching/state

### lib/shinami-client.ts — Initialize Clients

```typescript
import { GasStationClient } from "@shinami/clients/sui";
import { SuiClient } from "@mysten/sui/client";

if (!process.env.SHINAMI_GAS_STATION_ACCESS_KEY) {
  throw new Error("SHINAMI_GAS_STATION_ACCESS_KEY is not set");
}
if (!process.env.SHINAMI_NODE_ACCESS_KEY) {
  throw new Error("SHINAMI_NODE_ACCESS_KEY is not set");
}

export const gasStationClient = new GasStationClient(
  process.env.SHINAMI_GAS_STATION_ACCESS_KEY
);

export const suiClient = new SuiClient({
  url: `https://api.us1.shinami.com/node/v1/${process.env.SHINAMI_NODE_ACCESS_KEY}`,
});
```

- Single source of truth for Shinami clients
- These must be used server-side only

### lib/types.ts — Type Contracts

Define request and response interfaces to keep FE/BE in sync.

---

## Backend API Routes

### /api/buildSponsoredTx — Sponsor Move calls

**Flow**: Validate request → `buildGaslessTransaction()` → set sender → sponsor → return txBytes + sponsorSignature

```typescript
import { NextRequest, NextResponse } from "next/server";
import { buildGaslessTransaction } from "@shinami/clients/sui";
import { gasStationClient, suiClient } from "@/lib/shinami-client";

export async function POST(request: NextRequest) {
  const { sender, num1, num2 } = await request.json();
  if (!sender) return NextResponse.json({ error: "Missing sender" }, { status: 400 });

  const pkg = process.env.NEXT_PUBLIC_MOVE_PACKAGE_ID!;
  const gaslessTx = await buildGaslessTransaction(
    (txb) => {
      txb.moveCall({
        target: `${pkg}::math::add`,
        arguments: [txb.pure.u64(num1), txb.pure.u64(num2)],
      });
    },
    { sui: suiClient }
  );

  gaslessTx.sender = sender;
  const sponsored = await gasStationClient.sponsorTransaction(gaslessTx);

  return NextResponse.json({
    txBytes: sponsored.txBytes,
    sponsorSignature: sponsored.signature,
    digest: sponsored.txDigest,
  });
}
```

### /api/buildTransferTx — Sponsor SUI transfers

**Key rule**: In gasless flows you cannot use `txb.gas`; use the sender's coin.

```typescript
import { NextRequest, NextResponse } from "next/server";
import { buildGaslessTransaction } from "@shinami/clients/sui";
import { gasStationClient, suiClient } from "@/lib/shinami-client";

export async function POST(request: NextRequest) {
  const { sender, recipient, amount } = await request.json();
  if (!sender || !recipient) {
    return NextResponse.json({ error: "Missing address" }, { status: 400 });
  }

  const coins = await suiClient.getCoins({
    owner: sender,
    coinType: "0x2::sui::SUI",
  });
  if (!coins.data?.length) {
    return NextResponse.json({ error: "Sender has no SUI coins" }, { status: 400 });
  }

  const gaslessTx = await buildGaslessTransaction(
    (txb) => {
      const coinObjectId = coins.data[0].coinObjectId;
      const [split] = txb.splitCoins(coinObjectId, [amount]); // use sender's coin
      txb.transferObjects([split], recipient);
    },
    { sui: suiClient }
  );

  gaslessTx.sender = sender;
  const sponsored = await gasStationClient.sponsorTransaction(gaslessTx);

  return NextResponse.json({
    txBytes: sponsored.txBytes,
    sponsorSignature: sponsored.signature,
    digest: sponsored.txDigest,
  });
}
```

### /api/executeSponsoredTx — Backend Execution (optional)

```typescript
import { NextRequest, NextResponse } from "next/server";
import { suiClient } from "@/lib/shinami-client";

export async function POST(request: NextRequest) {
  const { txBytes, sponsorSignature, senderSignature } = await request.json();

  const result = await suiClient.executeTransactionBlock({
    transactionBlock: txBytes,
    signature: [senderSignature, sponsorSignature], // order matters
    options: { showEffects: true, showEvents: true, showObjectChanges: true },
  });

  return NextResponse.json({ digest: result.digest, effects: result.effects });
}
```

**When to use**: Need centralized logging, retries, or post-processing on the server

---

## Frontend Components

### components/Providers.tsx — App-wide Providers

```typescript
"use client";

import { useState } from "react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import {
  SuiClientProvider,
  createNetworkConfig,
  getFullnodeUrl,
} from "@mysten/dapp-kit";
import { WalletProvider } from "@mysten/dapp-kit";

const { networkConfig } = createNetworkConfig({
  testnet: { url: getFullnodeUrl("testnet") },
  mainnet: { url: getFullnodeUrl("mainnet") },
});
const network = process.env.NEXT_PUBLIC_SUI_NETWORK || "testnet";

export function Providers({ children }: { children: React.ReactNode }) {
  const [queryClient] = useState(
    () =>
      new QueryClient({
        defaultOptions: { queries: { retry: 1, refetchOnWindowFocus: false } },
      })
  );

  return (
    <QueryClientProvider client={queryClient}>
      <SuiClientProvider networks={networkConfig} defaultNetwork={network}>
        <WalletProvider autoConnect={false}>{children}</WalletProvider>
      </SuiClientProvider>
    </QueryClientProvider>
  );
}
```

### components/TransferForm.tsx — Gasless Transfer UI

```typescript
"use client";

import { useState } from "react";
import { useSignTransaction, useSuiClient } from "@mysten/dapp-kit";

export function TransferForm({ senderAddress }: { senderAddress: string }) {
  const [recipient, setRecipient] = useState("");
  const [amount, setAmount] = useState("0.1");
  const [loading, setLoading] = useState(false);
  const [result, setResult] = useState("");
  const [error, setError] = useState("");

  const { mutateAsync: signTransaction } = useSignTransaction();
  const suiClient = useSuiClient();

  const handleTransfer = async () => {
    try {
      setLoading(true);
      setError("");
      setResult("");

      if (!recipient) {
        setError("Please enter a recipient address");
        return;
      }

      const amountNum = parseFloat(amount);
      if (isNaN(amountNum) || amountNum <= 0) {
        setError("Invalid amount");
        return;
      }
      const amountInMist = Math.floor(amountNum * 1_000_000_000);

      // 1) Build & sponsor on backend
      const buildRes = await fetch("/api/buildTransferTx", {
        method: "POST",
        body: JSON.stringify({
          sender: senderAddress,
          recipient: recipient.trim(),
          amount: amountInMist,
        }),
      });
      const sponsored = await buildRes.json();
      if (!buildRes.ok) throw new Error(sponsored.error || "Build failed");

      // 2) User signs
      const { signature: senderSignature } = await signTransaction({
        transaction: sponsored.txBytes,
      });

      // 3) Execute with both signatures
      const exec = await suiClient.executeTransactionBlock({
        transactionBlock: sponsored.txBytes,
        signature: [senderSignature, sponsored.sponsorSignature],
      });

      setResult(`Transfer successful! Digest: ${exec.digest}`);
    } catch (e: any) {
      setError(e.message || "Transfer failed");
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="transfer-form">
      <input
        type="text"
        value={recipient}
        onChange={(e) => setRecipient(e.target.value)}
        placeholder="Recipient 0x..."
      />
      <input
        type="number"
        value={amount}
        onChange={(e) => setAmount(e.target.value)}
        step="0.01"
        min="0"
      />
      <button onClick={handleTransfer} disabled={loading || !recipient || !amount}>
        {loading ? "Processing..." : "Send SUI (Gas-Free)"}
      </button>
      {!!result && <div className="success">{result}</div>}
      {!!error && <div className="error">{error}</div>}
    </div>
  );
}
```

### app/layout.tsx — Root Layout

```typescript
import "./globals.css";
import "@mysten/dapp-kit/dist/index.css";
import { Providers } from "@/components/Providers";

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html>
      <body>
        <Providers>{children}</Providers>
      </body>
    </html>
  );
}
```

### app/page.tsx — Main Demo Page

```typescript
"use client";

import { ConnectButton, useCurrentAccount, useSignTransaction, useSuiClient } from "@mysten/dapp-kit";
import { useState } from "react";
import { TransferForm } from "@/components/TransferForm";

export default function Page() {
  const current = useCurrentAccount();
  const suiClient = useSuiClient();
  const { mutateAsync: signTransaction } = useSignTransaction();

  const [num1, setNum1] = useState("10");
  const [num2, setNum2] = useState("20");
  const [msg, setMsg] = useState("");

  const callFrontend = async () => {
    setMsg("");
    const res = await fetch("/api/buildSponsoredTx", {
      method: "POST",
      body: JSON.stringify({ sender: current?.address, num1: Number(num1), num2: Number(num2) }),
    });
    const s = await res.json();
    if (!res.ok) return setMsg(`Error: ${s.error || "build failed"}`);

    const { signature } = await signTransaction({ transaction: s.txBytes });
    const exec = await suiClient.executeTransactionBlock({
      transactionBlock: s.txBytes,
      signature: [signature, s.sponsorSignature],
      options: { showEffects: true, showEvents: true },
    });
    setMsg(`Success (frontend execute): ${exec.digest}`);
  };

  const callBackend = async () => {
    setMsg("");
    const res = await fetch("/api/buildSponsoredTx", {
      method: "POST",
      body: JSON.stringify({ sender: current?.address, num1: Number(num1), num2: Number(num2) }),
    });
    const s = await res.json();
    if (!res.ok) return setMsg(`Error: ${s.error || "build failed"}`);

    const { signature } = await signTransaction({ transaction: s.txBytes });

    const execRes = await fetch("/api/executeSponsoredTx", {
      method: "POST",
      body: JSON.stringify({
        txBytes: s.txBytes,
        sponsorSignature: s.sponsorSignature,
        senderSignature: signature,
      }),
    });
    const exec = await execRes.json();
    if (!execRes.ok) return setMsg(`Error: ${exec.error || "execute failed"}`);
    setMsg(`Success (backend execute): ${exec.digest}`);
  };

  return (
    <main>
      <h1>Gasless Transactions on Sui</h1>
      <ConnectButton />
      {!current ? (
        <p>Please connect your wallet</p>
      ) : (
        <>
          <h2>Gasless SUI Transfer</h2>
          <TransferForm senderAddress={current.address} />

          <h2>Sponsored Move Call (math::add)</h2>
          <input type="number" value={num1} onChange={(e) => setNum1(e.target.value)} />
          <input type="number" value={num2} onChange={(e) => setNum2(e.target.value)} />
          <div>
            <button onClick={callFrontend}>Submit on Frontend</button>
            <button onClick={callBackend}>Submit on Backend</button>
          </div>

          {!!msg && <pre>{msg}</pre>}
        </>
      )}
    </main>
  );
}
```

---

## Smart Contract Module (Move)

### move-example/sources/math.move

```rust
module move_example::math {
    use sui::event;

    public struct AdditionEvent has copy, drop {
        a: u64,
        b: u64,
        result: u64,
    }

    public entry fun add(a: u64, b: u64) {
        let result = a + b;
        event::emit(AdditionEvent { a, b, result });
    }

    public entry fun hello_world() {}
}
```

### Publish

```bash
cd move-example
sui client publish --gas-budget 100000000
# Copy the package id into NEXT_PUBLIC_MOVE_PACKAGE_ID
```

---

## End-to-End Flows

### Flow A — Frontend Execute

1. Frontend → `/api/buildSponsoredTx` with params
2. Backend builds gasless tx, sponsors via Shinami → returns txBytes + sponsor signature
3. Frontend wallet signs txBytes → gets user signature
4. Frontend executes with `[userSig, sponsorSig]`
5. Success digest shown

### Flow B — Backend Execute

Same as Flow A up to user signature, then:

4. Frontend POSTs to `/api/executeSponsoredTx` with txBytes, sponsorSig, userSig
5. Backend submits, returns digest and effects

---

## Live Demo Script

### 1. Connect Wallet

- Show Connect button and connected address

### 2. Gasless SUI Transfer

- Fill recipient + amount
- Click Send SUI (Gas-Free)
- Approve in wallet
- Show digest and gas not deducted from user

### 3. Move Function Call math::add

- Enter two numbers
- Run Submit on Frontend
- Show success and event on explorer
- Repeat with Submit on Backend (centralized execution)

### 4. Shinami Dashboard

- Show Gas Station fund balance and sponsored tx log

---

## Common Pitfalls & Fixes

- **"Invalid params"** → You built bytes manually. Use `buildGaslessTransaction()`
- **Using `txb.gas`** → Not available in gasless flows. Split from sender's coin
- **realFetch.call error** → Externalize `@shinami/clients` in next.config.ts
- **Empty wallet for transfers** → Sponsor pays gas, user must have the transfer amount
- **API keys in frontend** → Keep Gas Station & Node keys backend only
- **Wrong signature order** → Must be `[userSignature, sponsorSignature]`

---

## Production Considerations

- **Security**: input validation, rate limiting, auth/allowlists, per-user caps
- **Monitoring**: Gas Station balance alerts, sponsorship logs, failure alerts
- **Budgeting**: Prefer auto-budgeting; manual only if you know exact needs
- **Reliability**: retries on backend, idempotent APIs, structured logging
- **Back office**: webhook receipts, exports, and clean accounting mapping

---

## Testing & Verification

- Wallet connects on Testnet
- Transfer works if user has transfer amount
- Move call works even with empty wallet
- Sponsored transactions show two signatures in explorers
- Shinami dashboard lists your sponsored tx with gas cost

---

## Workshop Timeline (60 minutes)

- **00:00–10:00** Problem & solution, two-signature concept
- **10:00–20:00** Architecture and project structure
- **20:00–30:00** Code walk-through (clients, routes, FE, Move)
- **30:00–45:00** Live demos: transfer, move call, dashboard
- **45:00–55:00** Pitfalls, production checklist, Q&A
- **55:00–60:00** Resources & next steps

---

## Presenter Notes

- Use real tx links as proof; keep a fallback screenshot
- If network is flaky, have a short pre-recorded clip
- Repeat the mantra: user first, sponsor second (signature order)
- Emphasize: Gas Station pays gas; user provides assets
- Keep FE simple; move sensitive logic to backend

---

## Quick Reference

### Build → Sponsor → Sign → Execute

```typescript
// Backend
const gaslessTx = await buildGaslessTransaction(
  (txb) => {
    txb.moveCall({ target: `${pkg}::math::add`, arguments: [txb.pure.u64(a), txb.pure.u64(b)] });
  },
  { sui: suiClient }
);
gaslessTx.sender = sender;

const sponsored = await gasStationClient.sponsorTransaction(gaslessTx);
// Return: { txBytes: sponsored.txBytes, sponsorSignature: sponsored.signature }

// Frontend
const { signature: userSignature } = await signTransaction({ transaction: sponsored.txBytes });

await suiClient.executeTransactionBlock({
  transactionBlock: sponsored.txBytes,
  signature: [userSignature, sponsored.sponsorSignature],
});
```

### SUI Transfer Pattern (no txb.gas)

```typescript
const coins = await suiClient.getCoins({ owner: sender, coinType: "0x2::sui::SUI" });
const gaslessTx = await buildGaslessTransaction(
  (txb) => {
    const [split] = txb.splitCoins(coins.data[0].coinObjectId, [amountInMist]);
    txb.transferObjects([split], recipient);
  },
  { sui: suiClient }
);
```

---

## Next Steps & Resources

### Extend the app

- Add NFT transfers, token swaps, event indexing
- Build history UI and status polling
- Add auth and per-user sponsorship limits

### Docs & Links

- [Shinami Docs](https://docs.shinami.com)
- [Shinami Dashboard](https://app.shinami.com)
- [Sui Docs](https://docs.sui.io)
- [dApp Kit Docs](https://sdk.mystenlabs.com/dapp-kit)
- [Shinami Examples](https://github.com/shinamicorp/shinami-examples)

---

You now have a complete, production-ready template and a full workshop script — all in one file. Ship gasless UX with confidence! 🚀
