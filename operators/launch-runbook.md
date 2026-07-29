# forked-social Launch Runbook

This is the operator-level walkthrough of the network launch: the ordered
sequence from prerequisites to the Proof-of-Stake cutover. The **engineering
runbook with exact, copy-pasteable commands is
[../../backend/validators/README.md](../../backend/validators/README.md)** —
this document follows the same order and values and explains why each step
exists. If the two ever disagree, fix one of them; they are meant to say the
same thing.

Audience: the operators of the seed node and the three launch validators.
Everything below describes the mainnet variant (`--forknet`,
`node.forked.social`). The seed and all three validators run on **the same
server**: the seed keeps ports 42000/42001 and each validator gets its own
port pair — validator1 42100/42101, validator2 42200/42201, validator3
42300/42301 (P2P/API; see the port-layout table in the
[engineering runbook](../../backend/validators/README.md)). For a testnet
rehearsal, substitute `--forknet-testnet` / `FORKNET_TESTNET=true`,
`test.forked.social`, and ports 42420/42421.

## ⚠️ Read this first: the launch clock

> **The PoS cutover happens at block 300. The validator set for the first
> PoS epoch is snapshotted at block ~145. Blocks are 2 seconds. That means
> all three validators must be funded, registered, and staked within roughly
> FIVE MINUTES of the seed starting to mine.**
>
> - At 2-second blocks, height 145 arrives ~5 minutes after the seed starts.
> - The bootstrap tool warns and **aborts** if the chain is already past
>   height ~120 (override: `--continue-past-deadline` — do not use it on
>   launch day).
> - Registrations that miss the snapshot only join at the next epoch boundary
>   after the block-300 cutover — and until then the chain **stalls** at 300,
>   because no validator exists to produce PoS blocks.
> - **Because the genesis block is fixed, a missed window cannot be patched.**
>   The only recovery is wiping the data dir (chain + logs) on **every** node
>   and relaunching the network from genesis.

Everything in this runbook is ordered so that nothing time-boxed is left for
the five-minute window except the bootstrap run itself.

## Sequence at a glance

| Step | Where | Time-boxed? |
|---|---|---|
| 0. Prerequisites: DNS, TLS, images, data dirs | the server | No — do days early |
| 1. Secrets preparation and derivation checks | operator machine | No — offline |
| 2. Secrets into env on the seed host | the server | No |
| 3. Start the seed node | the server | Starts the clock |
| 4. Run the validator bootstrap tool | the server | **Yes — before height ~120, snapshot at ~145** |
| 5. Start the three validators | the server (per-validator dirs) | Before block 300 |
| 6. Watch the cutover at block 300 | anywhere | Height 300 |

## Step 0 — Prerequisites (do all of this before launch day)

