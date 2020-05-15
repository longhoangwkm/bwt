# Bitcoin Wallet Tracker

`bwt` is a lightweight wallet xpub tracker and query engine for Bitcoin, implemented in Rust.

:small_orange_diamond: Personal HD wallet indexer (EPS-like)<br>
:small_orange_diamond: Electrum JSON-RPC server (protocol version 1.4)<br>
:small_orange_diamond: HTTP REST API ([docs below](#http-api))<br>
:small_orange_diamond: Real-time notifications with Server-Sent-Events or Web Hooks

> ⚠️ This is early alpha software that is likely to be buggy, use with care.

- [Setup](#setup)
  - [Electrum-only mode](#electrum-only-mode)
  - [Real-time updates](#real-time-updates)
  - [Advanced options](#advanced-options)
- [HTTP API](#http-api)
  - [HD Wallets](#hd-wallets)
  - [Transactions](#transactions)
  - [Outputs](#outputs)
  - [Addresses / Scripthashes](#addresses-scripthashes)
  - [Server-Sent Events](#server-sent-events)
  - [Mempool & Fees](#mempool--fees)
  - [Miscellaneous](#miscellaneous)
- [Web Hooks](#web-hooks)
- [Example wallet client](#example-wallet-client)
- [Followup work](#followup-work)
- [Thanks](#thanks)
 
## Setup

Get yourself a synced bitcoin node (`txindex` is not required) and install bwt using one of the methods below:

#### From source

[Install Rust](https://rustup.rs/) and:

```bash
$ git clone https://github.com/shesek/bwt && cd bwt
$ cargo build --release
$ ./target/release/bwt  --network mainnet --xpub <xpub1> --xpub <xpub2>
```

#### With Docker

Assuming your bitcoin datadir is at `~/.bitcoin`,

```bash
$ docker run --net host -v ~/.bitcoin:/bitcoin shesek/bwt --network mainnet --xpub <xpub1> --xpub <xpub2>
```

#### Running bwt

`bwt --xpub <xpub>` should be sufficient to get you rolling.

You can also configure the `--network` (defaults to `mainnet`),
your `--bitcoind-url` (defaults to `http://127.0.0.1:<default-rpc-port>`),
`--bitcoind-dir` (defaults to `~/.bitcoin`) and
`--bitcoind-cred <user:pass>` (defaults to using the cookie file from `bitcoind-dir`).

You can set multiple `--xpub` to track. This also supports ypubs and zpubs.

Be default, the Electrum server will be bound on port `50001`/`60001`/`60401` (according to the network)
and the HTTP server will be bound on port `3060`. This can be controlled with `--electrum-rpc-addr`
and `--http-server-addr`.

> ⚠️ Both the HTTP API server and the Electrum server are *unauthenticated and unencrypted.*
If you're exposing them over the internet, they should be put behind something like an SSH tunnel,
VPN, or a Tor hidden service.

You may set `-v` to increase verbosity or `-vv` to increase it more.

See `--help` for the full list of options.

### Electrum-only mode

If you're only interested in the Electrum server, you may disable the HTTP API server
and the web hooks support by building bwt with `--no-default-features --features electrum`
or using the `shesek/bwt:electrum` docker image.

This removes several large dependencies and disables the `track-spends` database index
(which is not needed for the electrum server).


### Real-time updates

By default, bwt will query bitcoind for new blocks/transactions every 5 seconds.
This can be adjusted with `--poll-interval <seconds>`.

To get *real* real-time updates, you may configure your bitcoind node to send a `POST /sync` request to the bwt
http server whenever a new block or a wallet transaction is found, using the `walletnotify` and `blocknotify` options.

Example bitcoind configuration:
```
walletnotify=curl -X POST http://localhost:3060/sync
blocknotify=curl -X POST http://localhost:3060/sync
```

After verifying this works, you may increase your `--interval-poll` to avoid unnecessary indexing and reduce cpu usage.

If you're using the electrum-only mode without the http server, you may instead configure bwt to bind
on a unix socket using `--unix-listener-path <path>` and open a connection to it initiate an indexer sync.

For example, start with `--unix-listener-path /home/satoshi/bwt-sync-socket` and configure your bitcoind with:
```
walletnotify=nc -U /home/satoshi/bwt-sync-socket
blocknotify=nc -U /home/satoshi/bwt-sync-socket
```

(If `nc` is not available, you may also use `socat - UNIX-CONNECT:/home/satoshi/bwt-sync-socket`)

### Advanced options

##### Gap limit

You may configure the gap limit with `--gap--limit <N>` (defaults to 20).
The gap limit sets the maximum number of consecutive unused addresses to be imported before assuming there are no more used addresses to be discovered.

You can set a higher gap limit for the inital imports with `--initial-gap-limit <N>` (defaults to 50).
Higher value means less rescans.

##### Rescan policy / wallet birthday

You may specify a rescan policy with the key's birthday to indicate how far back it should scan,
using `--xpub <xpub>:<rescan>`, where `<rescan>` is one of  `all` (rescan from the beginning, the default),
`none` (don't rescan at all), the key birthday formatted as `yyyy-mm-dd`, or the birthday as a unix timestamp.

## HTTP API

All the endpoints return JSON. All bitcoin amounts are in satoshis.

### HD Wallets

> Note: Every `--xpub` specified will be represented as two wallet entries, one for the external chain (used for receive addresses)
and one for the internal chain (used for change addresses). You can associate the wallets to their parent xpub using the `origin` field.

#### `GET /hd`

Get a map of all tracked HD wallets, as a json object indexed by the fingerprint.

<details><summary>Expand...</summary><p></p>

See [`GET /hd/:fingerprint`](#get-hdfingerprint) below for the full wallet json format.

Example:
```
$ curl localhost:3060/hd

{
  "9f0d3265": {
    "master": "tpubDAfKzTwp5MBqcSM3PUqkGjftNDSgKYtKQNoT6CosG7oGDAKpQWgGmB7t2VBt5a9z2k1u7F7FZnMzDtDmdnUwRcdiVakHXt4n7uXCd8LFJzz",
    "origin": "3f37f4f0/0",
    "network": "regtest",
    ...
  },
  "53cc57c8": {
    "xpub": "tpubDAfKzTwp5MBqYacHmEvfVGZVAUSCFpSd3SrAhZkZSvL7XBNcAfSLH4rEGmFH5dePuRcuaJMxGyqRRRHqhdYAJq2TyvQqrVbrov7suU1aLkg",
    "origin": "3f37f4f0/1",
    "network": "regtest",
    ...
  },
  ...
}
```
</details>

#### `GET /hd/:fingerprint`

Get information about the HD wallet identified by the hex-encoded `fingerprint`.

<details><summary>Expand...</summary><p></p>

Returned fields:
- `xpub` - the xpubkey in base58 encoding
- `origin` - parent extended key this wallet is derived from, in `<fingerprint>/<derivation-index>` format
- `network` - the network this wallet belongs to (`bitcoin`, `testnet` or `regtest`)
- `script_type` - the scriptpubkey type used by this wallet (`p2pkh`, `p2wpkh` or `p2shp2wpkh`)
- `gap_limit` - the gap limited configured for this wallet
- `initial_gap_limit` - the gap limit used during the initial import
- `rescan_policy` - how far back rescanning should take place
- `max_funded_index` - the maximum derivation index that is known to have history
- `max_imported_index` - the maximum derivation index imported into bitcoind
- `done_initial_import` - a boolean indicating whether we're done importing addresses for this wallet

Example:
```
$ curl localhost:3060/hd/15cb9edc
{
  "xpub": "tpubDAFgf5upzxiDvMNWdTobtLv7roPaSbKRc4oK98pJ6D6egLTKm94QwkMu9k8Sf4oHcvWPaan5aTqYqjdDVkeSUwpmQpaY1zPADHDHLdhLJYx",
  "origin": "a9c5dde1/1",
  "network": "regtest",
  "script_type": "p2wpkh",
  "gap_limit": 20,
  "initial_gap_limit": 50,
  "rescan_policy": {
    "since": 0
  },
  "max_funded_index": 102,
  "max_imported_index": 122,
  "done_initial_import": true
}
```
</details>

#### `GET /hd/:fingerprint/:derivation_index`

Get the script entry for the HD key at derivation index `derivation_index`.

<details><summary>Expand...</summary><p></p>

Returned fields:
- `scripthash`
- `address`
- `origin`

Example:
```
$ curl localhost:3060/hd/15cb9edc/8

{
  "scripthash": "528f5ed05cddd6fa43881a1e02557ce5c5db2400a5e6887d74d2ce867c7eabae",
  "address": "bcrt1qcgzcp5dg52z2f3cyh8a0805a2hms9adg0zj3fu",
  "origin": "15cb9edc/8"
}
```
</details>

#### `GET /hd/:fingerprint/next`

Get the next unused address in the specified HD wallet.

<details><summary>Expand...</summary><p></p>

Issues a 307 redirection to the url of the next derivation index (`/hd/:fingerprint/:derivation-index`) *and* responds with the derivation index in the responses body.

Note that the returned address is not marked as used until receiving funds; If you wish to skip it and generate a different
address without receiving funds to it, you can specify an explicit derivation index instead.

Examples:
```
$ curl localhost:3060/hd/7caf9d54/next

< HTTP/1.1 307 Temporary Redirect
< Location: /hd/7caf9d54/104
104

# Follow the redirect to get the json with the address/scripthash

$ curl --location localhost:3060/hd/7caf9d54/next

{
  "scripthash": "3baba97cee91b29a96c1de055600c8a25bc0f877797824ef8d2e8750fc5e1afe",
  "address": "mizsjvWUjiiqxajrBZhcdrmmkbfa12ABSq",
  "origin": "7caf9d54/104"
}
```
</details>

#### `GET /hd/:fingerprint/gap`

Get the current maximum number of consecutive unused addresses in the specified HD wallet.

<details><summary>Expand...</summary><p></p>

Example:
```
$ curl localhost:3060/hd/15cb9edc/gap

7
```
</details>

### Transactions

#### `GET /tx/:txid`

Get the transaction with contextual wallet information about funded outputs and spent inputs.

<details><summary>Expand...</summary><p></p>

Only available for wallet transactions.

Returned fields:
- `txid`
- `status` - `confirmed` or `unconfirmed`
- `block_height` - the confirming block height (available for confirmed transactions only)
- `fee` (may not be available)
- `funding` - contains an entry for every output created by this transaction that is owned by the wallet
  - `vout` - the output index
  - `amount` - the output amount
  - `scripthash` - the scripthash funded by this output
  - `address` - the address funded by this output
  - `origin` - hd wallet origin information, in `<fingerprint>/<derivation-index>` format
  - `spent_by` - the transaction input spending this output in `txid:vin` format, or `null` for unspent outputs (only available with `track-spends`)
- `spending` - contains an entry for every wallet utxo spent by this transaction input
  - `vin` - the input index
  - `amount` - the amount of the previous output spent by this input
  - `scripthash` - the scripthash of the previous output spent by this input
  - `address` - the address of the previous output spent by this input
  - `origin` - hd wallet origin information
  - `prevout` - the `<txid>:<vout>` being spent
- `balance_change` - the net change to the wallet balance inflicted by this transaction

Example:
```
$ curl localhost:3060/tx/e700187477d262f370b4f1dfd17c496d108524ee2d440a0b7e476f66da872dda

{
  "txid": "e700187477d262f370b4f1dfd17c496d108524ee2d440a0b7e476f66da872dda",
  "status": "confirmed",
  "block_height": 113,
  "fee": 141,
  "funding": [
    {
      "vout": 1,
      "scripthash": "6bf2d435bc4e020d839900d708f02c721728ca6793024919c2c5bc029c00f033",
      "address": "bcrt1qu04qqzwkjvya65g2agwx5gnqvgzwpjkr6q5jvf",
      "origin": "364476e3/6",
      "amount": 949373,
      "spent_by": "950cc16e572062fa16956c4244738b35ea7b05e16c8efbd6b9812d561d68be3a:0"
    }
  ],
  "spending": [
    {
      "vin": 0,
      "scripthash": "a55c30f4f7d79600d568bdfa0b4f48cdce4e59b6ffbf286e99856c3e8699740d",
      "address": "bcrt1qxsvdm3jmwr79u67d82s08uykw6a82agzy42c6y",
      "origin": "364476e3/5",
      "amount": 1049514,
      "prevout": "70650243572b90705f7fe95c9f30a85a0cc55e4ea3159a8ada5f4d62d9841d7b:1"
    }
  ],
  "balance_change": -100141
}
```
</details>

#### `GET /tx/:txid/verbose`

Get the transaction in JSON as formatted by bitcoind's `getrawtransaction` with `verbose=true`.

#### `GET /tx/:txid/hex`

Get the raw transaction formatted as a hex string.

#### `GET /txs/since/:block-height`

Get all wallet transactions confirmed at or after `block-height`, plus all unconfirmed transactions,
for all tracked scripthashes.

<details><summary>Expand...</summary><p></p>

Returned in the same format as [`GET /tx/:txid`](https://gist.github.com/shesek/303a9ded987cad797266ded737b05641#get-txtxid).

*This is not yet very efficient.*

</details>

#### `GET /txs/since/:block-height/compact`

Get a compact minimal representation of all wallet transactions since `block-height`.

<details><summary>Expand...</summary><p></p>

Returns a simple JSON array of `[txid, block_height]` tuples, where `block_height` is null for unconfirmed transactions.

Example:
```
$ curl localhost:3060/txs/since/105/compcat
[
  ["859d5c41661426ab13a7816b9e845a3353b66f00a3c14bc412d20f87dcf19caa", 105],
  ["3c3c8722b493bcf43adab323581ea1da9f9a9e79628c0d4c89793f7fe21b68cf", 107],
  ["e51414f57bdee681d48a6ade696049c4d7569a062278803fb7968d9a022c6a96", null],
  ...
]
```

</details>

#### `POST /tx`

Broadcast a raw transaction to the Bitcoin network.

<details><summary>Expand...</summary><p></p>

Returns the `txid` on success.

Body parameters:
- `tx_hex` - the raw transaction encoded as a hex string

Example:

```
$ curl -X POST localhost:3060/tx -H 'Content-Type: application/json' \
       -d '{"tx_hex":"<hex-serialized-tx>"}'

33047288f0502eb3f2ad0729f6cfa24a8db87842f9c9a8eba7c0dbfaf7ea75b4
```

</details>

### Outputs

#### `GET /txo/:txid/:vout`

Get information about the specified wallet output.

<details><summary>Expand...</summary><p></p>

Returned fields:
- `txid` - the output transaction
- `vout` - the output index
- `amount` - the output amount
- `scripthash` - the scripthash funded by this output
- `address` - the address funded by this output
- `origin` - hd wallet origin information, in `<fingerprint>/<derivation-index>` format
- `status` - `confirmed` or `unconfirmed`
- `block_height` - the confirming block height (available for confirmed transactions only)
- `spent_by` - the transaction input spending this output in `txid:vin` format, or `null` for unspent outputs (only available with `track-spends`)


Example:
```
$ curl localhost:3060/txo/1b1170ac5996df9255299ae47b26ec3ad57c9801bc7bae68203b1222350d52fe/0
{
  "txid": "1b1170ac5996df9255299ae47b26ec3ad57c9801bc7bae68203b1222350d52fe",
  "vout": 0,
  "amount": 99791,
  "scripthash": "42c8d22a39047d79070acc984c7d3e6ee9cca69289c84c75900e05c52adb5e8e",
  "address": "bcrt1qknn7fg0w33j9gcsdtdd6k02llpjmqyg8a36728",
  "origin": "364476e3/15",
  "status": "confirmed",
  "block_height": 161,
  "spent_by": null
}
```
</details>

#### `GET /utxos`

Get all unspent wallet outputs.

<details><summary>Expand...</summary><p></p>

Returns a JSON array in the same format as [`GET /txo/:txid/:vout`](#get-txotxidvout).

Query string parameters:
- `min_conf` - minimum number of confirmations, defaults to 0
- `include_unsafe` - whether to include outputs that are not safe to spend (unconfirmed from outside keys or with RBF), defaults to true

Example:
```
$ curl localhost:3060/utxos?min_conf=1

[
  {
    "txid": "1973551cc7670237606561ba3f7579d46d38e7145a72cf6a55ff8975e7143fee",
    "vout": 0,
    "amount": 99791,
    ...
  },
  ...
]
```
</details>

### Addresses / Scripthashes


#### `GET /address/:address`
#### `GET /scripthash/:scripthash`

Get basic information about the provided scripthash/address.

<details><summary>Expand...</summary><p></p>

Returned fields:
- `scripthash`
- `address`
- `origin` - hd wallet origin information, in `<fingerprint>/<derivation-index>` format

Example:
```
$ curl localhost:3060/address/bcrt1qh0wa4uezedve99vd62dlungplq23e59cnw0j2s
{
  "scripthash": "c511375da743d7f6276db6cdaf9f03d7244c74d5569c9a862433e37c5bc84cb2",
  "address": "bcrt1qh0wa4uezedve99vd62dlungplq23e59cnw0j2s",
  "origin": "e583e3c5/6"
}
```

</details>

#### `GET /address/:address/stats`
#### `GET /scripthash/:scripthash/stats`

Get basic information and stats about the provided scripthash/address.

<details><summary>Expand...</summary><p></p>

Returned fields:
- `scripthash`
- `address`
- `origin` - hd wallet origin information, in `<fingerprint>/<derivation-index>` format
- `tx_count`
- `confirmed_balanace`
- `unconfirmed_balanace`

Example:
```
$ curl localhost:3060/address/bcrt1qh0wa4uezedve99vd62dlungplq23e59cnw0j2s/stats
{
  "scripthash": "c511375da743d7f6276db6cdaf9f03d7244c74d5569c9a862433e37c5bc84cb2",
  "address": "bcrt1qh0wa4uezedve99vd62dlungplq23e59cnw0j2s",
  "origin": "e583e3c5/6",
  "tx_count": 7,
  "confirmed_balance": 3240000,
  "unconfirmed_balance": 0
}
```
</details>

#### `GET /address/:address/utxos`
#### `GET /scripthash/:scripthash/utxos`

Get the list of unspent transaction outputs owned by the scripthash/address.

<details><summary>Expand...</summary><p></p>

Returns a JSON array in the same format as [`GET /txo/:txid/:vout`](#get-txotxidvout).

Query string parameters:
- `min_conf` - minimum number of confirmations, defaults to 0
- `include_unsafe` - whether to include outputs that are not safe to spend (unconfirmed from outside keys or with RBF), defaults to true

Examples:
```
$ curl localhost:3060/address/bcrt1qh0wa4uezedve99vd62dlungplq23e59cnw0j2s/utxos
[
  {
    "txid": "664fba0bcc745b05fda0fbf1f6fb6fc003afd82e64caad2c9fea0e3d566f6a58",
    "vout": 1,
    "amount": 1500000,
    "scripthash": "b24cc85b7ce33324869a9c56d5744c24d7039fafcdb66d27f6d743a75d3711c5",
    "address": "bcrt1qh0wa4uezedve99vd62dlungplq23e59cnw0j2s",
    "origin": "e583e3c5/6",
    "status": "confirmed",
    "block_height": 114,
    "spent_by": null
  },
  {
    "txid": "3a1c4dea8d376a2762dd9be1d39f7f13376b4c9ccb961725574689183c20cb90",
    "vout": 1,
    "amount": 1440000,
    "scripthash": "b24cc85b7ce33324869a9c56d5744c24d7039fafcdb66d27f6d743a75d3711c5",
    "address": "bcrt1qh0wa4uezedve99vd62dlungplq23e59cnw0j2s",
    "origin": "e583e3c5/6",
    "status": "confirmed",
    "block_height": 115,
    "spent_by": null
  },
  ...
]
$ curl localhost:3060/scripthash/c511375da743d7f6276db6cdaf9f03d7244c74d5569c9a862433e37c5bc84cb2/utxos?min_conf=1
```
</details>

#### `GET /address/:address/txs`
#### `GET /scripthash/:scripthash/txs`

Get the list of all transactions in the history of the address/scripthash.

<details><summary>Expand...</summary><p></p>

Returned in the same transaction format as `GET /tx/:txid`.

Example:
```
$ curl localhost:3060/address/bcrt1qh0wa4uezedve99vd62dlungplq23e59cnw0j2s/txs
[
  {
    "txid": "859d5c41661426ab13a7816b9e845a3353b66f00a3c14bc412d20f87dcf19caa",
    "status": "confirmed",
    "block_height": 105,
    "fee": 144,
    "funding": [ ... ],
    "spending": [ ...],
    "balance_change": 11000000
  },
  ...
]
```

</details>

#### `GET /address/:address/txs/compact`
#### `GET /scripthash/:scripthash/txs/compact`

Get a compact minimal representation of the scripthash/address history.

<details><summary>Expand...</summary><p></p>

Returns a simple JSON array of `[txid, block_height]` tuples, where `block_height` is null for unconfirmed transactions.

Example:
```
$ curl localhost:3060/scripthash/c511375da743d7f6276db6cdaf9f03d7244c74d5569c9a862433e37c5bc84cb2/txs/minimal
[
  ["859d5c41661426ab13a7816b9e845a3353b66f00a3c14bc412d20f87dcf19caa", 105],
  ["3c3c8722b493bcf43adab323581ea1da9f9a9e79628c0d4c89793f7fe21b68cf", 107],
  ["e51414f57bdee681d48a6ade696049c4d7569a062278803fb7968d9a022c6a96", null],
  ...
]
```

</details>

### Server-Sent Events

#### Available event categories

- `ChainTip(BlockHeight, BlockHash)` - emitted whenever a new block extends the best chain.
- `Reorg(BlockHeight, PrevBlockHash, CurrBlockHash)` - indicates that a re-org was detected on BlockHeight, with the previous block hash at this height and the current one.
- `Transaction(Txid, BlockHeight)` - emitted for new transactions as well as transactions changing their confirmation status (typically from unconfirmed to confirmed, possibly the other way around in case of reorgs).
- `TransactionReplaced(Txid)` - indicates that the transaction conflicts with another transaction and can no longer be confirmed (aka double-spent).
- `History(ScriptHash, Txid, BlockHeight)` - emitted whenever the history of a scripthash changes.
- `TxoCreated(OutPoint, BlockHeight)` - emitted for new unspent wallet outputs.
- `TxoSpent(OutPoint, TxInput, BlockHeight)` - emitted for wallet outputs spends.

For unconfirmed transactions, `BlockHeight` will be `null`.

#### `GET /stream`

Subscribe to a real-time notification stream of indexer update events.

<details><summary>Expand...</summary><p></p>

Query string parameters for filtering the event stream:
- `category`
- `scripthash`
- `outpoint`

Examples:
```bash
$ curl localhost:3060/stream
< HTTP/1.1 200 OK
< content-type: text/event-stream

data:{"category":"ChainTip","params":[630000,"138f2e0c6c0377ff9e5a7a6fb3386ba3ea5afcb0bd551010b4e79ff8ce337b53"]}

data:{"category":"Transaction","params":["1fbc4a22ace7abcdd7b76630d7a7d6f9cf158842a973ac796ad89db2ee8a846e",630000]}

data:{"category":"History","params":["796d12182f1a1dd89f478a44c2f439d8a68f66d6cc11e8f525bdcce69cd24c27","1fbc4a22ace7abcdd7b76630d7a7d6f9cf158842a973ac796ad89db2ee8a846e",630000]}

data:{"category":"History","params":["233fbc04a5fbd4527bded1fec6ddaf026d9db3b7171f73800e20dd1c455a562f","1fbc4a22ace7abcdd7b76630d7a7d6f9cf158842a973ac796ad89db2ee8a846e",630000]}

data:{"category":"TxoCreated","params":["1fbc4a22ace7abcdd7b76630d7a7d6f9cf158842a973ac796ad89db2ee8a846e:1",630000]}

data:{"category":"TxoSpent","params":["e51414f57bdee681d48a6ade696049c4d7569a062278803fb7968d9a022c6a96:1","1fbc4a22ace7abcdd7b76630d7a7d6f9cf158842a973ac796ad89db2ee8a846e:0",630000]}
```

```
$ curl localhost:3060/stream?category=ChainTip

data:{"category":"ChainTip","params":[630000,"661e3adef501b49457d5efc01f9700137a9c7340f1891895ef2a49ae35bfc069"]}

data:{"category":"ChainTip","params":[630001,"1c293df0c95d94a345e7578868ee679c9f73b905ac74da51e692af18e0425387"]}
```

```
$ curl localhost:3060/stream?outpoint=521d0991128a680d01e5a4f748f6ad8a6a1dbde485020f26db1031017be39554:0

data:{"category":"TxoSpent","params":["521d0991128a680d01e5a4f748f6ad8a6a1dbde485020f26db1031017be39554:0","1569f341184eda501b070b3cb8005bc06e6e293442422c41c2f9e451bb0a328c:0",630000]}
```

</details>

#### `GET /address/:address/stream`
#### `GET /scripthash/:scripthash/stream`

Subscribe to a real-time notification stream of indexer update events related to the specified scripthash/address.

<details><summary>Expand...</summary><p></p>

This is equivalent to `GET /stream?scripthash=<scripthash>`.

Example:
```
$ curl localhost:3060/scripthash/96a12ea14fa3fdbf93323980ca9ad8af4ad4fb00a8feaa4b92d4639ebbc5ff19/stream

data:{"category":"History","params":["96a12ea14fa3fdbf93323980ca9ad8af4ad4fb00a8feaa4b92d4639ebbc5ff19","1fbc4a22ace7abcdd7b76630d7a7d6f9cf1
58842a973ac796ad89db2ee8a846e",630000]}

data:{"category":"TxoCreated","params":["1fbc4a22ace7abcdd7b76630d7a7d6f9cf158842a973ac796ad89db2ee8a846e:0",630000]}
```
</details>


### Mempool & Fees

#### `GET /mempool/histogram`

Get the mempool feerate distribution histogram.

<details><summary>Expand...</summary><p></p>

Returns an array of `(feerate, vsize)` tuples, where each entry's `vsize` is the total vsize of transactions
paying more than `feerate` but less than the previous entry's `feerate` (except for the first entry, which has no upper bound).
This matches the format used by the Electrum RPC protocol for `mempool.get_fee_histogram`.

Cached for one minute.

Example:

```
$ curl localhost:3060/mempool/histogram

[[53.01, 102131], [38.56, 110990], [34.12, 138976], [24.34, 112619], [3.16, 246346], [2.92, 239701], [1.1, 775272]]
```

> In this example, there are transactions weighting a total of 102,131 vbytes that are paying more than 53 sat/vB,
110,990 vbytes of transactions paying between 38 and 53 sat/vB, 138,976 vbytes paying between 34 and 38, etc.

</details>

#### `GET /fee-estimate/:target`

Get the feerate estimate for confirming within `target` blocks.
Uses bitcoind's `smartestimatefee`.

<details><summary>Expand...</summary><p></p>

Returned in `sat/vB`, or `null` if no estimate is available.

Cached for one minute.

Example:
```
$ curl localhost:3060/fee-estimate/3

5.61
```

</details>

### Miscellaneous

#### `POST /sync`

Trigger an immediate indexer sync. See [Real-time updates](#real-time-updates).

#### `GET /dump`

Dumps the contents of the index store as JSON.

#### `GET /debug`

Dumps the contents of the index store as a debug string.

## Web Hooks

You can set `--webhook-url <url>` to have bwt send push notifications as a `POST` request to the provided `<url>`. Requests will be sent with a JSON-serialized *array* of one or more index updates as the body.

It is recommended to include a secret key within the URL to verify the authenticity of the request.

You can specify multiple `--webhook-url` to notify all of them.

Note that bwt currently attempts to send the webhook once and does not retry in case of failures.
It is recommended to occasionally catch up using the [`GET /txs/since/:block-height`](#get-txssinceblock-height) endpoint.

Tip: services like [webhook.site](https://webhook.site/) or [requestbin](http://requestbin.net/) can come in handy for debugging webhooks. (needless to say, for non-privacy-sensitive regtest/testnet use only)


## Example wallet client

An example JavaScript implementation utilizing the HTTP API for wallet tracking, using [`fetch`](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API) and [`EventSource`](https://developer.mozilla.org/en-US/docs/Web/API/EventSource)
(available on modern browsers and nodejs),
[is available here](https://github.com/shesek/bwt/blob/master/examples/wallet-tracker.js).

## Followup work

- Output descriptors! <3
- Coin selection
- Tests
- Similarly to electrum-personal-server, bwt manages its database entirely in-memory and has no persistent storage.
  This is expected to change at some point.

PRs welcome!

## Thanks

- [@romanz](https://github.com/romanz)'s [electrs](https://github.com/romanz/electrs) for the fantastic electrum server implementation that bwt is based on.

- [@chris-belcher](https://github.com/chris-belcher)'s [electrum-personal-server](https://github.com/chris-belcher/electrum-personal-server) for inspiring this project and the personal tracker model.

- [rust-bitcoin](https://github.com/rust-bitcoin), [rust-bitcoincore-rpc](https://github.com/rust-bitcoin/rust-bitcoincore-rpc) and the other incredible modules from the rust-bitcoin family.

## License

MIT