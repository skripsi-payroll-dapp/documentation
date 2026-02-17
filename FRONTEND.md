# Frontend Documentation (Next.js/React)

## Overview
The frontend is a **Next.js 14** application built with TypeScript, React, and Tailwind CSS. It serves as the primary interface for both Admin (Employee Management) and Employees (Salary Claims). It connects to the Solana blockchain via the **Solana Wallet Adapter**.

## Pages Structure

### 1. `app/page.tsx` (Dashboard)
This is the main dashboard that conditionally renders based on the user's role (Admin vs. Employee).
*   **Purpose:** Display current status and actions.
*   **Logic:**
    *   If wallet is not connected -> Show "Connect Wallet" button.
    *   If connected wallet is Admin -> Show "Manage Employees" panel.
    *   If connected wallet is Employee -> Show "Accrued Salary" and "Claim" button.

### 2. Components

#### `WalletButton.tsx`
A wrapper around `@solana/wallet-adapter-react-ui` to handle connection/disconnection.

#### `EmployeeCard.tsx`
Displays individual employee details:
*   Name (Pubkey)
*   Salary Rate (`salary_per_second`)
*   Claimable Amount (Live update every second, calculated client-side for UX).
*   "Claim" Button.

#### `AdminPanel.tsx`
*   Form specific to admins.
*   Fields: `Beneficiary Pubkey`, `Salary Rate`.
*   Action: Calls `add_employee` instruction.

## State Management

### Context API / Zustand
We use a lightweight state management solution (Zustand or React Context) to store:
*   `program`: The specific Anchor Program instance.
*   `balance`: The user's SOL and Token balance.
*   `employeeAccount`: The fetched `Employee` account data.
*   `lastFetchTime`: Timestamp of the last data fetch.

### Live Updates (Polling)
Since the `claimable_amount` increases every second, we implement a `setInterval` hook in the frontend to visually increment the displayed salary without making constantly repeated RPC calls.
**Formula for UI:**
`DisplayAmount = server_accrued + ((now - server_last_ts) * rate)`
This ensures the user sees their balance growing in real-time.

## Wallet Integration

### `@solana/wallet-adapter-react`
The application supports multiple wallets (Phantom, Solflare, etc.) via the standard adapter.
*   **ConnectionProvider:** Connects to the RPC endpoint (Devnet).
*   **WalletProvider:** Manages wallet selection state.
*   **useAnchorWallet:** Provides the `AnchorWallet` interface required for signing transactions using the Anchor client.

## API Setup (Anchor Client)

To interact with the smart contract, we initialize the program like this:

```typescript
import { AnchorProvider, Program } from "@coral-xyz/anchor";
import { useAnchorWallet, useConnection } from "@solana/wallet-adapter-react";
import idl from "../target/idl/bifrost.json";

const { connection } = useConnection();
const wallet = useAnchorWallet();

const provider = new AnchorProvider(connection, wallet, {});
const program = new Program(idl, programId, provider);
```
