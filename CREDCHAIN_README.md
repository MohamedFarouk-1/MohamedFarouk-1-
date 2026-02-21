# CredChain

> **Instant, tamper-proof credential verification anchored on the blockchain.**

Issue and verify academic degrees, professional certificates, and identity claims in seconds — with cryptographic proof that survives any institution's closure, database breach, or document fraud.

---

## Why I Built This

Credential fraud is a solved problem that nobody has actually solved.

The tools exist — cryptographic hashing, immutable ledgers, public smart contracts — but the systems still in use today have employers emailing registrars and waiting days for a response. Records get lost when institutions close. Documents get forged, and the only way to catch a forgery is to contact the same institution that issued it.

I built CredChain to demonstrate what a genuinely better architecture looks like: **credential fingerprints anchored on-chain, sensitive data kept off it.** The SHA-256 hash of a credential is permanent and publicly queryable. The name, grade, and personal details never touch the blockchain. Verification takes under a second, from anywhere in the world, with no intermediary.

This project also gave me hands-on experience with a design decision I find genuinely interesting from a product standpoint: **when does blockchain actually add value vs. add complexity?** CredChain is a case where the answer is clearly yes — the immutability and permissionless verifiability are the entire point. Contrast that with AgentPay, where I had to think hard about which parts of the system belong on-chain and which don't.

**What I learned:**
- How to design a system where privacy and transparency coexist (off-chain data, on-chain proof)
- The practical tradeoffs of on-chain vs. off-chain storage for sensitive data
- Smart contract design for real-world trust models (issuer-only revocation, no upgrades needed)
- Deployed and tested on Base Sepolia testnet

---

## The Problem

Credential verification is **slow, opaque, and centralized**.

Employers email registrars. Registrars respond in days. Records get lost when institutions close. Documents get forged — and the only way to catch forgeries is to contact the same institution that issued the document. The entire process depends on intermediaries and creates bottlenecks that cost everyone time and money.

---

## The Solution

CredChain anchors credential **fingerprints** on-chain:

- **Sensitive data stays off-chain.** Names, grades, and personal details never touch a public ledger.
- **A SHA-256 hash of the credential is written to a smart contract.** This 32-byte fingerprint is permanently immutable.
- **Verification is instant.** Any party — anywhere in the world — can re-hash the document and query the contract. No emails. No waiting. No intermediary.

If the hash is on-chain: the credential is genuine. If it isn't: the document was never issued or has been revoked.

---

## Architecture

```
  ISSUER                           BLOCKCHAIN (EVM)
  ──────                           ────────────────
  Fill in form                     CredentialRegistry.sol
       │                                  │
       │  sha256(fields)                  │
       ▼                                  │
  credentialHash  ──── issue(hash) ──────▶  mapping[hash] = {
       │                                       issuer,
       │  Tx confirmed                         subject,
       ▼                                       issuedAt,
  Credential ID returned                       revoked
  (share with verifier)                     }

  VERIFIER
  ────────
  Paste Credential ID ─── verify(hash) ──▶  return (exists, valid,
                                             issuer, issuedAt, revoked)
```

---

## Smart Contract

`CredentialRegistry.sol` stores a `mapping(bytes32 => Credential)`:

```solidity
struct Credential {
    address issuer;
    address subject;
    uint256 issuedAt;
    bool    revoked;
    string  metadataURI;
}

function issue(bytes32 credentialHash, address subject, string calldata metadataURI) external;
function revoke(bytes32 credentialHash) external;   // issuer only
function verify(bytes32 credentialHash) external view returns (bool exists, bool valid, ...);
```

Key properties:
- **No duplicates** — re-issuing the same hash reverts with `AlreadyIssued`
- **Issuer-only revocation** — any other address reverts with `NotIssuer`
- **No upgrades needed** — the mapping is permanent and queryable forever

---

## Tech Stack

| Layer | Technology |
|-------|------------|
| Smart contract | Solidity 0.8.20 |
| Contract tooling | Hardhat 2.22 |
| Hashing | Node.js `crypto` — SHA-256 (off-chain) |
| Blockchain client | ethers.js v6 |
| API server | Express 4 |
| Frontend | Vanilla HTML/CSS/JS |
| Testnet | Base Sepolia |

---

## Running Locally

```bash
cd credential-verification
npm install
npm run compile
npm run node          # Terminal 1 — local blockchain
npm run deploy        # Terminal 2 — deploys + writes .env
cd backend && node server.js
```

Open `http://localhost:3001`.

---

## API Reference

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/status` | Chain connectivity + issuer balance |
| `POST` | `/api/hash` | Compute hash without writing to chain |
| `POST` | `/api/issue` | Anchor credential hash on-chain |
| `POST` | `/api/verify` | Query on-chain status |
| `POST` | `/api/revoke` | Revoke credential (issuer only) |

---

## Use Cases

| Who | What |
|-----|------|
| **Universities** | Issue degrees at graduation; revoke if awarded in error |
| **HR / Recruiters** | Verify candidate credentials in <1 second |
| **Professional bodies** | Issue/revoke licences with on-chain audit trail |
| **DAOs** | Issue membership certificates and contributor badges |
| **Online courses** | Certify completions without relying on the platform staying online |

---

## Why Blockchain and Not a Database?

| | Traditional database | CredChain |
|--|--|--|
| **Tamper-proof** | No — admin can edit rows | Yes — immutable ledger |
| **Verifiable by anyone** | No — requires issuer API | Yes — public contract |
| **Survives institution closure** | No | Yes |
| **Privacy** | Data centralised, breach risk | Sensitive data never on-chain |

---

## License

MIT
