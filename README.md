---
description: Operator and developer documentation for the forked-social network
---

# forked-social Documentation

forked-social is a standalone blockchain network derived from the DeSo
codebase, prepared for launch as its own chain: its own genesis, its own
30,000,000-coin fixed supply, its own `FS1…` keys, and a
Proof-of-Stake validator set that takes over from a short PoW bootstrap at
block 300. This documentation set covers operating nodes and validators on
**this** network and building on its API. It does not describe the upstream
DeSo mainnet.

The canonical statement of what this network *is* — prefixes, ports,
domains, genesis, fork heights, peering — is
[intro/NETWORK.md](intro/NETWORK.md).

## Documentation map

**Run the network**

- [Network Reference](intro/NETWORK.md) — identity table, genesis and
  supply, fork heights, peering model, well-known public keys, endpoints.
- [Launch Runbook](operators/launch-runbook.md) — the ordered launch-day
  sequence with its hard timing constraints (the block-145 deadline).
  Complements the engineering runbook at
  `backend/validators/README.md`.
- [Running a Node](operators/run-a-node.md) — join the network as a
  non-validating backend: fresh data dir, `CONNECT_IPS`, checkpoints,
  ports, wiping, upgrades.
- [Running a Validator](operators/run-a-validator.md) — day-2 validator
  operations: the two-key identity model, registration anatomy, monitoring,
  unjailing.

**Understand the network**

- [Governance](intro/GOVERNANCE.md) — the founder-root key, param-updater
  mechanics, `UpdateGlobalParams`.
- [Node Architecture Overview](architecture-overview/) — the monorepo
  layout and how a node runs (rewritten for this fork).
- [Dev Setup](architecture-overview/dev-setup.md) — run the stack locally.

**Build on the network**

- [Backend configuration flags](deso-backend/configuration/README.md) and
  [data API](deso-backend/api/README.md) /
  [transaction construction API](deso-backend/construct-transactions/README.md) —
  the full REST surface served by every node on this network.
- [Identity service](deso-identity/identity/README.md) — the
  identity.forked.social signing model and its APIs.
- [Frontend getting started](deso-frontend/get-started.md) /
  [React example](deso-frontend/react-example.md) /
  [NextJS example](frontend-nextjs-example.md) — app quickstarts.
- Feature explainers: [associations](deso-features/associations.md),
  [creator coins](deso-features/creator-coins.md),
  [feeds & moderation](deso-features/feeds-and-moderation.md),
  [social NFTs](deso-features/social-nfts.md),
  [social tipping](deso-features/social-tipping.md).
- Protocol concepts: [consensus (PoW bootstrap → PoS)](deso-blockchain/consensus-pow-and-pos.md),
  [on-chain data](deso-blockchain/on-chain-data.md),
  [content moderation](deso-blockchain/content-moderation.md), and others
  under `deso-blockchain/`.

## Quick facts

| | |
|---|---|
| Mainnet variant | `--forknet` / `FORKNET=true` |
| Testnet variant | `--forknet-testnet` / `FORKNET_TESTNET=true` |
| Mainnet ports | P2P `42000`, API `42001` |
| Seed API | `https://node.forked.social` (testnet: `https://test.forked.social`) |
| Key prefixes | `FS1…` mainnet / `tFS2…` testnet, 54 chars |
| Supply | 30,000,000 coins, fixed at genesis — zero issuance |
| Consensus | PoW bootstrap to block 300 (2s blocks), then Fast-HotStuff PoS (144-block epochs) |
| Peering | No DNS seeds — `CONNECT_IPS=node.forked.social:42000` |

## Provenance and license

This codebase and documentation set derive from the open-source DeSo
project; inherited, generally-accurate API and feature reference material
is preserved and marked with fork-context notes where behavior differs
(the neutralized BitcoinExchange minting path, network prefixes, identity
URLs). See `LICENSE` in this directory.