**DNS.** A single A record for `node.forked.social`, pointing at the
**server IP** — the seed and the three validators share one host, kept
apart by distinct ports. All three validators register on-chain under
that same host as `node.forked.social:42100`, `node.forked.social:42200`,
and `node.forked.social:42300`; the validator mesh dials each registered
`host:port`. No per-validator records are needed: the consensus layer
imposes no cross-validator domain-uniqueness check (uniqueness is only
enforced within a single registration's own Domains list), so endpoint
uniqueness comes from the distinct ports.

**TLS.** Terminate TLS for `https://node.forked.social` on the seed host's
reverse proxy. The API port 42001 is published loopback-only; the reverse
proxy is what makes it public. `CHECKPOINT_SYNCING_PROVIDERS` and joining
nodes both use the HTTPS URL.

**Firewall.** Inbound TCP **42000, 42100, 42200, and 42300 open to the
internet** on the server — the P2P ports of the seed and validators 1–3.
The API ports 42001 / 42101 / 42201 / 42301 stay loopback-only.

**Images.** The backend image (`ghcr.io/forked-social/backend:<tag>`) built
and pulled once on the server — the seed and all three validators run the
same image. Additionally
build the **builder-stage container** for the bootstrap tool from the repo
root once (it carries the Go toolchain + libvips; host-native builds are not
possible):

```sh
podman build --target backend -f backend/Dockerfile -t forked-builder .
```

Any `go build` / `go run` inside that container must set
`CGO_CFLAGS="-std=gnu11"` — the bundled BLS C sources do not compile under
gcc-14+'s default `-std=gnu23`.

**Validator directories.** On the same server, install the compose + env
templates and create data dirs (exact commands in the
[engineering runbook, Step 0, item 5](../../backend/validators/README.md)):
`docker-compose.validator{1,2,3}.yml` plus
`validator{1,2,3}.env.example` (as `/opt/validatorN/validatorN.env`, mode
0600), and `/opt/validatorN/data/{chain,logs}`.

## Step 1 — Secrets preparation (offline)

You need **seven mnemonics**, all generated and stored offline before launch:

| Mnemonic | Count | Used as |
|---|---|---|
| Miner | 1 | `MINER_SEED` — funds the validators from the genesis allocation |
| Validator DeSo mnemonic (identity: signs register + stake) | 3 | `VALIDATOR{1,2,3}_SEED` |
| Validator BLS mnemonic (voting: signs PoS votes/blocks) | 3 | `VALIDATOR{1,2,3}_BLS_SEED` → later the validator's `POS_VALIDATOR_SEED` |

Rules:

- **The BLS mnemonics must have NO BIP39 passphrase anywhere in the
  pipeline.** No `--passphrase` when deriving, no extra text at account
  creation. The node derives its voting key from the bare mnemonic.
- The two mnemonics per validator are deliberately different keys; do not
  reuse one for the other role.
- Zero mnemonics are ever committed or logged. The repo, the compose files,
  and the deploy templates carry public keys only; the seeds are injected at
  runtime (env files at 0600, or manual shell exports).

Derive and record the public keys for cross-checking with the offline CLI
from `identity/`:

```sh
cd identity/
npm run derive-keys -- "<mnemonic>"            # mainnet-style FS1… pubkey
npm run derive-keys -- --testnet "<mnemonic>"  # testnet-style tFS2… pubkey
```

(`--passphrase`, `--account N`, and `--file <list>` also exist; the tool
runs fully offline and prints only public keys.) The three validator DeSo
pubkeys are expected to match the launch identities in the
[engineering runbook](../../backend/validators/README.md#launch-identities-public-keys--reference):

- Validator 1: `FS13wSsnnLhYeVfjNyvc8HE6f7iWGG6bHJx6Kb5wKR513QJrNQdq5W`
- Validator 2: `FS13vueAs3CmmPBjm5PnsyjbiUHegNHyNXmCcLdu1A6hh9tc2qHB74`
- Validator 3: `FS13w9xhKm1vbvDWQKSvZPCqNkQMLFDurJ2TRNw5zjsod4kM1DS1wE`

Optionally preview a validator's registered BLS voting pubkey with the
standalone generator from `validator-key-generator/`:

```sh
go run main.go --bls-seed-phrase="<BLS mnemonic>" \
  --deso-public-key="<validator DeSo pubkey>"
```

The bootstrap tool derives the same BLS key in-process with the node's own
`lib.NewBLSKeystore`, so identical mnemonics produce identical keys
everywhere.

## Step 2 — Secrets into the environment (seed host)

Export the seven mnemonics in the operator shell on the seed host (prefix
each line with a space to keep them out of shell history — exact block in
the [engineering runbook, Step 1](../../backend/validators/README.md)):

```sh
export MINER_SEED='<miner mnemonic>'
export VALIDATOR1_SEED='<v1 DeSo mnemonic>'
export VALIDATOR1_BLS_SEED='<v1 BLS mnemonic>'
export VALIDATOR2_SEED='<v2 DeSo mnemonic>'
export VALIDATOR2_BLS_SEED='<v2 BLS mnemonic>'
export VALIDATOR3_SEED='<v3 DeSo mnemonic>'
export VALIDATOR3_BLS_SEED='<v3 BLS mnemonic>'
# OPTIONAL per-validator domain override (defaults: node.forked.social:42100,
# node.forked.social:42200, node.forked.social:42300):
# export VALIDATOR1_DOMAIN=host1.example.com:42100
```

That is the tool's entire documented env surface: `MINER_SEED`,
`VALIDATOR{1,2,3}_SEED`, `VALIDATOR{1,2,3}_BLS_SEED`, optional
`VALIDATOR{1,2,3}_DOMAIN`, optional `API_URL` (the `--api-url` flag wins).

## Step 3 — Start the seed node (this starts the clock)

The seed runs from `backend/deploy.env` + `backend/podman-compose.yml`
(`FORKNET=true`, `API_PORT=42001`,
`CHECKPOINT_SYNCING_PROVIDERS=https://node.forked.social`, miner key
configured); its secrets (`POS_VALIDATOR_SEED`, `BLOCK_PRODUCER_SEED`,
`STARTER_DESO_SEED`) are runtime-injected and empty on a fresh checkout:

```sh
IMAGE=ghcr.io/forked-social/backend:<tag> \
  podman-compose -f /opt/backend/podman-compose.yml up -d

curl -s http://127.0.0.1:42001/api/v0/health-check    # -> 200
curl -s -X POST http://127.0.0.1:42001/api/v0/get-app-state \
  -H 'Content-Type: application/json' -d '{}'          # BlockHeight climbs
```

From this moment, 2-second blocks are being mined and the ~5-minute window
to block ~145 is running. **Do not start the seed until Steps 0–2 are done
and the bootstrap command for Step 4 is rehearsed.**

## Step 4 — Run the bootstrap tool (immediately)

Run the tool **on the seed host against the loopback API** — no TLS
dependency and lowest latency exactly when the window is tightest:

```sh
podman run --rm --network=host \
  -e CGO_CFLAGS="-std=gnu11" \
  -v "$HOME/forked-social":/src -w /src/backend \
  -e MINER_SEED -e VALIDATOR1_SEED -e VALIDATOR1_BLS_SEED \
  -e VALIDATOR2_SEED -e VALIDATOR2_BLS_SEED \
  -e VALIDATOR3_SEED -e VALIDATOR3_BLS_SEED \
  forked-builder go run ./scripts/pos/validator_bootstrap \
  --dry-run --api-url http://127.0.0.1:42001    # rehearsal, submits nothing

# the real run: same command without --dry-run
```

What it does, per validator, sequentially: **funds 5,125,000 coins** from
the miner key (the stake plus a 2.5% fee/burn buffer), submits the
**register-as-validator** transaction (deriving the BLS pubkey and voting
authorization in-process via `lib.NewBLSKeystore`, so the on-chain
registration is guaranteed consistent with the running node),
then **stakes exactly 5,000,000 coins** — three signed transactions per
validator, waiting for a fresh block between them. Totals: 15,375,000
coins funded (15,000,000 staked), leaving ~14,625,000 of the 30,000,000
genesis supply with the miner; the 5M-coin stakes are a launch-security
posture against 50%+ attacks, intended to be lowered via normal
unstake/unlock flows as the network grows. Expect ~1–2 minutes for
all three. The tool prints txids and pubkeys only, never mnemonics.

Guardrails:

- `--dry-run` prints the plan and submits nothing — rehearse with it
  **before** launch day (Step 0 of the engineering runbook).
- Past height 120 it prints a loud deadline warning and **aborts** unless
  `--continue-past-deadline` is passed. If you see this on launch day, the
  honest options are: race it (no), or stop, wipe, and relaunch.
- From anywhere other than the seed host: `--api-url
  https://node.forked.social` (slower; TLS).

## Step 5 — Start the three validators

On the server, for each validator (all three run there), satisfy the
identity requirement first:

> **`POS_VALIDATOR_SEED` in `/opt/validatorN/validatorN.env` must be the
> EXACT SAME mnemonic the bootstrap tool consumed as
> `VALIDATORN_BLS_SEED`.** The BLS voting pubkey registered on-chain was
> derived from that mnemonic; the container derives its signer from
> `POS_VALIDATOR_SEED`. If they differ, the node cannot sign for its
> registered identity, the mesh treats it as inactive, and the chain can
> stall at the cutover. This is the number-one launch-day failure mode.

```sh
sudoedit /opt/validator1/validator1.env   # POS_VALIDATOR_SEED=<v1 BLS mnemonic>
sudo chmod 0600 /opt/validator1/validator1.env
IMAGE=ghcr.io/forked-social/backend:<tag> \
  podman-compose -f /opt/validator1/docker-compose.validator1.yml up -d
```

The compose files already wire what a validator needs — do not change it:
`FORKNET=true`, `CONNECT_IPS=node.forked.social:42000` (there are no DNS
seeds on this network by design), and per-validator `PROTOCOL_PORT` /
`API_PORT` — validator1 42100/42101, validator2 42200/42201, validator3
42300/42301 (same-value host publishes) — with the P2P port public and the
API port loopback-only for admin. Sanity checks (validator 1 shown):
`podman logs validator1` shows fork-mainnet selection, `Protocol listening
on port 42100`, and peer connection;
`curl -s http://127.0.0.1:42101/api/v0/health-check` returns 200
(loopback-only: reach it via `ssh -L 42101:127.0.0.1:42101 <server>`);
logs appear under `/opt/validator1/data/logs`.

Validators can start any time before the cutover; starting them during the
bootstrap window is fine and recommended.

## Step 6 — The cutover at block 300

Watch from anywhere:

```sh
# Epoch/view progress and leader schedule:
curl -s https://node.forked.social/api/v0/current-epoch-progress | jq .

# Validator status — expect ACTIVE after cutover and
# TotalStakeAmountNanos = 5000000000000000 (5,000,000 coins):
curl -s https://node.forked.social/api/v0/validators/FS13wSsnnLhYeVfjNyvc8HE6f7iWGG6bHJx6Kb5wKR513QJrNQdq5W | jq .

# Self-stake (staker = validator pubkey):
curl -s https://node.forked.social/api/v0/stake/FS13wSsnnLhYeVfjNyvc8HE6f7iWGG6bHJx6Kb5wKR513QJrNQdq5W/FS13wSsnnLhYeVfjNyvc8HE6f7iWGG6bHJx6Kb5wKR513QJrNQdq5W | jq .

# Height must keep climbing THROUGH and past 300:
curl -s -X POST https://node.forked.social/api/v0/get-app-state \
  -H 'Content-Type: application/json' -d '{}' | jq .BlockHeight
```

Healthy cutover signals: `CurrentView` keeps increasing;
`CurrentLeader` rotates through the three validator pubkeys; the seed's PoW
miner stops producing at exactly 300; each validator's log shows
Fast-HotStuff view advancement.

Known route quirk: `/api/v0/current-epoch-progress` may return HTTP 500
while zero validators are registered (modulo on an empty leader schedule).
It becomes healthy after the first registration — before Step 4 completes it
is not, by itself, an alarm.

**If the chain stalls at or just after 300:** almost certainly a
registration missed the block-145 snapshot, or a `POS_VALIDATOR_SEED`
mismatch. Confirm with the `/api/v0/validators/…` calls above. Late
registrations join only at the next epoch boundary, so the chain stays
stalled until a snapshot picks up the registered set — and because genesis
is fixed, the realistic recovery is: stop all nodes, clear every node's data
dir (chain + logs), fix the cause, and relaunch from Step 0. Prevention —
the ordering in this runbook — is everything.
