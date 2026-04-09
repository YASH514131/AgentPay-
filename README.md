# AgentPay

> **The payment infrastructure layer for AI agents.** AgentPay gives every autonomous AI agent a programmable USDC payment rail on Solana — with user-controlled spending limits, an open agent registry, and protocol-level commission on every transaction.

[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Built on Solana](https://img.shields.io/badge/Built%20on-Solana-9945FF)](https://solana.com)
[![USDC](https://img.shields.io/badge/Token-USDC-2775CA)](https://www.circle.com/en/usdc)
[![Anchor](https://img.shields.io/badge/Framework-Anchor-FFA500)](https://anchor-lang.com)
[![Status: Concept](https://img.shields.io/badge/Status-Concept-lightgrey)](#status)

---

## Status

**This repository is currently a protocol design and architecture document. No code has been written yet.**

| Component | Status |
|-----------|--------|
| Anchor program (on-chain) | ⬜ Not started |
| Node.js backend | ⬜ Not started |
| Flutter app | ⬜ Not started |
| Agent SDK (npm) | ⬜ Not started |
| Devnet deployment | ⬜ Not started |

The README documents the intended architecture in full so contributors and potential agent integrators can evaluate and discuss the design before implementation begins. See [Roadmap](#roadmap) for the build sequence.

---

## Table of Contents

- [Status](#status)
- [Overview](#overview)
- [Quickstart](#quickstart)
- [How It Works](#how-it-works)
- [Architecture](#architecture)
  - [On-Chain Program (Anchor/Rust)](#on-chain-program-anchorrust)
  - [Backend Services (Node.js)](#backend-services-nodejs)
  - [Agent Registry](#agent-registry)
  - [Flutter Frontend](#flutter-frontend)
- [Commission Model](#commission-model)
- [Security Model](#security-model)
- [Scalability Design](#scalability-design)
- [Agent Integration Guide](#agent-integration-guide)
- [Roadmap](#roadmap)
- [Contributing](#contributing)
- [License](#license)

---

## Overview

The AI agent economy is arriving — but agents have no native way to pay for things. AgentPay solves this.

**AgentPay is not a wallet. It is the payment middleware layer** — the Stripe for AI agents — where autonomous transactions are authorized by users, executed on-chain, and settled in USDC.

### Core Primitives

| Primitive | What It Does |
|-----------|-------------|
| **PDA Wallet** | Per-user program-derived account that holds USDC. No private key. Only the program can move funds. |
| **Spending Limits** | Per-transaction cap, daily limit, and auto-approve threshold — enforced on-chain, not by the backend. |
| **Agent Registry** | Open on-chain + off-chain registry. Any AI service registers once and becomes a payable endpoint. |
| **execute_payment** | Single Anchor instruction that atomically transfers USDC to the agent and commission to the protocol treasury. |

### Example Flow

A user tells their AI assistant: *"Order me dinner."*

1. AgentPay forwards the intent to the **Food Agent** (e.g. Zomato)
2. Food Agent returns a **quote + basket**
3. AgentPay verifies it against the user's preferences and budget
4. If amount < `auto_approve_threshold` → executes automatically
5. Otherwise → push notification to Flutter app for one-tap approval
6. USDC transfers: `PDA Wallet → Food Agent ATA` (minus 0.5% protocol commission)
7. Food Agent receives a webhook confirming payment → processes the order

---

## Quickstart

> ⚠️ Nothing is implemented yet. These commands are the intended developer experience once the Anchor program exists. Use this section to understand what will be needed.

### Prerequisites

```bash
# Required tools
rustup install stable
sh -c "$(curl -sSfL https://release.solana.com/stable/install)"
cargo install --git https://github.com/coral-xyz/anchor anchor-cli
node --version   # 20+
flutter --version  # 3.x
```

### Build the Anchor Program (devnet)

```bash
git clone https://github.com/YASH514131/agentpay
cd agentpay/program

anchor build
anchor deploy --provider.cluster devnet

# Note the program ID printed — add it to your .env
```

### Environment Variables

```bash
# .env (backend)
HELIUS_RPC_URL=https://rpc.helius.xyz/?api-key=YOUR_KEY
AGENTPAY_PROGRAM_ID=<printed after anchor deploy>
USDC_MINT=EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v  # mainnet
# devnet USDC: 4zMMC9srt5Ri5X14GAgXhaHii3GnPAEERYPJgZJDncDU

BACKEND_KEYPAIR=<base58>   # hot wallet — execute_payment only, no withdraw access
DATABASE_URL=postgresql://user:pass@localhost:5432/agentpay
REDIS_URL=redis://localhost:6379
FCM_SERVER_KEY=<firebase_key>

COMMISSION_BPS=50           # 0.5% — changeable via ProtocolConfig
TREASURY_PUBKEY=<multisig>
```

### Not Yet Implemented

The following steps will be possible once the respective components are built:

```bash
# Backend (not started)
cd ../backend && npm run dev

# Flutter app (not started)
cd ../app && flutter run
```

---

## How It Works

```
User
 └─ tops up ──────────────────────► PDA Wallet (USDC, on-chain)
                                          │
                                          ▼
                                  AgentPay Core (backend)
                                          │
                         ┌────────────────┤
                         │                │
                         ▼                ▼
                   User Prefs DB    Agent Registry
                         │
                         ▼
                   Food Agent API  ◄── intent (preferences + budget)
                         │
                         ▼
                   Quote + Basket  ──► Verify Engine
                                           │
                              ┌────────────┴────────────┐
                              ▼                         ▼
                        auto-approve?            Show quote to user
                         (below threshold)        (Flutter push notification)
                              │                         │
                              └──────────┬──────────────┘
                                         ▼
                                  execute_payment (CPI)
                                  PDA Wallet ──► Agent ATA
                                             ──► Treasury ATA (0.5%)
                                         │
                                         ▼
                                  Webhook ──► Food Agent
                                             (order confirmed)
```

---

## Architecture

### On-Chain Program (Anchor/Rust)

The Anchor program has **4 core accounts** and **6 instructions**. Everything financial is on-chain. Everything relational is in the backend.

#### Accounts

```rust
// PDA: seeds = ["agentpay_wallet", user_pubkey]
#[account]
pub struct UserWallet {
    pub owner:                  Pubkey,  // user's wallet pubkey
    pub usdc_ata:               Pubkey,  // PDA's USDC associated token account
    pub spending_limit_per_tx:  u64,     // max USDC per transaction (6 decimals)
    pub auto_approve_threshold: u64,     // below this → no confirmation needed
    pub daily_limit:            u64,     // rolling 24h cap
    pub daily_spent:            u64,     // resets every epoch
    pub last_reset_ts:          i64,
    pub bump:                   u8,
}

// PDA: seeds = ["agent", agent_authority_pubkey]
#[account]
pub struct AgentRecord {
    pub authority:    Pubkey,     // agent's signing key
    pub usdc_ata:     Pubkey,     // where payments land
    pub name_hash:    [u8; 32],   // sha256(agent_name) — immutable
    pub fee_bps:      u16,        // agent's own service fee (on top of protocol fee)
    pub is_active:    bool,
    pub total_volume: u64,        // lifetime USDC processed
    pub tx_count:     u64,
    pub registered_at:i64,
    pub bump:         u8,
}

// PDA: seeds = ["protocol_config"] — singleton
#[account]
pub struct ProtocolConfig {
    pub authority:      Pubkey,   // AgentPay treasury multisig
    pub treasury_ata:   Pubkey,   // USDC ATA for commissions
    pub commission_bps: u16,      // default: 50 = 0.5%
    pub min_deposit:    u64,      // min SOL deposit for agent registration
    pub paused:         bool,     // emergency circuit breaker
    pub bump:           u8,
}
```

#### Instructions

| Instruction | Caller | Description |
|-------------|--------|-------------|
| `init_user_wallet` | User (first-time) | Creates PDA + USDC ATA, sets default spending limits |
| `update_spending_limits` | User | Updates per-tx cap, daily limit, auto-approve threshold |
| `top_up_wallet` | User | Transfers USDC from user's personal ATA → PDA ATA |
| `withdraw` | User | Pulls USDC from PDA back to personal wallet |
| `register_agent` | Agent operator | Pays SOL deposit, creates `AgentRecord` on-chain |
| `execute_payment` | AgentPay backend | Splits USDC: agent receives `amount - commission`, treasury receives `commission` |

#### execute_payment — Core Logic

```rust
pub fn execute_payment(
    ctx: Context<ExecutePayment>,
    amount: u64,
    order_ref: [u8; 16],  // UUID from backend — prevents replay
) -> Result<()> {
    let config = &ctx.accounts.protocol_config;
    let wallet = &mut ctx.accounts.user_wallet;

    // --- spending limit guards (enforced on-chain, not by backend) ---
    require!(amount <= wallet.spending_limit_per_tx, AgentPayError::ExceedsPerTxLimit);

    let now = Clock::get()?.unix_timestamp;
    if now - wallet.last_reset_ts > 86400 {
        wallet.daily_spent = 0;
        wallet.last_reset_ts = now;
    }
    require!(
        wallet.daily_spent + amount <= wallet.daily_limit,
        AgentPayError::DailyLimitExceeded
    );

    // --- commission split ---
    let commission = (amount as u128)
        .checked_mul(config.commission_bps as u128).unwrap()
        .checked_div(10_000).unwrap() as u64;
    let agent_amount = amount - commission;

    // --- PDA signs its own transfer ---
    let seeds = &[
        b"agentpay_wallet",
        ctx.accounts.user.key().as_ref(),
        &[wallet.bump],
    ];

    // transfer to agent
    token::transfer(
        CpiContext::new_with_signer(ctx.accounts.token_program.to_account_info(),
            Transfer {
                from: ctx.accounts.pda_usdc_ata.to_account_info(),
                to:   ctx.accounts.agent_usdc_ata.to_account_info(),
                authority: ctx.accounts.user_wallet.to_account_info(),
            },
            &[seeds],
        ),
        agent_amount,
    )?;

    // transfer commission to treasury
    token::transfer(
        CpiContext::new_with_signer(/* ... */, &[seeds]),
        commission,
    )?;

    wallet.daily_spent += amount;

    emit!(PaymentExecuted {
        user: ctx.accounts.user.key(),
        agent: ctx.accounts.agent_record.authority,
        amount,
        commission,
        timestamp: now,
        order_ref,
    });

    Ok(())
}
```

---

### Backend Services (Node.js)

#### Stack

| Layer | Technology |
|-------|-----------|
| API Gateway | Kong / AWS API Gateway (rate limiting, JWT auth, TLS) |
| Services | Node.js (Fastify), TypeScript |
| Queue | BullMQ (Redis) → Kafka at scale |
| Primary DB | PostgreSQL |
| Cache | Redis |
| Analytics | ClickHouse |
| Object Storage | S3 (receipts, assets) |
| Blockchain RPC | Helius (dedicated node) |
| TX Indexing | Solana Geyser webhooks + polling fallback |

#### Services

**Payment Orchestrator** — manages the full session lifecycle from intent to on-chain confirmation.

```typescript
export class PaymentOrchestrator {

  // Step 1: Receive intent from Flutter app
  async initiatePayment(dto: InitiatePaymentDto): Promise<PaymentSession> {
    const session = await this.db.createPaymentSession({
      userId:  dto.userId,
      agentId: dto.agentId,
      status:  'PENDING_QUOTE',
    });

    await this.queue.publish('payment.intent', {
      sessionId:     session.id,
      agentEndpoint: dto.agentEndpoint,
      userPrefs:     dto.preferences,
    });

    return session;
  }

  // Step 2: Quote received, decide auto-approve or ask user
  async processQuote(sessionId: string, quote: AgentQuote): Promise<void> {
    const wallet = await this.solana.getUserWalletState(quote.userId);

    if (quote.totalUsdc <= wallet.autoApproveThreshold) {
      return this.executeOnChain(sessionId, quote);  // skip confirmation
    }

    await this.notifications.sendConfirmationRequest(sessionId, quote);
    await this.db.updateSession(sessionId, { status: 'AWAITING_APPROVAL' });
  }

  // Step 3: Build and submit Anchor CPI
  async executeOnChain(sessionId: string, quote: AgentQuote): Promise<string> {
    const tx = await this.anchorProgram.methods
      .executePayment(
        new BN(quote.totalUsdc),
        Array.from(Buffer.from(sessionId.replace(/-/g, ''), 'hex'))
      )
      .accounts({ /* resolved from PDA seeds */ })
      .rpc();

    await this.db.updateSession(sessionId, { status: 'CONFIRMED', txHash: tx });
    await this.webhooks.dispatch(quote.agentId, {
      event:   'payment.confirmed',
      txHash:  tx,
      orderId: sessionId,
    });

    return tx;
  }
}
```

#### Database Schema (Key Tables)

```sql
CREATE TABLE users (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  wallet_pubkey TEXT UNIQUE NOT NULL,
  pda_pubkey    TEXT UNIQUE,
  email         TEXT,
  created_at    TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE agents (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  authority_pubkey TEXT UNIQUE NOT NULL,
  name             TEXT NOT NULL,
  category         TEXT,              -- food | travel | saas | finance | ...
  endpoint_url     TEXT NOT NULL,     -- AgentPay sends intents here
  webhook_url      TEXT,              -- AgentPay sends confirmations here
  rating           NUMERIC(3,2),
  tx_count         BIGINT DEFAULT 0,
  volume_usdc      BIGINT DEFAULT 0,
  is_verified      BOOLEAN DEFAULT FALSE,
  registered_at    TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE payment_sessions (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id      UUID REFERENCES users(id),
  agent_id     UUID REFERENCES agents(id),
  amount_usdc  BIGINT,
  commission   BIGINT,
  status       TEXT,   -- PENDING | QUOTED | AWAITING_APPROVAL | CONFIRMED | FAILED
  tx_hash      TEXT,
  quote_data   JSONB,
  order_ref    UUID,
  created_at   TIMESTAMPTZ DEFAULT NOW(),
  confirmed_at TIMESTAMPTZ
);

-- Performance indexes
CREATE INDEX idx_sessions_user   ON payment_sessions(user_id, created_at DESC);
CREATE INDEX idx_sessions_active ON payment_sessions(status)
  WHERE status NOT IN ('CONFIRMED', 'FAILED');
```

#### TX Listener (Solana Events → PostgreSQL)

```typescript
// Subscribes to on-chain PaymentExecuted events via Helius
const connection = new Connection(process.env.HELIUS_RPC_URL);

connection.onLogs(AGENTPAY_PROGRAM_ID, async (logs) => {
  const event = parsePaymentExecutedEvent(logs);
  if (!event) return;

  await db.paymentSessions.update({
    where: { orderRef: event.orderRef },
    data:  { status: 'CONFIRMED', txHash: event.signature },
  });

  await webhookQueue.add({ agentId: event.agent, payload: event });
}, 'confirmed');
```

---

### Agent Registry

Any AI service can register on AgentPay and become a payable endpoint. The registry is hybrid: core identity and payment address live on-chain; metadata (ratings, endpoint URLs, capabilities) lives in PostgreSQL.

#### Registration Steps

1. Submit registration via the AgentPay developer portal (name, category, Solana pubkey, endpoint URL, webhook URL)
2. Call `register_agent` on-chain — pay a SOL deposit (~0.5 SOL) as an anti-spam bond
3. AgentPay team reviews and sets `is_verified = true` (manual in beta; automated via criteria in v2)
4. Implement the **3 required protocol endpoints** (see below)

#### Required Agent Endpoints

Every registered agent must implement these three HTTP endpoints:

```
POST /agentpay/intent
```
AgentPay sends user preferences and budget. Agent returns a quote.

```json
// Request
{
  "session_id": "uuid",
  "intent_type": "food_order",
  "preferences": { "cuisine": "indian", "max_price_usdc": 5.0 },
  "user_budget_usdc": 8.0
}

// Response
{
  "quote_id": "uuid",
  "items": [...],
  "total_usdc": 4.25,
  "breakdown": { "food": 4.00, "delivery": 0.25 },
  "expires_at": "2025-06-01T10:05:00Z"
}
```

```
POST /agentpay/confirm
```
Called after on-chain confirmation. Agent processes the order.

```json
// Request
{ "session_id": "uuid", "tx_hash": "5xGfk...", "amount_usdc": 4.25 }

// Response
{ "order_id": "ZOM-12345", "eta_minutes": 30 }
```

```
GET /agentpay/health
```
AgentPay polls this to update agent status in the registry.

```json
{ "status": "ok", "version": "1.2.0" }
```

#### User-Facing Registry Features

- Browse agents by category (food, travel, SaaS, finance)
- Sort by rating, transaction volume, or recency
- Set a **default agent per category** (e.g. always use Zomato for food)
- Per-agent spending caps (override global limits)
- Rating submitted post-transaction; aggregate written back on-chain via crank

---

### Flutter Frontend

#### Screen Map

| Screen | Purpose |
|--------|---------|
| Wallet | PDA balance, top-up, withdraw, global spending limit controls |
| Agent Marketplace | Browse and select agents by category, set defaults |
| Pending Approvals | Active sessions needing confirmation — quote breakdown + approve / deny |
| Transaction History | All confirmed payments with agent, amount, Solana Explorer link |
| Settings | Per-agent caps, auto-approve threshold, notification preferences |

#### State Management

```dart
// Riverpod — unidirectional data flow

// PDA balance — polls every 30s
final walletProvider = StateNotifierProvider<WalletNotifier, WalletState>((ref) {
  return WalletNotifier(ref.read(solanaServiceProvider));
});

// Live approval requests — WebSocket stream from backend
final activeSessionsProvider = StreamProvider<List<PaymentSession>>((ref) {
  return ref.read(wsServiceProvider).sessionStream;
});

// Agent list — paginated, cached
final agentListProvider = FutureProvider.family<List<Agent>, AgentFilter>((ref, filter) {
  return ref.read(agentServiceProvider).fetchAgents(filter);
});
```

#### Push Notifications

Payment approval requests are delivered via **Firebase Cloud Messaging (FCM)**. When the backend emits an `AWAITING_APPROVAL` event, the Notification Service calls FCM, which wakes the Flutter app and presents the confirmation UI — even when the app is in the background.

---

## Commission Model

AgentPay charges **50 basis points (0.5%)** on every USDC transaction, deducted atomically inside `execute_payment`. No separate settlement step. No trust required.

### Fee Split

```
User pays: $10.00 USDC
  └─ Agent receives: $9.95
  └─ AgentPay treasury: $0.05
```

### Revenue Projections

| Stage | Monthly Volume | Monthly Revenue |
|-------|---------------|-----------------|
| Beta (100 users) | $10K | ~$50 |
| Early (10K users) | $1M | ~$5,000 |
| Growth (100K users) | $50M | ~$250,000 |
| Scale (1M users) | $2B | ~$10M+ |

### Additional Revenue Streams

- **Agent registration deposit** — one-time SOL bond (~$50 equivalent) per agent
- **Priority placement** — agents pay for featured listing in the marketplace
- **Enterprise API** — monthly flat fee for high-volume agents with SLA guarantees
- **Protocol licensing** — white-label AgentPay rails to other AI platforms

---

## Security Model

> ⚠️ **Not audited.** This protocol has not undergone a security audit. Do not deposit meaningful funds until an audit is completed. Contributions from security researchers are welcome.

### Threat Model

**If the backend hot keypair is compromised:**
The attacker can call `execute_payment` on behalf of any user, but only up to each user's `spending_limit_per_tx` per transaction and `daily_limit` per 24h window. Both limits are enforced on-chain — the backend cannot override them. The attacker cannot call `withdraw`.

**If an agent endpoint is malicious:**
A malicious agent can return an inflated quote. The Verify Engine checks the quot

Example : Here is the intended flow for integrating  SpiceJet AI agent with AgentPay. 

*Note: AgentPay is currently in the design and architecture phase, meaning no code has been written yet [1]. This outlines the exact flow designed for when the protocol is built.*

### **Phase 1: Registering the SpiceJet Agent**
To get your SpiceJet AI agent working with AgentPay, you first need to register it so it becomes a payable endpoint in the AgentPay registry [2, 3].

1. **Submit Details:** Register your agent through the AgentPay developer portal by providing your agent's name, category (e.g., Travel), Solana public key, endpoint URL, and webhook URL [3].
2. **Pay the Deposit:** Call the `register_agent` instruction on-chain and pay an anti-spam bond of approximately 0.5 SOL [3].
3. **Verification:** The AgentPay team will manually review your submission and set your status to verified [3].
4. **Implement Required Endpoints:** Your SpiceJet backend must implement three specific HTTP endpoints to communicate with AgentPay [3, 4]:
   * **Quote Endpoint:** AgentPay sends you user preferences and their budget; you return a flight quote [4].
   * **Fulfillment Webhook:** Called by AgentPay after the on-chain payment is confirmed so you can finalize the booking [4].
   * **Status Endpoint:** AgentPay polls this to update your agent's operational status in their registry [4].

### **Phase 2: The Customer Booking Flow**
Once your SpiceJet agent is integrated, here is how a customer's transaction would flow from request to final settlement [5]:

1. **User Intent:** A user tells their AI assistant, *"Book me a SpiceJet flight."* [5]
2. **Quote Generation:** AgentPay forwards the intent to your SpiceJet Agent. Your system calculates the fare and returns a **quote and basket** to AgentPay [5].
3. **Budget Verification:** AgentPay verifies your quote against the user's set preferences and budget limits [5].
4. **User Approval:** 
   * If the flight cost is **under the user's auto-approve threshold**, the payment is executed automatically [5].
   * If the cost is **over the threshold**, AgentPay sends a push notification to the user's Flutter app for a one-tap approval [5].
5. **On-Chain Payment:** Once approved, the `execute_payment` instruction runs on the Solana blockchain. The USDC funds are transferred instantly from the user's automated PDA wallet to your SpiceJet wallet [2, 5]. *Note: AgentPay automatically deducts a 0.5% protocol commission during this transfer [5, 6].*
6. **Ticket Issuance:** Your Fulfillment Webhook receives confirmation that the payment succeeded, and your SpiceJet agent processes the order and issues the ticket to the user [5].
