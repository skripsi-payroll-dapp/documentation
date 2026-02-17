# Smart Contract Documentation (Solana/Anchor)

## Overview
The smart contract is the core "Backend" of the application, deployed on the Solana Blockchain. It handles all financial logic, state management, and access control. We use the **Anchor Framework** to simplify development and ensure security best practices.

## Program ID
`declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");` (To be updated after deployment)

## Account Structures

### 1. Organization Account
This account holds global configuration for a specific company or payroll group.
*   **Seeds:** `[b"organization", admin_pubkey]`
*   **Size:** 8 (discriminator) + 32 (admin) + 32 (vault_mint) + 8 (total_employees) + 1 (vault_bump) = 81 bytes.

### 2. Employee Account
This account tracks the vesting progress for an individual employee.
*   **Seeds:** `[b"employee", organization_pubkey, beneficiary_pubkey]`
*   **Size:** 8 + 32 (beneficiary) + 8 (rate) + 8 (last_ts) + 8 (accrued) + 8 (withdrawn) + 1 (bump) = 73 bytes.

### 3. Vault Account (Token Account)
This is a standard SPL Token Account owned by the `Organization` PDA. It holds the funds.
*   **Seeds:** `[b"vault", organization_pubkey]`
*   **Authority:** The `Organization Account` itself (PDA signing).

## Instructions

### `initialize_organization`
*   **Input:** `vault_bump` (derived client-side).
*   **Logic:**
    1.  Initializes `Organization` account.
    2.  Initializes `Vault` token account using PDA as authority.
    3.  Sets `admin` and `vault_mint`.

### `add_employee`
*   **Input:** `employee_pubkey`, `salary_per_second`.
*   **Access Control:** `#[account(has_one = admin)]` ensuring only the admin can call this.
*   **Logic:**
    1.  Creates `Employee` PDA.
    2.  Sets initial state.

### `claim_salary`
*   **Input:** None.
*   **Access Control:** `#[account(has_one = beneficiary)]` ensuring only the employee can claim.
*   **Logic:**
    1.  Calculates `time_elapsed = now - last_update_ts`.
    2.  Calculates `earned = time_elapsed * salary_per_second`.
    3.  Updates `last_update_ts` to `now`.
    4.  Performs CPI to `spl_token::transfer` sending funds from Vault to User.

## Security Considerations

### Re-entrancy Guard
We strictly follow the **Checks-Effects-Interactions** pattern.
1.  **Check:** Validate signer and math.
2.  **Effect:** Update `accrued` and `last_update_ts` *before* transferring tokens.
3.  **Interaction:** Call `token_program`.

This ensures that even if the Token Program were malicious (which it isn't, but for theory), it couldn't call back into `claim_salary` to withdraw funds again before the state is updated.

### PDA Signers
The Vault is a PDA. It has no private key. To transfer funds, the program uses `CpiContext::new_with_signer` with the `Organization` seeds to "sign" for the Vault.
Seed construction:
```rust
let seeds = &[
    b"organization",
    organization.admin.as_ref(),
    &[organization.bump],
];
```

## Error Codes
*   `Unauthorized`: Signer is not authorized.
*   `InsufficientFunds`: Global vault is empty.
*   `MathOverflow`: Calculation exceeded u64 limits (unlikely with reasonable constraints).
