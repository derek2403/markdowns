# zkLogin: Complete Teaching Guide

## 📘 Part 1: What is zkLogin?

### **The Simple Explanation**

zkLogin is a way to create and use a blockchain wallet using your **Google account** (or other OAuth providers) instead of managing private keys and seed phrases.

**Traditional Blockchain Wallet:**
```
User → Generate Private Key → Write down 12 words → Store safely → Sign transactions
       ❌ Complex            ❌ Easy to lose      ❌ Scary for normal users
```

**With zkLogin:**
```
User → Sign in with Google → Get blockchain address → Sign transactions
       ✅ Familiar           ✅ No seed phrases     ✅ Easy for everyone
```

---

### **The Magic: Zero-Knowledge Proofs**

**The Problem:**
- Google knows your identity
- Blockchain needs to verify you own an address
- But we DON'T want Google and blockchain to be connected!

**The Solution:**
zkLogin uses **zero-knowledge proofs** to prove "I authenticated with Google" WITHOUT revealing:
- Your Google identity to the blockchain
- Your blockchain activity to Google

**It's like this:**
> Imagine showing a bouncer your ID through a magic curtain. The bouncer can verify your ID is real and valid, but can't see your name, photo, or any personal details. That's zero-knowledge proof!

---

### **Key Benefits**

1. **No Seed Phrases** - No more writing down 12 words
2. **Familiar UX** - Sign in like any website
3. **Account Recovery** - Reset via Google account recovery
4. **Privacy** - Google never sees blockchain activity
5. **Full Custody** - You own the address (not custodial)
6. **Multi-Device** - Same Google account = same address everywhere*

*Requires proper salt management

---

## 🔄 Part 2: zkLogin Steps (The Flow)

### **Overview Diagram**

```
┌──────────────┐
│     USER     │
└──────┬───────┘
       │
       │ 1. Click "Sign in with Google"
       ▼
┌─────────────────────────────────────┐
│  APP FRONTEND                       │
│  • Generate ephemeral keypair       │
│  • Create nonce                     │
│  • Save to sessionStorage           │
└──────┬──────────────────────────────┘
       │
       │ 2. Redirect to Google
       ▼
┌─────────────────────────────────────┐
│  GOOGLE OAUTH                       │
│  • User logs in                     │
│  • Google verifies identity         │
│  • Google creates JWT               │
└──────┬──────────────────────────────┘
       │
       │ 3. Redirect back with JWT
       ▼
┌─────────────────────────────────────┐
│  APP CALLBACK                       │
│  • Extract JWT                      │
│  • Get/generate salt                │
│  • Derive zkLogin address           │
└──────┬──────────────────────────────┘
       │
       │ 4. Request ZK proof
       ▼
┌─────────────────────────────────────┐
│  PROVER SERVICE                     │
│  • Verify JWT                       │
│  • Generate zero-knowledge proof    │
│  (takes 3-10 seconds)               │
└──────┬──────────────────────────────┘
       │
       │ 5. Return proof
       ▼
┌─────────────────────────────────────┐
│  APP FRONTEND                       │
│  • User authenticated!              │
│  • Can now sign transactions        │
└──────┬──────────────────────────────┘
       │
       │ 6. Create transaction
       ▼
┌─────────────────────────────────────┐
│  SIGN TRANSACTION                   │
│  • Sign with ephemeral key          │
│  • Combine with ZK proof            │
│  • Submit to blockchain             │
└──────┬──────────────────────────────┘
       │
       │ 7. Execute
       ▼
┌─────────────────────────────────────┐
│  SUI BLOCKCHAIN                     │
│  • Verify ZK proof                  │
│  • Verify ephemeral signature       │
│  • Execute transaction ✅           │
└─────────────────────────────────────┘
```

---

### **Detailed Steps**

#### **STEP 1: Generate Ephemeral Keys**

**What happens:**
```typescript
// App generates temporary keys
ephemeralKeyPair = new Ed25519Keypair()  // Random keypair
randomness = generateRandomness()         // Random value
maxEpoch = currentEpoch + 2               // Valid for ~24 hours
nonce = hash(publicKey, maxEpoch, randomness)
```

**Why:**
- Ephemeral key is temporary (expires in 2 epochs ≈ 24 hours)
- User never sees or manages this key
- Nonce links the OAuth session to this key
- Prevents replay attacks

**Where stored:**
- sessionStorage (cleared when browser closes)

---

#### **STEP 2: OAuth with Google**

**What happens:**
```
Redirect to: https://accounts.google.com/o/oauth2/v2/auth
Parameters:
  - client_id: Your Google app ID
  - redirect_uri: Where to return after login
  - nonce: The nonce we generated
  - scope: What info we want (email, profile)
```

**Why:**
- Google verifies the user's identity
- Google includes our nonce in the JWT
- This proves the JWT was created for our request

**What Google returns:**
A JWT (JSON Web Token) containing:
```json
{
  "iss": "https://accounts.google.com",
  "sub": "1234567890",  // User's unique Google ID
  "aud": "your-client-id.apps.googleusercontent.com",
  "nonce": "the-nonce-we-sent",
  "email": "user@gmail.com",
  "exp": 1735123456  // Expiration timestamp
}
```

---

#### **STEP 3: Process Callback & Get Salt**

