# ⚡ Manifest AI Commerce Engine

> **Production-grade AI-to-AI (A2A) e-commerce merchant platform built on Razorpay Test Mode APIs, deterministic cryptographic authorization, and tamper-evident WORM audit anchoring.**

[![Build](https://img.shields.io/badge/Build-Passing-brightgreen?style=flat-square&logo=typescript)](./package.json)
[![Tests](https://img.shields.io/badge/Tests-67%2F67%20Passing%20(100%25)-brightgreen?style=flat-square&logo=vitest)](./tests)
[![Node](https://img.shields.io/badge/Node-v20%2B-blue?style=flat-square&logo=node.js)](https://nodejs.org)
[![Fastify](https://img.shields.io/badge/Server-Fastify%205.x-black?style=flat-square&logo=fastify)](https://fastify.dev)
[![PostgreSQL](https://img.shields.io/badge/Database-PostgreSQL%2014%2B-blue?style=flat-square&logo=postgresql)](https://www.postgresql.org)
[![Security](https://img.shields.io/badge/Audit-KMS%20WORM%20Immutable-orange?style=flat-square&logo=amazon-aws)](./src/audit)
[![Q&A](https://img.shields.io/badge/Q%26A-Architecture%20Deep%20Dive-blueviolet?style=flat-square)](./QA.md)
[![License](https://img.shields.io/badge/License-MIT-green?style=flat-square)](./LICENSE)

---

## 📖 Table of Contents

1. [Executive Overview](#-executive-overview)
2. [System Architecture](#-system-architecture)
3. [Core Architectural Pillars & Invariants](#-core-architectural-pillars--invariants)
   - [1. M2M Capability Discovery](#1-machine-to-machine-capability-discovery)
   - [2. Buyer AI ↔ Seller AI Communication Protocol](#2-buyer-ai--seller-ai-communication-protocol)
   - [3. Dual-Rail Payment Architecture](#3-dual-rail-payment-architecture)
   - [4. Authentic Razorpay Webhook Provenance & Direct API Verification](#4-authentic-razorpay-webhook-provenance--direct-api-verification)
   - [5. Webhook Secret Rotation with Dual-Verification Grace Period](#5-webhook-secret-rotation-with-dual-verification-grace-period)
   - [6. WebAuthn Passkey Assertion Ceremony & Step-Up Lifecycle](#6-webauthn-passkey-assertion-ceremony--step-up-lifecycle)
   - [7. Cryptographic Delegations & Monotonic Narrowing Rule](#7-cryptographic-delegations--monotonic-narrowing-rule)
   - [8. Five Deterministic State Machines](#8-five-deterministic-state-machines)
   - [9. Atomic Inventory Reservation & DB Invariants](#9-atomic-inventory-reservation--db-invariants)
   - [10. Multi-Tier Sybil Hoarding Defense](#10-multi-tier-sybil-hoarding-defense)
   - [11. Transactional Outbox & Audit Atomicity Monitor](#11-transactional-outbox--audit-atomicity-monitor)
   - [12. KMS-Signed WORM Storage Anchoring & DB Compromise Detection](#12-kms-signed-worm-storage-anchoring--db-compromise-detection)
   - [13. Fulfilment Exceptions, Auto-Refunds & Uncertainty Reconciliation](#13-fulfilment-exceptions-auto-refunds--uncertainty-reconciliation)
   - [14. Fixed-Point Basis-Point Financial Arithmetic](#14-fixed-point-basis-point-financial-arithmetic)
4. [State Machine Specifications](#-state-machine-specifications)
5. [Interactive Merchant Control Tower Dashboard](#-interactive-merchant-control-tower-dashboard)
6. [API & Protocol Reference](#-api--protocol-reference)
7. [Directory Structure](#-directory-structure)
8. [Prerequisites & Setup](#-prerequisites--setup)
9. [Execution & Demonstration Commands](#-execution--demonstration-commands)
10. [Test Suite Verification (67/67 Invariants)](#-test-suite-verification-6767-invariants)
11. [Security, Governance & Threat Model](#-security-governance--threat-model)
12. [Architecture, Security & Economics Q&A (QA.md)](./QA.md)

---

## 🌟 Executive Overview

**Manifest** is an autonomous AI-to-AI e-commerce merchant platform. When autonomous buyer agents interact with merchants on the open web, traditional e-commerce backends break due to:
- Prompt injection and unauthorized autonomous spending.
- Inventory exhaustion caused by multi-agent Sybil hoarding attacks.
- Non-atomic commerce mutations creating un-auditable "ghost" states.
- Webhook race conditions, out-of-order deliveries, and floating-point currency rounding drifts.
- Database compromises silently rewriting transaction ledgers.

Manifest resolves these challenges by introducing a **deterministic execution layer** below the AI agents. Agents handle natural language, search, negotiation, and intent interpretation, but **all pricing, stock reservations, policy enforcement, payment capture, state transitions, and audit records are executed strictly by deterministic, mathematically verifiable engines.**

```
┌────────────────────────────────────────────────────────────────────────┐
│                        AI REASONING LAYER                              │
│   Buyer Agent (Planner / LLM)  ◄───►  Seller Agent Façade (Interpreter) │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ Structured Protocol Intent
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│                     DETERMINISTIC MANIFEST ENGINES                     │
│  ┌──────────────────────┐ ┌──────────────────────┐ ┌─────────────────┐ │
│  │ Capability Discovery │ │ Deterministic Policy │ │ Fixed-Point Math│ │
│  └──────────────────────┘ └──────────────────────┘ └─────────────────┘ │
│  ┌──────────────────────┐ ┌──────────────────────┐ ┌─────────────────┐ │
│  │ Row-Locked Inventory │ │ 5 Legal State Mach.  │ │ Sybil Risk Agg. │ │
│  └──────────────────────┘ └──────────────────────┘ └─────────────────┘ │
│  ┌──────────────────────┐ ┌──────────────────────┐ ┌─────────────────┐ │
│  │ Razorpay Webhooks    │ │ Transactional Outbox │ │ KMS WORM Ledger │ │
│  └──────────────────────┘ └──────────────────────┘ └─────────────────┘ │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ ACID Atomic Mutation
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│                     PERSISTENCE & IMMUTABLE ANCHORS                    │
│   PostgreSQL Kernel (ACID)  ◄───►  AWS KMS + Immutable S3 WORM Vault   │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 🏛️ System Architecture

Manifest is structured into modular, decoupled subsystems:

```
                    ┌─────────────────────────┐
                    │  GET /v1/capabilities   │ ◄── Zero-Prior-Knowledge Discovery
                    └───────────┬─────────────┘
                                │
                    ┌───────────▼─────────────┐
                    │    Seller Agent Façade   │ ◄── A2A Interpretation & Dialogue
                    └───────────┬─────────────┘
                                │
     ┌──────────────────────────┼──────────────────────────┐
     ▼                          ▼                          ▼
┌──────────────┐      ┌──────────────────┐       ┌──────────────────┐
│ Catalog &    │      │ Policy & Mandate │       │ Row-Locked Stock │
│ Negotiation  │      │ Validator (P0.3) │       │ Engine (P0.4)    │
└──────┬───────┘      └─────────┬────────┘       └────────┬─────────┘
       │                        │                         │
       └────────────────────────┼─────────────────────────┘
                                │
                     ┌──────────▼──────────┐
                     │   Checkout Service  │
                     └──────────┬──────────┘
                                │
          ┌─────────────────────┴─────────────────────┐
          │                                           │
  [Autonomous Path]                           [Interactive Path]
  Ed25519 Mandate Verified                    Standard Razorpay Rail
  Payment Rail: MOCK_SIMULATED_RAIL           Payment Rail: RAZORPAY_STANDARD
  Environment: TEST_ONLY                      Status: REQUIRES_USER_ACTION
  Status: AUTO_EXECUTABLE                     Redirect: Hosted Checkout URL
          │                                           │
          │                                           │ Customer Completes Payment
          │                                           ▼
          │                           ┌────────────────────────────────┐
          │                           │ Razorpay Webhook Ingestion     │
          │                           │ - Constant-time HMAC-SHA256    │
          │                           │ - x-razorpay-event-id Dedup    │
          │                           │ - Direct REST API Verification │
          │                           └───────────────┬────────────────┘
          │                                           │
          └─────────────────────┬─────────────────────┘
                                │
                     ┌──────────▼──────────┐
                     │ Transactional Outbox│ (Atomic PG Transaction)
                     └──────────┬──────────┘
                                │ Worker Drain
                     ┌──────────▼──────────┐
                     │ Merkle Hash Chain   │ (RFC 8785 Canonical JSON)
                     └──────────┬──────────┘
                                │ Checkpoint Trigger
                     ┌──────────▼──────────┐
                     │ AWS KMS Ed25519 Sign│ ➔ Immutable WORM Object Store
                     └─────────────────────┘
```

---

## 🛡️ Core Architectural Pillars & Invariants

### 1. Machine-to-Machine Capability Discovery
Before attempting any transaction, an external Buyer Agent with zero prior knowledge performs protocol discovery via `GET /v1/capabilities`. The merchant returns a machine-readable schema:
```json
{
  "search": true,
  "negotiation": true,
  "payment_rails": {
    "razorpay_standard": "INTERACTIVE",
    "simulated_rail": "TEST_ONLY"
  },
  "mandates": {
    "manifest_signed": "SUPPORTED",
    "ap2": "EXPERIMENTAL"
  }
}
```
This guarantees strict transparency: the buyer agent knows upfront that Razorpay rails require interactive step-up authentication, and that autonomous zero-human purchases run via cryptographically signed purchase mandates on simulated test rails.

### 2. Buyer AI ↔ Seller AI Communication Protocol
Transactions execute over an explicit multi-agent dialogue protocol:
- **Reference Buyer Agent**: Decomposes user intent into goals, performs search, evaluates items, requests price negotiations, reserves stock, signs mandates, and verifies cryptographic provenance.
- **Seller Agent Façade**: Serves as the merchant's machine interface. It handles language interpretation, dialogue formulation, and policy explanations, but **delegates 100% of decisions to deterministic backend engines.**
- **Strict Margin Confidentiality**: Internal gross margin floors (e.g. `1200 bps`) and COGS remain private. Buyer-Agent explanations return:
  > `"Offer satisfies the merchant's private pricing and promotion policies. Internal margin thresholds are confidential."`

### 3. Dual-Rail Payment Architecture
Manifest maintains an uncompromised distinction between payment rails:
1. **Autonomous Simulated Rail (`MOCK_SIMULATED_RAIL`)**:
   - Execution Mode: `AUTO_EXECUTABLE`
   - Authorization: Verified Ed25519 Purchase Mandate Token
   - Environment: `TEST_ONLY` (`Funds Moved: FALSE`, `Fulfilment Allowed: FALSE`)
2. **Live Interactive Razorpay Rail (`RAZORPAY_STANDARD`)**:
   - Execution Mode: `REQUIRES_USER_ACTION`
   - Authorization: Requires Human OTP/UPI Interactive Step-Up
   - Environment: Razorpay Test Mode API (`/checkout/hosted/:id`)

### 4. Authentic Razorpay Webhook Provenance & Direct API Verification
To prevent replay and local forgery attacks:
- Webhooks are received over dedicated HTTPS endpoints.
- Verifies constant-time raw body `HMAC-SHA256` signatures against active secrets.
- Deduplicates using the authoritative provider `x-razorpay-event-id` header.
- **Direct Provider REST API Verification**: Before committing status changes, Manifest actively queries the Razorpay API (`GET /v1/payments/{payment_id}`) to cross-verify `payment_id`, `order_id`, `amount`, `currency`, and `captured` state against provider records, emitting `networkProvenance: 'VERIFIED'`.

### 5. Webhook Secret Rotation with Dual-Verification Grace Period
Merchants rotate webhook secrets periodically. In-flight retries of older webhook events must not fail during rotation. Manifest implements:
- Dual-verification algorithm: Verifies incoming signature against the `PRIMARY` secret; on mismatch, falls back to the `PREVIOUS_ACTIVE` secret if within the `valid_until` retry horizon.
- Zero plaintext secrets in PostgreSQL: Secrets are referenced via KMS/Secret Manager handles (`secret_handle`).

### 6. WebAuthn Passkey Assertion Ceremony & Step-Up Lifecycle
High-value human-to-agent delegations require authentic biometric / hardware security key authentication:
1. **Asymmetric Ceremony**: Backend generates a cryptographic challenge (300s TTL). Client authenticator produces an Ed25519/ECDSA signature over `authenticatorData || SHA-256(clientDataJSON)`.
2. **Flags Inspection**: Validates User Presence (`UP` = `0x01`) and User Verification (`UV` = `0x04` biometric/PIN).
3. **Session Issuance**: Upon successful verification, issues a short-lived (5-minute) `STEP_UP_VERIFIED` session token.

### 7. Cryptographic Delegations & Monotonic Narrowing Rule
High-level human principals delegate bounded spending power to autonomous agents:
- **Monotonic Narrowing (`validateDelegationSubset`)**: A child mandate or sub-delegation can never exceed the parent envelope:
  $$\text{Amount}_{\text{child}} \le \text{Amount}_{\text{parent}}, \quad \text{Expiry}_{\text{child}} \le \text{Expiry}_{\text{parent}}, \quad \text{Categories}_{\text{child}} \subseteq \text{Categories}_{\text{parent}}$$
- **Delegation Revocation Cascade**: Revoking a parent delegation immediately invalidates all downstream child mandates in real-time.
- **Dynamic Version-Bound Budget Constraints**: Reducing a delegation ceiling (e.g. ₹10,000 $\rightarrow$ ₹5,000) dynamically constrains existing child mandates.
- **Self-Delegation Defense**: Agents attempting self-delegation are rejected with `403 AGENT_CANNOT_SELF_DELEGATE`.

### 8. Five Deterministic State Machines
Manifest strictly partitions domain lifecycles into five non-overlapping state machines:
1. `CheckoutStatus`: `OPEN` $\rightarrow$ `AWAITING_AUTH` $\rightarrow$ `PAID` / `CANCELLED` / `PAYMENT_ATTEMPT_FAILED`
2. `ReservationStatus`: `ACTIVE` $\rightarrow$ `COMMITTED` / `RELEASED` / `EXPIRED`
3. `PaymentStatus`: `PENDING` $\rightarrow$ `AUTHORIZED` $\rightarrow$ `CAPTURED` $\rightarrow$ `REFUNDED` / `PARTIALLY_REFUNDED`
4. `OrderStatus`: `PENDING_CONFIRMATION` $\rightarrow$ `CONFIRMED` $\rightarrow$ `PROCESSING` $\rightarrow$ `SHIPPED` $\rightarrow$ `DELIVERED` (or exception: `FULFILMENT_EXCEPTION`)
5. `RefundStatus`: `PENDING` $\rightarrow$ `PROCESSED` / `FAILED` / `REFUND_UNCERTAIN`

### 9. Atomic Inventory Reservation & DB Invariants
- **Row-Lock Decoupling**: Database row locks (`SELECT ... FOR UPDATE`) are held for $<15\text{ms}$ during reservation creation, while reserving inventory hold TTL for 10 minutes.
- **Canonical Lock-Order Deadlock Prevention**: Multi-SKU reservation lock-order deadlocks prevented through canonical ascending variant UUID sorting (`reserveMultiStock`).
- **Database-Level Invariants**: Enforced by PostgreSQL constraints:
  ```sql
  CHECK (reserved_qty <= stock_qty),
  CHECK (stock_qty >= 0),
  CHECK (reserved_qty >= 0)
  ```
- **State Transition Idempotency**: Transition `ACTIVE` $\rightarrow$ `RELEASED` is idempotent; duplicate calls cannot double-decrement reserved counters.

### 10. Multi-Tier Sybil Hoarding Defense
Protects scarce/hot SKUs against distributed botnets and multi-agent hoarding:
- **`DEFINITE_LINK` (High Confidence)**: Linked by Human Principal ID, Account/Payment Hash, or Agent ID $\rightarrow$ Strict limit of **1 active reservation per linked identity cluster per SKU**.
- **`PROBABILISTIC_LINK` (Medium Confidence)**: Linked by Subnet cluster, Device fingerprint, or Shipping address $\rightarrow$ Triggers risk score elevation and step-up challenge.

### 11. Transactional Outbox & Audit Atomicity Monitor
Prevents un-audited "ghost" state mutations if a node crashes after committing database state:
- **Atomic Insertion**: Every business state mutation and its corresponding `audit_outbox` entry are committed inside the exact same PostgreSQL ACID transaction.
- **Sequential Outbox Worker**: Asynchronously drains outbox events into the hash-chained Merkle ledger.
- **Atomicity Invariant Monitor**: Background auditor continuously verifies that every order, payment, reservation, and refund has an unbroken audit trace (`verifyCommerceAuditAtomicity()`).

### 12. KMS-Signed WORM Storage Anchoring & DB Compromise Detection
- **Cryptographic Hash Chain**: Every event is hashed with SHA-256 over RFC 8785 canonical JSON, linking to the previous event hash.
- **KMS Ed25519 Signing**: Audit checkpoints are signed with AWS KMS key (`alias/manifest-audit-checkpoint-signer-v2`).
- **Immutable WORM Vault**: Anchored to external Write-Once-Read-Many object storage (`s3://.../checkpoints/`) under configurable retention policies:
  - `STANDARD_AUDIT_1_YEAR`
  - `PAYMENT_PCI_3_YEARS`
  - `COMPLIANCE_MODE_7_YEARS` (Object Lock Legal Hold active)
  - `LEGAL_HOLD_INDEFINITE`
- **60-Second Detection SLO**: Daemon verifies local PostgreSQL against external WORM storage every $\le 60\text{s}$. Direct database tampering is immediately detected and flagged.

### 13. Fulfilment Exceptions, Auto-Refunds & Uncertainty Reconciliation
- **Late `AUTHORIZED` Webhook Handling**: Late-arriving webhooks on expired reservations check available stock; if exhausted, the engine flags `DO_NOT_CAPTURE` and triggers an auto-refund.
- **Post-Capture Fulfilment Exceptions**: If payment captures after inventory is exhausted, Order enters `FULFILMENT_EXCEPTION`, Refund enters `REQUIRED` $\rightarrow$ `PROCESSED`, and Payment resolves to `REFUNDED`.
- **Refund Uncertainty Reconciliation**: Network timeouts transition refunds to `REFUND_UNCERTAIN`, triggering autonomous reconciliation with exponential backoff and jitter.

### 14. Fixed-Point Basis-Point Financial Arithmetic
All financial calculations use pure integer basis-point math with deterministic `roundDivision(numerator, denominator)` (`ROUND_HALF_UP`), eliminating floating-point binary rounding errors (e.g. $0.1 + 0.2 \neq 0.3$).

---

## 🔄 State Machine Specifications

```
  [CheckoutStatus]
       ┌──────────────┐
       │     OPEN     │
       └──────┬───────┘
              │ Start Checkout
       ┌──────▼───────┐
  ┌───►│AWAITING_AUTH ├──────────────┐
  │    └──────┬───────┘              │
  │ Retry     │ Success              │ Decline
  │    ┌──────▼───────┐       ┌──────▼──────────────┐
  └────┤     PAID     │       │PAYMENT_ATTEMPT_FAILED
       └──────────────┘       └─────────────────────┘

  [ReservationStatus]
       ┌──────────────┐
       │    ACTIVE    ├──────────────┐ Expire TTL
       └──────┬───────┘              │
              │                      ▼
       ┌──────┴───────┐       ┌──────────────┐
       │              │       │   EXPIRED    │
       ▼              ▼       └──────────────┘
  ┌──────────┐  ┌──────────┐
  │COMMITTED │  │ RELEASED │
  └──────────┘  └──────────┘

  [PaymentStatus]
       ┌──────────────┐
       │   PENDING    │
       └──────┬───────┘
              │ Provider Auth
       ┌──────▼───────┐
       │  AUTHORIZED  │
       └──────┬───────┘
              │ Provider Capture
       ┌──────▼───────┐
       │   CAPTURED   ├──────────────┐ Full Refund
       └──────┬───────┘              │
              │ Partial Refund       ▼
              ▼               ┌──────────────┐
       ┌──────────────┐       │   REFUNDED   │
       │PARTIALLY_REF.│       └──────────────┘
       └──────────────┘

  [OrderStatus]
       ┌────────────────────┐
       │PENDING_CONFIRMATION│
       └─────────┬──────────┘
                 │ Payment Captured (Stock Valid)
       ┌─────────▼──────────┐
       │     CONFIRMED      ├──────────────┐ Fulfilment Failure
       └─────────┬──────────┘              │
                 │ Fulfilment Dispatch     ▼
       ┌─────────▼──────────┐       ┌────────────────────┐
       │     PROCESSING     │       │FULFILMENT_EXCEPTION│
       └─────────┬──────────┘       └────────────────────┘
                 │ Shipped
       ┌─────────▼──────────┐
       │     SHIPPED        │
       └─────────┬──────────┘
                 │ Delivered
       ┌─────────▼──────────┐
       │    DELIVERED       │
       └────────────────────┘

  [RefundStatus]
       ┌──────────────┐
       │   REQUIRED   │
       └──────┬───────┘
              │ Submit to Provider
       ┌──────▼───────┐
       │  SUBMITTED   │
       └──────┬───────┘
              │ Provider Processing
       ┌──────▼───────┐
       │  PROCESSING  ├──────────────┐ Timeout
       └──────┬───────┘              │
              │ Confirmed            ▼
       ┌──────▼───────┐       ┌──────────────┐
       │  PROCESSED   │       │REFUND_UNCERT.│
       └──────────────┘       └──────────────┘
```

---

## 🎛️ Interactive Merchant Control Tower Dashboard

Manifest features a real-time visual storytelling Control Tower UI accessible at `http://localhost:3000`:
- **Live Visual Transaction Pipeline (6 Stages)**:
  - **Stage 1: A2A Protocol**: Visualizes Buyer Agent intent matching Seller Agent Façade with protected, confidential economics (`🔒 REDACTED`).
  - **Stage 2: Deterministic Policy Gate**: Decomposes `[AUTHORIZATION]`, `[PAYMENT CAPABILITY]`, `[RISK]`, and `[EXECUTION PLAN]`.
  - **Stage 3: Atomic Inventory Hold**: Projects real-time row-lock duration ($<15\text{ms}$ released) and 10-minute hold TTL countdown with Sybil link confidence.
  - **Stage 4: Payment & Rail (Rail-Aware)**: Dynamically distinguishes `MOCK_SIMULATED_RAIL` (Autonomous `TEST_ONLY` Mandates) from `RAZORPAY_STANDARD` (Authentic HMAC & REST API Match).
  - **Stage 5: Commerce State Resolution**: Visualizes independent state machine transitions across Checkout, Reservation, Payment, Order, and Refund lifecycles.
  - **Stage 6: Audit, Provenance & Immutable Anchor**: Displays RFC 8785 canonical hash-chaining, AWS KMS Ed25519 root signatures, and S3 WORM Object-Lock verification.
- **Persistent System Invariants Strip**: Live real-time counters asserting `OVERSELLS: 0 ✓`, `POLICY ESCAPES: 0 ✓`, `AUDIT FORKS: 0 ✓`, and `LEAKS: 0 ✓`.
- **5 Scenario Playbooks (Real API-Driven)**:
  1. *Autonomous Manifest Mandate* (`TEST-ONLY Delegated Payment`, `AP2-inspired`)
  2. *Razorpay Interactive Checkout* (Default judge playbook: Interactive Step-Up + Live Webhook Verification)
  3. *Payment Failure ➔ Retry ➔ Recovery* (Stock hold preservation during decline $\rightarrow$ UPI Intent retry)
  4. *Captured Payment ➔ Fulfilment Exception ➔ Compensation* (Hold expiration $\rightarrow$ sub-second automated refund initiation)
  5. *Adversarial Agent Attack* (Prompt injection & margin probing strictly blocked by policy)
- **Progressive Disclosure Technical Inspectors**: Click any visual stage to inspect raw cryptographic Merkle proofs, HMAC calculations, KMS signatures, and Transactional Outbox atomicity logs.

---

## 📡 API & Protocol Reference

### Merchant HTTP REST Endpoints

| Method | Endpoint | Authentication | Description |
| :--- | :--- | :--- | :--- |
| `GET` | `/v1/capabilities` | Public | Machine-discoverable capability profile |
| `GET` | `/v1/catalog/products` | Public / Agent | Search catalog with structured budget and category filters |
| `GET` | `/v1/catalog/products/:id` | Public / Agent | Retrieve SKU variant specifications and inventory state |
| `POST` | `/v1/cart` | Agent JWT / Gateway | Initialize secure shopping cart |
| `POST` | `/v1/cart/items` | Agent JWT / Gateway | Atomically reserve inventory with 10-minute hold TTL |
| `POST` | `/v1/cart/negotiate` | Agent JWT / Gateway | Request policy-bounded dynamic price negotiation |
| `POST` | `/v1/checkout/start` | Agent JWT / Mandate | Start checkout (Autonomous Mandate or Razorpay Session) |
| `GET` | `/v1/orders/:id` | Agent JWT / Principal | Query order status and Merkle provenance receipt |
| `POST` | `/v1/webhooks/razorpay` | HMAC-SHA256 Signature | Authoritative Razorpay network webhook ingestion endpoint |
| `POST` | `/v1/auth/webauthn/challenge` | Session | Generate WebAuthn FIDO2 challenge |
| `POST` | `/v1/auth/webauthn/verify` | Passkey Signature | Verify Passkey assertion and issue `STEP_UP_VERIFIED` token |
| `POST` | `/v1/delegations` | Human Step-Up JWT | Create or update monotonic human-to-agent delegation grant |
| `DELETE`| `/v1/delegations/:id` | Human Step-Up JWT | Revoke delegation grant (triggers downstream mandate cascade) |
| `GET` | `/v1/audit/verify` | Merchant Admin | Run cryptographic hash-chain and WORM cross-verification |

### Replay-Resistant Signed Gateway Protocol Headers
Internal microservices and API gateways authenticate using signed envelopes:
- `X-Gateway-Timestamp`: Unix epoch milliseconds (validated within 60s freshness window)
- `X-Gateway-Nonce`: Unique UUIDv4 (tracked in memory/cache to prevent replay attacks)
- `X-Gateway-Signature`: `HMAC-SHA256(secret, timestamp + "." + nonce + "." + method + "." + path + "." + sha256(body))`

---

## 📂 Directory Structure

```
.
├── src/
│   ├── agent/                 # AI-to-AI multi-agent layer
│   │   ├── capabilities.ts    # GET /v1/capabilities discovery schema
│   │   ├── seller-agent.ts    # Seller Agent Façade (Dialogue & Interpreter)
│   │   ├── buyer-agent.ts     # Reference Buyer Agent Planner
│   │   └── llm-router.ts      # LLM & Deterministic tool router
│   ├── audit/                 # Cryptographic ledger & WORM storage
│   │   ├── hash-chain.ts      # RFC 8785 Canonical JSON SHA-256 Ledger
│   │   ├── merkle.ts          # Merkle tree provenance proofs
│   │   ├── anchoring.ts       # AWS KMS Ed25519 signing & S3 WORM vault
│   │   └── outbox.ts          # Transactional Outbox & Invariant Monitor
│   ├── cart/                  # Cart lifecycle management
│   ├── catalog/               # Product catalog & SKU management
│   ├── checkout/              # Checkout orchestration service
│   ├── db/                    # Database migrations, schema & seeders
│   │   ├── schema.sql         # PostgreSQL schema with DB-level constraints
│   │   ├── migrate.ts         # Database migration runner
│   │   └── seed.ts            # 30+ SKU catalog and test mandate seeder
│   ├── inventory/             # Inventory reservation & concurrency
│   │   └── reservation.ts     # Row locking, Sybil risk & DB invariants
│   ├── middleware/            # Fastify middleware (Auth, Gateway, Security)
│   │   └── auth.ts            # Replay-resistant gateway verification & JWT auth
│   ├── negotiation/           # Policy-bounded price negotiation engine
│   ├── payments/              # Payment gateways & reconciliation
│   │   ├── adapters/          # Razorpay Test Mode API adapter
│   │   ├── refunds.ts         # Automated compensation refunds
│   │   └── reconciliation.ts  # Autonomous refund reconciliation engine
│   ├── policy/                # Deterministic authorization layer
│   │   ├── policy-engine.ts   # Mandate, budget & scope policy evaluator
│   │   ├── delegation.ts      # Human delegation & monotonic narrowing
│   │   ├── mandates.ts        # Ed25519 purchase mandate token lifecycle
│   │   ├── webauthn.ts        # FIDO2 Passkey assertion ceremony engine
│   │   └── state-machines.ts  # Five deterministic lifecycle state machines
│   ├── pricing/               # Fixed-point basis-point arithmetic
│   ├── scripts/               # End-to-end demo & evaluation scripts
│   │   ├── run-demo-e2e.ts    # Full 5-Act E2E Live Demonstration
│   │   └── run-buyer-agent.ts # Standalone Autonomous Buyer Agent script
│   ├── webhooks/              # Razorpay webhook ingestion & rotation
│   │   ├── razorpay.ts        # Webhook processor & direct API verification
│   │   └── secrets.ts         # Webhook secret rotation & grace period
│   └── server.ts              # Fastify HTTP server entry point
├── dashboard/                 # Control Tower Web UI (HTML/CSS/JS)
├── tests/                     # 67 Automated Invariant Tests (Vitest)
│   ├── audit/                 # Hash-chain, Outbox atomicity & WORM tests
│   ├── concurrency/           # Reservation race conditions & Sybil tests
│   ├── policy/                # Policy engine, delegation & state machine tests
│   ├── pricing/               # Pure basis-point financial math tests
│   ├── security/              # WebAuthn, Gateway replay & privilege tests
│   └── webhooks/              # Webhook HMAC, API verification & rotation tests
├── package.json
├── tsconfig.json
└── walkthrough.md             # Complete architectural walkthrough & logs
```

---

## ⚙️ Prerequisites & Setup

### Requirements
- **Node.js**: `v20.0.0` or higher
- **PostgreSQL**: `v14` or higher (running locally or via Docker)
- **Package Manager**: `npm`

### 1. Clone & Install Dependencies
```bash
git clone https://github.com/your-org/manifest-ai-commerce.git
cd manifest-ai-commerce
npm install
```

### 2. Configure Environment Variables
Copy `.env.example` to `.env` and adjust database credentials and Razorpay test keys:
```bash
cp .env.example .env
```

Example `.env` configuration:
```ini
PORT=3000
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/manifest_db
RAZORPAY_KEY_ID=rzp_test_YourKeyIdHere
RAZORPAY_KEY_SECRET=YourKeySecretHere
RAZORPAY_WEBHOOK_SECRET=demo_webhook_secret_abcde
GATEWAY_SHARED_SECRET=manifest_internal_gateway_secret_2026
JWT_SECRET=manifest_jwt_super_secret_signing_key_2026
NODE_ENV=development
```

### 3. Initialize Database Schema & Seed Data
```bash
# Run PostgreSQL schema migrations
npm run db:migrate

# Seed 30+ SKUs across categories, AP2-inspired mandate tokens, and test accounts
npm run db:seed
```

---

## 🚀 Execution & Demonstration Commands

### 1. Run Complete E2E Live Demonstration (Acts 0 - 4)
Executes the full automated multi-agent scenario across all 5 acts:
```bash
npm run demo:e2e
```
**Demonstration Sequence**:
- **Act 0**: Human WebAuthn Passkey Ceremony $\rightarrow$ Step-Up Session $\rightarrow$ M2M Capability Discovery (`GET /v1/capabilities`).
- **Act 1**: Buyer AI $\leftrightarrow$ Seller AI Autonomous Purchase Flow (`MOCK_SIMULATED_RAIL`, `TEST_ONLY`).
- **Act 2**: Standard Interactive Razorpay Checkout (`REQUIRES_USER_ACTION`).
- **Act 2A**: Authentic Razorpay Webhook Ingestion & Provenance Cross-Verification (`GET /v1/payments/{id}`).
- **Act 3A**: Graceful Payment Failure $\rightarrow$ Stock Preservation $\rightarrow$ Alternate Rail Recovery.
- **Act 3B**: Post-Capture Fulfilment Exception $\rightarrow$ Automated Compensation Refund.
- **Act 4**: Cryptographic Hash-Chain Verification $\rightarrow$ AWS KMS Ed25519 WORM Anchoring $\rightarrow$ Outbox Atomicity Monitor.

### 2. Run Standalone Autonomous Buyer Agent
```bash
npm run demo:agent
```

### 3. Start Development Server with Merchant Control Tower
```bash
npm run dev
```
Open **`http://localhost:3000`** in your browser to view the Control Tower.

### 4. Production Build & Start
```bash
npm run build
npm run start
```

---

## 🧪 Test Suite Verification (67/67 Invariants)

Run all 67 deterministic invariant tests across all 6 suites:
```bash
npm test
```

### Test Suite Summary

```text
 ✓ tests/policy/policy_engine.test.ts (16 tests)
 ✓ tests/concurrency/reservation_race.test.ts (11 tests)
 ✓ tests/audit/audit_integrity.test.ts (8 tests)
 ✓ tests/security/adversarial_attacks.test.ts (12 tests)
 ✓ tests/webhooks/webhook_verification.test.ts (15 tests)
 ✓ tests/pricing/money_rounding.test.ts (5 tests)

 Test Files  6 passed (6)
      Tests  67 passed (67)
   Duration  4.18s
```

### Target Invariant Test Suites
- **Concurrency & Invariants** (`tests/concurrency/reservation_race.test.ts`):
  - 20 concurrent requests competing for 1 unit of stock yields exactly 1 winner and 19 rejections.
  - Multi-SKU canonical UUID lock-order deadlock prevention.
  - Sybil anti-hoarding multi-agent cluster blocking.
  - Database row lock durations $<15\text{ms}$.
- **Webhooks & Rotation** (`tests/webhooks/webhook_verification.test.ts`):
  - Constant-time raw HMAC-SHA256 verification.
  - Webhook secret rotation with primary and previous active grace periods.
  - Provider REST API cross-verification.
  - Out-of-order monotonic webhook precedence.
- **Security & Governance** (`tests/security/adversarial_attacks.test.ts`):
  - WebAuthn FIDO2 asymmetric Passkey ceremony validation.
  - Replay-resistant signed gateway timestamp and nonce verification.
  - Agent self-delegation attack rejection (`403`).
  - Cross-tenant isolation verification.
- **Policy & State Machines** (`tests/policy/policy_engine.test.ts`):
  - Monotonic delegation narrowing rule (`validateDelegationSubset`).
  - Delegation revocation cascade across child mandates.
  - Reserve $\rightarrow$ Commit $\rightarrow$ Release mandate budget accounting.
  - Five distinct lifecycle state machines.
  - Seller Agent dialogue confidentiality.
- **Audit & WORM Anchoring** (`tests/audit/audit_integrity.test.ts`):
  - RFC 8785 canonical hash-chain integrity.
  - Transactional Outbox atomicity and ghost mutation detection.
  - External WORM storage cross-verification and local DB compromise detection.
- **Financial Precision** (`tests/pricing/money_rounding.test.ts`):
  - Fixed-point basis-point arithmetic and rounding invariants.

---

## 🔒 Security, Governance & Threat Model

| Threat / Attack Vector | Mitigation Strategy | Manifest Implementation |
| :--- | :--- | :--- |
| **Agent Prompt Injection / Overspending** | Deterministic Policy Engine below LLM | Hard boundary checks: Amount, Scope, Expiration, and Category whitelist. |
| **Agent Privilege Escalation** | Step-Up Authentication Requirement | High-power delegations require Human Principal with FIDO2 WebAuthn Passkey MFA. |
| **Sybil Hoarding Attack** | Multi-Entity Risk Aggregator | Limits hot items to strictly 1 active reservation per linked identity cluster. |
| **Database Compromise / Log Rewrite** | KMS-Signed External WORM Anchoring | Audit roots signed with KMS Ed25519 key and stored in immutable WORM S3 vault with 60s detection SLO. |
| **Ghost Commerce Mutations** | Transactional Outbox Pattern | Commerce mutations and audit events committed in the exact same ACID transaction. |
| **Webhook Replay / Secret Leaks** | Timestamp Freshness & Secret Rotation | Dual-verification grace period, KMS secret handles, and provider event ID dedup. |
| **Gateway Request Forgery** | Signed Envelopes with Nonces | HMAC-SHA256 over timestamp (60s freshness) + unique nonce + body hash. |
| **Financial Rounding Drifts** | Fixed-Point Integer Math | Basis-point integer arithmetic with half-up rounding (`roundDivision`). |

---

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](./LICENSE) file for details.
# Razorpay


Off Campus Code QMTC3G
