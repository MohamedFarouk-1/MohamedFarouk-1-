# AgentPay

> **On-chain payment infrastructure for autonomous AI agents.**

An AI agent that can think but can't pay is fundamentally incomplete. AgentPay solves the last-mile problem: funds deposit USDC into a vault, authorize specific agents, and set hard spending limits — agents then transact autonomously within those limits, no human approval required per transaction.

---

## Why I Built This

Autonomous AI agents are becoming real, but they have a critical gap: they have no native way to hold or spend money. Every payment still requires a human in the loop.

I designed AgentPay to test whether **on-chain spending limits could replace manual approval flows** — giving agents real financial autonomy within defined constraints. The architecture ended up being a full-stack system: Solidity vault contracts on Base and Arbitrum, a FastAPI backend for off-chain state, and a React frontend for fund and agent management.

After shipping v1, I dug deeper into the competitive landscape and discovered that Coinbase's x402 protocol was solving the same problem at the infrastructure level — with distribution advantages I couldn't match as a solo builder. That informed a deliberate decision to **pivot toward product roles** in AI infrastructure rather than compete directly. The codebase stands as a technical proof-of-concept and a record of how I think through system design under real constraints.

**What I learned:**
- On-chain authorization patterns (agent allowances, daily limits, per-tx caps)
- Where smart contracts make sense vs. where they add unnecessary friction
- How to evaluate a market and recognize when distribution beats technology
- The x402 / HTTP payment channel design and why it's a stronger primitive

---

## Architecture

```
agentpay/
├── contracts/   # Solidity smart contracts (Hardhat)
├── backend/     # REST API (FastAPI + SQLAlchemy)
└── frontend/    # Web app (React + Vite + wagmi)
```

### contracts/
Solidity 0.8.20 contracts deployed on Base and Arbitrum.
- `AgentPaymentVault.sol` — core vault: deposits, withdrawals, agent authorization, per-transaction and daily spending limits, platform fee routing
- `MockUSDC.sol` — ERC-20 mock for local development and testing
- Deployment scripts and a 50+ case Hardhat test suite

### backend/
FastAPI service for off-chain state that the contracts don't store.
- Fund and agent registry
- Transaction history indexing
- Wallet-signature authentication (eth-account)
- Async SQLAlchemy (SQLite in dev, PostgreSQL in production)
- Docker-ready

### frontend/
React 18 + Vite single-page app.
- RainbowKit wallet connection (MetaMask, WalletConnect, Coinbase Wallet)
- wagmi v2 hooks for all on-chain interactions (deposit, withdraw, authorize agent, set limits)
- Bot/agent marketplace
- Transaction history dashboard
- TailwindCSS responsive UI

---

## How It Works

1. **Fund deposits USDC** into the vault contract and authorizes one or more agents.
2. **Fund sets limits** — a daily cap and a per-transaction cap for each agent.
3. **Agent calls `purchaseOnBehalf()`** — the contract validates authorization and limits, deducts a configurable platform fee (default 2%), and transfers USDC to the service provider.
4. **Backend records** the transaction; the frontend shows it in the history dashboard.

---

## Supported Networks

| Network          | Chain ID |
|-----------------|----------|
| Base Mainnet    | 8453     |
| Base Sepolia    | 84532    |
| Arbitrum One    | 42161    |
| Arbitrum Sepolia| 421614   |

---

## Quick Start

### 1. Contracts
```bash
cd contracts
npm install
cp .env.example .env   # fill in PRIVATE_KEY and RPC/API keys
npm test
npm run deploy:base-sepolia
```

### 2. Backend
```bash
cd backend
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt
cp .env.example .env
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

API docs at `http://localhost:8000/docs`.

### 3. Frontend
```bash
cd frontend
npm install
cp .env.example .env
npm run dev
```

App at `http://localhost:5173`.

---

## Security

- Reentrancy protection via OpenZeppelin `ReentrancyGuard`
- Fund balances are isolated — no cross-fund access
- Solidity 0.8.20 built-in overflow/underflow protection
- Daily limits reset automatically; per-transaction cap enforced on every call
- Platform fee capped at 10% in the contract

---

## License

MIT