**What happens:**
```typescript
// Extract JWT from URL
jwt = params.get('id_token')

// Get or create salt
if (first time login) {
  salt = await fetchSaltFromServer(jwt)  // Or generate locally
  localStorage.setItem('user_salt', salt)
} else {
  salt = localStorage.getItem('user_salt')
}
```

**Why salt is CRITICAL:**
- Makes address derivation deterministic but unique
- Same Google account + same salt = always same address
- Different salt = different address (even with same Google account)
- **If you lose salt, you LOSE ACCESS to the address forever!**

**Storage:**
- localStorage (persists forever)
- In production: Store on backend server

---

#### **STEP 4: Derive zkLogin Address**

**What happens:**
```typescript
// Combine JWT + salt to create address
addressSeed = hash(
  salt,
  "sub",           // Key claim name
  jwt.sub,         // User's Google ID
  jwt.aud          // Your app's client ID
)

zkLoginAddress = deriveAddress(addressSeed)
```

**Why:**
- Deterministic: Same inputs = same output
- Unique: Your salt makes your address different from others
- Secure: Can't reverse engineer from address to Google ID

**Example:**
```
Google ID: 1234567890
Salt: 9876543210
→ zkLogin Address: 0x1a2b3c4d5e6f7890...
```

---

#### **STEP 5: Generate Zero-Knowledge Proof**

**What happens:**
```typescript
// Send to prover service
request = {
  jwt: "eyJhbGci...",
  extendedEphemeralPublicKey: "...",
  maxEpoch: 12345,
  jwtRandomness: "...",
  salt: "...",
  keyClaimName: "sub"
}

// Prover does complex cryptographic computation
zkProof = await proverService.generateProof(request)
```

**Why:**
- Proves "I have a valid JWT from Google"
- WITHOUT revealing the JWT itself!
- Blockchain can verify the proof
- But cannot extract your Google identity from it

**How long:**
- 3-10 seconds (complex cryptography)
- Only needs to be done once per session

---

#### **STEP 6: Sign Transaction**

**What happens:**
```typescript
// Create transaction
transaction = {
  sender: zkLoginAddress,
  action: "transfer 1 SUI to 0xabc..."
}

// Sign with ephemeral key
ephemeralSignature = ephemeralKey.sign(transaction)

// Combine into zkLogin signature
zkLoginSignature = {
  ephemeralSignature,
  zkProof,
  addressSeed,
  maxEpoch
}

// Submit to blockchain
blockchain.execute(transaction, zkLoginSignature)
```

**Why:**
- Ephemeral key signs the transaction data
- ZK proof proves you own the address
- Together they create a valid signature
- Blockchain can verify WITHOUT seeing JWT

---

#### **STEP 7: Blockchain Verification**

**What blockchain verifies:**
```
✓ Is the ZK proof valid?
✓ Does the proof match the address?
✓ Is the ephemeral signature valid?
✓ Is the ephemeral key still valid (not expired)?
✓ Does everything match up?
```

**What blockchain NEVER sees:**
- Your JWT token
- Your Google email
- Your Google ID
- Any connection to Google

**Result:** Transaction executes! 🎉

---

## 💻 Part 3: Implementing zkLogin

### **Architecture Overview**

```
Project Structure:
├── lib/zklogin/          # Core logic
│   ├── config.ts         # Configuration
│   ├── types.ts          # Type definitions
│   └── utils.ts          # Main implementation
├── components/           # UI components
│   ├── zklogin-auth.tsx  # Auth button
│   └── transaction-demo.tsx  # Transaction UI
└── app/
    ├── page.tsx          # Main page
    └── auth/callback/    # OAuth handler
        └── page.tsx
```

---

### **Prerequisites**

**What you need:**
1. Google OAuth Client ID
2. Sui SDK (`@mysten/sui`)
3. Next.js or React app
4. Understanding of async/await JavaScript

**Setup Steps:**
```bash
# 1. Install dependencies
npm install @mysten/sui jwt-decode

# 2. Get Google OAuth Client ID
# Go to console.cloud.google.com
# Create OAuth 2.0 Client ID
# Add redirect URI: http://localhost:3000/auth/callback

# 3. Set environment variables
NEXT_PUBLIC_GOOGLE_CLIENT_ID=your_id_here
NEXT_PUBLIC_FULLNODE_URL=https://fullnode.devnet.sui.io
NEXT_PUBLIC_PROVER_URL=https://prover-dev.mystenlabs.com/v1
```

---

## 📂 Part 4: Repository File-by-File Explanation

### **🔧 Configuration Files**

#### **1. `lib/zklogin/config.ts`**

**What it does:** Stores all configuration and environment variables

**Code:**
```typescript
export const ZKLOGIN_CONFIG = {
  // Your Google OAuth app ID
  GOOGLE_CLIENT_ID: process.env.NEXT_PUBLIC_GOOGLE_CLIENT_ID || '',
  
  // Where Google redirects after login
  REDIRECT_URI: process.env.NEXT_PUBLIC_REDIRECT_URI || 'http://localhost:3000/auth/callback',
  
  // Sui blockchain RPC endpoint
  FULLNODE_URL: process.env.NEXT_PUBLIC_FULLNODE_URL || 'https://fullnode.devnet.sui.io',
  
  // Service that generates ZK proofs
  PROVER_URL: process.env.NEXT_PUBLIC_PROVER_URL || 'https://prover-dev.mystenlabs.com/v1',
  
  // Service that provides user salts (optional)
  SALT_SERVER_URL: process.env.NEXT_PUBLIC_SALT_SERVER_URL || 'https://salt.api.mystenlabs.com/get_salt',
};

// Keys for browser storage
export const STORAGE_KEYS = {
  EPHEMERAL_KEY_PAIR: 'ephemeral_key_pair',  // sessionStorage
  RANDOMNESS: 'randomness',                   // sessionStorage
  MAX_EPOCH: 'max_epoch',                     // sessionStorage
  USER_SALT: 'user_salt',                     // localStorage ⚠️
  JWT_TOKEN: 'jwt_token',                     // sessionStorage
  ZKLOGIN_ADDRESS: 'zklogin_address',         // sessionStorage
};
```

