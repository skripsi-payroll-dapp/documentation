# BAB 3: ANALISIS DAN PERANCANGAN SISTEM

## 3.1 Arsitektur Umum (High-Level Architecture)

Sistem *Decentralized Streaming Payroll* ini dirancang menggunakan pendekatan arsitektur *Hub-and-Spoke* untuk memaksimalkan efisiensi biaya komputasi (*Compute Units*) dan pengelolaan likuiditas pada jaringan Solana. Dalam model ini, akun organisasi bertindak sebagai pusat (*Hub*) yang mengelola konfigurasi global dan penyimpanan dana terpusat, sementara akun karyawan bertindak sebagai jari-jari (*Spokes*) yang menyimpan status individual (*User State*) masing-masing penerima gaji.

### 3.1.1 Model Hub-and-Spoke
Arsitektur ini terdiri dari dua entitas utama:
1.  **Organization Account (Hub):** Sebuah *Program Derived Address* (PDA) tunggal yang menyimpan konfigurasi global seperti *Public Key* admin, alamat *Mint* token (misalnya USDC atau IDR Stablecoin), dan jumlah total karyawan. Akun ini juga memiliki otoritas atas *Token Vault* global.
2.  **Employee Account (Spoke):** PDA unik untuk setiap karyawan yang menyimpan data spesifik seperti tarif gaji per detik (*salary rate*), saldo yang belum ditarik (*accrued amount*), dan penanda waktu terakhir klaim (*last update timestamp*).

### 3.1.2 Efisiensi Gas dan Likuiditas
Keunggulan utama dari arsitektur ini dibandingkan model streaming tradisional (dimana setiap aliran dana memerlukan deposit terpisah) adalah efisiensi likuiditas. Employer hanya perlu melakukan deposit satu kali ke dalam *Global Vault*. Kontrak pintar tidak memisahkan dana per karyawan secara fisik, melainkan secara logis melalui perhitungan matematika (*lazy evaluation*). Hal ini mengurangi beban transaksi admin dan mengoptimalkan penggunaan sewa (*rent*) penyimpanan data di blockchain.

## 3.2 Perancangan Smart Contract (On-Chain)

Sistem ini memanfaatkan kemampuan Solana untuk memisahkan logika program (*stateless*) dan data akun (*stateful*). Berikut adalah definisi struktur data dalam bahasa pemrograman Rust menggunakan *framework* Anchor.

### 3.2.1 Data Structures
Berikut adalah definisi *struct* untuk akun utama:

**A. Global State (`Organization` Account)**
Akun ini dibuat satu kali oleh perusahaan pada saat inisialisasi sistem.
*   `admin`: Pubkey (32 bytes) - Otoritas administrasi.
*   `vault_mint`: Pubkey (32 bytes) - Alamat Mint token pembayaran.
*   `total_employees`: u64 (8 bytes) - Jumlah total karyawan.
*   `vault_bump`: u8 (1 byte) - Bump seed untuk validasi PDA Vault.

**B. User State (`Employee` Account)**
Akun ini dibuat untuk setiap karyawan.
*   `beneficiary`: Pubkey (32 bytes) - Penerima gaji.
*   `salary_per_second`: u64 (8 bytes) - Tarif gaji (satuan token terkecil/detik).
*   `last_update_ts`: i64 (8 bytes) - Waktu terakhir perhitungan/klaim dilakukan.
*   `accrued_amount`: u64 (8 bytes) - Gaji tertunda yang belum diklaim.
*   `total_withdrawn`: u64 (8 bytes) - Total gaji yang sudah ditarik.

### 3.2.2 PDA Seeds Strategy
Strategi derivasi alamat *Program Derived Address* (PDA) sangat penting untuk mencegah kolisi dan memastikan determinisme:
*   **Organization PDA Seed:** `[b"organization", admin_pubkey]`
*   **Vault PDA Seed:** `[b"vault", organization_pubkey]`
*   **Employee PDA Seed:** `[b"employee", organization_pubkey, beneficiary_pubkey]`

