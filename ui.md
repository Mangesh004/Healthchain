HealthChain — UI/UX Design Specification
1. Design System
The visual identity must communicate trust, security, and clinical professionalism. 
 
•	Colors:
o	Primary: Deep clinical blue/teal. Avoid generic "Bootstrap-blue-and-white" generic CRUD looks. 
o	Neutrals: Slate greys for backgrounds, text, and borders. 
 
o	Accent: Amber, reserved strictly for pending/warning states to maintain its meaning. 
 
•	Typography: Clean, readable sans-serif fonts. Large numeric type paired with small labels for statistical cards. 
 
•	Cards: Minimalist stat cards showing values (e.g., records count, pending requests) without being overloaded with icons. 
 
•	Tables: Zebra-free with generous row heights. A status column must always use a StatusBadge. 
•	Status Badges:
o	Pending = Amber. 
 
o	Granted = Green. 
 
o	Revoked = Grey. 
 
o	Expired = Red-outline. 
 
•	Modals: Used only for destructive or consent actions (e.g., Revoke, Reject) requiring a one-line confirmation. They are not for primary navigation. 
•	Toast Notifications: Used for immediate system feedback (success/error), complementing a persistent bell dropdown for history. 
 
2. Global Layout
The application utilizes a consistent dashboard layout across all authenticated roles: 
 
•	Left Sidebar: Fixed, containing role-specific navigation links. 
 
•	Top Navbar: Contains global search, a notifications bell, and user avatar with role badge. 
 
•	Content Area: Responsive main area with consistent 24px padding. 
 
3. Important UX States
•	Pending/Warning: Amber accents used to highlight pending requests or system warnings. 
 
•	Granted/Authentic: Green indicators for active consent and verified records. 
 
•	Revoked/Archived: Grey indicators for withdrawn access or old states. 
 
•	Expired/Tampered: Red/Red-outline for lapsed consent or severe security warnings (hash mismatch). 
 
•	Loading: Skeleton loaders or subtle spinners during API calls and blockchain transactions.
4. Reusable UI Modules
Blockchain UI Elements
•	Transaction Status: Follows a strict three-state flow: Submitting → Confirming → Confirmed. 
 
•	Transaction Hash Display: Hashes are shown truncated (e.g., 0x8d91c...93af), copyable, and link to a local block explorer view. 
 
•	Verification Badge: A large, unambiguous banner indicating integrity. 
 
o	Success: Green check "✓ Record Verified — hashes match". 
 
o	Failure: Red warning "⚠ Hash Mismatch — do not trust this file". Subtle inline text is strictly forbidden for this safety-relevant signal. 
Consent UI Elements
•	Request Card: Displays the requester's identity, requested record types as checkboxes (checked if already requested), a duration selector, and explicit Grant/Reject buttons. 
 
5. Screen Specifications by Role
A. Authentication UI (Login & Registration)
•	Purpose: Secure entry point for all users.
•	UI Elements: Email/password inputs, role selection (for registration), submit buttons.
•	Actions: Register account, Login.
•	Data Displayed: Form validation text.
•	Success/Error States: Invalid credentials error, successful login redirect.
•	Allowed Roles: Unauthenticated users.
B. Patient UI
Patient Dashboard
•	Purpose: Overview of patient account. 
 
•	UI Elements: Stat cards, recent records list, pending consent requests list. 
 
•	Actions: Grant/reject requests inline. 
 
•	Allowed: PATIENT.
Patient Records & Record Details
•	Purpose: List all owned records and view specific metadata. 
 
•	UI Elements: Table of records, detail view card, action buttons. 
 
•	Actions: Filter, open detail, Verify, download, share. 
 
•	Allowed: PATIENT.
Upload Record
•	Purpose: Patient self-uploads a document. 
 
•	UI Elements: File dropzone, record type selector. 
 
•	Actions: Upload, tag type. 
 
•	Allowed: PATIENT.
Consent Requests & Active Consents
•	Purpose: Manage incoming and existing access. 
 
•	UI Elements: Consent request cards, Active consent table. 
 
•	Actions: Grant/Reject (Requests), Revoke (Active). 
 
•	Allowed: PATIENT.
Access History
•	Purpose: Audit view of who viewed what, and when. 
 
•	UI Elements: Audit log table. 
 
•	Actions: Filter by doctor/date. 
 
•	Allowed: PATIENT.
Verify Record
•	Purpose: Manual integrity check. 
 
