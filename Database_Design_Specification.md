HealthChain Database Design Specification
1. Database Overview
•	Database: PostgreSQL. 

•	Purpose: Acts as the primary system of record for all non-cryptographic data, relational metadata, user profiles, and UI-facing audit logs. 

•	Data Boundaries:


o	PostgreSQL: Stores patient/doctor/hospital profiles, PII, off-chain record metadata (file pointers), UI notifications, full audit logs, and a fast-read mirror of consent states. 

o	Encrypted File Storage: Stores the actual medical documents (PDFs, images). Never stored directly in the database. 

o	Blockchain: Stores only cryptographic proofs—record hashes, record IDs, pseudonymous wallet addresses, consent statuses, access events, and timestamps. 

2. Complete Schema
The HealthChain database consists of the following exact tables: 


1.	users

2.	patients

3.	doctors

4.	hospitals

5.	labs

6.	medical_records

7.	record_versions

8.	consents

9.	audit_logs

10.	blockchain_transactions

11.	notifications

Note on access_requests: As recommended in the blueprint for MVP simplicity, access_requests is merged into the consents table by utilizing the PENDING status to represent the request state, avoiding duplicate tables. 


3. Every Table
users
Column	Type	Key	Nullable	Unique	Default	Purpose DOCX
id	UUID	PK	No	Yes	Gen	User identity
name	varchar	-	No	No	-	Display name
email	varchar	-	No	Yes	-	Login
password_hash	varchar	-	No	No	-	bcrypt hash
role	enum	-	No	No	-	RBAC (PATIENT, DOCTOR, HOSPITAL, LAB, ADMIN)
wallet_address	varchar	-	Yes	Yes	-	On-chain identifier
hospital_id	UUID	FK	Yes	No	-	Doctor/lab affiliation
created_at	timestamp	-	No	No	Now	Creation time
patients
Column	Type	Key	Nullable	Unique	Default	Purpose DOCX
id	UUID	PK, FK	No	Yes	-	Extends users where role=PATIENT
date_of_birth	date	-	Yes	No	-	Patient DOB
blood_group	varchar	-	Yes	No	-	Emergency-access field
emergency_contact	varchar	-	Yes	No	-	Contact details
doctors
Column	Type	Key	Nullable	Unique	Default	Purpose DOCX
id	UUID	PK, FK	No	Yes	-	Extends users
specialization	varchar	-	No	No	-	Doctor specialty (implied)
license_no	varchar	-	No	Yes	-	Medical license (implied)
hospital_id	UUID	FK	No	No	-	Affiliated hospital
hospitals
Column	Type	Key	Nullable	Unique	Default	Purpose DOCX
id	UUID	PK, FK	No	Yes	-	Extends users
address	varchar	-	No	No	-	Physical location
registration_no	varchar	-	No	Yes	-	Institution registration
labs
Column	Type	Key	Nullable	Unique	Default	Purpose DOCX
id	UUID	PK, FK	No	Yes	-	Extends users
address	varchar	-	No	No	-	Physical location
accreditation_no	varchar	-	No	Yes	-	Lab accreditation
medical_records
Column	Type	Key	Nullable	Unique	Default	Purpose DOCX
id	UUID	PK	No	Yes	Gen	Record identity
patient_id	UUID	FK	No	No	-	Owner (references patients)
uploaded_by	UUID	FK	No	No	-	Doctor/lab/patient who uploaded
hospital_id	UUID	FK	Yes	No	-	Affiliated hospital
record_type	enum	-	No	No	-	Prescription/Blood Test/X-Ray...
file_url	varchar	-	No	No	-	Pointer to encrypted storage
file_hash	varchar(64)	-	No	No	-	SHA-256, mirrors on-chain hash
onchain_record_id	varchar	-	Yes	Yes	-	Maps to smart contract record ID
status	enum	-	No	No	ACTIVE	ACTIVE or ARCHIVED
created_at	timestamp	-	No	No	Now	Creation time
updated_at	timestamp	-	No	No	Now	Last updated time
record_versions
Column	Type	Key	Nullable	Unique	Default	Purpose DOCX
id	UUID	PK	No	Yes	Gen	Version identity
record_id	UUID	FK	No	No	-	References medical_records
file_url	varchar	-	No	No	-	Pointer to version file
file_hash	varchar(64)	-	No	No	-	SHA-256 of version
version_no	int	-	No	No	-	Version increment
created_at	timestamp	-	No	No	Now	Creation time
consents
Column	Type	Key	Nullable	Unique	Default	Purpose DOCX
id	UUID	PK	No	Yes	Gen	Consent identity
patient_id	UUID	FK	No	No	-	Granter
requester_id	UUID	FK	No	No	-	Doctor/hospital
record_id	UUID	FK	Yes	No	-	null = whole-patient scope
status	enum	-	No	No	PENDING	PENDING, GRANTED, REJECTED, REVOKED, EXPIRED
start_time	timestamp	-	Yes	No	-	Access start
expiry_time	timestamp	-	Yes	No	-	Access end
transaction_hash	varchar(66)	-	Yes	Yes	-	On-chain tx proof
created_at	timestamp	-	No	No	Now	Creation time
audit_logs
Column	Type	Key	Nullable	Unique	Default	Purpose DOCX
id	UUID	PK	No	Yes	Gen	Log identity
user_id	UUID	FK	No	No	-	Actor
action	varchar	-	No	No	-	Action performed
target_type	varchar	-	No	No	-	Entity type affected
target_id	UUID	-	No	No	-	Entity ID affected
metadata	jsonb	-	Yes	No	-	Additional data
ip_address	varchar	-	Yes	No	-	Network origin
created_at	timestamp	-	No	No	Now	Action time
blockchain_transactions
Column	Type	Key	Nullable	Unique	Default	Purpose DOCX
id	UUID	PK	No	Yes	Gen	Identity
tx_hash	varchar	-	No	Yes	-	Blockchain hash
related_entity_type	varchar	-	No	No	-	RECORD or CONSENT
related_entity_id	UUID	-	No	No	-	Linked record/consent ID
action	varchar	-	No	No	-	Transaction intent
block_number	int	-	Yes	No	-	Chain block
status	varchar	-	No	No	-	Tx status
created_at	timestamp	-	No	No	Now	Action time
notifications
Column	Type	Key	Nullable	Unique	Default	Purpose DOCX
id	UUID	PK	No	Yes	Gen	Notification identity
user_id	UUID	FK	No	No	-	Recipient
type	varchar	-	No	No	-	Event type
message	varchar	-	No	No	-	Display text
is_read	boolean	-	No	No	False	Read status
created_at	timestamp	-	No	No	Now	Time sent
(Note: Data types like string limits or default implementations not explicitly sized in the blueprint are standard assumptions based on the provided generic types.) 


