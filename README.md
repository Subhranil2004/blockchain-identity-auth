# Academic Certificate Verification Portal

A blockchain-based academic certificate issuance and verification system using Ethereum smart contracts, MetaMask, Ganache, a React frontend, and an Express backend. Certificates are issued as on-chain credentials with tamper-proof hashes, QR codes for easy sharing, and immutable audit trails.

## Current Truth (Important)

- **Academic Certificates**: Phase 2 issues certificates with fields: student wallet, student name, college, course, grade, passing year.
- **QR Codes**: Each issued certificate generates a shareable QR code containing the credential ID and verification URL.
- **Wallet-Signed Writes**: All writes (DID registration and certificate issuance/verification) are signed by student/admin wallets via MetaMask in the frontend.
- **Backend Read-Only**: Backend does not sign blockchain transactions; write routes return HTTP 410 to enforce wallet-based signing.
- **Issuer RBAC**: Certificate issuance is restricted to authorized issuers by on-chain role-based access control.
- **Audit Trail**: All certificate operations (issued, verified) are logged immutably on-chain and queryable via backend APIs.
- **Ganache Chain ID**: Defaults to 1337 for local development.

## Project Structure

> `project_ppt.pdf`[[Canva Link]](https://canva.link/3yncwy1epa877lv) and `project_report.pdf` contain detailed architecture diagrams, flowcharts, and explanations of the system design.

```text
blockchain_auth/modern/
├── contracts/
│   ├── DIDRegistry.sol
│   ├── CredentialRegistry.sol
│   └── AuditLog.sol
├── backend/
│   ├── server.js
│   ├── routes.js
│   ├── config.js
│   ├── contractABIs.js
│   └── .env.example
├── frontend/
│   └── src/
│       ├── App.js
│       └── services/web3Service.js
├── scripts/deploy.js
├── hardhat.config.js
└── test/credential.js
```

## Prerequisites

- Node.js 16+
- Ganache running on `http://127.0.0.1:7545`
- MetaMask browser extension

## Setup

### 1. Install dependencies (root)

```bash
npm install
cd backend && npm install
cd ../frontend && npm install
```

### 2. Deploy contracts

From project root:

```bash
npx hardhat compile
npx hardhat run scripts/deploy.js --network ganache
```

Deployment order in `scripts/deploy.js`:

1. `DIDRegistry`
2. `AuditLog`
3. `CredentialRegistry(auditLogAddress)`

The script appends deployed addresses to `backend/.env`.

### 3. Backend configuration

Create `backend/.env` from `backend/.env.example` and ensure:

```env
GANACHE_RPC_URL=http://127.0.0.1:7545
GANACHE_NETWORK_ID=1337
PORT=5000

DID_REGISTRY_ADDRESS=<deployed address>
CREDENTIAL_REGISTRY_ADDRESS=<deployed address>
AUDIT_LOG_ADDRESS=<deployed address>
```

### 4. Frontend configuration

Create `frontend/.env.local`:

```env
REACT_APP_API_URL=http://localhost:5000/api/v1
```

Optional (fallback if backend health response is unavailable):

```env
REACT_APP_DID_REGISTRY_ADDRESS=<deployed address>
REACT_APP_CREDENTIAL_REGISTRY_ADDRESS=<deployed address>
REACT_APP_AUDIT_LOG_ADDRESS=<deployed address>
```

### 5. Start services

Backend:

```bash
cd backend
npm run dev
```

Frontend:

```bash
cd frontend
npm start
```

## API Reality

Base URL: `http://localhost:5000/api/v1`

### Root Endpoint

```
GET /
```

Returns system information, version, and available endpoints.

### Read routes (active)

**DID Operations:**

- `GET /did/:didHash` - Retrieve DID document with owner, DID string, public key, schema hash, and timestamps

**Credential Operations:**

- `GET /credential/:credentialId` - Retrieve credential details including issuer/subject DIDs, type, and timestamps

**Audit Trail Operations:**

- `GET /audit/did/:didHash` - Retrieve full audit trail for a specific DID with enriched data
- `GET /audit/did/:didHash/ids` - Retrieve audit record IDs only for a DID (lightweight, faster variant)
- `GET /audit/credential/:credentialId` - Retrieve audit trail for a specific credential
- `GET /audit/recent/:limit` - Retrieve the N most recent audit records system-wide

**System Health:**

- `GET /health` - Check blockchain connection status, block number, gas price, and contract deployment status

### Write routes (intentionally disabled server-side)

These routes return HTTP 410 with guidance to use MetaMask in frontend:

- `POST /did/register-schema`
- `POST /did/register`
- `POST /credential/issue`
- `POST /credential/verify`

## Smart Contract Highlights

### DIDRegistry

- Stores schema hashes and DID documents.
- `registerSchema` and `registerDID` are called directly from frontend using MetaMask signer.

### CredentialRegistry

- Constructor requires `auditLog` address.
- **Academic Certificate Issuance**:
  - `issueCredential(issuerDID, subjectDID, credentialType, credentialHash, expiryDays)`
  - Accepts credential type (set to `"ACADEMIC_CERTIFICATE"` in Phase 2)
  - Stores only hash of certificate payload on-chain (full data stored off-chain)
  - Default expiry: 3650 days (10 years)
  - Emits `CredentialIssued` event with credential ID
- **RBAC fields**:
  - `owner` (deployer)
  - `mapping(address => bool) public isIssuer` (authorized issuers only)
- **Access control**:
  - `addIssuer(address)` only by owner
  - `issueCredential(...)` requires `onlyIssuer` modifier
  - `verifyCredential(credentialId, submittedHash)` performs hash-based integrity verification before marking credential verified
  - `verifyCredentialIntegrity(credentialId, payloadHash)` is a view helper for frontend checks
  - Hash mismatch detection: rejects credentials if payload hash differs from stored `credentialHash` (detects grade/data tampering)
- **Audit integration**:
  - Logs `CREDENTIAL_ISSUED` when certificate issued
  - Logs `CREDENTIAL_VERIFIED` when certificate verified and integrity confirmed
  - Both logged to `AuditLog` with timestamp and actor address
- **Field naming**:
  - Uses `proofHash` for tamper-proof hash of certificate payload

### AuditLog

- Stores immutable audit records and provides read APIs.
- Exposes `getDIDAuditTrail` and `getRecentAuditRecords`.

## Frontend Flow

In `frontend/src/App.js`:

- `getSigner()` uses `ethers.BrowserProvider(window.ethereum)` to connect MetaMask.
- **Phase 1 - DID Registration**: Student registers identity (binds wallet to DID). The UI auto-registers a basic schema and stores an empty publicKey.
- **Phase 2 - Academic Certificate Portal**:
  - Form fields: student wallet address, student name, college, course, grade, passing year
  - Input validation: all fields required; student wallet must be valid Ethereum address
  - Off-chain payload: JSON object with certificate metadata + timestamp
  - Hash storage: `proofHash = keccak256(toUtf8Bytes(JSON.stringify(payload)))`
  - Smart contract call: `issueCredential(issuerDID, subjectDID, "ACADEMIC_CERTIFICATE", proofHash, 3650)`
  - QR generation: creates shareable QR code with URL containing credential ID
  - Verification: accepts raw credential ID or QR URL (auto-normalizes)
  - Payload storage: saved in browser localStorage for integrity checks during later verification
- **Phase 3 - Audit Inspection**: Backend read APIs for audit trails and certificate details

## Test Status

Contract test exists and passes:

```bash
$env:HARDHAT_DISABLE_VSCODE_EXTENSION_PROMPT='true'; npx hardhat test
```

Expected:

- `CredentialRegistry` test passes (`should issue a credential`).

## Troubleshooting

- If writes fail in frontend, confirm MetaMask is connected to chain ID 1337 and correct account is selected.
- If issue credential reverts with authorization error, ensure deployer (owner) has added intended issuer via `addIssuer`.
- If backend health fails, verify `backend/.env` contract addresses and Ganache RPC URL.

## Security & Integrity Verification

### Hash-Based Credential Integrity

When a credential is issued:

1. The full certificate data (name, grade, college, etc.) is stored off-chain as JSON.
2. A `keccak256` hash of this payload is computed and stored on-chain as `credentialHash`.
3. This creates a tamper-proof commitment: any change to the certificate data will produce a different hash.

During verification:

1. The verifier provides the credential ID and, if available, the full certificate payload.
2. The frontend recomputes the hash of the provided payload.
3. This computed hash is compared against the stored `credentialHash` on-chain.
4. **If hashes match**: The credential is verified and marked on-chain with a confirmation.
5. **If hashes differ**: Verification fails with "Credential data mismatch - possible tampering detected" error.

### Tampering Detection

This design detects:

- **Grade tampering**: If someone tries to change the grade after issuance, the hash will differ.
- **Payload modification**: Any modification to student name, college, course, or passing year will be detected.
- **Partial changes**: Even a single field change invalidates the hash.

### Limitations

- If the certificate payload is not available during verification (e.g., only the credential ID is provided), the UI warns and submits the on-chain hash only (integrity is not fully checked).
- The system does not prevent unauthorized data storage off-chain; it only ensures on-chain integrity of registered hashes.

## Implementation Notes

- **Certificate Storage Model**: Full certificate data (student name, college, course, grade, year, issuance date) is kept off-chain. Only a keccak256 hash of the certificate payload is stored on-chain. This ensures tamper-proof integrity while minimizing blockchain storage cost.
- **QR Code Workflow**: Each issued certificate generates a QR code containing a verification URL with the credential ID. Students and employers can scan the QR to quickly access verification.
- **No Zero-Knowledge Proof**: The system uses hash-based proof storage (`proofHash`), not cryptographic zero-knowledge verification. This is a practical trade-off for a local demo system.
- **Issuer Control**: Only authorized issuers (managed by contract owner via `addIssuer()`) can issue certificates. This prevents credential inflation.

---

## Contributors

- [Pallab Mandal](https://github.com/PallabMandal)
- [Harshit Kumar](https://github.com/Harshit-Kr01)
- [Subhranil Nandy](https://github.com/Subhranil2004)
