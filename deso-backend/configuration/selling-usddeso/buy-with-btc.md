---
description: Description of flags related to selling DESO for BTC on your Node
---

# Buy with BTC

> **forked-social note:** the BitcoinExchange minting path this feature
> configures is **economically neutralized on this network**
> (`DeSoNanosPurchasedAtGenesis = MaxNanos`) — effectively no new coins can
> be minted by burning BTC. The transaction type and these flags still
> exist in the code, so the reference is kept verbatim below.

## Buy DESO BTC Address

`--buy-deso-btc-address`

Type: String

Default: None

BTC Address that will receive BTC for all Wyre Wallet Orders and 'Buy With BTC' purchases

## BlockCypher API Key

`--block-cypher-api-key`

Type: String

Default: None

When specified, this key is used to power the BitcoinExchange flow and to check for double-spends in the mempool
