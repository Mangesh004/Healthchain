# HealthChain

> **"Secure Healthcare Records. Patient-Controlled Access."**  
> Blockchain-Based Healthcare Digital Records Exchange

HealthChain is a secure, patient-centric electronic medical records exchange platform. It empowers patients to own, manage, and grant granular consent for their healthcare records across hospitals, clinics, and diagnostic laboratories while leveraging Ethereum smart contracts for immutable audit trails and tamper-proof cryptographic integrity.

---

## 📌 Current Status

**ALL 14 PHASES COMPLETE AND LOCKED**:
- [x] **Phase 1**: Project Setup & Frontend Visual Foundation (Medivra-inspired healthcare design system)
- [x] **Phase 2**: Authentication & RBAC (JWT, bcrypt 12 rounds, secure sessions)
- [x] **Phase 3**: PostgreSQL Database Foundation (11 tables, sequential migrations, seeds, transactional role profiles)
- [x] **Phase 4**: Role Modules (Patient, Doctor, Hospital, Lab, Admin functional views)
- [x] **Phase 5**: Record Upload & AES-256-GCM Encryption
- [x] **Phase 6**: Off-Chain Consent Management
- [x] **Phase 7**: Smart Contract Implementation (Solidity 0.8.24 / OpenZeppelin)
- [x] **Phase 8**: Blockchain Integration (ethers.js v6)
- [x] **Phase 9**: Hash Verification & Tamper Detection
- [x] **Phase 10**: Immutable Audit Trail Logging
- [x] **Phase 11**: Emergency Break-Glass Access Protocol
- [x] **Phase 12**: Professional Healthcare UI/UX Polish
- [x] **Phase 13**: Automated Testing Expansion & Quality Assurance (351 tests, 6/6 suites)
- [x] **Phase 14**: Deployment, Reproducible Configuration & Demo Guide (10/10 Smoke Tests)

**DOCUMENTATION & GUIDES**:
- [Deployment & Operations Manual](docs/DEPLOYMENT.md)
- [Live Demonstration Guide](docs/DEMO_GUIDE.md)
- [System Architecture Specification](docs/ARCHITECTURE.md)

---

## 🏛️ Authoritative Architecture & Schema

```
                  ┌────────────────────────────────────────┐
                  │          React + Vite Frontend         │
                  │   Tailwind CSS / Medivra Design System │
                  └───────────────────┬────────────────────┘
                                      │ REST API / Axios
                                      ▼
                  ┌────────────────────────────────────────┐
                  │           Node.js / Express API        │
                  │       (Off-Chain Business Logic)       │
                  └─────────────┬────────────────────┬─────┘
                                │                    │
             PostgreSQL Queries │                    │ ethers.js v6
                                ▼                    ▼
        ┌───────────────────────────────┐   ┌───────────────────────────────┐
        │       PostgreSQL Database     │   │     Ethereum / Hardhat Node   │
        │  11 Tables (Off-Chain Data):  │   │  - SHA-256 Record Hashes      │
        │  - users, patients, doctors   │   │  - On-Chain Consent Registry  │
        │  - hospitals, labs            │   │  - Immutable Audit Events     │
        │  - medical_records (metadata) │   └───────────────────────────────┘
        │  - record_versions, consents  │
        │  - audit_logs, notifications  │
        │  - blockchain_transactions    │
        └───────────────────────────────┘
```

### Core Architecture Principle

- **OFF-CHAIN (Privacy & Compliance Layer)**:
  - Medical records & diagnostic reports (stored in encrypted file storage, NEVER in DB)
  - Protected Health Information (PHI)
  - User accounts and identity credentials
  - Record metadata & off-chain application storage
- **ON-CHAIN (Trust & Verification Layer)**:
  - Cryptographic SHA-256 hashes of medical records for tamper detection
  - Patient consent and access permissions
  - Immutable access logs & consent event history
  - Emergency break-glass event verification

*Medical files and PHI are NEVER stored directly in PostgreSQL or on the blockchain.*

---

## 🛠️ Technology Stack

| Layer | Technology |
|---|---|
| **Frontend** | React 18, Vite, Tailwind CSS, React Router v6, Axios, Lucide React, Plus Jakarta Sans |
| **Backend** | Node.js, Express.js, CORS, Morgan, dotenv, pg |
| **Database** | PostgreSQL, node-pg pool, transactional migrations & seeders |
| **Blockchain** | Solidity 0.8.24, Hardhat, ethers.js v6, OpenZeppelin AccessControl |
| **Security & Cryptography** | AES-256-GCM (off-chain file encryption), SHA-256 (hashing), JWT / bcrypt (12 rounds) |

---

## 🚀 Migration & Database Commands

```bash
# Run all 11 migrations in strict sequential order
npm run db:migrate

# Seed demo accounts (Admin, Hospital, 2 Doctors, Lab, 2 Patients)
npm run db:seed

# Reset development database
npm run db:reset

# Run automated database verification & API regression tests
npm run test:db
```

---

## 🗺️ 14-Phase Development Roadmap

1. **Phase 1: Project Setup & Frontend Visual Foundation** ✅ *(Completed)*
2. **Phase 2: Authentication & RBAC** ✅ *(Completed)*
3. **Phase 3: Database Schema & PostgreSQL Setup** ✅ *(Completed)*
4. **Phase 4: Role Modules** ✅ *(Completed)*
5. **Phase 5: Record Upload & AES-256-GCM Encryption** ✅ *(Completed)*
6. **Phase 6: Consent Management** ✅ *(Completed)*
7. **Phase 7: Smart Contract Implementation** ✅ *(Completed)*
8. **Phase 8: Blockchain Integration** ✅ *(Completed)*
9. **Phase 9: Hash Verification** ✅ *(Completed)*
10. **Phase 10: Audit Trail Logging** ✅ *(Completed)*
11. **Phase 11: Emergency Access Protocol** ✅ *(Completed)*
12. **Phase 12: UI Polish & User Experience** ✅ *(Completed)*
13. **Phase 13: Testing & Quality Assurance** ✅ *(Completed)*
14. **Phase 14: Deployment & Final Demo** ✅ *(Completed)*

---

## 📄 License

MIT © HealthChain Team
