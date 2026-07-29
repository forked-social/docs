# forked-social Network Reference

This document is the canonical reference for the forked-social standalone
network: its two variants, address prefixes, ports, domains, genesis
allocation, fork heights, and peering model. When another document disagrees
with this one, this one wins. Values are defined in code in
`core/lib/fork_params.go` (parameters) and `core/lib/constants.go` (defaults).

forked-social is a standalone blockchain derived from the DeSo (formerly
BitClout) codebase. It shares DeSo's transaction types, API surface, and
Fast-HotStuff Proof-of-Stake machinery, but it has its own genesis block,
its own supply, and its own validator set. It does **not** sync with, bridge
to, or depend on the DeSo mainnet in any way.

## Network variants

| | ForkMainnet | ForkTestnet |
|---|---|---|
| CLI flag | `--forknet` | `--forknet-testnet` |
| Environment variable | `FORKNET=true` | `FORKNET_TESTNET=true` |
| Public-key prefix | `FS1…` (54 chars) | `tFS2…` (54 chars) |
| Public-key Base58Check prefix bytes | `0x05, 0x01, 0xED` | `0x11, 0xC8, 0x7D` |
| P2P protocol port | `42000` | `42420` |
| REST API port | `42001` | `42421` |
| Seed node API | `https://node.forked.social` | `https://test.forked.social` |

Notes:

- The two flags are mutually exclusive and cannot be combined with the
  inherited `--testnet` flag (enforced at startup in `core/cmd/config.go`).
- Public keys are 54 characters long on both variants. The prefix bytes are
  fixed and byte-compatible with the identity service — they must not change
  after launch.
- The upstream `--testnet`/default (DeSo mainnet) modes still exist in the
  code, but they are **not supported paths** for this network. Supported ways
  to run a node are `--forknet` and `--forknet-testnet` only.

## Domains

| Domain | Role |
|---|---|
| `node.forked.social` | Mainnet node host: seed API (P2P `:42000`, API `:42001` behind TLS) — also the shared host of the three launch validators' P2P endpoints (`:42100`/`:42200`/`:42300`) |
| `test.forked.social` | Testnet seed node API (P2P `:42420`, API `:42421`) |
| `forked.social` | Frontend web app |
| `identity.forked.social` | Identity service (signing iframe) |
| `explorer.forked.social` | Block explorer |
| `media.forked.social` | Image/video media host |

The seed and all three validators run on **one server**: all four nodes
share the single `node.forked.social` A record above (one host, one server
IP), with distinct port pairs keeping the listeners apart (seed 42000/42001;
validators registered as `node.forked.social:42100`/`:42200`/`:42300`, API
ports 42101/42201/42301 loopback-only — P2P/API). The authoritative port
layout is the engineering
reference, [../../backend/validators/README.md](../../backend/validators/README.md).

## Genesis and supply

- **Genesis hash:**
  `3cc0c2827a5f7b8b32514d214e4a4c5f7ea7c20c3a715799554b6131b186d8cc`
- **Allocation:** a single genesis output granting 100% of the supply —
  **30,000,000 coins** (30,000,000 × 10⁹ nanos = 3×10¹⁶ nanos, `MaxNanos`) —
  to the miner key `FS13xm5oxJvGKd7194u5f2paA9WaaQpZSvz7Tzu8uXDngxromi3Lnv`.
  The genesis hash is verified at node boot; the block cannot be changed
  without wiping every node.
- **Zero issuance.** No mechanism mints new coins:
  - PoW bootstrap block rewards are disabled (`DisablePoWBlockRewards`), so
    mining the bootstrap window pays **0 coins** per block.
  - After the cutover to Proof of Stake, block producers earn **fees only**.
  - Staking rewards APY defaults to **0%**
    (`DefaultStakingRewardsAPYBasisPoints = 0`).
- **BitcoinExchange is economically neutralized**
  (`DeSoNanosPurchasedAtGenesis = MaxNanos`): the purchase schedule is priced
  astronomically past the genesis allocation, so effectively no coins can be
  minted through it. The transaction type still exists in the code and in the
  API surface, but it cannot expand supply.

## Chain parameters and fork heights

The fork network starts every legacy DeSo upgrade at height 0 (they are
considered already applied). Everything interesting happens in the first
~300 blocks:

| Height | Event |
|---|---|
| 0 | Genesis block |
| 0 | All legacy fork heights (balance-model-era upgrades) — already applied |
| 1 | Balance model, lockups, and PoS state setup activate (`ProofOfStake1StateSetupBlockHeight`) |
| 1 | Validator registration opens — validators are **not** genesis-baked; they register on-chain |
| ~145 | End of epoch 1 (blocks 2–145) — **epoch-1 snapshot seeds the first PoS validator set** |
| 300 | Proof-of-Stake consensus cutover (`ProofOfStake2ConsensusCutoverBlockHeight`) |

Related parameters:

- **Epoch duration:** 144 blocks. Snapshots taken at epoch boundaries use a
  two-epoch lookback, so the set signing blocks from height 290 onward is the
  one snapshotted at height 145.