**Why we need this:**
- Central place for all configuration
- Easy to change environments (devnet → testnet → mainnet)
- Clear documentation of what each setting does

---

#### **2. `lib/zklogin/types.ts`**

**What it does:** TypeScript type definitions for type safety

**Code:**
```typescript
// Structure of Google's JWT token
export interface JwtPayload {
  iss?: string;          // Issuer (accounts.google.com)
  sub?: string;          // Subject (user's Google ID) - IMPORTANT!
  aud?: string | string[]; // Audience (your client ID)
  exp?: number;          // Expiration timestamp
  iat?: number;          // Issued at timestamp
  nonce?: string;        // The nonce we sent
}

// Structure of zero-knowledge proof
export interface PartialZkLoginSignature {
  proofPoints: {
    a: string[];         // Proof component A
    b: string[][];       // Proof component B
    c: string[];         // Proof component C
  };
  issBase64Details: {
    value: string;       // Issuer details
    indexMod4: number;   // Index position
  };
  headerBase64: string;  // JWT header
}

// Application state
export interface ZkLoginState {
  ephemeralKeyPair: string | null;
  randomness: string | null;
  maxEpoch: number | null;
  userSalt: string | null;
  jwtToken: string | null;
  zkLoginAddress: string | null;
}
```

**Why we need this:**
- Type safety in TypeScript
- Auto-completion in IDE
- Catch errors at compile time
- Self-documenting code

---

### **⚙️ Core Implementation**

#### **3. `lib/zklogin/utils.ts` ⭐ MOST IMPORTANT FILE**

This file contains ALL the core zkLogin logic. Let's break it down function by function.

---

##### **Function 1: Generate Ephemeral Keys**

**Code:**
```typescript
export async function generateEphemeralKeyPair() {
  // 1. Connect to Sui blockchain to get current epoch
  const suiClient = new SuiClient({ url: ZKLOGIN_CONFIG.FULLNODE_URL });
  const { epoch } = await suiClient.getLatestSuiSystemState();
  
  // 2. Calculate when this key expires (2 epochs from now ≈ 24 hours)
  const maxEpoch = Number(epoch) + 2;
  
  // 3. Generate random Ed25519 keypair (standard elliptic curve crypto)
  const ephemeralKeyPair = new Ed25519Keypair();
  
  // 4. Generate random value for nonce
  const randomness = generateRandomness();
  
  // 5. Create nonce = hash(publicKey + maxEpoch + randomness)
  //    This links the OAuth session to this specific key
  const nonce = generateNonce(
    ephemeralKeyPair.getPublicKey(), 
    maxEpoch, 
    randomness
  );
  
  // 6. Return everything
  return {
    ephemeralKeyPair,  // The keypair (public + private)
    randomness,        // Random value used in nonce
    nonce,             // The nonce to send to Google
    maxEpoch,          // When this key expires
  };
}
```

**Why each part:**
- **Get current epoch:** Need to know when key should expire
- **maxEpoch = epoch + 2:** Keys valid for ~24 hours, then must re-authenticate
- **Ed25519Keypair:** Standard, secure, fast elliptic curve
- **generateRandomness():** Adds unpredictability to nonce
- **generateNonce():** Links OAuth session to this key (prevents attacks)

**When called:** When user clicks "Sign in with Google"

---

##### **Function 2: Build Google OAuth URL**

