# Running a forked-social Validator

Day-2 operations for validators on the forked-social network: how
registration works, what a validator host needs, how to monitor one, and
how unjailing works. The initial launch of the three founder validators is
covered in [launch-runbook.md](launch-runbook.md) and in the engineering
runbook, [../../backend/validators/README.md](../../backend/validators/README.md).

## The two-key identity model

Every validator has **two independent mnemonics**:

| Key | Signs | Where the seed lives |
|---|---|---|
| DeSo identity key (`FS1…`) | the register + stake transactions (it is a normal on-chain account) | operator vault only — never on the validator host |
| BLS voting key | PoS votes/proposals after the cutover (`POS_VALIDATOR_SEED` on the host) | `/opt/validatorN/validatorN.env`, 0600, runtime-injected |

The on-chain registration binds them: the register-as-validator transaction
records the BLS **voting public key** (derived from the BLS mnemonic) and a
voting authorization for the DeSo identity key. **The mnemonic configured
as `POS_VALIDATOR_SEED` must be the same one whose public key was
registered** — if they differ, the node cannot sign for its registered
identity, the mesh treats it as inactive, and with enough such validators
the chain stalls. The BLS mnemonic must have **no BIP39 passphrase**
anywhere in the pipeline.

All key-derivation paths (the bootstrap tool, the node, and
`validator-key-generator/`) use the same lib calls
(`lib.NewBLSKeystore` + `lib.CreateValidatorVotingAuthorizationPayload`),
so identical mnemonics guarantee identical keys.

## Anatomy of registration

Validators are **not** genesis-baked. Registration is open to any funded
key from block height 1 and consists of ordinary on-chain transactions:

1. **Fund** the validator's DeSo identity key (it needs coins to stake and
   to pay fees). At launch the bootstrap tool moved 5,125,000 coins to each
   validator identity from the miner key (a 5,000,000-coin stake plus a
   2.5% fee/burn buffer).
2. **Register** — a register-as-validator transaction that carries:
    - the **domain** the validator will be dialed on (the launch validators
      all registered under the existing `node.forked.social` record as
      `node.forked.social:42100`, `node.forked.social:42200`,
      `node.forked.social:42300` — same host, distinct P2P ports; the mesh
      dials whatever `host:port` is registered), and
   - the **BLS voting public key** + voting authorization derived from the
     BLS mnemonic.
   Backend route: `POST /api/v0/validators/register`.
3. **Stake** — a stake transaction locking coins behind the validator
   (launch validators stake exactly 5,000,000 coins = 5×10¹⁵ nanos, visible
   as `TotalStakeAmountNanos = 5000000000000000` — a launch-security
   posture against 50%+ attacks, intended to be lowered via normal
   unstake transactions as the network grows). Stake carries a lockup
   of `DefaultStakeLockupEpochDuration` = 3 epochs.

A registration takes effect in the leader schedule two epochs after the
epoch in which it landed (snapshots use a two-epoch lookback; epochs are
144 blocks). This is why launch-day registration is time-critical and why
a late registration appears to "do nothing" for a couple of epochs.

`POST /api/v0/validators/unregister` reverses a registration.

## Running a new validator (post-launch)

1. Provision a host with a **DNS A record** you will register (e.g.
   `validator4.example.net`) and the P2P port open (`42000` by default;
   register a distinct port if the host already runs another node, as the
   launch server does for validators 1–3).
2. Create the two mnemonics offline. Derive the DeSo public key with the
   identity CLI (`cd identity && npm run derive-keys -- "<mnemonic>"`).
   Optionally preview the BLS pubkey with `validator-key-generator/`
   (`go run main.go --bls-seed-phrase=… --deso-public-key=…`).
3. Fund the DeSo identity key with enough coins for the stake you intend
   plus fees. Remember supply is fixed at 30,000,000 coins and staking
   APY is 0% — validator economics are fees-only.
4. Register and stake (routes above, or the same flow the bootstrap tool
   automates for the launch validators).
5. Configure the node like any fork node (see
   [run-a-node.md](run-a-node.md)) plus:
   `POS_VALIDATOR_SEED=<the BLS mnemonic>` and
   `CONNECT_IPS=node.forked.social:42000`.
6. After the two-epoch lookback, confirm ACTIVE status and rotation
   through the leader schedule (monitoring below).

One node per BLS mnemonic. Never commit the mnemonic; the repo and compose
templates carry public keys only.

## Monitoring

| Check | Command |
|---|---|
| Liveness | `GET /api/v0/health-check` |
| This validator's status/stake | `GET /api/v0/validators/<DeSo pubkey of validator>` |
| Stakes behind a validator | `GET /api/v0/stake/<validator>/<staker>` |
| Epoch & leader view | `GET /api/v0/current-epoch-progress` |

Healthy signals: status `ACTIVE`; the validator's pubkey rotating through
`CurrentLeader` in `current-epoch-progress`; `CurrentView` advancing;
Fast-HotStuff view advancement lines in the node logs (`/opt/validatorN/data/logs`).

Note: `current-epoch-progress` is the real route — anything mentioning
`get-current-epoch-progress` is wrong — and it can return HTTP 500 while
zero validators are registered.

## Unjailing

A validator that misses its duties is jailed: it is excluded from block
production for `DefaultValidatorJailEpochDuration` = 3 epochs (inactivity
can escalate — the inactive-validator grace period is
`DefaultJailInactiveValidatorGracePeriodEpochs` = 48 epochs). A jailed
validator's node keeps syncing; it just stops being scheduled while the
jail holds. The operator lifts a jail early with the explicit unjail
transaction: `POST /api/v0/validators/unjail`.

## Chain wiped/stalled?

- A stalled chain shortly after height 300 means the first PoS validator
  set was empty or non-signing — see the launch runbook's failure section.
- Wiping a validator = clearing its data dir (chain + logs) and re-syncing
  from checkpoints, exactly like any node. Its registration, stake, and
  identity live on-chain and survive a node wipe; the same
  `POS_VALIDATOR_SEED` brings the same identity back.
