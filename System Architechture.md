I. Project Overview and Foundation
1. Document Information
•	Document Title: Master Architecture & System Design Specification. 
•	Project Name: HealthChain. 
•	Project Description: Blockchain-Based Healthcare Digital Records Exchange. 
•	Document Purpose: To serve as the master technical architecture document for project implementation. 
•	Source of Truth: HealthChain — Complete Architecture & Development Blueprint. 
•	Intended Audience: Development team, project guides, and examiners. 
•	Version: Not specified in the source blueprint.
•	Status: Not specified in the source blueprint.
2. Project Overview
•	HealthChain is a secure, patient-controlled medical record sharing and consent management system. 
•	It solves the problem of siloed medical records and manual sharing by giving patients cryptographic control over who can view their data. 
•	Healthcare records require controlled sharing because patients currently lack verifiable control and cryptographic guarantees against silent record alteration. 
•	Patient-controlled consent is enforced via smart contracts rather than easily altered database flags. 
•	Medical record integrity is guaranteed by comparing the hash of an off-chain file against an immutable on-chain hash. 
•	Blockchain does NOT store medical records, PHI, patient names, or actual files. 
•	Blockchain stores only record hashes, record IDs, patient/doctor wallet addresses, consent statuses, access grants/revocations, and timestamps. 
•	PostgreSQL serves as the system of record for everything except cryptographic proofs (e.g., patient/doctor profiles, notifications, UI audit logs). 
•	Encrypted file storage holds the actual PDFs and images off-chain. 
3. Problem Statement
•	Medical records are currently siloed per hospital or lab. 
•	Record sharing relies on manual transfers (e.g., PDFs over email, printouts). 
•	There is a lack of verifiable, patient-controlled consent trails. 
•	There is a lack of cryptographic integrity guarantees to prove a record hasn't been silently edited post-issuance. 
•	Infrastructure/resource constraints: Not specified in the source blueprint beyond standard manual sharing limitations. 
•	There is a critical need for an auditable access trail that cannot be quietly rewritten by system admins. 
•	Problems caused by changing access requirements are solved by allowing patients to revoke access at any time. 
4. Proposed Solution
•	Patient: Uploads records, controls access, and revokes permissions. 
•	Doctor: Requests access to patient records, views authorized files, and uploads diagnoses. 
•	Hospital: Manages doctor accounts and views institution-wide activity. 
•	Lab: Uploads test reports directly against a patient's record. 
•	Admin: Manages users and views full system audit logs. 
•	Backend: Acts as the single trust boundary, handling authentication, business logic, and blockchain transaction signing. 
•	PostgreSQL: Stores relational metadata, user profiles, and a fast-read mirror of consent states. 
•	Encrypted file storage: Securely stores the actual medical documents. 
•	Blockchain: A private Ethereum-compatible network that records immutable consent events and document hashes. 
•	Smart contract: Enforces consent rules and integrity checks. 
•	Audit trail: Dual-layered, with UI-facing events in PostgreSQL and cryptographic proofs on-chain. 
5. Core Design Principles
•	Blockchain does not store PHI or medical data. 
•	PostgreSQL is the system of record for non-cryptographic data. 
•	Medical files remain off-chain and encrypted at rest. 
•	Blockchain proves integrity (file hasn't changed) and consent (access was authorized). 
•	The backend is the trust boundary, handling all authorization checks. 
•	The backend is the sole blockchain signer in the prototype (custodial model). 
•	The smart contract enforces consent; it is the only component immune to admin tampering. 
•	Actual record access always rechecks blockchain consent, rather than trusting only the database. 
•	Sensitive actions are audited both in PostgreSQL and on-chain. 
•	Blockchain should remain intentionally small to optimize performance and reduce complexity. 
•	No unnecessary healthcare data should be stored on-chain. 
II. System and Component Architecture
6. System Architecture
•	React Frontend: Provides role-aware UI and never talks directly to the blockchain. 
•	REST/JWT: Secures communication between frontend and backend. 
•	Node.js + Express Backend: The single choke point for authorization, business logic, and blockchain interaction. 
•	PostgreSQL + Encrypted File Storage: Manages relational data and encrypted documents. 
•	blockchainService.js: The exclusive module authorized to construct and sign blockchain transactions. 
•	ethers.js: The library used for blockchain communication. 
•	HealthcareRecords.sol: The smart contract enforcing logic. 
•	Hardhat Local / Private Ethereum-Compatible Network: The local blockchain environment for the prototype. 
7. Architecture Diagram
Code snippet
graph TD
    U[Users: Patient, Doctor, Hospital, Lab, Admin] -->|HTTPS| F[React + Vite + Tailwind frontend]
    F -->|REST API| B[Express REST API]
    B -->|Authentication / Authorization| S[Services]
    S --> DB[(PostgreSQL)]
    S --> FS[(Encrypted File Storage)]
    S --> BS[blockchainService.js]
    BS -->|ethers.js| SC[HealthcareRecords.sol]
    SC -->|Deployed on| HL[Hardhat Local Blockchain]
    
    classDef offchain fill:#f9f,stroke:#333,stroke-width:2px;
    classDef onchain fill:#bbf,stroke:#333,stroke-width:2px;
    class F,B,S,DB,FS,BS offchain;
    class SC,HL onchain;
8. User Roles
•	Patient: Uploads/receives records, grants or rejects consent requests, revokes access, and views access history. 
•	Doctor: Requests access to patient records, views authorized files, and uploads diagnoses/prescriptions. 
•	Hospital: Manages its doctors, adds/removes staff, and views institution-wide activity logs. 
•	Lab: Uploads test reports directly against a patient record and views recent uploads. 
•	Admin: Performs CRUD operations on user accounts and views full system audit logs. 
9. Frontend Architecture
•	Technology: React, Vite, Tailwind, React Router, Axios. 
•	Folder Structure: 
o	frontend/
	src/
	components/: Reusable UI elements (buttons, modals, transaction status badges). 
	pages/: Role-specific directories (auth, patient, doctor, hospital, lab, admin). 
	layouts/: Layout wrappers (DashboardLayout, AuthLayout). 
	services/: Axios instances (api.js), auth, record, consent, and read-only blockchain services. 
	hooks/: Custom hooks like useAuth, useRole. 
	context/: Global state (AuthContext, NotificationContext). 
	utils/: Helpers (formatDate, roleGuard). 
	assets/: Static files. 
	App.jsx: Root component. 
•	Role-Specific Pages: Dashboards tailored for Patient, Doctor, Hospital, Lab, and Admin. 
•	Core UI Elements: Authentication, Dashboard, Records list/details, Consent requests/active, Verification badges, Notifications, and Blockchain transaction status UI. 
10. Backend Architecture
•	Folder Structure: 
o	backend/
	controllers/: Thin handlers that parse requests and shape responses. 
	routes/: Express routers grouped by resource. 
	models/: ORM models mirroring DB tables. 
	middleware/: Authentication (auth.js), role verification (roleCheck.js), error handling, and Multer uploads. 
	services/: Core business logic (recordService, consentService, auditService, file service, hashService.js). 
	blockchain/: Houses blockchainService.js and provider setup. 
	utils/: Helpers like filename sanitization. 
	config/: Environment and DB configuration. 
	uploads/: Local encrypted file storage. 
	tests/: Backend test suites. 
	server.js: Application entry point. 
•	Data Flow: Routes → Controllers → Services → Models / blockchainService. 
•	Critical Rule: Controllers must NOT directly call ethers.js or the database. 
11. Database Architecture
•	Tables & Purpose: 
o	users: Base identity (PK: id, fields: email, password_hash, role, wallet_address). 
o	patients: 1:1 extension of users. 
o	doctors: Details specialization, hospital_id FK. 
o	hospitals: Institution details. 
o	labs: Laboratory details. 
o	medical_records: Off-chain record metadata (fields: file_url, file_hash, status). 
o	record_versions: Supports advanced versioning features. 
o	consents: Manages access grants (fields: status, expiry_time, transaction_hash). 
o	audit_logs: Tracks all sensitive system actions. 
o	blockchain_transactions: Logs on-chain actions. 
o	notifications: User event feed. 
•	Note: access_requests is recommended to be merged into consents using a PENDING status. 
•	ER Diagram:
Code snippet
erDiagram
    USERS ||--o| PATIENTS : extends
    USERS ||--o{ CONSENTS : requester
    USERS ||--o{ AUDIT_LOGS : performs
    USERS ||--o{ NOTIFICATIONS : receives
    PATIENTS ||--o{ MEDICAL_RECORDS : owns
    PATIENTS ||--o{ CONSENTS : grants
    HOSPITALS ||--o{ DOCTORS : employs
    HOSPITALS ||--o{ LABS : employs
    HOSPITALS ||--o{ MEDICAL_RECORDS : hosts
    MEDICAL_RECORDS ||--o{ RECORD_VERSIONS : has
    CONSENTS ||--o| BLOCKCHAIN_TRANSACTIONS : proves
16. Blockchain Architecture
•	Technology: Solidity, Hardhat, ethers.js v6, OpenZeppelin. 
•	Environment: Uses a local Hardhat blockchain with a backend signer acting on behalf of users. 
•	Structure: 
o	blockchain/
	contracts/: Solidity files.
	scripts/: Deployment scripts.
	test/: Contract test suites.
	deployments/: Exports ABI and contract address.
	hardhat.config.js: Network config.
17. Smart Contract Architecture
•	Contract: HealthcareRecords.sol. 
•	Record Struct: recordId, patient, uploader, fileHash, timestamp, active. 
•	Consent Struct: patient, doctor, recordId, expiry, active. 
•	Functions: 
o	registerRecord(): Stores a new record's hash.
o	grantAccess(): Patient authorizes access.
o	revokeAccess(): Patient withdraws access.
o	hasAccess(): Read-only verification check.
o	verifyRecord(): Compares provided hash against stored hash.
o	logAccess(): Records a view event on-chain.
o	grantEmergencyAccess(): Logs an emergency access override.
•	Events: RecordRegistered, AccessGranted, AccessRevoked, RecordAccessed, EmergencyAccessGranted. 
III. Data, Security, and Workflows
12. On-Chain vs Off-Chain Architecture
Data/Storage	Location	Reason DOCX
Names, PII, Patient profiles	Off-chain	Privacy; no PHI on shared ledgers.
Medical files, Diagnoses, Prescriptions	Off-chain	Size constraints and privacy.
Notifications, UI audit, File locations	Off-chain	App-specific utility metadata.
Record hash, Record ID	On-chain	Cryptographic proof of integrity.
Patient/doctor addresses	On-chain	Pseudonymous identities for consent mapping.
Consent status, grants/revocations	On-chain	Immutable proof of authorization.
Expiry, Timestamps, Blockchain events	On-chain	Provable timelines independent of DB.
•	Architectural Rule: If information describes the contents of a medical record, it goes off-chain; if it proves something happened or guarantees integrity, it goes on-chain. 
13. File Storage Architecture
•	Flow: Upload → MIME validation → Size validation → Filename sanitization → SHA-256 → AES-256 encryption → Local encrypted storage → PostgreSQL metadata → Blockchain hash registration. 
•	Details: Supports PDF/JPG/PNG. 10MB prototype limit. Files get UUID filenames. Encrypted via AES-256-GCM using a .env key. Architecture is S3-ready. 
•	Security: Authenticated downloads only; no public static file serving. 
14. Data Integrity Architecture
•	Upload: Original file → SHA-256 → Database hash → Blockchain hash. 
•	Verify: Stored file → decrypt → SHA-256 again → compare with blockchain hash. 
•	Outcomes: A match yields a VALID result; a mismatch yields a HASH MISMATCH (tampering detected). 
•	Authority: The blockchain hash is the ultimate source of truth. 
15. Consent Architecture
•	Lifecycle: Doctor requests access → Patient receives notification → Patient reviews → Patient grants/rejects → Blockchain transaction → Database update → Doctor receives notification → Doctor accesses authorized record. 
•	Scope: Supports both record-specific and whole-patient consent (recordId = 0/null). 
•	Revocation: Revoking immediately updates the on-chain hasAccess() to false, instantly blocking access even before DB sync. 
18. Blockchain Security Architecture
•	Utilizes OpenZeppelin AccessControl for Uploader, Emergency, and DEFAULT_ADMIN_ROLE. 
•	Applies ReentrancyGuard. 
•	Relies on Solidity ^0.8 built-in integer protections. 
•	Strictly no PII on-chain and no unbounded arrays. 
•	Maintains separate deployer and operational backend signer keys, stored in environment secrets. 
•	Limitation/Evolution: The backend signer is trusted in the prototype; future evolution requires user-held wallets. 
19. Backend–Blockchain Integration
•	Flow: React → Express Controller → Service → blockchainService.js → ethers.js → HealthcareRecords.sol → Hardhat. 
•	blockchainService.js is the absolute only module allowed to construct and sign transactions. 
•	It manages the provider, signer, contract instance, ABI, and contract address. 
•	Transactions await tx.wait() to receive a receipt, storing the tx hash in the database and handling errors cleanly. 
20. Authentication and Authorization
•	Uses bcrypt (12 rounds) for passwords and JWT for access/refresh tokens. 
•	JWT payload includes sub, role, and exp. 
•	Enforces Role-Based Access Control (RBAC) and record-level authorization. 
•	Critical Rule: A doctor cannot access a patient record without valid, unexpired consent verified against the blockchain. 
21. Emergency Access Architecture
•	Requires a specific EMERGENCY_ROLE and a mandatory reason for access. 
•	Access is time-limited and immediately logs an event on-chain, visible permanently to the patient. 
•	Production systems should evolve this to require two-person approval. 
22. Audit Architecture
•	PostgreSQL: Logs logins, uploads, views, downloads, consent requests/grants/revokes, verifications, and emergency access. 
•	Blockchain: Emits RecordRegistered, RecordAccessed, AccessGranted, AccessRevoked, EmergencyAccessGranted. 
•	UI actions (like viewing a file) live in DB; immutable state changes go on-chain. 
23. API Architecture
•	Auth: POST /api/auth/register, POST /api/auth/login (No auth required). 
•	Patient: GET /api/patients/profile, GET /api/patients/records (Auth + PATIENT role). 
•	Records: POST /api/records, GET /api/records/:id, GET /api/records/:id/download, GET /api/records/:id/verify (Auth + specific roles/consent checks). 
•	Consent: POST /api/consents/request, POST /api/consents/:id/grant, POST /api/consents/:id/revoke. 
•	Audit: GET /api/access-history. 
•	Labs: POST /api/labs/reports. 
•	Emergency: POST /api/emergency-access. 
•	Admin: GET /api/admin/users, GET /api/admin/audit. 
24. Security Architecture
•	Prototype: JWT, bcrypt, RBAC, input validation, rate limiting, parameterized ORMs, MIME/size validation, filename sanitization, AES-256 encryption, env secrets, restricted CORS, audit logging, no PHI on blockchain/logs. 
•	Production: Separates prototype features from required production features (MFA, WAF, KMS, AV scanning). 
25. Complete Data Flows
Code snippet
sequenceDiagram
    participant Patient
    participant Backend
    participant DB as PostgreSQL
    participant BC as Smart Contract
    Patient->>Backend: POST /consents/:id/grant
    Backend->>DB: Verify patient owns record
    Backend->>BC: grantAccess(patient, doctor, recordId, expiry)
    BC-->>Backend: emit AccessGranted, return txHash
    Backend->>DB: Update consents.status = GRANTED, store txHash
    Backend-->>Patient: 200 OK
IV. Execution, Deployment, and Summary
26. Complete Project Folder Structure
•	HealthChain/
o	frontend/: React UI applications. 
o	backend/: Express APIs and business logic. 
o	blockchain/: Solidity contracts and Hardhat config. 
o	database/: Migrations and schemas. 
o	docs/: Architecture documentation. 
o	tests/: End-to-end tests. 
o	README.md, .gitignore, .env.example, docker-compose.yml. 
27. Environment Configuration
•	Requires: DATABASE_URL, JWT_SECRET, JWT_EXPIRY, PORT, BLOCKCHAIN_RPC_URL, PRIVATE_KEY, CONTRACT_ADDRESS, UPLOAD_DIR, FILE_ENCRYPTION_KEY, FRONTEND_ORIGIN. 
•	Crucial: .env must never be committed; only .env.example should be tracked. 
28. Development Roadmap
1.	Project Setup: Scaffolding. 
2.	Authentication: JWT/bcrypt logic. 
3.	Database: Schemas and constraints. 
4.	Role Modules: Role-specific UI dashboards. 
5.	Record Upload: File storage and hashing. 
6.	Consent (Off-Chain): DB-level consent flows. 
7.	Smart Contract: Solidity compilation and testing. 
8.	Blockchain Integration: Wire blockchainService.js to APIs. 
9.	Hash Verification: Integrity check endpoint. 
10.	Audit Trail: Logging sensitive actions. 
11.	Emergency Access: Override protocols. 
12.	UI Polish: Design system application. 
13.	Testing: Unit/E2E coverage. 
14.	Deployment/Demo: Docker scripts and demo prep. 
29. Testing Architecture
•	Coverage: Smart contracts (Hardhat/Mocha), Backend (Jest/Supertest), Frontend (RTL), E2E (Playwright/Cypress). 
•	Critical Scenarios: Unauthorized access blocks, correct grant/revoke propagation, expiry denials, tampered file detection, hash mismatches, role restrictions, and authorized downloads. 
30. MVP Scope
•	MUST: Auth, record upload/view, off-chain consent, basic smart contract consent/hashing, basic audit. 
•	SHOULD: Admin/Hospital/Lab panels, time-limited expiry, notifications. 
•	ADVANCED: Emergency access, versioning, blockchain audit UI. 
•	FUTURE: QR sharing, self-custody wallets, Fabric migration (Not to be built in MVP). 
31. Deployment Architecture
•	Prototype: Localhost running Vite (React), Node/Express, PostgreSQL, and Hardhat (optionally via Docker Compose). 
•	Future Production: Vercel/Netlify for frontend, Cloud VM for backend, Managed Postgres, S3 for files, and a Private Ethereum/Fabric network. 
32. Known Limitations
•	The backend signer acts on behalf of users (trusted model). 
•	Local blockchain lacks true decentralization. 
•	Uses local file storage instead of scalable cloud buckets. 
•	Blockchain addresses are pseudonymous, not fully anonymous. 
•	Gas costs are bypassed locally but remain a production concern. 
•	HIPAA/healthcare compliance is not fully implemented. 
•	Emergency access requires only single-doctor authorization currently. 
33. Production Evolution
•	Evolves toward user-held wallets, KMS for encryption, HSM/MPC for keys, cloud storage, mandatory HTTPS, MFA, WAF, AV scanning, and deployment to permissioned chains like Hyperledger Fabric with stronger emergency approvals. 
34. Architecture Decision Summary
Component	Technology	Responsibility
Frontend	React/Vite/Tailwind	UI 
Backend	Node.js/Express	Business logic 
Database	PostgreSQL	System of record 
File Storage	Encrypted local disk	Medical files 
Blockchain	Hardhat/Eth-compatible	Proof/consent 
Smart Contract	Solidity	Consent/integrity rules 
Blockchain Library	ethers.js v6	Blockchain communication 
Contract Security	OpenZeppelin	Access control 
Authentication	JWT + bcrypt	Identity/authentication 
Hashing	SHA-256	Integrity 
Encryption	AES-256-GCM	File protection 
35. Final Architecture Summary HealthChain operates fundamentally as an off-chain healthcare application, leveraging conventional secure web architectures (Node.js, PostgreSQL, AES-encrypted file storage) for processing and storing sensitive medical data. The blockchain is utilized selectively and strictly as a decentralized ledger for cryptographic integrity, patient consent enforcement, and provable access events. No actual healthcare records or protected health information (PHI) are ever stored on the blockchain.