**Code:**
```typescript
export function getGoogleAuthURL(nonce: string): string {
  // Build query parameters for Google OAuth
  const params = new URLSearchParams({
    client_id: ZKLOGIN_CONFIG.GOOGLE_CLIENT_ID,     // Your app ID
    redirect_uri: ZKLOGIN_CONFIG.REDIRECT_URI,      // Where to return
    response_type: 'id_token',                      // We want JWT
    scope: 'openid email profile',                  // What info we want
    nonce: nonce,                                   // Our nonce!
  });
  
  // Return full Google OAuth URL
  return `https://accounts.google.com/o/oauth2/v2/auth?${params.toString()}`;
}
```

**Why each part:**
- **client_id:** Tells Google which app is requesting
- **redirect_uri:** Where Google sends user after login (MUST match Google Console!)
- **response_type: 'id_token':** We want JWT directly (not authorization code)
- **scope:** What user info we need (email, profile)
- **nonce:** Critical! Links this OAuth request to our ephemeral key

**When called:** Right before redirecting to Google

---

##### **Function 3: Decode JWT**

**Code:**
```typescript
export function decodeJwt(token: string): JwtPayload {
  // JWT is base64 encoded. Decode it to get user info
  return jwtDecode<JwtPayload>(token);
}
```

**Why:**
- JWT is encoded, we need to read what's inside
- Get user's Google ID (sub), client ID (aud), issuer (iss)
- These are needed for address derivation

**When called:** After receiving JWT from Google

---

##### **Function 4: Get User Salt**

**Code:**
```typescript
export async function getUserSalt(jwt: string): Promise<string> {
  // Try to get salt from Mysten Labs server
  try {
    const response = await fetch(ZKLOGIN_CONFIG.SALT_SERVER_URL, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ token: jwt }),
    });
    
    if (!response.ok) {
      throw new Error(`Salt server returned ${response.status}`);
    }
    
    const data = await response.json();
    console.log('✅ Got salt from server');
    return data.salt;
    
  } catch (error) {
    // EXPECTED: Most apps aren't whitelisted for Mysten's server
    console.log('ℹ️ Salt server not accessible, using local fallback');
    
    // Fallback: Generate random salt locally
    // This is secure for demo/testing
    const fallbackSalt = BigInt(
      Math.floor(Math.random() * Number.MAX_SAFE_INTEGER)
    ).toString();
    
    console.log('✅ Generated local salt');
    return fallbackSalt;
  }
}
```

**Why each part:**
- **Try server first:** In production, backend manages salts
- **Catch error:** Mysten's server requires whitelisting
- **Fallback:** Generate random salt locally (secure for demo)
- **Store in localStorage:** MUST persist forever!

**CRITICAL WARNING:**
```
⚠️ If user loses salt, they LOSE ACCESS to address permanently!
   In production: Store salt on your backend server!
```

**When called:** First time user logs in, or when localStorage is empty

---

##### **Function 5: Get zkLogin Address**

**Code:**
```typescript
export function getZkLoginAddress(jwt: string, userSalt: string): string {
  // Use Sui SDK function to derive address
  // Internally: hash(jwt.sub + jwt.aud + salt) → address
  return jwtToAddress(jwt, userSalt);
}
```

**Why:**
- Deterministic: Same JWT + same salt = same address
- Secure: Can't reverse from address to Google ID
- Simple: Sui SDK handles the complex crypto

**When called:** After getting JWT and salt

---

##### **Function 6: Get Zero-Knowledge Proof** ⭐

**Code:**
```typescript
export async function getZkProof(
  jwt: string,
  ephemeralKeyPair: Ed25519Keypair,
  maxEpoch: number,
  randomness: string,
  userSalt: string
): Promise<PartialZkLoginSignature> {
  
  // 1. Get extended public key (includes extra data)
  const extendedEphemeralPublicKey = getExtendedEphemeralPublicKey(
    ephemeralKeyPair.getPublicKey()
  );
  
  // 2. Build request payload
  const payload = {
    jwt,                                          // The JWT from Google
    extendedEphemeralPublicKey: extendedEphemeralPublicKey.toString(),
    maxEpoch: maxEpoch.toString(),               // When key expires
    jwtRandomness: randomness,                   // Random value from nonce
    salt: userSalt,                              // User's salt
    keyClaimName: 'sub',                         // Which JWT field has user ID
  };
  
  // 3. Send to prover service
  const response = await fetch(ZKLOGIN_CONFIG.PROVER_URL, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(payload),
  });
  
  if (!response.ok) {
    throw new Error(`Prover request failed: ${response.statusText}`);
  }
  
  // 4. Return the zero-knowledge proof
  const proof = await response.json();
  return proof as PartialZkLoginSignature;
}
```

**Why each part:**
- **extendedEphemeralPublicKey:** Includes additional data prover needs
- **Send everything:** Prover needs all info to create proof
- **keyClaimName: 'sub':** Tells prover to use Google ID field
- **Takes 3-10 seconds:** Complex cryptographic computation

**What prover does:**
```
Input: JWT + ephemeral key + salt
Process: Complex zero-knowledge proof generation
Output: Proof that proves "I have valid JWT" without revealing JWT
```

**When called:** After OAuth callback, before transactions

---

##### **Function 7: Generate Address Seed**

**Code:**
```typescript
export function generateAddressSeed(
  userSalt: string,
  sub: string,        // User's Google ID
  aud: string | string[]  // Your client ID
): string {
  // Handle aud as array or string
  const audString: string = Array.isArray(aud) ? aud[0] : aud;
  
  // Use Sui SDK to generate address seed
  return genAddressSeed(
    BigInt(userSalt),
    'sub',      // Key claim name
    sub,        // User ID value
    audString   // Client ID value
  ).toString();
}
```

**Why:**
- Needed when signing transactions
- Combines salt + user ID + client ID
- Must match the address derivation

**When called:** When creating transaction signature

---

##### **Function 8: Storage Utilities**

**Code:**
```typescript
export const storage = {
  // Save ephemeral keypair
  saveEphemeralKeyPair: (keyPair: Ed25519Keypair) => {
    if (typeof window !== 'undefined') {
      sessionStorage.setItem(
        STORAGE_KEYS.EPHEMERAL_KEY_PAIR, 
        keyPair.getSecretKey()  // Store private key
      );
    }
  },
  
  // Load ephemeral keypair
  loadEphemeralKeyPair: (): Ed25519Keypair | null => {
    if (typeof window !== 'undefined') {
      const stored = sessionStorage.getItem(STORAGE_KEYS.EPHEMERAL_KEY_PAIR);
      if (stored) {
        return Ed25519Keypair.fromSecretKey(stored);  // Reconstruct keypair
      }
    }
    return null;
  },
  
  // Save user salt (⚠️ CRITICAL)
  saveUserSalt: (salt: string) => {
    if (typeof window !== 'undefined') {
      localStorage.setItem(STORAGE_KEYS.USER_SALT, salt);  // localStorage = persists forever
    }
  },
  
  // Load user salt
  loadUserSalt: (): string | null => {
    if (typeof window !== 'undefined') {
      return localStorage.getItem(STORAGE_KEYS.USER_SALT);
    }
    return null;
  },
  
  // Clear session (logout)
  clearAll: () => {
    if (typeof window !== 'undefined') {
      sessionStorage.clear();
      // Note: Don't clear localStorage (keep salt!)
    }
  },
  
  // Clear everything including salt (⚠️ DANGEROUS)
  clearAllIncludingSalt: () => {
    if (typeof window !== 'undefined') {
      sessionStorage.clear();
      localStorage.clear();  // This will lose the address!
    }
  },
  
  // ... more storage functions for randomness, maxEpoch, JWT, etc.
};
```

**Storage Strategy:**

**sessionStorage (temporary, cleared on browser close):**
- Ephemeral keypair
- Randomness
- Max epoch
- JWT token
- zkLogin address
- ZK proof

**localStorage (permanent, never cleared):**
- User salt ⚠️ CRITICAL

**Why this separation:**
- Session data should expire when browser closes (security)
- Salt must persist forever (or user loses address)

---

### **🎨 UI Components**

#### **4. `components/zklogin-auth.tsx`**

**What it does:** Provides the "Sign in with Google" button and authentication UI

**Key code blocks:**

**A. Component State**
```typescript
const [isLoading, setIsLoading] = useState(false);
const [error, setError] = useState<string | null>(null);
const [zkLoginAddress, setZkLoginAddress] = useState<string | null>(null);

