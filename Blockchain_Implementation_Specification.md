HealthChain Blockchain Implementation Specification
1. Blockchain Architecture
•	Network: Ethereum-compatible blockchain utilizing a local Hardhat network for development. 

•	Smart Contract Language: Solidity. 

•	Integration Library: ethers.js v6. 

•	Security Framework: OpenZeppelin. 

•	Trust Boundary: The Node.js/Express backend serves as the single choke point and trust boundary. 

•	Signer Model: The backend acts as the sole operational signer in a custodial model, signing all transactions on behalf of users. 

2. On-Chain vs Off-Chain Data
Data Type	Storage Location	Rationale DOCX
Record hashes & IDs	On-chain	Cryptographic proof of integrity. 

Patient/Doctor addresses	On-chain	Pseudonymous identifiers for mapping consent. 

Consent status & expiry	On-chain	Immutable proof of access rights. 

Access/Grant/Revoke events	On-chain	Tamper-evident audit trail. 

Medical files (PDFs/images)	Encrypted File Storage	Never on-chain due to size and privacy. 

Patient/Doctor names, PII	PostgreSQL	Never on-chain to prevent privacy leakage. 

Diagnoses, Prescriptions	PostgreSQL	Never on-chain; handled as off-chain metadata. 

3. Smart Contract (HealthcareRecords.sol)
The contract is intentionally minimal, functioning as a consent and integrity ledger. 


Structures
•	Record: recordId (bytes32), patient (address), uploader (address), fileHash (bytes32), timestamp (uint256), active (bool). 

•	Consent: patient (address), doctor (address), recordId (bytes32, where 0x0 = all records), expiry (uint256), active (bool). 

Mappings
•	records: Maps bytes32 (record ID) to the Record struct. 

•	consents: Maps bytes32 (derived from keccak256(patient, doctor, recordId)) to the Consent struct. 

Functions
•	registerRecord(recordId, patient, hash): Stores a new record's hash. 

•	grantAccess(patient, doctor, recordId, expiry): Patient (via backend) authorizes access. 

•	revokeAccess(patient, doctor, recordId): Patient withdraws access immediately. 

•	hasAccess(patient, doctor, recordId): Read-only check to verify active, unexpired consent. 

•	verifyRecord(recordId, hash): Compares a provided hash against the stored hash for integrity. 

•	logAccess(patient, doctor, recordId): Records a successful read event. 

•	grantEmergencyAccess(patient, doctor, reason): Time-boxed, logged override restricted to emergency roles. 

4. Events
Every state change emits an indexable event. 


•	RecordRegistered: Emitted when a new record hash is saved (Triggers: registerRecord()). 

•	AccessGranted: Emitted when a patient approves access (Triggers: grantAccess()). 

•	AccessRevoked: Emitted when a patient withdraws access (Triggers: revokeAccess()). 

•	RecordAccessed: Emitted when a doctor successfully views a file (Triggers: logAccess()). 

•	EmergencyAccessGranted: Emitted when an emergency override occurs (Triggers: grantEmergencyAccess()). 

5. Smart Contract Security
•	OpenZeppelin AccessControl: Used for role administration. 

•	Roles: UPLOADER_ROLE (doctors/labs/patients), EMERGENCY_ROLE (emergency doctors), DEFAULT_ADMIN_ROLE (contract deployer). 

•	ReentrancyGuard: Applied as defense-in-depth on state-changing functions. 

•	Integer Overflow: Protected natively by Solidity ^0.8.x. 

•	Patient Authorization: grantAccess and revokeAccess require msg.sender == patient (handled via backend passing verified patient requests). 

•	Storage Limitations: No PII on-chain and no unbounded arrays. 

•	Key Management: The deployer key (DEFAULT_ADMIN_ROLE) is kept separate from the operational backend signer key. Keys are stored securely in environment secrets. 

6. Backend Integration
Rule: blockchainService.js is the only module permitted to construct and sign blockchain transactions. Controllers must not interact directly with ethers.js. 


Code snippet
graph LR
    A[React] -->|REST API| B[Express Controller]
    B --> C[Service Layer]
    C --> D[blockchainService.js]
    D -->|ethers.js v6| E[HealthcareRecords.sol]
    E --> F[Hardhat Node]
7. Transaction Lifecycle
1.	Creation & Signing: blockchainService.js constructs the transaction and signs it using the backend's operational private key. 

2.	Submission: The transaction is broadcasted to the Hardhat network. 

