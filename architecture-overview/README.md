# Node Architecture Overview

How the forked-social codebase is put together and how a node on this
network runs. This is a rewritten-for-the-fork version of the upstream
DeSo architecture walkthrough: the machinery is inherited, but this repo is
a **monorepo** and the network parameters are ours (see
[../intro/NETWORK.md](../intro/NETWORK.md)).

## The four components (one repo)

Upstream DeSo ships as four separate repositories. Here they live in one
monorepo, already wired to the forked-social network:

| Directory | What it is |
|---|---|
| `core/` | The consensus kernel in Go: chain parameters (`core/lib/fork_params.go`), P2P server, mempool, block/transaction processing, PoS (Fast-HotStuff) |
| `backend/` | Embeds `core` as a library and exposes the REST API (`backend/routes/`), plus deployment artifacts (`backend/deploy.env`, `backend/podman-compose.yml`, `backend/validators/`) and ops tooling (`backend/scripts/`) |
| `frontend/` | The Angular web app served at forked.social (built with the `forked` environment) |
| `identity/` | The embeddable signing app (iframe) served at identity.forked.social, plus the offline key-derivation CLI (`identity/scripts/derive-pubkey.ts`) |

Two small extras: `explorer/` (the block explorer app) and
`validator-key-generator/` (standalone BLS pubkey / voting-auth generator).

Common conventions:

- Flags are managed with Viper: every flag has an environment equivalent
  (`--data-dir` ↔ `DATA_DIR`, `--api-port` ↔ `API_PORT`, …), which is what
  the deployment env files rely on.
- Network selection is explicit: `--forknet` (mainnet-style) or
  `--forknet-testnet`, resolved in `core/cmd/config.go`. Default ports are
  42000 (P2P) / 42001 (API) on mainnet, 42420 / 42421 on testnet.

## Lifecycle of a node

### Startup and peering

- Entry point: `backend/main.go` → `backend/cmd/run.go` → node startup in
  `backend/cmd/node.go`, driving `core` via `core/cmd/config.go`.
- **This network has no DNS seeds** (empty by design in
  `core/lib/fork_params.go`). Peering is explicit:
  - `--connect-ips node.forked.social:42000` pins connections to given
    peers (what joining nodes and validators use);
  - any node's P2P port accepts inbound peers (42000 by default; the
    three launch validators run on the seed's server and publish distinct
    P2P ports 42100/42200/42300 — see
    [../../backend/validators/README.md](../../backend/validators/README.md)).
- Checkpoint syncing defaults to `https://node.forked.social`
  (`core/lib/constants.go`), so nodes bootstrap trust from the seed's API.

The `ConnectionManager` (`core/lib/connection_manager.go`) owns peer
connections; connected peers go through a Bitcoin-style version
negotiation and are handed to the node's single-threaded main loop
(`core/lib/server.go`), which processes control and peer messages.

### Syncing

Sync is headers-first: `server.go` picks a sync peer, exchanges header
bundles, then downloads and validates blocks. Block application lives in
`core/lib/blockchain.go` → `ConnectBlock` → `ConnectTransaction` per
transaction, executed against a copy-on-write "view" (`core/lib/block_view.go`)
that is flushed to disk only after the whole block validates — the same
mechanism the mempool (`core/lib/pos_mempool.go`) uses to validate
individual transactions. Steady-state operation is INV-driven: new blocks
and transactions announced by peers are requested and processed
on demand.

### Block production

- **Blocks 1–300 (PoW bootstrap):** block templates are produced by
  `core/lib/block_producer.go` and mined by the seed's miner
  (zero block reward on this network — supply is fixed; see
  [../intro/NETWORK.md](../intro/NETWORK.md)). The window at 2-second
  block targets lasts roughly ten minutes and exists solely to reach the
  PoS cutover.
- **Block 300 onward (PoS):** Fast-HotStuff view-driven production by the
  registered validator set (epoch snapshots every 144 blocks). Validators
  sign with BLS voting keys; the seed stops mining at the cutover height.
- A remote miner harness also exists (`backend/remote_miner_main.go`,
  get-block-template / submit-block API routes) for PoW mining against any
  node — economically pointless here (no reward) but part of the inherited
  API surface.

### BitcoinExchange machinery

The inherited Bitcoin-heavy path still exists in code (`core/lib/bitcoin_burner.go`,
`core/lib/block_view_bitcoin.go`): nodes can validate Bitcoin burn proofs
for `BitcoinExchange` minting transactions. On this network it is
**economically neutralized** (`DeSoNanosPurchasedAtGenesis = MaxNanos`): the
transaction type remains valid consensus-wise, but effectively no new coins
can be minted through it.

## Transactions, identity, and the apps

- Transaction types and their metadata are defined in `core/lib/network.go`
  (UTXO-based, one signing public key, a flexible `ExtraData` map). The
  REST API in `backend/routes/server.go` (route constants at the top of the
  file) constructs and submits them — full reference under
  [../deso-backend/api/](../deso-backend/api/README.md) and
  [../deso-backend/construct-transactions/](../deso-backend/construct-transactions/README.md).
- Signing never happens in the frontend. The app at forked.social requests
  an unsigned transaction from its backend, sends it to the
  identity.forked.social iframe (via `postMessage`, with the seed encrypted
  per-host in `encryptedSeedHex`), the iframe signs and returns it, and the
  app submits the signed transaction to the API. This is the inherited DeSo
  identity model, described in detail under
  [../deso-identity/](../deso-identity/identity/README.md).
- The frontend talks to its backend through the same REST API; the
  deso-protocol npm dependency it uses is pinned to this network via a
  patch-package override (`frontend/patches/deso-protocol+2.6.4.patch`).

## Where to go next

- [dev-setup.md](dev-setup.md) — run the stack locally.
- [../operators/run-a-node.md](../operators/run-a-node.md) — join the
  network.
- [../operators/run-a-validator.md](../operators/run-a-validator.md) —
  validate.
- [../../backend/validators/README.md](../../backend/validators/README.md) —
  launch-day engineering runbook.
