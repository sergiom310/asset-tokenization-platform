# Asset Tokenization Platform

🇬🇧 English | [🇪🇸 Español](README.es.md)

> Blockchain infrastructure for instant, tamper-proof verification of certificates and official documents — diplomas, permits, government records — built for institutions like universities, notaries and government agencies, and extending to real estate, vehicles and other real-world assets. Mutable metadata, on-chain verification, non-transferable records where required, and an onboarding experience that requires no crypto knowledge from institutions or end users.

This is the engineering behind **[Fidukio](https://fidukio.com)**, a Colombian SaaS platform for digital certification of documents, credentials and assets — currently going through SENA's seed-funding process (Fondo Emprender) toward formal incorporation and a Base mainnet launch. This repo is documentation only: it's the map of a working, three-repo system — read it first, then go read the code.

| | |
|---|---|
| ⚙️ [**Smart Contracts**](https://github.com/sergiom310/asset-tokenization-smart-contracts) | Solidity, OpenZeppelin, Beacon Proxy (EIP-1967), ERC-721/2981/4906 |
| 🔌 [**Backend API**](https://github.com/sergiom310/asset-tokenization-api) | Node.js, Express 5, multi-tenant institutions, ethers.js |
| 💻 [**Frontend DApp**](https://github.com/sergiom310/asset-tokenization-dapp) | React 19, Ethers.js v6, wallet + no-wallet UX |

---

## The problem

Institutions that issue or manage records of real-world assets — land registries, transport secretariats, universities, notaries, IP offices, banks — depend on centralized databases and paper trails that are slow to verify, easy to forge, and expensive to reconcile across parties. Blockchain solves the trust problem, but only if the technology stays invisible to the people who actually have to use it: an institution's back-office staff, and the third party checking a record months or years later.

This platform is built around that constraint: **institutions operate through a normal web app with email/password login, end users never need a wallet to receive a record, and anyone can verify one from a public link with no account at all.**

It runs today as a shared multi-tenant platform, and the same codebase deploys as a dedicated white-label instance for a single institution or government agency that wants its own — see [Deployment model](#deployment-model).

## Use cases

| Asset type | Real-world example | Transferable? |
|---|---|---|
| Certificate | Diploma, professional license, training credential | **No — soulbound** |
| Document | Government-issued ID document, official permit | **No — soulbound** |
| Real estate | Property title / deed | Yes |
| Vehicle | Vehicle registration, technical inspection record | Yes |
| Intellectual property | Patent, trademark, copyright registration | **No — soulbound** |
| Artwork / Luxury item | Provenance and authenticity record | Yes |

---

## Architecture

```mermaid
flowchart TB
    subgraph Institution["Institution staff"]
        Login[Email/password login<br/>no wallet needed]
    end

    subgraph FE["Frontend DApp"]
        UI[Mint form · gallery ·<br/>admin panel · public verify page]
    end

    subgraph BE["Backend API"]
        Auth[JWT auth]
        Routes[Mint · verify · metadata ·<br/>transfer · admin]
        DB[(MariaDB<br/>institutions, users, credits)]
    end

    IPFS[(IPFS / Pinata)]
    Operator[Operator wallet<br/>pays gas, holds contract roles]

    subgraph Chain["Smart Contracts"]
        SC[ERC-721 + ERC-2981 + ERC-4906<br/>Beacon Proxy EIP-1967 · soulbound · role-gated]
    end

    Public(("Third-party verifier<br/>no login, no wallet")):::public
    EndUser[End user's own wallet<br/>only for transfer/approve]

    Login --> UI
    UI -->|Bearer JWT| Auth
    Auth --> Routes
    Routes <--> DB
    Routes -->|upload| IPFS
    Routes -->|sign & submit tx| Operator
    Operator -->|mint / verify / update| SC
    EndUser -.->|sign transfer/approve| SC
    Public -->|GET /verificar/:tokenId| UI
    UI -->|read-only| Routes
    Routes -->|read on-chain state| SC

    classDef public fill:#e8f4ff,stroke:#0969da
```

**Why the contract is the least-trusted-required layer:** every mint, verification and soulbound restriction that's already on-chain remains independently checkable against the contract itself, even if the backend is redeployed, rewritten, or temporarily down. The backend and frontend exist to make that trust *usable*, not to hold it themselves.

---

## Engineering approach: build, then audit yourself

Each repo's `SECURITY.md` documents its access control model — and, honestly, what was found wrong during review and fixed, rather than only what looks good:

- **Smart contracts**: found and fixed a mint-time reentrancy issue where a malicious contract minted as the recipient of a soulbound token (a certificate, a legal document) could have transferred it away during the mint callback, before the soulbound flag took effect — a checks-effects-interactions violation. [Details →](https://github.com/sergiom310/asset-tokenization-smart-contracts/blob/main/SECURITY.md)
- **Backend API**: found and fixed a near-total authentication bypass — almost every endpoint that signs a transaction with the operator wallet (which holds every contract role) had no auth middleware, while CORS was open to any origin. Every affected route now requires the correct auth. [Details →](https://github.com/sergiom310/asset-tokenization-api/blob/main/SECURITY.md)
- **Frontend DApp**: reviewed directly — no XSS sinks, no hardcoded secrets — with the one real tradeoff (JWT in `localStorage`) explained rather than hidden. [Details →](https://github.com/sergiom310/asset-tokenization-dapp/blob/main/SECURITY.md)

None of this was live in production when found. The point isn't "everything is perfect" — each `TESTING.md` states plainly what's still a gap rather than glossing over it. The point is that the review process itself — reading the code adversarially, not just functionally — is part of how this was built, and it's documented instead of hidden.

Each repo now also has an automated suite that runs in CI on every push, not just locally:

| Repo | Suite | What it covers |
|---|---|---|
| Smart contracts | 36 Hardhat tests | access control, soulbound logic, royalties, pause, the reentrancy regression above, and beacon-upgrade safety (storage layout, cross-institution upgrade propagation) |
| Backend API | 77 Bun tests | integration tests run against a **real local Hardhat node + MariaDB service** in CI (not mocked ethers calls) — see `.github/workflows/test.yml` |
| Frontend DApp | 64 Vitest unit tests | components, services and pages, including the mint-form signature flow |

Still open, tracked rather than hidden: a third-party security audit of the smart contracts (quoted, execution pending seed funding) and the Base mainnet migration — both called out explicitly below instead of implied as done.

---

## Deployment model

The [backend](https://github.com/sergiom310/asset-tokenization-api)'s institution registration, approval and credits model already supports two deployment shapes without any redesign:

- **Shared platform** — multiple institutions on one deployment, each isolated by account, as it runs today.
- **Dedicated / white-label instance** — the same codebase deployed for a single institution or government agency that wants its own instance rather than sharing infrastructure.

Data locality follows the same split either way: personal and institutional data lives only in the database, never on-chain — see the backend's [SECURITY.md](https://github.com/sergiom310/asset-tokenization-api/blob/main/SECURITY.md#data-locality--compliance-readiness) for what that means for KYC/AML and data-protection requirements. This is architectural readiness, not a compliance certification — no jurisdiction-specific legal review has been done.

---

## How I work with institutions

This is a repeatable engagement, not a one-off demo:

1. **Analysis** — map the institution's existing records and workflow, decide which asset categories apply, and scope soulbound vs. transferable per category.
2. **Development** — configure or extend the platform for the institution: branding, credit/plan setup, and any category-specific fields.
3. **DevOps** — deploy as a shared-tenant account or a dedicated white-label instance (see [Deployment model](#deployment-model) above), with the same encryption and data-locality guarantees either way.
4. **QA** — test the full mint → verify → (if applicable) transfer lifecycle against the institution's actual document types before go-live.
5. **Maintenance** — ongoing support: credit top-ups, operator wallet balance monitoring (already built in), and platform updates that never touch records already minted.

---

## Repository breakdown

### [asset-tokenization-smart-contracts](https://github.com/sergiom310/asset-tokenization-smart-contracts)

An ERC-721 contract supporting 8 asset categories, per-token soulbound flags, dynamic metadata for assets whose state changes over time, on-chain metadata history, ERC-2981 royalties, and role-based access control. A factory deploys one dedicated Beacon Proxy (EIP-1967) per institution, all pointing at a single shared `UpgradeableBeacon` — upgrading the beacon once upgrades every institution's contract at the same time, with no per-proxy transaction. Deployed and verified on Base Sepolia testnet.

### [asset-tokenization-api](https://github.com/sergiom310/asset-tokenization-api)

Express backend that lets institutions register, get approved, and mint on behalf of their end users without those users ever holding a wallet — the backend's operator wallet pays gas and signs. Handles IPFS uploads, credit-based multi-tenancy, and public no-login verification.

### [asset-tokenization-dapp](https://github.com/sergiom310/asset-tokenization-dapp)

React frontend covering the full lifecycle: minting form, personal gallery, transfer/approval flows signed by the end user's own wallet, an admin panel for institution/role/royalty management, and the public verification page third parties actually load.

---

## Tech stack

| Layer | Technology |
|---|---|
| Smart contracts | Solidity 0.8.28, OpenZeppelin 5.x, Hardhat, Beacon Proxy (EIP-1967) |
| Backend | Node.js 24, Express 5, ethers.js v6, MariaDB, JWT |
| Frontend | React 19, Vite 6, Ethers.js v6, React Router 7 |
| Storage | IPFS (Pinata) for static metadata; live API for dynamic metadata |
| Networks | Base (live on Sepolia testnet today; Ethereum/Polygon configured in Hardhat but unused) |
| Tooling | Bun, GitHub Actions (compile/lint/test on every push, all three repos) |

---

## Roadmap

- Third-party smart contract security audit — quoted, execution pending seed funding
- Base mainnet migration (live today on Base Sepolia testnet only)
- Formal external legal review of Terms of Service / Privacy Policy / SLA (in progress)

---

## About

Founded and built end-to-end by **Sergio Melo** — contracts, backend and frontend — as the technical foundation of [Fidukio](https://fidukio.com), not an isolated demo. Fidukio doesn't issue the underlying documents; institutions (schools, notaries, and other entities) do, as they always have — this platform adds the certification and verification layer on top of what they already emit.

- Fidukio: [fidukio.com](https://fidukio.com)
- GitHub: [@sergiom310](https://github.com/sergiom310)
- LinkedIn: [linkedin.com/in/sergiom310](https://linkedin.com/in/sergiom310)

Open to discussing tokenization projects, technical roles, or consulting engagements in this space.