useEffect(() => {
  // Check if already authenticated
  const savedAddress = storage.loadZkLoginAddress();
  if (savedAddress) {
    setZkLoginAddress(savedAddress);
    onAuthenticated?.(savedAddress);
  }
}, []);
```

**Why:**
- Track loading state (show spinner)
- Track errors (show error message)
- Track address (show when authenticated)
- Check on mount if already logged in

---

**B. Handle Google Login**
```typescript
const handleGoogleLogin = async () => {
  setIsLoading(true);
  setError(null);

  try {
    // Step 1: Generate ephemeral key pair
    console.log('Generating ephemeral key pair...');
    const { ephemeralKeyPair, randomness, nonce, maxEpoch } = 
      await generateEphemeralKeyPair();
    
    // Step 2: Save to session storage
    storage.saveEphemeralKeyPair(ephemeralKeyPair);
    storage.saveRandomness(randomness);
    storage.saveMaxEpoch(maxEpoch);
    
    // Step 3: Redirect to Google OAuth
    const authUrl = getGoogleAuthURL(nonce);
    console.log('Redirecting to Google OAuth...');
    window.location.href = authUrl;  // Browser navigates away
    
  } catch (err) {
    console.error('Error during login:', err);
    setError(err instanceof Error ? err.message : 'Failed to initiate login');
    setIsLoading(false);
  }
};
```

**Why each step:**
1. **Generate keys:** Need ephemeral key and nonce
2. **Save locally:** Will need these after redirect
3. **Redirect:** User goes to Google to log in

---

**C. Handle Logout**
```typescript
const handleLogout = () => {
  storage.clearAll();  // Clear sessionStorage (keep salt!)
  setZkLoginAddress(null);
  window.location.reload();
};
```

**Why:**
- Clear session data
- Keep salt in localStorage (user can log back in)
- Reload page to reset state

---

**D. UI Rendering**
```typescript
// If authenticated, show address and logout button
if (zkLoginAddress) {
  return (
    <div>
      <p>Address: {zkLoginAddress}</p>
      <button onClick={handleLogout}>Logout</button>
      <button onClick={handleReset}>Reset All Data</button>
    </div>
  );
}