3.	Confirmation: The backend awaits confirmation via await tx.wait(). 

4.	Database Sync: The transaction receipt (including block number and tx hash) is saved to the PostgreSQL blockchain_transactions table and mirrored to the corresponding record/consent row. 

5.	Frontend Status: Reflected in the UI strictly as Submitting → Confirming → Confirmed. 

8. Record Hash Flow
•	Upload: Original File → SHA-256 hash generated → Hash stored in PostgreSQL file_hash AND registered on-chain via registerRecord(). 

•	Verification: To detect tampering, verifyRecord() takes the decrypted file's current SHA-256 hash and checks it against the authoritative on-chain fileHash. A match is VALID, a mismatch flags HASH MISMATCH. 

9. Consent Flow
Code snippet
sequenceDiagram
    participant D as Doctor
    participant P as Patient
    participant B as Backend
    participant DB as PostgreSQL
    participant C as Smart Contract
    D->>B: Request Access
    B->>DB: Insert PENDING request
    P->>B: Grant Access
    B->>C: grantAccess() transaction
    C-->>B: tx hash & receipt
    B->>DB: Update to GRANTED
    D->>B: View Record
    B->>C: hasAccess() check
    C-->>B: returns true
    P->>B: Revoke Access
    B->>C: revokeAccess() transaction
    C-->>B: access immediately blocked
•	Scope: Supported via recordId = 0/null (whole-patient) or specific recordId. 

•	Expiry: Checked dynamically within hasAccess() (block.timestamp < expiry); naturally expires without requiring a transaction. 

10. Emergency Access
•	Role: Caller must possess EMERGENCY_ROLE. 

•	Constraints: A reason string is mandatory; access is strictly time-boxed. 

•	Logging: Triggers an EmergencyAccessGranted event on-chain and writes to the PostgreSQL audit log. 

•	Visibility: The event is permanently visible to the patient in their Access History. 

11. Deployment
•	Node: Deployed to a local Hardhat network. 

•	Export: Deployment scripts generate the ABI and contract address, exporting them to deployments/localhost.json. 

•	Environment Variables: The backend requires BLOCKCHAIN_RPC_URL, PRIVATE_KEY, and CONTRACT_ADDRESS to interact with the deployed contract. 

12. Blockchain Testing
Test suites (using Hardhat/Mocha/Chai) must verify: 


•	Authorized vs. unauthorized record registration. 

•	Only patients can grant/revoke their consent. 

•	Revoking access immediately blocks hasAccess(). 

•	Expired consent correctly denies access. 

•	verifyRecord() correctly flags tampered hashes. 

•	Role restrictions for EMERGENCY_ROLE. 

•	Accurate event emission for all state changes. 

13. Error Handling
•	Reverts: Transactions wrapped in try/catch within blockchainService.js. The backend parses smart contract revert reasons and maps them to clean HTTP 4xx responses for the frontend, avoiding raw stack traces. 

14. Security Limitations & Production Evolution
•	Prototype Limitation: The backend operational signer is fully trusted and signs on behalf of the patient (custodial model). 

•	Evolution Path: Future iterations will migrate to user-held wallets (MetaMask) or HSM/MPC key management, removing the backend's ability to forge consent. Production deployment may target a permissioned network (e.g., Hyperledger Fabric) or private Ethereum instance to satisfy compliance requirements, paired with AWS KMS for encryption. 

15. Blockchain Implementation Checklist
•	[ ] Implement HealthcareRecords.sol using Solidity ^0.8.x and OpenZeppelin AccessControl. 

•	[ ] Define Record and Consent structs and mappings. 

•	[ ] Implement all 7 required functions (registerRecord, grantAccess, revokeAccess, hasAccess, verifyRecord, logAccess, grantEmergencyAccess). 

•	[ ] Implement and trigger all 5 events. 

•	[ ] Apply ReentrancyGuard and role modifiers. 

•	[ ] Ensure no PII or unbounded arrays exist in the contract. 

•	[ ] Create Hardhat deployment script that exports ABI/Address to deployments/localhost.json. 

•	[ ] Configure blockchainService.js in Node.js to read environment secrets and manage all ethers.js interactions. 

•	[ ] Ensure hasAccess() is called prior to returning decrypted files. 

•	[ ] Ensure backend correctly parses reverts to HTTP 4xx errors. 

•	[ ] Write Mocha/Chai tests covering all scenarios specified in Section 12. 

