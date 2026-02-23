# SSS Forge

SSS Forge is a reference implementation of the Solana Stablecoin Standard (SSS) with a modular structure for:

- on-chain program (Anchor/Rust)
- TypeScript SDK + operator CLI
- backend watchtower services
- admin dashboard frontend

Live frontend (latest deployed UI): https://sss-forge.vercel.app

## Table of Contents

- [Overview](#overview)
- [Standards and Presets](#standards-and-presets)
- [Architecture](#architecture)
- [Repository Structure](#repository-structure)
- [Bounty Deliverables Mapping](#bounty-deliverables-mapping)
- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [SDK Usage Example](#sdk-usage-example)
- [CLI Command Reference](#cli-command-reference)
- [Backend API Overview](#backend-api-overview)
- [Devnet Deployment Notes](#devnet-deployment-notes)
- [Documentation Index](#documentation-index)
- [Current Status](#current-status)
- [Troubleshooting](#troubleshooting)
- [License](#license)

## Overview

This monorepo follows a three-layer model inspired by the bounty specification:

1. Base SDK and core token lifecycle operations.
2. Optional compliance modules and services.
3. Opinionated standards (SSS-1 and SSS-2) for common issuance scenarios.

The project includes a full developer/operator surface:

- SDK (`sdk/`) for application integration.
- CLI (`sss-token`) for fast operator workflows.
- Backend watchtower (`backend/`) with API, audit, and webhook hooks.
- Frontend dashboard (`src/`) for admin visibility.

## Standards and Presets

| Preset | Purpose | Key Capabilities |
| --- | --- | --- |
| `sss-1` | Minimal stablecoin profile | Mint, burn, freeze/thaw, pause/unpause, metadata-oriented flow |
| `sss-2` | Compliance-oriented profile | SSS-1 + blacklist actions + seize flow + transfer-hook style compliance gating |

## Architecture

| Layer | Path | Stack | Responsibility |
| --- | --- | --- | --- |
| Layer 1 | `programs/sss_forge` | Rust, Anchor 0.30.1, Token-2022 libs | Core on-chain logic, roles, compliance instructions |
| Layer 2 | `sdk/` | TypeScript, web3.js, commander | Developer SDK and operator CLI |
| Layer 3 | `backend/` | Node.js, Express, Docker | Watchtower API, compliance service, indexing hooks |
| UI | `src/` | React + Vite | Admin dashboard for operational workflows |

## Repository Structure

```text
SSS-Forge/
|- programs/
|  |- sss_forge/
|     |- src/
|     |- Cargo.toml
|- sdk/
|  |- src/
|  |- bin/
|  |- package.json
|- backend/
|  |- src/
|  |- Dockerfile
|  |- package.json
|- src/
|  |- components/
|  |- data/
|- docs/
|  |- ARCHITECTURE.md
|  |- SDK.md
|  |- OPERATIONS.md
|  |- SSS-1.md
|  |- SSS-2.md
|  |- COMPLIANCE.md
|  |- API.md
|- tests/
|  |- sss_forge.ts
|- Anchor.toml
|- Cargo.toml
|- docker-compose.yml
|- package.json
|- README.md
```

## Bounty Deliverables Mapping

| Deliverable Area | Location in Repo |
| --- | --- |
| On-chain configurable stablecoin program | `programs/sss_forge/src` |
| TypeScript SDK (`@stbr/sss-token`) | `sdk/src/index.ts` |
| Admin CLI (`sss-token`) | `sdk/src/cli.ts`, `sdk/bin/sss-token.js` |
| Backend mint/burn and compliance services | `backend/src/services` |
| Webhook and monitoring API | `backend/src/index.ts`, `backend/src/services/webhook.service.ts` |
| Operator and architecture docs | `docs/*.md` |
| Frontend admin example | `src/components/*` |
| Integration test scaffolding | `tests/sss_forge.ts` |

## Prerequisites

Recommended environment:

- Node.js 20+
- npm 10+
- Rust toolchain (stable)
- Solana CLI (1.18+)
- Anchor CLI (0.30.1 preferred for this repo)
- Docker Desktop (optional, for backend container run)
- On Windows: WSL2 is strongly recommended for Anchor/Solana toolchain

## Quick Start

### 1) Frontend dashboard

```bash
npm install
npm run dev
```

Default local URL: `http://localhost:5173`

### 2) Backend watchtower

```bash
cd backend
npm install
npm run dev
```

Default local URL: `http://localhost:3000`

### 3) SDK build and CLI

```bash
cd sdk
npm install
npm run build
node ./bin/sss-token.js --help
```

You can also run with `npx` from `sdk/` after build.

### 4) Dockerized backend (optional)

```bash
docker compose up --build
```

### 5) Anchor program flow

```bash
anchor build
anchor test
```

For devnet deployment:

```bash
anchor deploy --provider.cluster devnet
```

## SDK Usage Example

```ts
import { Connection, Keypair } from "@solana/web3.js";
import { SolanaStablecoin, Presets } from "@stbr/sss-token";

const connection = new Connection("https://api.devnet.solana.com", "confirmed");
const authority = Keypair.generate();

const stablecoin = await SolanaStablecoin.create(connection, {
  preset: Presets.SSS_2,
  name: "My USD",
  symbol: "MYUSD",
  decimals: 6,
  authority,
});

await stablecoin.mint({
  recipient: authority.publicKey,
  amount: 1_000_000,
});

const supply = await stablecoin.getTotalSupply();
console.log({ supply });
```

## CLI Command Reference

Main command:

```bash
sss-token --help
```

Core workflows:

```bash
sss-token configure --rpc https://api.devnet.solana.com --program-id <PROGRAM_ID> --authority <PUBKEY>
sss-token init --preset sss-1 --name "Stable One" --symbol S1
sss-token init --preset sss-2 --name "Stable Two" --symbol S2
sss-token mint <RECIPIENT> <AMOUNT>
sss-token burn <AMOUNT> --from <ADDRESS>
sss-token freeze <ADDRESS>
sss-token thaw <ADDRESS>
sss-token pause
sss-token unpause
sss-token status
sss-token supply
sss-token holders --min-balance 1000
sss-token minters list
sss-token minters add <ADDRESS>
sss-token minters remove <ADDRESS>
sss-token transfer-authority <ADDRESS>
sss-token audit-log --action mint
```

Compliance workflows (SSS-2):

```bash
sss-token blacklist add <ADDRESS> --reason "OFAC match"
sss-token blacklist remove <ADDRESS>
sss-token seize <ADDRESS> --to <TREASURY> --amount 1000
```

CLI local state file: `.sss-token-state.json`

## Backend API Overview

Base URL: `http://localhost:3000`

| Method | Endpoint | Purpose |
| --- | --- | --- |
| GET | `/health` | Health check |
| GET | `/api/status` | Runtime service status |
| POST | `/api/mint` | Mint request |
| POST | `/api/burn` | Burn request |
| GET | `/api/supply` | Current total supply |
| GET | `/api/holders` | Holder list (`minBalance` query optional) |
| GET | `/api/minters` | Minter list |
| POST | `/api/minters/add` | Add minter |
| POST | `/api/minters/remove` | Remove minter |
| POST | `/api/compliance/blacklist` | Add blacklist entry |
| POST | `/api/compliance/blacklist/remove` | Remove blacklist entry |
| GET | `/api/compliance/blacklist` | List blacklist entries |
| GET | `/api/compliance/audit-log` | Compliance audit log |
| GET | `/api/audit-log` | Combined audit log |

Environment template: `backend/.env.example`

## Devnet Deployment Notes

Before sharing proof with judges/reviewers:

1. Generate or set a deploy wallet keypair.
2. Ensure `solana config get` points to devnet and correct signer.
3. Run `anchor deploy --provider.cluster devnet`.
4. Capture and store:
   - deployed Program ID
   - deployment transaction signature
   - key proof transactions (mint, freeze, blacklist, seize)
5. Update PR description with Explorer links.

Important: `Anchor.toml` currently includes a localnet program key under `[programs.localnet]`. Replace values appropriately for your deployment flow.

## Documentation Index

- Architecture: `docs/ARCHITECTURE.md`
- SDK Guide: `docs/SDK.md`
- Operations Runbook: `docs/OPERATIONS.md`
- SSS-1 Spec: `docs/SSS-1.md`
- SSS-2 Spec: `docs/SSS-2.md`
- Compliance Notes: `docs/COMPLIANCE.md`
- Backend API Reference: `docs/API.md`

## Current Status

Implemented in repository:

- Frontend admin dashboard and operational pages.
- SDK package and CLI command surface.
- Backend watchtower services and dockerization.
- Documentation suite under `docs/`.
- Anchor program source and test scaffolding.

Notes:

- Local and devnet behavior can differ based on Solana/Anchor toolchain versions.
- Keep Anchor, Solana CLI, and Rust toolchain aligned to avoid build mismatches.

## Troubleshooting

### `solana` or `solana-keygen` not found

- Ensure Solana CLI bin path is exported in shell.
- On WSL, run `source ~/.bashrc` after install.

### Devnet airdrop rate-limit

- `solana airdrop` may be rate-limited.
- Retry after delay or use approved devnet faucet alternatives.

### Anchor version mismatch warning

- Repo uses `anchor_version = "0.30.1"` in `Anchor.toml`.
- Use matching Anchor CLI version to minimize compatibility errors.

### Docker backend healthcheck fails

- Confirm backend dependencies are installed and container port `3000` is free.

## License

This project is licensed under the MIT License.
See `LICENSE` for details.
