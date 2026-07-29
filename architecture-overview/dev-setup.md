# Dev Setup: Running the Stack Locally

How to run a forked-social node and apps on a development machine. For
production node operation see [../operators/run-a-node.md](../operators/run-a-node.md).

This is a monorepo — no multi-repo checkout is needed.

## Prerequisites

- **Go** (for `core` + `backend`).
- **Node.js / npm** (Angular apps: `frontend`, `identity`, `explorer`).
- **libvips** (`brew install vips` or `apt install libvips-tools`) — image
  processing in the backend.
- **On gcc-14+ systems**, any direct `go build`/`go run` of the backend or
  core must set `CGO_CFLAGS="-std=gnu11"` (the bundled BLS C sources do not
  compile under `-std=gnu23`). The provided scripts and images handle this.

## Backend: a local fork testnet node

The fastest local setup is the rewritten dev script
(`backend/scripts/nodes/`), which runs a regtest-style **fork testnet**
node:

```sh
cd backend/scripts/nodes
./n0_test
```

What it does (see the script for the exact flags): builds `backend`, starts
it with `--forknet-testnet --regtest`, API on `localhost:42421`, P2P on
`42420`, mining with `NUM_MINING_THREADS=1` and the miner key
`FS13xm5oxJvGKd7194u5f2paA9WaaQpZSvz7Tzu8uXDngxromi3Lnv`, data under
`/tmp/fork_n0_test`, and CORS pre-opened for the local frontend origins.
Sibling scripts (`n1_test`, `n2_test`, `n0`, …) run additional local peers
on shifted ports. Validator/secrets flags are runtime-injected only —
never commit them.

Manual equivalent, from the repo root:

```sh
go build -o backend backend/main.go   # add CGO_CFLAGS="-std=gnu11" if gcc >= 14
./backend run --forknet-testnet --regtest \
  --api-port=42421 --protocol-port=42420 \
  --data-dir=/tmp/fork_dev --num-mining-threads=1
```

Wipe local state by deleting the data dir (`/tmp/fork_n0_test` etc.).

## Frontend (forked.social web app)

```sh
cd frontend
npm install
npm start        # serves on http://localhost:4200
```

Production builds bake in the forked-social endpoints via the `forked`
environment: either `npm run build_prod` (runs `patch-package` first —
this applies `patches/deso-protocol+2.6.4.patch`, which repoints the
deso-protocol library to the fork network) or the Docker build with
`--build-arg environment=forked`.

## Identity

```sh
cd identity
npm install
npm start        # serves on http://localhost:4201
```

Production build: `npm run build_prod`.

## App runtime configuration (frontend & identity)

Three optional app features are operator-configurable at container start via
compose environment variables; each ships blank, and a blank value degrades
gracefully:

| Variable | Compose file | Blank behavior |
|---|---|---|
| `WEBPUSH_VAPID_PUBLIC_KEY` | `frontend/podman-compose.yml` | Web push subscriptions disabled |
| `WALLET_CONNECT_PROJECT_ID` | `identity/podman-compose.yml` | WalletConnect pairing disabled; attempts return an explanatory error |
| `GOOGLE_DRIVE_CLIENT_ID` | `identity/podman-compose.yml` | "Back up seed to Google Drive" option hidden |

Mechanism: the container entrypoint `run.sh` regenerates `/env-config.js`
at startup from the compose environment, and the Angular app merges
`window.__FORKED_RUNTIME_ENV__` over its compiled environment defaults.
For local (non-container) development, set values in the app's
`src/env-config.js` or define the `window.__FORKED_RUNTIME_ENV__` global
manually — both app READMEs (`frontend/`, `identity/`) document this.

## Explorer

```sh
cd explorer
npm install
npm start
```

Production build: `npm run build_prod` (uses
`NODE_OPTIONS=--openssl-legacy-provider` for the legacy Angular builder).

## Deriving keys for local testing

The offline derivation CLI prints the public keys a mnemonic maps to —
useful for wiring test identities:

```sh
cd identity
npm run derive-keys -- "<mnemonic>"            # FS1… (fork mainnet style)
npm run derive-keys -- --testnet "<mnemonic>"  # tFS2… (fork testnet style)
```

Options: `--passphrase <text>` (BIP39 passphrase), `--account <N>`
(sub-account), `--file <path>` (batch). Remember: BLS validator mnemonics
must have **no** passphrase anywhere in the pipeline. See
[../operators/launch-runbook.md](../operators/launch-runbook.md) for the
validator two-key model.
