HealthChain — Complete Architecture & Development Blueprint
Blockchain-Based Healthcare Digital Records Exchange Secure, Patient-Controlled Medical Record Sharing and Consent Management System
Blockchain isn't storing the medical records; it makes consent, integrity, and important access events provable and tamper-evident.
________________________________________
SECTION 1 — Executive Project Overview
Objective: Give patients cryptographic control over who can see their medical records, and give every stakeholder (patient, doctor, hospital, lab) tamper-evident proof of what was accessed, when, and whether a record has been altered since it was issued.
Problem being solved: Medical records today are siloed per hospital/lab. Sharing between providers relies on manual transfer (PDF over email, printouts, portals that don't talk to each other), with no verifiable, patient-controlled consent trail and no cryptographic guarantee that a record hasn't been silently edited after being issued.
Target users: Patients, Doctors, Hospitals (admin staff), Labs, System Admin.
Main use cases:
•	Patient uploads/receives records, controls who can view them, revokes access anytime.
•	Doctor requests access to a patient's records, views only what's authorized, uploads diagnoses/prescriptions.
•	Hospital manages its doctors and sees institution-wide activity.
•	Lab uploads test reports directly against a patient record.
•	Anyone can verify a record hasn't been tampered with since issuance.
Core innovation: Consent and integrity are enforced by a smart contract, not by a database flag a backend admin could quietly edit. A GRANTED or REVOKED state, and a record's hash, live in an append-only, publicly-verifiable ledger.
Why blockchain is used: To make three things provable independent of the application's own database: (1) a record's content hasn't changed since it was hashed, (2) a patient did/did not consent to a specific doctor viewing specific records, (3) an access or consent event happened at a specific time and can't be silently rewritten later.
What blockchain does NOT do: Store medical data, PHI, patient names, or files. It does not replace the database — PostgreSQL remains the system of record for everything except integrity/consent proofs.
On-chain: record hash, record ID, patient/doctor identifiers (addresses, not names), consent status, access grant/revoke events, timestamps, expiry.
Off-chain (PostgreSQL + encrypted file storage): patient/doctor/hospital profiles, actual medical files, diagnoses, prescriptions, all PII, notifications, UI-facing audit logs.
One-paragraph description (report/presentation/viva)
HealthChain is a healthcare digital records exchange that lets patients control access to their medical data across hospitals, doctors, and labs. Medical documents are encrypted and stored off-chain in a conventional database and file store, while a private Ethereum-compatible blockchain records only a cryptographic hash of each document, patient-granted consent events, and access logs. This gives the system tamper-evident integrity verification and an immutable, patient-controlled consent trail — without ever exposing sensitive health information on a public or shared ledger.
30-second examiner explanation
"Medical records are fragmented across hospitals, and patients have no real control over who accesses their data. HealthChain keeps the actual records off-chain, encrypted, like a normal healthcare app would. What goes on the blockchain is just a hash of each record and a log of consent — who granted access to whom, for how long. So if anyone tampers with a file, the hash won't match anymore and the app flags it immediately. And because consent is recorded on-chain, nobody — not even our own backend — can quietly grant themselves access; the smart contract enforces it."
________________________________________
SECTION 2 — Complete System Architecture
                                   ┌───────────────┐
                                   │     USERS     │
                                   │ Patient/Doctor│
                                   │ Hospital/Lab  │
                                   │    Admin      │
                                   └───────┬───────┘
                                           │ HTTPS
                                           ▼
                                ┌─────────────────────┐
                                │   REACT FRONTEND     │
                                │  (Vite, Tailwind,    │
                                │   React Router,      │
                                │   Axios)             │
                                └──────────┬───────────┘
                                           │ REST (JWT)
                                           ▼
                                ┌─────────────────────┐
                                │  EXPRESS BACKEND     │
                                │ ── Auth middleware   │
                                │ ── Role middleware   │
                                │ ── Controllers       │
                                │ ── Services          │
                                └───┬───────────┬──────┘
                      ┌────────────┘           └────────────┐
                      ▼                                      ▼
            ┌───────────────────┐                  ┌─────────────────────┐
            │    PostgreSQL      │                  │  Encrypted File      │
            │  (users, records   │                  │  Storage (off-chain) │
            │  metadata, consent │                  │  PDFs/images         │
            │  mirror, audit)    │                  └─────────────────────┘
            └───────────────────┘

                      Backend also calls:
                      ┌─────────────────────┐
                      │  blockchainService.js│
                      │      (ethers.js)      │
                      └──────────┬────────────┘
                                 ▼
                      ┌─────────────────────┐
                      │  Smart Contract       │
                      │  HealthcareRecords.sol│
                      └──────────┬────────────┘
                                 ▼
                      ┌─────────────────────┐
                      │ Hardhat Local/Private │
                      │  Ethereum Network      │
                      └───────────────────────┘
Component roles:
•	React frontend — role-aware UI; never talks to the blockchain directly (no MetaMask requirement for the prototype).
•	Express backend — single choke point for auth, authorization, business logic, and blockchain signing. This is the trust boundary.
•	PostgreSQL — system of record for everything except cryptographic proofs.
•	Encrypted file storage — actual documents, encrypted at rest, never referenced directly by URL without an authorization check.
•	blockchainService.js — the only module allowed to construct/sign transactions.
•	Smart contract — enforces who can grant/revoke/verify; the only thing that can't be silently edited by an app admin.
Cross-role data flow (example — Doctor requests, Patient grants):
Doctor → POST /api/consents/request → Backend validates doctor+record exist
      → DB: access_requests row (PENDING) → Notification to Patient
Patient → POST /api/consents/:id/grant → Backend verifies patient owns record
      → blockchainService.grantAccess() → Smart Contract emits AccessGranted
      → Backend stores txHash in DB, updates consents row to GRANTED
      → Notification to Doctor
Doctor → GET /api/records/:id → Backend calls hasAccess() on-chain (or cached DB mirror)
      → if true, decrypt & serve file + log access event
________________________________________
SECTION 3 — Frontend Architecture
frontend/
├── src/
│   ├── components/       # Reusable: Button, Card, Table, Modal, StatusBadge, TxLink
│   │   ├── common/
│   │   ├── records/
│   │   ├── consent/
│   │   └── blockchain/   # TransactionStatus, HashDisplay, VerifyBadge
│   ├── pages/
│   │   ├── auth/
│   │   ├── patient/
│   │   ├── doctor/
│   │   ├── hospital/
│   │   ├── lab/
│   │   └── admin/
│   ├── layouts/           # DashboardLayout (sidebar+navbar per role), AuthLayout
│   ├── services/          # api.js (axios instance), authService, recordService,
│   │                       # consentService, blockchainService (read-only calls via backend)
│   ├── hooks/              # useAuth, useRole, useNotifications
│   ├── context/            # AuthContext, NotificationContext
│   ├── utils/               # formatDate, hashDisplay, roleGuard
│   ├── assets/
│   └── App.jsx
This is close to your original structure — the additions are layouts/ split by role (each role's dashboard has a different sidebar) and a components/blockchain/ folder, since transaction status and hash verification badges are reused across many pages.
Page inventory (purpose / key API calls / actions — condensed)
Page	Purpose	Key API	Key Actions
Patient Dashboard	Overview stats, pending requests	GET /patients/dashboard	Grant/reject inline
Patient Records	List all records	GET /records?patientId=	Filter, open detail
Record Details	View one record	GET /records/:id	Verify, download, share
Upload Record	Patient self-uploads a doc	POST /records	Upload, tag type
Consent Requests	Incoming access requests	GET /consents/requests	Grant / Reject
Active Consents	Currently granted access	GET /consents/active	Revoke
Access History	Who viewed what, when	GET /access-history	Filter by doctor/date
Verify Record	Manual integrity check	GET /records/:id/verify	Show hash match/mismatch
Notifications	Event feed	GET /notifications	Mark read
Doctor Dashboard	Patients, pending requests	GET /doctors/dashboard	—
Doctor Patients	Patient list	GET /doctors/patients	Open patient
Patient Details (doctor view)	Authorized records only	GET /doctors/patients/:id	Request more access
Access Requests (doctor)	Requests sent, status	GET /consents/requests?by=doctor	—
Upload Diagnosis	Attach new record to patient	POST /records (role DOCTOR)	Upload
Hospital Dashboard	Institution stats	GET /hospitals/dashboard	—
Hospital Doctors	Manage doctor accounts	GET/POST /hospitals/doctors	Add/remove
Access Logs (hospital)	All activity by hospital staff	GET /audit?hospitalId=	Filter
Lab Dashboard	Upload queue, recent reports	GET /labs/dashboard	—
Upload Lab Report	Attach report to patient	POST /labs/reports	Search patient, upload
Admin: User/Hospital/Doctor/Lab Mgmt	CRUD on accounts	GET/POST/PATCH /admin/...	Approve/suspend
Admin: System Audit	Full audit log + chain events	GET /admin/audit	Filter, export
________________________________________
SECTION 4 — UI/UX Design
Design system: Deep clinical blue/teal primary, neutral slate greys, one accent (amber) reserved only for pending/warning states so it stays meaningful. Avoid default Bootstrap-blue-and-white — that's the "generic CRUD" look.
•	Layout: Fixed left sidebar (role-specific nav) + top navbar (search, notifications bell, avatar/role badge) + content area with consistent 24px padding.
•	Cards: Stat cards (records count, active consents, pending requests) use large numeric type + small label, not overloaded with icons.
•	Tables: Zebra-free, generous row height, status column always uses StatusBadge (Pending=amber, Granted=green, Revoked=grey, Expired=red-outline).
•	Blockchain transaction UI: A TransactionStatus component with three states — Submitting → Confirming → Confirmed, each with the tx hash shown truncated (0x8d91c...93af) and copyable, linking to a local block explorer view (even a simple in-app page listing Hardhat blocks is enough for a demo).
•	Verification UI: Big clear pass/fail state — green check "✓ Record Verified — hashes match" or red warning "⚠ Hash Mismatch — do not trust this file," never a subtle inline text that could be missed.
•	Consent UI: A request card shows requester identity, requested record types as checkboxes (already checked = requested), duration selector, Grant/Reject buttons — matches the example flow in your original doc.
•	Modals: Used only for destructive/consent actions (Revoke, Reject) with a one-line confirmation, not for primary navigation.
•	Notifications: Toast for immediate feedback + persistent bell dropdown for history.
Per-role dashboard contents:
•	Patient: record count, active consents, pending requests, recent access — then records list, then pending consent requests.
•	Doctor: assigned/active patients, pending access requests, recently viewed records.
•	Hospital: total patients, doctors, records, pending requests, recent activity feed.
•	Lab: upload queue, recent reports submitted, verification status of last few uploads.
________________________________________
SECTION 5 — Backend Architecture
backend/
├── controllers/        # Thin — parse req, call service, shape response
├── routes/              # Express routers, grouped by resource
├── models/               # Sequelize/Prisma models (1:1 with DB tables)
├── middleware/            # auth.js (JWT verify), roleCheck.js, errorHandler.js, upload.js (Multer)
├── services/               # Business logic: recordService, consentService, auditService
├── blockchain/               # blockchainService.js, contractABI.json, provider setup
├── utils/                      # hashService.js (SHA-256), sanitizeFilename, logger
├── config/                      # db.js, env.js
├── uploads/                      # local encrypted file store (dev only)
├── tests/
└── server.js
Layering (clean architecture): Routes → Controllers (HTTP concerns only) → Services (business rules, transaction orchestration) → Models (DB) / blockchainService (chain). Controllers never talk to ethers.js or the DB directly — this keeps the blockchain swappable and testable.
•	Auth/Authorization middleware: auth.js verifies JWT and attaches req.user; roleCheck.js is a factory (roleCheck('DOCTOR')) used per-route.
•	Blockchain service: owns the provider, signer, and contract instance; exposes registerRecordHash(), grantAccess(), revokeAccess(), hasAccess(), verifyRecord() — controllers only ever call these, never raw ethers calls.
•	File service: validates MIME/size, sanitizes filename, encrypts, writes to disk (S3-ready interface).
•	Hash service: pure SHA-256 utility, used both at upload time and at verify time.
•	Audit service: writes to audit_logs on every sensitive action; separately, blockchain-relevant actions also emit a chain event via blockchainService.
________________________________________
SECTION 6 — Database Architecture
Tables
users
Column	Type	Key	Nullable	Purpose
id	UUID	PK	no	user identity
name	varchar		no	display name
email	varchar unique		no	login
password_hash	varchar		no	bcrypt hash
role	enum(PATIENT,DOCTOR,HOSPITAL,LAB,ADMIN)		no	RBAC
wallet_address	varchar		yes	on-chain identifier (custodial or external)
hospital_id	UUID	FK→hospitals	yes	doctor/lab affiliation
created_at	timestamp		no	
patients (1:1 extension of users where role=PATIENT)
Column	Type	Key	Nullable	Purpose
id	UUID	PK, FK→users	no	
date_of_birth	date		yes	
blood_group	varchar		yes	emergency-access field
emergency_contact	varchar		yes	
doctors | id (FK→users) | specialization | license_no | hospital_id (FK) |
hospitals | id (FK→users) | address | registration_no |
labs | id (FK→users) | address | accreditation_no |
medical_records
Column	Type	Key	Nullable	Purpose
id	UUID	PK	no	
patient_id	UUID	FK→patients	no	owner
uploaded_by	UUID	FK→users	no	doctor/lab/patient who uploaded
hospital_id	UUID	FK→hospitals	yes	
record_type	enum		no	Prescription/Blood Test/X-Ray/...
file_url	varchar		no	pointer to encrypted storage
file_hash	varchar(64)		no	SHA-256, mirrors on-chain hash
onchain_record_id	varchar		yes	maps to smart contract record ID
created_at	timestamp		no	
updated_at	timestamp		no	
status	enum(ACTIVE,ARCHIVED)		no	
record_versions (supports the versioning advanced feature) | id | record_id (FK) | file_url | file_hash | version_no | created_at |
consents
Column	Type	Key	Nullable	Purpose
id	UUID	PK	no	
patient_id	UUID	FK	no	
requester_id	UUID	FK→users	no	doctor/hospital
record_id	UUID	FK→medical_records	yes	null = whole-patient scope
status	enum(PENDING,GRANTED,REJECTED,REVOKED,EXPIRED)		no	
start_time	timestamp		yes	
expiry_time	timestamp		yes	
transaction_hash	varchar(66)		yes	on-chain tx proof
created_at	timestamp		no	
access_requests — kept distinct from consents if you want to separate "request" (pre-decision) from "consent" (post-decision) lifecycle; can be merged with consents for MVP simplicity (recommended: merge, use status=PENDING as the request state, to avoid duplicate tables).
audit_logs | id | user_id (FK) | action | target_type | target_id | metadata (jsonb) | ip_address | created_at |
blockchain_transactions | id | tx_hash | related_entity_type (RECORD/CONSENT) | related_entity_id | action | block_number | status | created_at |
notifications | id | user_id (FK) | type | message | is_read | created_at |
ER Diagram (ASCII)
USERS
 ├── PATIENTS ──┬── MEDICAL_RECORDS ──┬── RECORD_VERSIONS
 │              │                      └── (file_hash mirrors chain)
 │              └── CONSENTS ── BLOCKCHAIN_TRANSACTIONS
 ├── DOCTORS ───── CONSENTS (as requester)
 ├── HOSPITALS ─── DOCTORS, LABS, MEDICAL_RECORDS
 └── LABS ──────── MEDICAL_RECORDS (as uploader)

AUDIT_LOGS ── references USERS + any entity
NOTIFICATIONS ── references USERS
Recommended indexes
•	medical_records(patient_id), medical_records(file_hash)
•	consents(patient_id, status), consents(requester_id, status)
•	audit_logs(user_id, created_at)
•	users(email) unique, users(wallet_address)
________________________________________
SECTION 7 — Medical Record Data Model
Field	PostgreSQL	Blockchain
Record ID	✅ (UUID PK)	✅ (mirrored as bytes32/uint)
Patient ID	✅	✅ (as address, not name)
Doctor/Uploader ID	✅	✅ (as address)
Hospital/Lab ID	✅	❌ (not needed on-chain)
Record type	✅	❌
File location	✅	❌ never
File hash	✅ (mirror)	✅ (source of truth)
Created/updated date	✅	✅ (block timestamp is authoritative)
Status	✅	partially (active/revoked flags only)
Rule of thumb: if a field describes what is in the record, it's off-chain; if it proves that something happened or whether something is unchanged, it's on-chain.
________________________________________
SECTION 8 — File Storage Architecture
Upload → MIME/size validation → Filename sanitization → SHA-256 hash
      → AES-256 encryption → Write to storage (local disk /uploads,
        S3-ready path convention: {patientId}/{recordId}/{version})
      → DB row (medical_records: file_url, file_hash)
      → blockchainService.registerRecordHash(recordId, hash, patientAddr)
•	Allowed types: PDF, JPG, PNG. Validate by real MIME sniffing (not just extension).
•	Size limit: e.g. 10MB per file for the prototype (Multer limits).
•	Filename sanitization: strip path traversal characters, store under a generated UUID filename, keep original name only as metadata.
•	Encryption: AES-256-GCM, key from .env (or per-file key wrapped by a master key) — never commit keys.
•	Access control: files are never served by direct static URL; every download goes through GET /api/records/:id/download, which re-checks consent server-side before decrypting and streaming.
•	Secure download: signed short-lived download token or direct authenticated stream.
•	Integrity verification: re-hash on download, compare to file_hash — this is also your on-demand "Verify" button logic.
Swap point for production: replace the local disk adapter with an S3 adapter behind the same fileService interface — nothing above it changes.
________________________________________
SECTION 9 — Hashing and Record Integrity
Upload time:
Medical File → SHA-256 → File Hash (e.g. 8e93a8c0...)
             → stored in medical_records.file_hash
             → registerRecordHash() on smart contract (source of truth)

Verify time (user clicks "Verify"):
Stored File → SHA-256 → Current Hash
            → compare against on-chain hash (authoritative) 
            → also compare against DB mirror (should always match; mismatch
              here alone = DB tampering, distinct from file tampering)
1.	When generated: immediately after upload, before encryption is finalized (hash the plaintext file).
2.	Where stored: DB (file_hash column, fast reads) and blockchain (authoritative, via registerRecordHash).
3.	How blockchain stores it: as bytes32 in the record's struct, emitted also as a RecordRegistered event for cheap off-chain indexing.
4.	Verification: backend re-hashes the currently stored (decrypted) file and calls verifyRecord(recordId, currentHash) on-chain, which compares against the stored hash and returns bool.
5.	If modified: hashes diverge → UI shows HASH MISMATCH, record flagged, event logged.
Result: VALID           Result: HASH MISMATCH
Blockchain Hash: ABC123   Blockchain Hash: ABC123
Current Hash:    ABC123   Current Hash:    XYZ999
Status: ✓ AUTHENTIC       Status: ⚠ TAMPERED — DO NOT TRUST
UI: large, unambiguous banner, never a subtle inline label — this is a safety-relevant signal.
________________________________________
SECTION 10 — Consent Management
Doctor → POST /consents/request {patientId, recordIds[], duration}
Patient notified → reviews on Consent Requests page
Patient → POST /consents/:id/grant
   Backend verifies caller IS the patient who owns the record(s)
   → blockchainService.grantAccess(patientAddr, doctorAddr, recordId, expiry)
   → smart contract stores {patient, doctor, recordHash, expiry, active=true}
   → emits AccessGranted event
   Backend updates consents.status = GRANTED, stores tx_hash
Doctor → GET /records/:id
   Backend calls hasAccess(patientAddr, doctorAddr, recordId) on-chain
   → true → decrypt + serve + log access
   → false → 403 with "consent required"
•	Revoke: POST /consents/:id/revoke → revokeAccess() on-chain → active=false → any subsequent hasAccess() call returns false immediately, even before the DB row updates (chain is authoritative, DB is a fast-read mirror).
•	Time-limited access: expiry stored on-chain; hasAccess() checks block.timestamp < expiry in addition to active.
•	Record-specific vs whole-patient: support both — recordId = 0/null sentinel means "all records," otherwise scoped to one record.
•	Expired access: a scheduled job (or lazy check on read) marks DB rows EXPIRED once expiry_time passes; chain state naturally reflects this via the timestamp check without needing a transaction.
________________________________________
SECTION 11 — Blockchain Architecture
blockchain/
├── contracts/
│   └── HealthcareRecords.sol
├── scripts/
│   ├── deploy.js
│   └── seed.js          # optional: seed roles for demo
├── test/
│   └── HealthcareRecords.test.js
├── deployments/
│   └── localhost.json    # contract address + ABI export for backend
└── hardhat.config.js
Recommended contract functions
Function	Purpose	Access
registerRecord(recordId, patient, hash)	store a new record's hash	uploader roles (doctor/lab/patient), via backend signer
grantAccess(patient, doctor, recordId, expiry)	patient authorizes access	only the patient (or backend acting on verified patient request)
revokeAccess(patient, doctor, recordId)	patient withdraws access	only the patient
hasAccess(patient, doctor, recordId)	read-only check	anyone (view function, no gas)
verifyRecord(recordId, hash)	compare hash	anyone (view function)
logAccess(patient, doctor, recordId)	record that a view actually happened	called by backend after a successful read
grantEmergencyAccess(patient, doctor, reason)	logged emergency override	restricted role, always emits event
Kept intentionally small — this is not a general-purpose EHR contract, it's a consent/integrity ledger. Resist the urge to add record content, patient profile fields, or messaging on-chain.
________________________________________
SECTION 12 — Smart Contract Data Structure
struct Record {
    bytes32 recordId;
    address patient;
    address uploader;
    bytes32 fileHash;
    uint256 timestamp;
    bool active;
}

struct Consent {
    address patient;
    address doctor;
    bytes32 recordId;   // 0x0 = all records
    uint256 expiry;
    bool active;
}

mapping(bytes32 => Record) public records;
mapping(bytes32 => Consent) public consents; // key = keccak256(patient, doctor, recordId)

event RecordRegistered(bytes32 indexed recordId, address indexed patient, bytes32 fileHash, uint256 timestamp);
event AccessGranted(address indexed patient, address indexed doctor, bytes32 recordId, uint256 expiry);
event AccessRevoked(address indexed patient, address indexed doctor, bytes32 recordId);
event RecordAccessed(address indexed doctor, bytes32 indexed recordId, uint256 timestamp);
event EmergencyAccessGranted(address indexed patient, address indexed doctor, string reason, uint256 timestamp);
On-chain: hashes, addresses, timestamps, boolean flags, expiry. Off-chain only: names, file contents, diagnoses, addresses(physical), phone numbers, everything PHI.
________________________________________
SECTION 13 — Smart Contract Security Review
Concern	Mitigation
Unauthorized record registration	onlyRole(UPLOADER_ROLE) via OpenZeppelin AccessControl; also assert msg.sender matches backend's registered signer
Unauthorized consent modification	grantAccess/revokeAccess require msg.sender == patient (or backend acting strictly on a verified, signed patient request — see Section 14)
Reentrancy	Functions here don't send ETH/value, so risk is low, but apply OpenZeppelin ReentrancyGuard on any state-changing function as defense-in-depth
Integer issues	Solidity ^0.8.x has built-in overflow/underflow checks — no SafeMath needed
Event design	Every state change emits an event — this is your audit trail's cheap, indexable source
Privacy leakage	Never store names/PII on-chain; addresses alone are pseudonymous but note in your report this is not full anonymity (a known limitation)
Excessive on-chain storage	No arrays that grow unbounded per record; keep structs minimal (as above)
Transaction replay	Not a signature-replay design since backend directly signs; if you later add off-chain patient signatures (EIP-712), add nonces
Role administration	Use AccessControl's DEFAULT_ADMIN_ROLE for granting/revoking UPLOADER_ROLE/EMERGENCY_ROLE, held by a secured deployer key, not the everyday backend signer
Key management	Backend signer key in .env/secrets manager, never in source; separate deployer key from operational signer key
________________________________________
SECTION 14 — Backend ↔ Blockchain Integration
React → Express controller → Service layer → blockchainService.js
     → ethers.js Contract instance (ABI + address + signer)
     → Hardhat local network
•	Provider: new ethers.JsonRpcProvider(process.env.BLOCKCHAIN_RPC_URL) — points at local Hardhat node in dev.
•	Signer: backend holds one operational wallet (private key in env) that signs all transactions on behalf of users — custodial model. Individual users do not need their own wallet/MetaMask for the prototype.
•	Contract instance: new ethers.Contract(address, abi, signer).
•	ABI/address: exported by Hardhat deploy script into deployments/localhost.json, imported by backend at startup.
•	Env vars: BLOCKCHAIN_RPC_URL, PRIVATE_KEY, CONTRACT_ADDRESS.
•	Transaction handling: tx = await contract.grantAccess(...), then await tx.wait() for confirmation before responding 200 to the frontend — the UI shows Submitting → Confirming → Confirmed to reflect this.
•	Error handling: wrap in try/catch; on revert, parse the revert reason and return a clean 4xx to the frontend rather than a raw ethers stack trace.
•	Tx hash storage: every chain call's resulting hash is written to blockchain_transactions and mirrored onto the relevant consents/medical_records row.
•	DB/chain sync: chain is authoritative for consent/hash truth; DB is a fast-read cache. On any read where consistency matters (an actual record view), backend re-checks hasAccess() on-chain rather than trusting only the DB flag — this is what makes the whole thing more than "a badge that says blockchain."
Who signs / who pays gas / production path
•	Local dev: backend's single funded Hardhat account signs everything; gas is free (test ETH from Hardhat's default accounts).
•	Who pays: in this custodial model, the application effectively pays gas (from its own signer's balance) — normal for an academic prototype and even common in real consumer apps ("gasless" UX).
•	Account/user mapping: each users row gets a wallet_address — for the prototype this can simply be a deterministic derived address per user (or all actions signed by one shared backend key with the patient's address passed as a function parameter rather than msg.sender, since the backend is the actual caller). Document this clearly: the backend is a trusted intermediary in this architecture — a real limitation, addressed below.
•	Production evolution: move to a model where the patient signs consent transactions with their own wallet (MetaMask or a custodial-but-user-keyed wallet service), so msg.sender really is the patient and the backend can no longer forge a grant on their behalf. This removes the "trusted backend" limitation.
________________________________________
SECTION 15 — API Design (representative set)
Method	URL	Auth	Role	Purpose
POST	/api/auth/register	No	—	create account
POST	/api/auth/login	No	—	issue JWT
GET	/api/patients/profile	Yes	PATIENT	own profile
GET	/api/patients/records	Yes	PATIENT	list own records
POST	/api/records	Yes	PATIENT/DOCTOR/LAB	upload record
GET	/api/records/:id	Yes	consent-checked	view metadata
GET	/api/records/:id/download	Yes	consent-checked	stream decrypted file
GET	/api/records/:id/verify	Yes	consent-checked	hash verification result
POST	/api/consents/request	Yes	DOCTOR	request access
POST	/api/consents/:id/grant	Yes	PATIENT (owner)	grant on-chain
POST	/api/consents/:id/reject	Yes	PATIENT	reject request
POST	/api/consents/:id/revoke	Yes	PATIENT	revoke on-chain
GET	/api/access-history	Yes	PATIENT/DOCTOR	audit view
POST	/api/labs/reports	Yes	LAB	upload report against a patient
POST	/api/emergency-access	Yes	DOCTOR (emergency role)	logged override
GET	/api/admin/users	Yes	ADMIN	user management
GET	/api/admin/audit	Yes	ADMIN	full system audit
Every mutating endpoint requires JWT + role check; record-level endpoints additionally require a server-side consent check (DB fast-path + on-chain confirm for the actual read).
________________________________________
SECTION 16 — Authentication and Authorization
Register → bcrypt(password, 12 rounds) → store hash
Login → bcrypt.compare → issue JWT {sub, role, exp: 1h (+ refresh 7d)}
Every request → auth.js verifies JWT signature+expiry → req.user
             → roleCheck.js('DOCTOR') gate
             → controller → record-level check: 
                does a GRANTED, non-expired consent exist for this
                (doctor, record) pair? (DB fast check + on-chain hasAccess()
                for the actual file read)
•	JWT structure: { sub: userId, role, iat, exp }, signed with JWT_SECRET.
•	Expiry: short-lived access token (1h) + refresh token (7d, httpOnly cookie) for the prototype; simpler alternative for MVP is a single longer-lived token if refresh flow is out of scope.
•	Record-level authorization is the critical guarantee: "A doctor cannot access a patient's record unless proper consent exists" is enforced at the controller layer, re-checked against the chain, not just trusted from the JWT role claim.
________________________________________
SECTION 17 — Emergency Access
Patient unconscious → Emergency Doctor (role flag EMERGENCY_ROLE)
→ POST /api/emergency-access {patientId, reason}
→ reason is mandatory, min length enforced
→ backend calls grantEmergencyAccess() on-chain (always logged, always time-boxed e.g. 24h)
→ access granted, EmergencyAccessGranted event emitted
→ patient sees this in Access History with reason + doctor + timestamp, permanently
Security concerns: this must never be a silent bypass. Constraints: only accounts explicitly flagged EMERGENCY_ROLE by an admin can call it; every use requires a non-empty reason; access is time-boxed and auto-expires; it is always surfaced to the patient after the fact, non-deletable from their history. It should be rare-use, heavily logged, and ideally require two-person sign-off in a production system (out of scope for prototype, mention in report as a limitation/future improvement).
________________________________________
SECTION 18 — Audit Trail
Event	PostgreSQL audit_logs	Blockchain event
Login	✅	❌
Record upload	✅	✅ (RecordRegistered)
Record view	✅	✅ (RecordAccessed)
Record download	✅	❌ (download is a UI action on an already-authorized view)
Consent request	✅	❌ (pre-decision, off-chain only)
Consent grant	✅	✅ (AccessGranted)
Consent revoke	✅	✅ (AccessRevoked)
Record verification	✅	read-only, no new event
Emergency access	✅	✅ (EmergencyAccessGranted)
Blockchain tx (generic)	✅ (blockchain_transactions)	is the source
Rule: anything that needs to be provable to a third party or immutable goes on-chain; everything needed for UI/UX and general activity logging stays in PostgreSQL, which is cheaper and richer (can store IP, user agent, free-text metadata).
________________________________________
SECTION 19 — Security Architecture Checklist
Area	Prototype	Production addition
Authentication	JWT + bcrypt	add MFA, account lockout
Authorization	role + record-level check	fine-grained policy engine
Password security	bcrypt 12 rounds	password breach checks
API security	input validation (Joi/Zod), rate limiting on auth routes	WAF, full rate limiting
SQL injection	parameterized queries / ORM	same + periodic scans
XSS	React escapes by default; sanitize any rendered HTML	CSP headers
CSRF	N/A for pure JWT-bearer API	if cookies used, CSRF tokens
File upload security	MIME/size validation, sanitized filenames	AV scanning
Encryption	AES-256 at rest	KMS-managed keys, TLS everywhere
Secrets management	.env (gitignored)	vault/secrets manager
Blockchain key security	single signer key in env	HSM or MPC wallet, key rotation
Rate limiting	basic express-rate-limit	full API gateway
Logging	console + audit_logs table	centralized logging (ELK)
CORS	restricted to frontend origin	same, stricter
HTTPS	not required locally	mandatory in production
Sensitive data exposure	never log PHI, never put PHI on-chain	DLP tooling
________________________________________
SECTION 20 — Complete Data Flows (condensed sequence diagrams)
FLOW E — Patient Grants Consent
Patient → Frontend: click "Grant"
Frontend → Backend: POST /consents/:id/grant
Backend → DB: verify caller owns the record/consent request
Backend → blockchainService: grantAccess(patient, doctor, recordId, expiry)
blockchainService → Smart Contract: tx submitted
Smart Contract → Blockchain: mined, event emitted
blockchainService → Backend: tx receipt (hash, block)
Backend → DB: update consents.status=GRANTED, tx_hash
Backend → Frontend: 200 OK {status, txHash}
Backend → Notification service: notify Doctor

FLOW H — Record Integrity Verification
User → Frontend: click "Verify"
Frontend → Backend: GET /records/:id/verify
Backend → File storage: fetch + decrypt file
Backend → hashService: SHA-256(file)
Backend → Smart Contract: verifyRecord(recordId, currentHash) [view call]
Smart Contract → Backend: bool match
Backend → Frontend: {status: VALID | HASH_MISMATCH, hashes}
(Flows A–D, F, G, I, J follow the same pattern — Frontend → Backend → [DB and/or Chain] → Backend → Frontend, with the chain step only present for hash-registration, consent grant/revoke, and emergency access.)
________________________________________
SECTION 21 — Complete Project Folder Structure
HealthChain/
├── frontend/
├── backend/
├── blockchain/
├── database/
│   └── migrations/          # SQL migration files (or Prisma/Sequelize migrations)
├── docs/
│   ├── diagrams/
│   ├── api-spec.md
│   └── viva-prep.md
├── tests/                     # cross-cutting e2e tests (Playwright/Cypress)
├── README.md
├── .gitignore
├── .env.example
└── docker-compose.yml           # optional: postgres + backend + hardhat node
________________________________________
SECTION 22 — Environment Variables
DATABASE_URL=postgresql://user:pass@localhost:5432/healthchain
JWT_SECRET=<random 32+ byte string>
JWT_EXPIRY=1h
PORT=5000
BLOCKCHAIN_RPC_URL=http://127.0.0.1:8545
PRIVATE_KEY=<backend signer key — Hardhat default account in dev>
CONTRACT_ADDRESS=<deployed address, from deployments/localhost.json>
UPLOAD_DIR=./uploads
FILE_ENCRYPTION_KEY=<32-byte key for AES-256>
FRONTEND_ORIGIN=http://localhost:5173
All consumed via config/env.js in the backend and hardhat.config.js for blockchain vars. Never commit .env — commit .env.example only.
________________________________________
SECTION 23 — Development Phases
Phase	Objective	Key files	Depends on	Result	Test checklist
1	Project setup	repo scaffold, configs	—	Frontend/backend boot, DB connects	health-check endpoint returns 200
2	Authentication	auth routes, JWT, bcrypt	1	Register/login works for all roles	wrong password rejected, token expiry works
3	Database	migrations, all tables	1	Schema matches Section 6	FK constraints hold
4	Role modules	patient/doctor/hospital/lab CRUD	2,3	Each role has a dashboard shell	role-gated routes 403 correctly
5	Record upload	file service, hash service	3,4	Upload → stored + hashed	re-download matches hash
6	Consent (off-chain only)	consents table, request/grant/reject/revoke logic	5	Full flow works without chain	doctor blocked until granted
7	Smart contract	HealthcareRecords.sol, Hardhat tests	— (parallel with 1-6)	Contract compiles, unit tests pass	grant/revoke/verify tested in isolation
8	Blockchain integration	blockchainService.js, wire into consent/record services	6,7	On-chain calls fire on grant/revoke/upload	tx hash stored, hasAccess() matches DB
9	Hash verification	verify endpoint + UI	8	Tamper demo works	modified file correctly flagged
10	Audit trail	audit_logs writes everywhere, admin audit page	4-9	Full activity visible	every sensitive action logged
11	Emergency access	emergency endpoint + contract function	8	Logged override works	reason mandatory, always visible to patient
12	UI polish	design system applied consistently	all above	Looks industry-grade	no unstyled/default-browser elements
13	Testing	unit + integration + e2e + contract tests	all above	CI-passable test suite	coverage on critical paths
14	Deployment/demo	local docker-compose, demo script	all above	Runs from clean clone	fresh clone → working demo in <10 min
________________________________________
SECTION 24 — MVP vs Advanced Features
MUST HAVE (MVP): Auth (4 roles minus admin optional), record upload/view, off-chain consent working end-to-end, smart contract hash registration + consent grant/revoke, verify-record feature, basic audit log.
SHOULD HAVE: Admin panel, notifications, hospital/lab dashboards fully built out, time-limited consent expiry.
ADVANCED: Emergency access, record versioning, full blockchain audit UI with tx explorer links.
FUTURE (mention in report, don't build): QR-based sharing, digital signatures, MetaMask/self-custody wallets, multi-hospital federation, Hyperledger Fabric production migration.
________________________________________
SECTION 25 — Testing Strategy
•	Smart contract (Hardhat/Mocha/Chai): grantAccess only callable by patient; revoke immediately blocks hasAccess; verifyRecord correctly flags tampered hash; role-gated registerRecord reverts for unauthorized caller.
•	Backend (Jest/Supertest): auth flow, role middleware 403s, record upload → hash matches, consent grant → DB+chain both updated, download blocked without consent.
•	Frontend (React Testing Library): protected routes redirect unauthenticated users, StatusBadge renders correct states, verify button shows correct pass/fail UI.
•	E2E (Playwright/Cypress): full demo scenario — register patient+doctor, upload, request, grant, view, revoke, denied, verify.
•	Key test cases: tampered file correctly fails verification; expired consent correctly denies access even if DB row is stale; revoke takes effect immediately.
________________________________________
SECTION 26 — Demo Scenario (10–15 min)
1.	Patient & Doctor login (pre-seeded accounts)
2.	Lab uploads a new report for the patient → show hash generated + on-chain tx
3.	Doctor requests access to that report
4.	Patient reviews request, grants 7-day access → show on-chain AccessGranted tx
5.	Doctor views the report (previously blocked, now works)
6.	Patient opens Verify page → shows ✓ Authentic
7.	Tamper demo: modify the stored file directly on disk (simulating an attacker), re-run Verify → ⚠ Hash Mismatch
8.	Patient revokes access
9.	Doctor attempts to view again → 403 Access Denied
10.	Patient opens Access History → full trail of grant/view/revoke with tx hashes
11.	(Optional) Emergency access demo with a different doctor + mandatory reason
12.	Admin audit page showing system-wide activity
________________________________________
SECTION 27 — Viva Preparation (selected 30 Q&A)
1.	Why blockchain? To make consent and integrity provable independent of trusting our own database/admins.
2.	Why not store records on blockchain? PHI on a shared ledger is a privacy and regulatory risk, and blockchains are expensive/slow for large files — hashes are enough to prove integrity.
3.	Why Ethereum-compatible over Hyperledger Fabric? Faster to build/demo for an academic timeline; Solidity/Hardhat tooling is mature and well-documented; contract-level role checks give us permissioned-like behavior without Fabric's operational complexity.
4.	Why Hardhat? Local, fast, free test network with excellent testing/debugging tools.
5.	Why Solidity? Industry-standard EVM smart contract language with strong tooling and security libraries (OpenZeppelin).
6.	Why PostgreSQL? Relational integrity for structured healthcare metadata, mature, free, well-supported by Node.
7.	Why React? Component reuse across five role-specific dashboards with shared design system.
8.	Why Node.js/Express? Same language as frontend, lightweight, huge ecosystem, easy ethers.js integration.
9.	What is SHA-256? A one-way cryptographic hash function — any change to input produces a completely different, unpredictable output, making it ideal for tamper detection.
10.	What is a smart contract? Code deployed to a blockchain that executes deterministically and whose state changes are enforced by network consensus, not a single trusted party.
11.	What is a blockchain transaction? A signed, submitted state change that gets included in a block once validated.
12.	Who pays gas? The backend's operational wallet, in this custodial prototype design.
13.	Who signs transactions? The backend, on behalf of the acting user, after verifying their identity/authorization via JWT.
14.	What happens if consent is revoked? hasAccess() immediately returns false; any subsequent access attempt is blocked at the API layer.
15.	What happens if a file is modified? Its SHA-256 hash changes; verification against the on-chain hash fails and the UI flags it clearly.
16.	How is patient privacy maintained? No PHI on-chain — only pseudonymous addresses and hashes; actual data stays encrypted off-chain behind consent checks.
17.	What happens if the backend is compromised? It's an acknowledged limitation of the custodial signer model — an attacker with backend access could, in theory, forge grants; production fix is user-held keys (see Section 14).
18.	Why not Hyperledger Fabric? Its operational complexity (org/channel/peer setup) isn't justified for a prototype demonstrating the same integrity/consent concepts.
19.	How can this become production-ready? User-held wallets, KMS-managed keys, cloud file storage, HTTPS everywhere, compliance review, possibly a permissioned chain like Fabric at scale.
20.	What are the limitations? Trusted backend signer, local-only blockchain, no real regulatory compliance review, emergency access is single-person-authorized.
(Full 30-question set available on request — the remaining ~10 follow directly from Sections 9, 10, 13, and 19 above: gas cost implications, event-based audit trail rationale, why OpenZeppelin AccessControl over a custom mapping, why events over on-chain arrays for logs, scalability of a local chain, etc.)
________________________________________
SECTION 28 — Project Report Structure
Standard 23-section report; each maps directly onto a section of this blueprint: Abstract (Section 1) → Introduction/Problem/Existing/Proposed System (Section 1) → Objectives/Scope (Section 1, 24) → Functional/Non-Functional Requirements (Sections 3–20) → System Architecture (Section 2) → Tech Stack (this whole doc's stack choices) → Database Design (Section 6) → Blockchain Design (Sections 11–14) → Smart Contract (Sections 11–13) → API Design (Section 15) → Security (Section 19) → Implementation (Section 23 phases, written up after building) → Testing (Section 25) → Results (screenshots + demo walkthrough) → Limitations (Section 32) → Future Scope (Section 24 "Future") → Conclusion → References.
________________________________________
SECTION 29 — Diagrams Required
System Architecture (Section 2 diagram) · Use Case Diagram (5 actors × main use cases from Section 1) · DFD Level 0 (single process, external entities = 5 roles) · DFD Level 1 (broken into Auth/Records/Consent/Blockchain sub-processes) · ER Diagram (Section 6) · Database Schema (table list w/ FKs) · Blockchain Architecture (Section 2, chain half) · Smart Contract Flow (function call sequence) · Record Upload Flow, Consent Flow, Record Verification Flow (Section 20 style) · Sequence Diagrams (Section 20) · Deployment Diagram (Section 30).
________________________________________
SECTION 30 — Deployment Architecture
Local development: React (Vite dev server) + Node/Express + PostgreSQL (local or Docker) + Hardhat local node, all on localhost, orchestrated optionally via docker-compose.yml.
Possible production path (not required now): Frontend → Vercel/Netlify; Backend → cloud VM/container service; Database → managed PostgreSQL (RDS/Supabase); Files → S3-compatible storage; Blockchain → a private Ethereum-compatible network (or Hyperledger Fabric if scaling to real multi-hospital consortium) with proper key management.
________________________________________
SECTION 31 — Cost and Complexity
Category	Tools
Free/local	React, Node, Express, PostgreSQL (local), Hardhat, Solidity, ethers.js, OpenZeppelin
Free tier possible	Vercel (frontend hosting), a small managed Postgres instance
May cost money (only if going to production)	Cloud VM, S3 storage, mainnet/testnet gas (irrelevant for local Hardhat)
Fully offline/local demo	Everything — the entire MVP can be demoed on a single laptop with no internet dependency, which is ideal for a viva
________________________________________
SECTION 32 — Risks and Limitations
Limitation	Production mitigation
Blockchain privacy (addresses pseudonymous, not anonymous)	zero-knowledge techniques, permissioned chain
Key management (single backend signer)	user-held wallets, HSM/MPC
Scalability (local chain, single node)	real testnet/mainnet or Fabric consortium
File storage (local disk)	S3 + CDN
Gas costs (irrelevant locally)	layer-2 or permissioned chain to remove cost concern
Healthcare compliance (HIPAA/local regs)	dedicated compliance review, BAAs with vendors
Emergency access (single-doctor authorization)	two-person rule, stricter time-boxing
Backend trust (custodial signer)	user-signed transactions
Local blockchain limitations (no real decentralization)	multi-node permissioned network in production
________________________________________
SECTION 33 — Final Locked Architecture
Layer	Technology	Responsibility
Frontend	React + Vite + Tailwind + React Router + Axios	Role-aware UI, no direct chain access
Backend	Node.js + Express	Business logic, auth, sole blockchain signer
Database	PostgreSQL	System of record for all non-cryptographic data
File Storage	Local encrypted disk (S3-ready)	Off-chain medical documents
Blockchain	Solidity + Hardhat (local) + ethers.js v6	Consent + hash + audit event ledger
Smart Contract	HealthcareRecords.sol w/ OpenZeppelin AccessControl	Enforces who can register/grant/revoke/verify
Authentication	JWT + bcrypt + RBAC	Identity and role gating
Security	AES-256 file encryption, input validation, audit logging	Defense in depth
Testing	Jest/Supertest, Hardhat/Mocha, RTL, Playwright	Coverage across all layers
Deployment	Local dev now; documented cloud path for later	Realistic for a 2nd-year academic timeline
Why this is right for this project: every layer has a clear, singular job; the blockchain is scoped to exactly what only a blockchain can credibly provide (tamper-evidence, provable consent); everything else uses boring, well-understood, well-documented tools a 2nd-year team can actually finish. Database, API, frontend, and contract designs all reference the same entities (record, consent, patient, doctor) consistently, so there's no architectural drift between layers.
________________________________________
FINAL SUMMARY
A. Final Tech Stack: React/Vite/Tailwind · Node/Express · PostgreSQL · Solidity/Hardhat/ethers.js/OpenZeppelin · JWT/bcrypt · AES-256 encrypted local file storage.
B. Final Folder Structure: See Section 21.
C. Final Database Tables: users, patients, doctors, hospitals, labs, medical_records, record_versions, consents, audit_logs, blockchain_transactions, notifications. (access_requests merged into consents — see Section 6 note.)
D. Final API List: See Section 15 (representative; full CRUD set follows same pattern per resource).
E. Final Smart Contract Functions: registerRecord, grantAccess, revokeAccess, hasAccess, verifyRecord, logAccess, grantEmergencyAccess.
F. Final Frontend Pages: See Section 3 table (22 pages across 5 roles).
G. Final Development Roadmap: 14 phases, Section 23.
H. Final Demo Flow: 12-step scenario, Section 26.
I. Final Viva Topics: Section 27 (blockchain rationale, hashing, consent lifecycle, security limitations, production path).
________________________________________
STOP — architecture complete. Say "START PHASE 1" when ready to begin implementation, and we'll go phase-by-phase without skipping ahead.