•	UI Elements: Verification Badge (Pass/Fail). 
 
•	Actions: Trigger hash comparison. 
 
•	Allowed: PATIENT.
C. Doctor UI
Doctor Dashboard & Patients List
•	Purpose: View assigned patients and pending access requests. 
 
•	UI Elements: Patient list table, stat cards. 
 
•	Actions: Open patient profile. 
 
•	Allowed: DOCTOR.
Patient Details & Access Requests
•	Purpose: View authorized patient records and request more access. 
 
•	UI Elements: Record list (authorized only), Request Access modal. 
 
•	Actions: View record, request access. 
 
•	Allowed: DOCTOR.
Upload Diagnosis
•	Purpose: Attach a new record to a patient. 
 
•	UI Elements: File upload form. 
 
•	Actions: Upload. 
 
•	Allowed: DOCTOR.
D. Hospital UI
Dashboard & Doctor Management
•	Purpose: Manage institution staff and view stats. 
 
•	UI Elements: Stat cards, doctor table. 
 
•	Actions: Add/remove doctors. 
 
•	Allowed: HOSPITAL.
Access Logs
•	Purpose: View all activity by hospital staff. 
 
•	UI Elements: Comprehensive audit table. 
 
•	Allowed: HOSPITAL.
E. Lab UI
Dashboard & Upload Lab Report
•	Purpose: View upload queue and attach reports to patients. 
 
•	UI Elements: Recent uploads table, patient search, file dropzone. 
 
•	Actions: Search patient, upload. 
 
•	Allowed: LAB.
F. Admin UI
User Management & System Audit
•	Purpose: Full system administration and chain event verification. 
 
•	UI Elements: User CRUD tables, full system audit log table with chain events. 
 
•	Actions: Approve/suspend accounts, filter/export logs. 
 
•	Allowed: ADMIN.
6. Navigation and User Flows (Mermaid)
1. Login Flow
Code snippet
graph LR
    A[Enter Credentials] --> B{Valid?}
    B -- Yes --> C[Check Role]
    C --> D[Redirect to Role Dashboard]
    B -- No --> E[Show Error Toast]
2. Patient Uploads Record Flow
Code snippet
graph LR
    A[Select File & Type] --> B[Click Upload]
    B --> C[Show Loading/Tx Status]
    C --> D[File Hashed & Encrypted]
    D --> E[Blockchain Tx Confirmed]
    E --> F[Show Success Toast]
3. Doctor Requests Access & Patient Grants Consent
Code snippet
graph TD
    A[Doctor: Clicks 'Request Access'] --> B[Select Record & Duration]
    B --> C[Submit Request]
    C --> D[Patient: Receives Notification]
    D --> E[Patient: Reviews Request Card]
    E --> F[Patient: Clicks 'Grant']
    F --> G[Blockchain Tx: Submitting -> Confirmed]
    G --> H[Doctor: Receives Notification & Access]
4. Record Verification Flow
Code snippet
graph TD
    A[User: Clicks 'Verify Record'] --> B[Backend Decrypts & Hashes]
    B --> C[Fetch On-Chain Hash]
    C --> D{Match?}
    D -- Yes --> E[Show Green Authentic Badge]
    D -- No --> F[Show Red Tampered Warning]
7. Frontend Component Structure
The UI must be built adhering to the specified React folder structure: 
 
Plaintext
frontend/
├── src/
│   ├── components/       
│   │   ├── common/       # Buttons, Cards, Tables, Modals, StatusBadges
│   │   ├── records/      # Record lists, Upload dropzones
│   │   ├── consent/      # Request cards, Duration selectors
│   │   └── blockchain/   # TransactionStatus, HashDisplay, VerifyBadge
│   ├── pages/
│   │   ├── auth/         # Login, Register
│   │   ├── patient/      # Dashboard, Records, History
│   │   ├── doctor/       # Dashboard, Patients, Upload
│   │   ├── hospital/     # Dashboard, Staff Management
│   │   ├── lab/          # Dashboard, Upload
│   │   └── admin/        # User Management, System Audit
│   ├── layouts/          # DashboardLayout (sidebar+navbar), AuthLayout
│   ├── services/         # api.js, authService, recordService, consentService, blockchainService
│   ├── hooks/            # useAuth, useRole, useNotifications
│   ├── context/          # AuthContext, NotificationContext
│   ├── utils/            # formatDate, hashDisplay, roleGuard
│   ├── assets/           # Images, generic styles
│   └── App.jsx           # Root routing