4. Relationships
Code snippet
erDiagram
    USERS ||--o| PATIENTS : "1:1 role=PATIENT"
    USERS ||--o| DOCTORS : "1:1 role=DOCTOR"
    USERS ||--o| HOSPITALS : "1:1 role=HOSPITAL"
    USERS ||--o| LABS : "1:1 role=LAB"
    
    HOSPITALS ||--o{ DOCTORS : "employs"
    HOSPITALS ||--o{ LABS : "employs"
    
    PATIENTS ||--o{ MEDICAL_RECORDS : "owns"
    USERS ||--o{ MEDICAL_RECORDS : "uploaded_by"
    HOSPITALS ||--o{ MEDICAL_RECORDS : "hosted_at"
    
    MEDICAL_RECORDS ||--o{ RECORD_VERSIONS : "has"
    
    PATIENTS ||--o{ CONSENTS : "grants"
    USERS ||--o{ CONSENTS : "requested_by"
    MEDICAL_RECORDS ||--o{ CONSENTS : "authorizes"
    
    USERS ||--o{ AUDIT_LOGS : "performs"
    USERS ||--o{ NOTIFICATIONS : "receives"
5. Indexes & Constraints
The following indexes are explicitly required by the blueprint: 


•	medical_records(patient_id)

•	medical_records(file_hash)

•	consents(patient_id, status)

•	consents(requester_id, status)

•	audit_logs(user_id, created_at)

•	users(email) (Must be UNIQUE)

•	users(wallet_address) (Must be UNIQUE)

6. Medical Record Design
•	Storage: The actual file is never stored in PostgreSQL; it lives in the Encrypted File Storage. The database only holds a file_url pointer. 

•	Integrity Hash: The file_hash stores the SHA-256 hash of the unencrypted file. This column strictly mirrors the hash stored on the blockchain. 

•	On-Chain ID: onchain_record_id maps the relational record to the smart contract's recordId. 

•	Versioning: Supported via record_versions as an advanced feature, where each version has its own URL, hash, and version number. 

•	Status: Stored as an ENUM (ACTIVE or ARCHIVED). 

7. Consent Database Design
•	Statuses: Uses a unified consents table flowing through PENDING (request phase) → GRANTED or REJECTED (decision) → REVOKED or EXPIRED (termination). 

•	Scope:


o	Record-specific: The record_id FK points to a specific document. 

o	Whole-patient: A record_id = 0/null sentinel value indicates access is granted to all of a patient's records. 

•	Expiry: Stored in expiry_time. A scheduled job or lazy-read check marks the row as EXPIRED once this time passes. 

•	Proof: The transaction_hash stores the receipt of the on-chain event. 

8. Audit & Blockchain Transaction Storage
•	audit_logs: Captures UI/UX activity and generic system events (logins, record views, file downloads, consent requests). Rich metadata like IP address and user agents can be stored here. 

•	blockchain_transactions: Strictly stores data about state-changing transactions sent to the blockchain (RecordRegistered, AccessGranted, AccessRevoked, EmergencyAccessGranted). 

•	Distinction: Anything needing third-party provability goes to the blockchain (and blockchain_transactions); standard system activity stays in audit_logs. 

9. Database Security
•	Passwords: Stored exclusively as password_hash using bcrypt (12 rounds); no plaintext passwords. 

•	Data Isolation: No medical files (PHI) or file contents are stored in PostgreSQL. 

•	Private Keys: Not specified in the database blueprint; the operational backend signer key lives in .env/secrets. 

•	Queries: Must use parameterized ORM queries to prevent SQL injection. 

•	Access Control: Handled by the backend application logic (Express + Services) enforcing RBAC before database access. 

10. Database ↔ Backend Integration
•	Flow: React UI makes a REST API call to Express Controllers → Business Services → PostgreSQL ORM Models. Controllers never query the database directly. 

•	Authentication: Checks the users table via email and verifies password_hash. 

•	Medical Records: Resolves metadata and authorization before calling the file service to decrypt and serve the file. 

•	Consent: The database acts as a fast-read cache. However, on actual file read, the backend confirms the consent status on-chain (hasAccess()) rather than trusting the database flag alone. 

•	Events: Triggers writes to notifications and audit_logs during core user actions. 

11. Database ↔ Blockchain Boundary
Data	PostgreSQL	Blockchain	Authority DOCX
File Contents	Off-chain (Encrypted)	❌ Never	-
Patient/Doctor Names	✅ Yes	❌ Never	PostgreSQL
File Hash	✅ DB Mirror (file_hash)	✅ bytes32 fileHash	Blockchain
Record ID	✅ onchain_record_id	✅ bytes32 recordId	Blockchain
Consent Status	✅ DB Mirror (status)	✅ bool active	Blockchain
Expiry	✅ DB Mirror (expiry_time)	✅ uint256 expiry	Blockchain
Access Logs	✅ UI activity	✅ RecordAccessed event	Blockchain (for proofs)
12. Migration Structure
To ensure foreign key constraints function properly, migrations (stored in database/migrations/) should execute in this order:


1.	users

2.	hospitals (Needs users for id PK/FK, but users may need hospitals for hospital_id. Note: Handle circular dependency by making hospital_id in users nullable and adding the FK constraint after hospitals creation.)

3.	patients, doctors, labs

4.	medical_records

5.	record_versions

6.	consents

7.	audit_logs

8.	blockchain_transactions

9.	notifications

13. Seed Data
For the prototype/demo phase, seed accounts must represent each role: 


•	1x Admin (Full system audit access).

•	1x Hospital Admin.

•	2x Doctors (Assigned to the hospital).

•	1x Lab.

•	2x Patients (With initialized off-chain profiles).

•	(Not specified in the source blueprint: Additional dummy data like pre-uploaded historical records, though useful, is not strictly mandated by the core architecture document).

14. Backup & Production Considerations
•	Managed Hosting: For production, use managed PostgreSQL (e.g., RDS, Supabase). 

•	(Not specified in the source blueprint: Specific backup frequency, replication strategies, or migration tool requirements (e.g., Prisma vs Sequelize), other than ORMs should be parameterized).

15. Final Implementation Checklist
•	[ ] PostgreSQL is configured as the primary system of record.

•	[ ] All 11 tables exist with correct columns and data types.

•	[ ] access_requests is properly merged into consents using PENDING.

•	[ ] Medical files are NOT stored in the database.

•	[ ] Passwords use bcrypt hashing.

•	[ ] file_hash and onchain_record_id mirror the blockchain structures.

•	[ ] Required indexes (especially patient_id and file_hash) are applied.

•	[ ] recordId = 0/null is supported for whole-patient consent.

•	[ ] Database migrations are ordered to respect foreign keys.

