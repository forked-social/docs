# Running a forked-social Node

How to join the forked-social network with a full (non-validating,
non-mining) backend node — the configuration exchanges, explorers, or
app-backends use. If you want to produce blocks instead, see
[run-a-validator.md](run-a-validator.md). For what the network *is*, see
[../intro/NETWORK.md](../intro/NETWORK.md).

Two variants exist; this page shows mainnet (`FORKNET=true`,
`node.forked.social`, ports 42000/42001). For testnet use
`FORKNET_TESTNET=true`, `test.forked.social`, and ports 42420/42421.

## What a joining node needs

The fork network has **no DNS seeds by design**. A joining node finds the
network through exactly two facilities:

- **Explicit peering:** `CONNECT_IPS=node.forked.social:42000` — a pinned
  connection to the seed (add more peers as the network grows, e.g. the
  validator endpoints `node.forked.social:42100`,
  `node.forked.social:42200`, `node.forked.social:42300`).
- **Checkpoint syncing:** `CHECKPOINT_SYNCING_PROVIDERS` defaults to
  `https://node.forked.social` already (`core/lib/constants.go`), so a
  joining node checkpoints from the seed out of the box. Set it explicitly
  only if you want a different/uniform provider list.

Everything else is a standard backend deployment: point it at an empty data
dir, publish the P2P port, keep the API port private.

## Quick start (compose)

The deployment artifacts of the seed (`backend/deploy.env` +
`backend/podman-compose.yml`) are a working template for any node; a
joining node strips the seed-only lines (miner config, admin keys — and it
does not need `POS_VALIDATOR_SEED`/`BLOCK_PRODUCER_SEED`, which stay empty).
The binary maps env names onto its viper flags (`DATA_DIR` → `--data-dir`,
`API_PORT` → `--api-port`, …), so any flag has an env equivalent.

1. **Fresh data dir.** Create the host directories that will hold chain data
   and logs. A node that once ran another network against the same dir will
   not start cleanly against the fork genesis — when in doubt, start empty.

   ```sh
   sudo install -d -o deploy -g deploy -m 0755 /opt/backend/data/chain /opt/backend/data/logs
   ```

2. **Env file** (adapted from `backend/deploy.env`):

   ```sh
   DATA_DIR=/data
   LOG_DIR=/logs
   FORKNET=true
   API_PORT=42001
   CONNECT_IPS=node.forked.social:42000
   CHECKPOINT_SYNCING_PROVIDERS=https://node.forked.social
   ```

3. **Ports.** Publish P2P 42000 publicly (peers need to reach you; run the
   container with host networking or `"42000:42000"`). Publish the API port
   loopback-only (`"127.0.0.1:42001:42001"`) and put it behind your own
   reverse proxy if you intend to expose an API.

4. **Run:**

   ```sh
   IMAGE=ghcr.io/forked-social/backend:<tag> \
     podman-compose -f /opt/backend/podman-compose.yml up -d
   ```

## Quick start (bare binary)

The equivalent minimal flag set when running the binary directly
(`backend` image entrypoint or a local build — remember
`CGO_CFLAGS="-std=gnu11"` if you build the image's toolchain yourself):

```sh
backend run \
  --forknet \
  --data-dir=/path/to/chain \
  --connect-ips=node.forked.social:42000 \
  --api-port=42001 \
  --protocol-port=42000
```

`--forknet` selects the fork mainnet parameters (prefixes, ports, genesis,
fork heights). It cannot be combined with `--testnet` or
`--forknet-testnet`.

## Verifying the node

```sh
curl -s http://127.0.0.1:42001/api/v0/health-check    # liveness: 200
curl -s -X POST http://127.0.0.1:42001/api/v0/get-app-state \
  -H 'Content-Type: application/json' -d '{}'          # BlockHeight as it syncs
```

A full node catches up via checkpoints + headers-first sync; when
`BlockHeight` matches the seed
(`https://node.forked.social/api/v0/get-app-state`), you are synced. The
API surface available from your node is documented under
[../deso-backend/api/](../deso-backend/api/README.md).

## Operational notes

- **Do not mine.** Leave `MINER_PUBLIC_KEYS` unset. The PoW bootstrap window
  is over after block 300 and block rewards are zero anyway — the entire
  supply lives in the genesis allocation.
- **Trusted-producer posture (optional).** Upstream's
  `trusted-block-producer-public-keys` restriction defaults to **empty** on
  this network (the restriction is inactive). A deliberate launch posture
  can be restored via the `--trusted-block-producer-public-keys` CLI flag or
  a config file; note there is **no environment variable** for this setting.
- **Staying up to date.** Track this repository's releases and re-pull the
  backend image; upgrades are `podman-compose down` / `up -d` with the same
  data dirs. Read release notes before major versions — a resync is
  occasionally required, in which case you simply clear the data dir and
  re-sync from checkpoints.
- **Wiping a node** means clearing its data dir (chain + logs), nothing
  else. Against this network a wiped node re-syncs from the fork genesis
  via `node.forked.social` checkpoints.
- **Sync modes.** `--hypersync` (default on) accelerates initial sync. If
  you disable it, initial sync takes considerably longer.
