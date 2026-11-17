# 🌾 AgriTrace (Food Traceability System)

A blockchain-enabled food traceability platform designed to track food batches from farm harvest through processing, distribution, and retail to consumers. It ensures end-to-end provenance and tamper detection by combining a relational SQL database with immutable smart contract event verification.

---

## 📌 Table of Contents
1. [Architecture Overview](#-architecture-overview)
2. [Technology Stack](#-technology-stack)
3. [How Data is Stored: SQL + Blockchain](#-how-data-is-stored-sql--blockchain)
   - [Where the SQL Database is Stored](#1-where-the-sql-database-is-stored)
   - [Where the Blockchain is Stored](#2-where-the-blockchain-is-stored)
   - [How They Work Together](#3-how-they-work-together-hybrid-architecture)
4. [How to Run the Project](#-how-to-run-the-project)
   - [Option 1: Full-Stack Mode with Blockchain (Recommended)](#option-1-full-stack-mode-with-blockchain-recommended)
   - [Option 2: Quick Start Mode (Without Blockchain)](#option-2-quick-start-mode-without-blockchain)
5. [Pre-Seeded Demo Accounts & Credentials](#-pre-seeded-demo-accounts--credentials)
6. [Testing & Demonstration Scripts](#-testing--demonstration-scripts)
7. [API & Port Reference](#-api--port-reference)
8. [Project Completion & Status](#-project-completion--status)

---

## 🏗 Architecture Overview

The system is split into three main components:

```
┌──────────────────────────────────────────────────────────┐
│                   Frontend (React + Vite)                │
│       Role Dashboards • QR Scanner • Public Verification │
└───────────────▲───────────────────────────▲──────────────┘
                │                           │
                ▼                           │
┌───────────────────────────────┐           │
│     Backend (Node/Express)    │           │
│  Auth • REST API • QR Builder │           │
└───────┬───────────────┬───────┘           │
        │               │                   │
        ▼               ▼                   │
┌──────────────┐ ┌────────────────────┐     │
│ SQL Database │ │ Ethers.js v6 Wallet│     │
│ (sql.js / pg)│ └─────────┬──────────┘     │
└──────────────┘           │                │
                           ▼                │
                 ┌────────────────────┐     │
                 │  Hardhat Blockchain│─────┘
                 │ (Solidity Contract)│ (Direct verification)
                 └────────────────────┘
```

1. **`contracts/`**: Solidity smart contract (`TraceabilityRegistry.sol`) deployed to a local Hardhat EVM node.
2. **`backend/`**: Express.js REST API managing authentication, role-based access, event logging, SHA-256 data hashing, and contract interaction.
3. **`frontend/`**: Modern React 19 single-page application with customized dashboards for each supply chain participant, live camera QR code scanner, visual provenance timeline, and tamper simulator.

---

## 💻 Technology Stack

| Layer | Technologies Used | Description |
| :--- | :--- | :--- |
| **Smart Contracts** | **Solidity `0.8.24`**, **Hardhat** | Immutable registry contract storing cryptographic event proofs (`EventProof`). |
| **Blockchain Client** | **Ethers.js `v6`** | Connects the backend via JSON-RPC, signs transactions, and interacts with contract ABIs. |
| **Database** | **SQLite (`sql.js`)** / **PostgreSQL** | Relational SQL database for fast app queries, user profiles, batch metadata, and coordinates. |
| **Backend** | **Node.js**, **Express.js**, **JWT**, **Zod** | REST API with input validation, role-based access control (RBAC), and logging. |
| **Frontend** | **React 19**, **Vite**, **Lucide Icons**, **Tailwind/Vanilla CSS** | Fast responsive UI with role-based routing and dark-mode styling. |
| **QR & Scanning** | **`qrcode`**, **`html5-qrcode`** | QR code generation for batches and live camera scanning in browser. |
| **Cryptography** | **Node.js `crypto` (SHA-256)** | Deterministic canonical JSON sorting and SHA-256 hashing. |

---

## 🗄 How Data is Stored: SQL + Blockchain

### 1. Where the SQL Database is Stored
- **Active Engine**: **SQLite** via `sql.js` (WebAssembly-based SQLite engine).
- **Physical File**: [`food-trace/backend/data/food_trace.db`](file:///d:/soft_pro/food-trace/backend/data/food_trace.db).
- **Optional Engine**: PostgreSQL can be enabled anytime by setting `DB_TYPE=postgres` and `DATABASE_URL` in `backend/.env`.
- **SQL Tables**:
  - `users`: User credentials, bcrypt password hashes, organizations, and roles (`ADMIN`, `FARMER`, `PROCESSOR`, `DISTRIBUTOR`, `RETAILER`, `CONSUMER`).
  - `batches`: Core product batch details (crop type, origin location, quantity, harvest date, owner ID).
  - `events`: Individual supply chain steps (`HARVEST`, `PROCESSING`, `PACKAGING`, `TRANSPORT`, `STORAGE`, `RETAIL_RECEIPT`, `CORRECTION`).
  - `reports`: Consumer discrepancy and alert reports.

### 2. Where the Blockchain is Stored
- **Active Engine**: **Hardhat EVM Local Node** at `http://127.0.0.1:8545` (Chain ID `31337`).
- **Smart Contract**: [`TraceabilityRegistry.sol`](file:///d:/soft_pro/food-trace/contracts/contracts/TraceabilityRegistry.sol) deployed at `0x5FbDB2315678afecb367f032d93F642f64180aa3`.
- **In-Memory Ledger**: Hardhat keeps the blocks, state tree, and transaction history in memory while running. In production, this points to an Ethereum testnet (e.g. Sepolia), Polygon, or a private Hyperledger Besu consortium.
- **On-Chain Data**: Only hashes and IDs are stored on-chain to protect business privacy and minimize gas fees:
  - `batchId`: String ID of the batch.
  - `dataHash`: 32-byte (`bytes32`) SHA-256 hash representing the exact state of that event.
  - `timestamp`: Blockchain block timestamp.
  - `submitter`: Wallet address that signed the entry.

### 3. How They Work Together (Hybrid Architecture)
1. When a participant (e.g. Farmer or Distributor) logs an action, the backend saves the full record in the **SQL database**.
2. The backend canonicalizes the data and calculates its **SHA-256 hash** (`data_hash`).
3. The backend sends a signed transaction to the **Solidity smart contract**, recording `(eventId, batchId, dataHash)` permanently on-chain.
4. The transaction hash (`tx_hash`) and block number are saved back into the **SQL database**.
5. When a consumer scans the QR code:
   - The backend reads the event from SQL and recalculates the SHA-256 hash.
   - It fetches the on-chain hash directly from the smart contract.
   - If they match: Status displays **`VERIFIED` (Authentic & Untampered)**.
   - If someone modified the database directly: Hashes mismatch and status flips to **`MISMATCH` (Tampering Detected)**.

---

## 🚀 How to Run the Project

### Option 1: Full-Stack Mode with Blockchain (Recommended)

This mode runs all three layers, enabling live on-chain event anchoring and verification.

#### Terminal 1: Start the Local Blockchain Node
```powershell
cd d:\soft_pro\food-trace\contracts
npx hardhat node
```
*Leave this running. It exposes the JSON-RPC interface at `http://127.0.0.1:8545`.*

#### Terminal 2: Deploy the Smart Contract
```powershell
cd d:\soft_pro\food-trace\contracts
npm run deploy:local
```
*This compiles the contract, deploys it to the local node, and automatically writes the new contract address into `backend/.env`.*

#### Terminal 3: Start the Backend API
```powershell
cd d:\soft_pro\food-trace\backend
npm run dev
```
*Starts the API on `http://localhost:4000`.*

> **Optional Database Reset**:
> If you ever want to wipe and re-seed the SQLite database:
> ```powershell
> cd d:\soft_pro\food-trace\backend
> npm run migrate
> npm run seed
> ```

#### Terminal 4: Start the Frontend App
```powershell
cd d:\soft_pro\food-trace\frontend
npm run dev
```
*Open **http://localhost:5173** in your browser.*

---

### Option 2: Quick Start Mode (Without Blockchain)

If you only need to inspect the UI or test REST endpoints without starting Hardhat:

1. **Terminal 1 (Backend)**:
   ```powershell
   cd d:\soft_pro\food-trace\backend
   npm run dev
   ```
2. **Terminal 2 (Frontend)**:
   ```powershell
   cd d:\soft_pro\food-trace\frontend
   npm run dev
   ```
3. Visit **http://localhost:5173**. *(Blockchain interactions will gracefully fallback to `DB_VERIFIED` mode).*

---

## 👥 Pre-Seeded Demo Accounts & Credentials

The seed script initializes accounts with different role permissions:

| Role | Email | Password | Allowed Capabilities |
| :--- | :--- | :--- | :--- |
| **Admin** | `admin@trace.io` | `Admin@123` | View all batches, manage user statuses, review reports, trigger tamper demo. |
| **Farmer** | `farmer@trace.io` | `Farmer@123` | Create new batches, log `HARVEST` events, generate QR codes. |
| **Processor** | `processor@trace.io` | `Processor@123` | Log `PROCESSING` and `PACKAGING` events. |
| **Distributor** | `distributor@trace.io` | `Distributor@123` | Log `TRANSPORT` and `STORAGE` events. |
| **Retailer** | `retailer@trace.io` | `Retailer@123` | Log `RETAIL_RECEIPT` events. |
| **Consumer** | *(Public / No login)* | N/A | Scan QR codes, verify provenance, submit discrepancy reports. |

**Pre-seeded Demo Batch ID**: `BATCH-2026-0001` (Organic Honeycrisp Apples).

---

## 🧪 Testing & Demonstration Scripts

### 1. Run Smart Contract Tests (17 tests)
```powershell
cd d:\soft_pro\food-trace\contracts
npm test
```
*Verifies contract deployment, access control, event storage, batch queries, and duplicate rejection.*

### 2. Run Backend API Tests
```powershell
cd d:\soft_pro\food-trace\backend
npm test
```
*Executes Jest integration tests for health, authentication, RBAC, event flow, and tamper checks.*

### 3. Run Live Tamper Simulation Script
```powershell
cd d:\soft_pro\food-trace\backend
npm run tamper-demo
```
*Simulates an unauthorized bad actor editing the SQL database directly (e.g. altering coordinates or product name) and demonstrates the cryptographic verification failure.*

---

## 🔌 API & Port Reference

- **Frontend**: `http://localhost:5173`
- **Backend API**: `http://localhost:4000/api`
- **Hardhat Blockchain Node**: `http://127.0.0.1:8545`

### Primary Endpoints
- `POST /api/auth/login` - User login & JWT issuance
- `GET  /api/auth/me` - Authenticated user profile
- `POST /api/batches` - Register a new crop batch (Farmer only)
- `GET  /api/batches` - List batches for current user/role
- `GET  /api/batches/:batchId` - Get full batch details & events
- `GET  /api/batches/:batchId/qr` - Generate QR code image
- `POST /api/batches/:batchId/events` - Append a new supply chain step
- `GET  /api/verify/:batchId` - **Public verification** comparing SQL vs Blockchain hashes
- `POST /api/reports` - Public consumer discrepancy report submission
- `POST /api/admin/tamper` - Admin tamper demo trigger

---

## 📊 Project Completion & Status

- **Smart Contracts**: **95%** (Completed, unit tested, deployed)
- **Backend API**: **85%** (Completed, dual SQLite/Postgres support, cryptographic pipeline)
- **Frontend App**: **85%** (Completed, 8 pages, camera QR scanner, visual timeline)
- **DevOps & Public Testnet**: **45%** (Ready for Docker containerization & Sepolia/Polygon testnet config)
- **Overall Completion**: **~82%** (Full working MVP)
# AgriTrace
