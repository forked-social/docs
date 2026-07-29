# forked-social Governance

This document describes who can change the forked-social network's on-chain
parameters and how those changes are applied. The launch-day parameters
themselves are listed in [NETWORK.md](NETWORK.md).

## The founder-root key

forked-social launches with a single governance key:

```
FS13vrbSsSRefiNeoJK5uHMKgCwzJJ3NMCFdEKpui7qPX4X7xSLSko
```

In the code this key is `FounderRootPubKeyBase58Check`
(`core/lib/fork_params.go`) and it replaces the upstream DeSo "Architect"
key everywhere that key was used (`core/lib/constants.go` sets
`ArchitectPubKeyBase58Check = FounderRootPubKeyBase58Check`). It is:

- the **sole param-updater**: the only public key whose on-chain
  `UpdateGlobalParams` transactions are valid, and
- the **admin and super-admin** of the seed node deployment
  (`ADMIN_PUBLIC_KEYS` / `SUPER_ADMIN_PUBLIC_KEYS` in `backend/deploy.env`).

The mnemonic backing this key is runtime-injected and never committed; only
the public key appears in the repository.

Distinct from the founder-root key, a **founder-treasury** key
(`FS13zB3hyv7V3nnMx5Dikp8NMGpDfrcQ1MWVz8nptRygd446NTVawA`) receives the
10-basis-point trading fee on $DESO-denominated DEX markets — where
upstream routed this fee to Openfund treasuries, the fork repoints it at
its own treasury (fee economics are unchanged;
`backend/routes/dao_coin_exchange_with_fees.go`).

## On-chain parameter updates (UpdateGlobalParams)

Protocol-level global parameters (network fee floors, per-action fees such as
`CreateNFTFeeNanos`, USD/Bitcoin exchange metadata, and related knobs read
via `GetCurrentGlobalParamsEntry`) are changed with an
`UpdateGlobalParams` transaction:

1. The transaction is an on-chain transaction of type
   `UPDATE_GLOBAL_PARAMS`; its entire payload travels in the transaction's
   `ExtraData` map (`UpdateGlobalParamsMetadata` in `core/lib/network.go`).
2. Consensus accepts the transaction **only if it is signed by the
   founder-root key** (enforced in `_connectUpdateGlobalParams` via
   `GetParamUpdaterPublicKeys`, `core/lib/block_view.go`). An
   `UpdateGlobalParams` transaction from any other key is invalid and cannot
   be mined.
3. Once mined, the new values apply from that block forward on every node —
   there is no voting period and no validator sign-off. Validator upgrades
   weigh in only when a change requires shipping new node software.

## Backend admin surface

The seed node additionally exposes an admin API that proxies parameter
changes and node administration:

- `POST /api/v0/admin/update-global-params` — constructs and submits the
  `UpdateGlobalParams` transaction described above. On the seed deployment
  this route is restricted to the super-admin key.
- Other `/api/v0/admin/…` endpoints (see
  [../deso-backend/api/admin-endpoints.md](../deso-backend/api/admin-endpoints.md))
  follow the same admin/super-admin gating, which each deployment controls
  with its own `ADMIN_PUBLIC_KEYS` / `SUPER_ADMIN_PUBLIC_KEYS` settings
  (documented in
  [../deso-backend/configuration/admins.md](../deso-backend/configuration/admins.md)).

A self-hosted node operator who does not set admin keys simply has no admin
endpoints — none of this affects block following or consensus.

## What governance does not cover

- **Consensus rules and fork heights** (block 1 setup, block 300 cutover,
  epoch duration) are fixed in `core/lib/fork_params.go` and cannot be
  changed by parameter updates; changing them is a hard fork that requires
  every node to upgrade.
- **Total supply** cannot be governed upward: 30,000,000 coins exist, PoW
  rewards are disabled, staking APY defaults to 0, and the BitcoinExchange
  minting path is economically neutralized (see
  [NETWORK.md](NETWORK.md#genesis-and-supply)). Parameter updates do not
  re-enable coin issuance.
- **Validators and staking** are driven by on-chain registration/stake
  transactions from any funded key — the founder-root key does not approve
  validator sets (see
  [../operators/run-a-validator.md](../operators/run-a-validator.md)).