// Otherwise, show login button
return (
  <div>
    <button onClick={handleGoogleLogin} disabled={isLoading}>
      {isLoading ? 'Loading...' : 'Sign in with Google'}
    </button>
  </div>
);
```

---

#### **5. `components/transaction-demo.tsx`**

**What it does:** UI for viewing balance, requesting faucet, and sending transactions

**Key code blocks:**

**A. Load Balance**
```typescript
const loadBalance = async () => {
  setIsLoadingBalance(true);
  try {
    // Connect to Sui blockchain
    const client = new SuiClient({ url: ZKLOGIN_CONFIG.FULLNODE_URL });
    
    // Get balance for this address
    const balanceResult = await client.getBalance({ owner: address });
    
    // Convert from MIST to SUI (1 SUI = 1_000_000_000 MIST)
    setBalance((Number(balanceResult.totalBalance) / 1_000_000_000).toFixed(4));
    
  } catch (err) {
    console.error('Error loading balance:', err);
    setError('Failed to load balance');
  } finally {
    setIsLoadingBalance(false);
  }
};
```

**Why:**
- Need to know balance before sending transactions
- MIST is the smallest unit (like satoshis for Bitcoin)
- Error handling for network issues

---

**B. Request Faucet**
```typescript
const requestFaucet = async () => {
  try {
    // Call devnet faucet API
    const response = await fetch('https://faucet.devnet.sui.io/gas', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        FixedAmountRequest: {
          recipient: address,  // Send to our zkLogin address
        },
      }),
    });

    if (!response.ok) {
      throw new Error('Faucet request failed');
    }

    alert('Test tokens requested! Please wait a few seconds and refresh balance.');
    setTimeout(() => loadBalance(), 3000);  // Auto-refresh after 3 seconds
    
  } catch (err) {
    setError('Failed to request from faucet. You may have exceeded the rate limit.');
  }
};
```

**Why:**
- Need SUI tokens to pay for gas
- Devnet faucet gives free test tokens
- Rate limited (can't spam it)

---

**C. Send Transaction** ⭐ MOST COMPLEX

```typescript
const sendTransaction = async () => {
  setIsSending(true);
  setError(null);

  try {
    // STEP 1: Load required data from storage
    const ephemeralKeyPair = storage.loadEphemeralKeyPair();
    const jwt = storage.loadJwtToken();
    const maxEpoch = storage.loadMaxEpoch();
    const userSalt = storage.loadUserSalt();
    const zkProofStr = sessionStorage.getItem('zk_proof');

    // Verify we have everything
    if (!ephemeralKeyPair || !jwt || maxEpoch === null || !userSalt || !zkProofStr) {
      throw new Error('Missing authentication data. Please log in again.');
    }

    // Parse stored data
    const decodedJwt = decodeJwt(jwt);
    const partialZkProof: PartialZkLoginSignature = JSON.parse(zkProofStr);

    // STEP 2: Create transaction
    const client = new SuiClient({ url: ZKLOGIN_CONFIG.FULLNODE_URL });
    const txb = new Transaction();
    txb.setSender(address);  // This zkLogin address is the sender

    // Split gas coin and transfer
    const [coin] = txb.splitCoins(
      txb.gas,  // Use gas coin
      [BigInt(Math.floor(parseFloat(amount) * 1_000_000_000))]  // Convert SUI to MIST
    );
    txb.transferObjects([coin], recipientAddress);

    // STEP 3: Sign transaction with ephemeral key
    const { bytes, signature: userSignature } = await txb.sign({
      client,
      signer: ephemeralKeyPair,  // Sign with ephemeral key
    });

    // STEP 4: Generate address seed
    const addressSeed = generateAddressSeed(
      userSalt,
      decodedJwt.sub!,   // User's Google ID
      decodedJwt.aud as string  // Your client ID
    );

    // STEP 5: Assemble zkLogin signature
    const zkLoginSignature = getZkLoginSignature({
      inputs: {
        ...partialZkProof,  // The ZK proof
        addressSeed,         // Links to address
      },
      maxEpoch,              // When key expires
      userSignature,         // Ephemeral key signature
    });

    // STEP 6: Execute transaction on blockchain
    const result = await client.executeTransactionBlock({
      transactionBlock: bytes,
      signature: zkLoginSignature,  // Our zkLogin signature!
    });

    console.log('Transaction result:', result);
    setTxResult(result.digest);  // Show transaction hash
    
    // Refresh balance after transaction
    setTimeout(() => loadBalance(), 2000);
    
  } catch (err) {
    console.error('Transaction error:', err);
    setError(err instanceof Error ? err.message : 'Transaction failed');
  } finally {
    setIsSending(false);
  }
};
```

**Why each step:**
1. **Load data:** Need all auth data from earlier
2. **Create transaction:** Standard Sui transaction
3. **Sign with ephemeral key:** Creates user signature
4. **Generate address seed:** Needed for zkLogin signature
5. **Assemble zkLogin signature:** Combine all pieces
6. **Execute:** Submit to blockchain for verification

**What blockchain verifies:**
- ZK proof is valid
- Ephemeral signature is valid
- Address matches
- Key hasn't expired
- Transaction is valid

---

### **📄 Page Components**

#### **6. `app/page.tsx`**

**What it does:** Main landing page that orchestrates everything

**Key code:**
```typescript
export default function Home() {
  const [authenticatedAddress, setAuthenticatedAddress] = useState<string | null>(null);

  return (
    <div>
      {/* Header */}
      <h1>Sui zkLogin Demo</h1>
      
      {/* Authentication Section */}
      <ZkLoginAuth onAuthenticated={setAuthenticatedAddress} />
      
      {/* Transaction Section - only shown when authenticated */}
      {authenticatedAddress && (
        <TransactionDemo address={authenticatedAddress} />
      )}
      
      {/* Information and instructions */}
      <div>
        <h3>How it works</h3>
        <ol>
          <li>Sign in with Google</li>
          <li>Complete authentication</li>
          <li>Get zkLogin address</li>
          <li>Request test tokens</li>
          <li>Send transactions</li>
        </ol>
      </div>
    </div>
  );
}
```

**Why:**
- Simple orchestration
- Pass authenticated address to transaction component
- Conditional rendering (only show transactions when logged in)

---

#### **7. `app/auth/callback/page.tsx`** ⭐ CRITICAL

**What it does:** Handles the OAuth callback from Google

This is where the MAGIC happens after Google redirects back!

**Full flow with explanations:**

```typescript
export default function AuthCallback() {
  const router = useRouter();
  const [status, setStatus] = useState('Processing authentication...');

  useEffect(() => {
    const processCallback = async () => {
      try {
        // ═══════════════════════════════════════════════════════
        // STEP 1: Extract JWT from URL fragment
        // ═══════════════════════════════════════════════════════
        setStatus('Extracting JWT token...');
        
        // Google returns JWT in URL fragment (after #)
        // Example: http://localhost:3000/auth/callback#id_token=eyJhb...
        const hash = window.location.hash.substring(1);  // Remove #
        const params = new URLSearchParams(hash);
        const idToken = params.get('id_token');

        if (!idToken) {
          throw new Error('No JWT token found in callback URL');
        }

        // ═══════════════════════════════════════════════════════
        // STEP 2: Decode JWT
        // ═══════════════════════════════════════════════════════
        setStatus('Decoding JWT...');
        
        const decodedJwt = decodeJwt(idToken);
        console.log('Decoded JWT:', decodedJwt);
        // Contains: sub (user ID), aud (client ID), iss, nonce, etc.

        // ═══════════════════════════════════════════════════════
        // STEP 3: Load ephemeral key pair from session storage
        // ═══════════════════════════════════════════════════════
        setStatus('Loading ephemeral key pair...');
        
        // Remember: We saved these BEFORE redirecting to Google
        const ephemeralKeyPair = storage.loadEphemeralKeyPair();
        const randomness = storage.loadRandomness();
        const maxEpoch = storage.loadMaxEpoch();

        // If missing, user needs to start over
        if (!ephemeralKeyPair || !randomness || maxEpoch === null) {
          throw new Error('Missing ephemeral key pair data. Please try logging in again.');
        }

        // ═══════════════════════════════════════════════════════
        // STEP 4: Get or create user salt ⚠️ CRITICAL
        // ═══════════════════════════════════════════════════════
        setStatus('Getting user salt...');
        
        let userSalt = storage.loadUserSalt();
        
        if (!userSalt) {
          // FIRST TIME LOGIN
          console.log('🔐 First time login - generating user salt...');
          userSalt = await getUserSalt(idToken);
          storage.saveUserSalt(userSalt);  // Save to localStorage
          console.log('✅ User salt saved to localStorage');
        } else {
          // RETURNING USER
          console.log('✅ Using existing user salt from localStorage');
        }

        // ═══════════════════════════════════════════════════════
        // STEP 5: Derive zkLogin address
        // ═══════════════════════════════════════════════════════
        setStatus('Deriving zkLogin address...');
        
        // This is deterministic: same JWT + same salt = same address
        const zkLoginAddress = getZkLoginAddress(idToken, userSalt);
        console.log('✅ zkLogin address derived:', zkLoginAddress);

        // ═══════════════════════════════════════════════════════
        // STEP 6: Get zero-knowledge proof
        // ═══════════════════════════════════════════════════════
        setStatus('Generating zero-knowledge proof (this may take 3-10 seconds)...');
        
        try {
          console.log('🔄 Requesting zero-knowledge proof from prover service...');
          
          const zkProof = await getZkProof(
            idToken,
            ephemeralKeyPair,
            maxEpoch,
            randomness,
            userSalt
          );
          
          console.log('✅ Zero-knowledge proof obtained successfully');

          // Store the proof for later use (when signing transactions)
          sessionStorage.setItem('zk_proof', JSON.stringify(zkProof));
          
        } catch (proofError) {
          console.error('❌ Error getting ZK proof:', proofError);
          console.warn('⚠️ Continuing without ZK proof - will generate on first transaction');
          // Note: We could continue without proof and generate it later
        }

        // ═══════════════════════════════════════════════════════
        // STEP 7: Save authentication data
        // ═══════════════════════════════════════════════════════
        storage.saveJwtToken(idToken);
        storage.saveZkLoginAddress(zkLoginAddress);

        // ═══════════════════════════════════════════════════════
        // STEP 8: Redirect to home
        // ═══════════════════════════════════════════════════════
        setStatus('Authentication successful! Redirecting...');
        setTimeout(() => {
          router.push('/');  // Go back to main page
        }, 1000);

      } catch (err) {
        console.error('Authentication error:', err);
        setError(err.message);
        
        // Clear potentially corrupted data
        storage.clearAll();
        
        // Redirect back to home after showing error
        setTimeout(() => {
          router.push('/');
        }, 3000);
      }
    };

    processCallback();
  }, [router]);

  // UI shows status or error
  return (
    <div>
      {error ? (
        <div>Authentication Failed: {error}</div>
      ) : (
        <div>
          <div className="spinner"></div>
          <p>{status}</p>
        </div>
      )}
    </div>
  );
}
```

**Why each step matters:**

**Step 1 - Extract JWT:**
- Google puts JWT in URL fragment (after #)
- Fragment doesn't go to server (stays in browser)
- This is more secure than query parameter

**Step 2 - Decode JWT:**
- Need to read user ID and other claims
- JWT is just base64 encoded JSON

**Step 3 - Load ephemeral data:**
- We need the keys we generated earlier
- If missing, authentication process was interrupted

**Step 4 - Get salt (MOST CRITICAL):**
- First time: Generate and save salt
- Returning: Load existing salt
- **If salt changes, address changes!**
- This is why salt must be in localStorage

**Step 5 - Derive address:**
- Combine JWT + salt → address
- Deterministic and secure

**Step 6 - Get ZK proof:**
- This is the expensive part (3-10 seconds)
- Proves we have valid JWT
- Needed for signing transactions

**Step 7 - Save everything:**
- Store JWT, address, proof
- Ready for transactions!

**Step 8 - Redirect:**
- Go back to main page
- User now authenticated

---

## 🎯 Part 5: Why We Do These Things

### **Why Ephemeral Keys?**

**Problem:** If we used a permanent private key, we'd need to:
- Generate it securely
- Store it safely
- Manage it carefully
- Users would be back to seed phrases!

**Solution:** Ephemeral keys
- Generated fresh each session
- Expire after 24 hours
- User never sees them
- App manages automatically
- Lost key? Just log in again!

---

### **Why Nonce?**

**Problem:** Without nonce:
- Attacker could reuse old JWT
- Could create fake OAuth sessions
- Security vulnerability!

**Solution:** Nonce
- Links OAuth session to ephemeral key
- Can't reuse JWT with different key
- Prevents replay attacks
- Google includes it in JWT

---

### **Why Salt?**

**Problem:** Without salt:
- Everyone with same Google account gets same address
- No way to have multiple addresses
- Privacy issues

**Solution:** Salt
- Makes address derivation unique
- Same Google + different salt = different address
- Allows multiple addresses per Google account
- Must be stored securely!

---

### **Why Zero-Knowledge Proof?**

**Problem:** How do we prove to blockchain we own the address?
- Can't send JWT (exposes Google identity)
- Can't send just ephemeral signature (doesn't prove ownership)
- Need to link Google auth to blockchain address

**Solution:** Zero-knowledge proof
- Proves "I have valid JWT from Google"
- WITHOUT revealing JWT
- Blockchain can verify
- Privacy preserved!

**The cryptographic guarantee:**
```
Blockchain learns:
✓ You authenticated with Google
✓ Your authentication is valid
✓ You own this address