### 3.2.3 Algoritma Vesting (The Math)
Sistem menggunakan *Lazy Evaluation* dengan rumus matematika berikut:

$$
Claimable(t_{now}) = Accrued + [(t_{now} - t_{last}) \times R]
$$

Dimana:
*   $t_{now}$: Waktu saat ini (Unix Timestamp dari `Clock` Sysvar).
*   $t_{last}$: Waktu update terakhir.
*   $R$: Tarif gaji per detik (`salary_per_second`).
*   $Accrued$: Saldo akumulasi dari periode tarif sebelumnya.

## 3.3 Alur Logika Instruksi (Instruction Logic)

### 3.3.1 Initialize Stream (Organization)
1.  **Validasi:** Pastikan `signer` memiliki cukup SOL untuk rent.
2.  **Inisialisasi Akun:** Buat akun `Organization` dan `Vault` Token Account.
3.  **Set Data:** Simpan `admin_pubkey` dan `mint_address` ke dalam state.

### 3.3.2 Add Employee
1.  **Validasi:** Cek apakah `signer == organization.admin`.
2.  **Inisialisasi Akun:** Buat akun PDA `Employee` baru.
3.  **Set Data:** Simpan `beneficiary`, `rate`, dan set `last_update_ts = now`.

### 3.3.3 Claim Salary (Withdraw)
1.  **Validasi:** Cek `signer == employee.beneficiary`.
2.  **Perhitungan:** Hitung jumlah `claimable` menggunakan rumus di atas.
3.  **Update State (Effects):**
    *   `last_update_ts = now`
    *   `accrued = 0`
    *   `total_withdrawn += claimable`
4.  **Transfer (Interactions):** Panggil CPI ke `spl-token` untuk transfer dari Vault ke dompet User.

## 3.4 Diagram Sekuens (Mermaid)

```mermaid
sequenceDiagram
    actor User as Employee
    participant Client as Client App
    participant Program as Solana Program
    participant State as User PDA
    participant Token as SPL Token Program

    User->>Client: Click "Withdraw"
    Client->>Program: claim_salary(ctx)
    Program->>State: Fetch Data (Rate, LastTS, Accrued)
    Note right of Program: Calculate Delta = Now - LastTS<br/>Total = Accrued + (Delta * Rate)
    Program->>State: Update State (LastTS = Now, Accrued = 0)
    Program->>Token: Transfer(Vault -> UserWallet, Total)
    Token-->>Program: Success
    Program-->>Client: Success
    Client-->>User: Show Notification
```

## 3.5 Analisis Keamanan & Skalabilitas

### 3.5.1 Parallelism (Sealevel Runtime)
Berbeda dengan EVM yang memproses transaksi secara serial, Solana menggunakan model *Sealevel* yang memungkinkan pemrosesan paralel.
*   Karena setiap karyawan memiliki akun PDA (`Employee State`) yang terpisah, transaksi klaim gaji dari Karyawan A dan Karyawan B tidak saling memblokir (tidak ada *state contention* yang signifikan pada akun user, hanya sedikit pada akun Vault yang dilock sebentar saat mutasi saldo).
*   Hal ini memungkinkan ribuan karyawan melakukan klaim secara bersamaan tanpa lonjakan biaya gas yang ekstrem.

### 3.5.2 Trustless Architecture
Sistem ini bersifat *trustless* karena:
*   **Vault Control:** Dana disimpan dalam PDA Vault yang tidak memiliki *Private Key*. Hanya program yang bisa menandatangani pengeluaran dana sesuai logika yang telah diprogramkan.
*   **Immutable Rules:** Aturan vesting tertanam di smart contract. Employer (Admin) tidak bisa menarik kembali gaji yang sudah menjadi hak karyawan (sudah melewati waktu kerja) secara sepihak tanpa mekanisme khusus (misal: fitur clawback yang disepakati, jika ada).
