# Backend Documentation (Next.js API / Off-Chain)

## Overview
In a decentralized application (dApp), the primary "backend" is the blockchain itself (Solana). However, we still utilize Next.js API Routes (`/pages/api/...`) for specific off-chain tasks that improve user experience or handle administrative functions that are not strictly on-chain.

## Responsibilities

### 1. Metadata Hosting (Optional)
While SPL Tokens themselves hold balance, additional metadata (like Employee Profile Pictures, detailed job descriptions) might be stored off-chain to save on rent costs.
*   **JSON API:** We can serve JSON metadata from `/api/metadata/[id]`.
*   **Storage:** Could be IPFS or a simple database (PostgreSQL/Supabase) if configured.

### 2. Transaction Monitoring (Indexer)
Sometimes the frontend needs historical data that is hard to query directly from RPC nodes (e.g., "All past salary claims").
*   **Webhook Listener:** We can set up a Helius or Shyft webhook to listen for `ClaimSalary` events and store them in a local database for analytics.
*   **Analytics API:** An endpoint `/api/analytics/total-payouts` that aggregates this off-chain data for the Admin Dashboard.

### 3. Notification Service
To notify employees of salary updates or low balance warnings for the employer:
*   **Email/Telegram Bot:** A cron job running via Vercel Cron or GitHub Actions that checks the `Vault` balance daily and alerts the Admin if funds are low.

## API Routes Structure (Next.js)

### `GET /api/employee/[pubkey]`
Fetches off-chain details for an employee (e.g., Department, Email) which are not stored on-chain to save space.

### `POST /api/webhook/solana`
Receives transaction notifications from Helius/Shyft.
*   **Event:** `programId` matches our contract.
*   **Action:** Parse instruction logs.
*   **Store:** Save `amount`, `timestamp`, `txSignature` to database.

## Security
*   **Rate Limiting:** Use Upstash/Redis to prevent API abuse.
*   **Signature Verification:** Determine if the request comes from a valid source (e.g., verifying the `Ed25519` signature of a request if it claims to be from an admin wallet).

## Why No Traditional Database?
Solana accounts (`Organization`, `Employee`) already serve as a highly available, decentralized database. We only use off-chain storage for:
1.  **Indexing/Querying:** Complex queries (SQL-like) that the blockchain cannot do.
2.  **Privacy:** Data that should not be public (e.g., PII - Personally Identifiable Information).
3.  **Performance:** Caching frequently accessed data.