Blockchain NEVER learns:
✗ Your Google email
✗ Your Google ID
✗ Your JWT contents
✗ Any personal information
```

---

### **Why Two Storage Types?**

**sessionStorage (temporary):**
- Ephemeral keys (should expire)
- JWT token (should expire)
- ZK proof (should expire)
- zkLogin address (recalculated each session)

**localStorage (permanent):**
- User salt (MUST persist forever)

**Why this separation:**
- Security: Session data auto-expires
- Convenience: Salt lets you log back in
- Best practice: Don't store sensitive data permanently

---

### **Why This Architecture?**

**Frontend-first approach:**
- No backend required for basic zkLogin
- Easy to get started
- Lower infrastructure costs
- User maintains full custody

**When you need backend:**
- Production salt management
- User account management
- Analytics and monitoring
- Enhanced security

---

## 🎓 Teaching Tips

### **Start with "Why"**

Before diving into code, show the problem:
1. Demo a traditional wallet (show seed phrase)
2. Ask: "Has anyone lost their seed phrase?"
3. Then introduce zkLogin as the solution

---

### **Use Analogies**

**zkLogin = Hotel Key Card**
- Old way: You own the hotel (private key)
- New way: Hotel gives you room key (OAuth)
- Key expires, but you can get a new one anytime
- Zero-knowledge proof: Proves you're a guest without showing ID

---

### **Show Don't Just Tell**

**Live demo beats everything:**
1. Show fresh browser
2. Click sign in
3. Show browser DevTools (storage)
4. Complete OAuth
5. Show address derived
6. Send transaction
7. Show on blockchain explorer

---

### **Emphasize Critical Parts**

**Repeat these multiple times:**
- ⚠️ Salt is CRITICAL - lose it = lose address
- ⏱️ ZK proof takes time (3-10 seconds)
- 🔄 Ephemeral keys expire (~24 hours)
- 🔒 Privacy preserved (Google ≠ Blockchain)

---

## 📚 Common Questions & Answers

**Q: Is this production-ready?**
A: YES, but you need:
- Your own salt management backend
- Mysten Labs prover approval (or run your own)
- Proper error handling
- Key rotation strategy

**Q: What if Google shuts down?**
A: zkLogin supports multiple providers:
- Google, Facebook, Twitch, Apple, etc.
- User can link multiple OAuth accounts
- Diversifies risk

**Q: Can I recover my account?**
A: YES, if you have:
- Same OAuth account
- Same salt
Then you can always regenerate the same address

**Q: What about multi-device?**
A: Need salt on all devices
- Solution: Backend salt management
- Sync salt securely across devices

**Q: Is this secure?**
A: YES, cryptographically proven:
- Zero-knowledge proofs are mathematically sound
- No private key exposure
- OAuth security + blockchain security

---

## 🚀 Summary

**zkLogin in one sentence:**
> Sign blockchain transactions using Google login through zero-knowledge proofs that preserve privacy.

**Key components:**
1. **Ephemeral Keys** - Temporary signing keys
2. **Nonce** - Links OAuth to keys
3. **Salt** - Makes addresses unique
4. **JWT** - Proves Google verified you
5. **ZK Proof** - Proves JWT without revealing it

**The flow:**
```
User → Google OAuth → JWT → (JWT + Salt) → zkLogin Address
                              ↓
                        Ephemeral Key + ZK Proof → Sign Transactions
```

**Why it matters:**
- Web2 UX for Web3
- No seed phrases
- Privacy preserved
- Full custody
- Production ready

---

**Now you're ready to teach zkLogin! Start with this guide, use your working code as examples, and watch developers' minds blown by zero-knowledge proofs! 🎉**