- **PoW bootstrap block target:** 2 seconds, so the ~300-block window takes
  roughly ten minutes wall-clock. Height 145 is reached about **5 minutes**
  after the seed starts mining.
- Block production interval under PoS: 1500 ms; PoS block size caps are
  32,000 bytes (max) / 16,000 bytes (soft).

## Consensus model

1. **Bootstrap (blocks 1–300):** the seed node mines PoW blocks with zero
   block reward. During this window validators must be funded, registered,
   and staked (see the [launch runbook](../operators/launch-runbook.md) —
   the deadline is hard).
2. **Epoch 1 snapshot (~block 145):** the on-chain validator set at this
   height determines the first PoS leader schedule.
3. **Cutover (block 300):** consensus switches to Fast-HotStuff
   Proof-of-Stake. A registration that missed the epoch-1 snapshot joins at
   the next epoch boundary **after** cutover.

Staking and jailing parameters of note (defaults from `fork_params.go`):

| Parameter | Default |
|---|---|
| `DefaultEpochDurationNumBlocks` | 144 |
| `DefaultStakeLockupEpochDuration` | 3 epochs |
| `DefaultValidatorJailEpochDuration` | 3 epochs |
| `DefaultJailInactiveValidatorGracePeriodEpochs` | 48 epochs |
| `DefaultValidatorSetMaxNumValidators` | 1000 |
| `DefaultLeaderScheduleMaxNumValidators` | 100 |
| `DefaultStakingRewardsMaxNumStakes` | 10000 |
| `DefaultStakingRewardsAPYBasisPoints` | 0 |
| `StakeFeeBasisPoints` | 1000 (10%) |
| `MaxStakeMultipleBasisPoints` | 1000000 (100×) |

## Peering model

- **DNS seeds are empty by design.** The fork has no DNS bootstrapping;
  the only supported way to find the network is explicit peering.
- Joining nodes set `CONNECT_IPS=node.forked.social:42000` (mainnet) or
  `CONNECT_IPS=test.forked.social:42420` (testnet).
- **Checkpoint providers default to the fork domains**
  (`DefaultMainnetCheckpointProvider = https://node.forked.social`,
  `DefaultTestnetCheckpointProvider = https://test.forked.social` in
  `core/lib/constants.go`), so even the inherited non-fork modes no longer
  reference upstream infrastructure.
- The seed node does not pin `CONNECT_IPS`; it must accept all peers.

## Well-known public keys

All seeds/mnemonics behind these keys are runtime-injected and never
committed. Only public keys appear in this repo.

| Role | Public key |
|---|---|
| Founder root (governance, sole param-updater, seed admin/super-admin) | `FS13vrbSsSRefiNeoJK5uHMKgCwzJJ3NMCFdEKpui7qPX4X7xSLSko` |
| Miner (holds 100% of genesis supply, mines the bootstrap window) | `FS13xm5oxJvGKd7194u5f2paA9WaaQpZSvz7Tzu8uXDngxromi3Lnv` |
| Block producer (founder-operated PoS producer) | `FS13yDA1PGpsRQ2zDSVhiwyJVP6zhYdjaiekE2bvfYC6ChZg95ZemM` |
| Starter-DESO faucet (testnet onboarding drips) | `FS13yCzGJGuQNmrsDn6Et74LphomuxnbsshZ3Rwo9k7iuxtLVRqyFV` |
| Validator 1 (DeSo identity key) | `FS13wSsnnLhYeVfjNyvc8HE6f7iWGG6bHJx6Kb5wKR513QJrNQdq5W` |
| Validator 2 (DeSo identity key) | `FS13vueAs3CmmPBjm5PnsyjbiUHegNHyNXmCcLdu1A6hh9tc2qHB74` |
| Validator 3 (DeSo identity key) | `FS13w9xhKm1vbvDWQKSvZPCqNkQMLFDurJ2TRNw5zjsod4kM1DS1wE` |

Governance mechanics for the founder-root key are covered in
[GOVERNANCE.md](GOVERNANCE.md). Validator BLS voting keys are distinct from
the DeSo keys above; see
[../../backend/validators/README.md](../../backend/validators/README.md) for
the two-mnemonic identity model.

## Useful endpoints

Served by every node's API port (e.g. `https://node.forked.social/api/v0/…`):

| Endpoint | Purpose |
|---|---|
| `GET /api/v0/health-check` | Liveness probe |
| `GET /api/v0/current-epoch-progress` | Epoch/view progress, leader schedule. **Note:** there is no `get-current-epoch-progress` route — this is the real one. May return HTTP 500 before any validator has registered (modulo on an empty leader schedule); this resolves itself after the first registration. |
| `GET /api/v0/validators/<pubkey>` | Registration, domain, voting pubkey, status, stake |
| `GET /api/v0/stake/<validator>/<staker>` | Stake of `<staker>` with `<validator>` (order: validator first — route name `GetStakeForValidatorAndStaker`) |
