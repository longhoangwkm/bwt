# bwt-daemon

A nodejs library for programmatically controlling the Bitcoin Wallet Tracker daemons
using the `libbwt` ffi interface.

### Install

```
$ npm install bwt-daemon
```

This will download the `libbwt` library for your platform.
The currently supported platforms are linux-x64, macos-x64, windows-x64, linux-arm32v7 and linux-arm64v8.

The hash of the downloaded library is verified against the
[`SHA256SUMS`](https://github.com/shesek/bwt/blob/master/contrib/nodejs-bwt-daemon/SHA256SUMS)
file that ships with the npm package.

The library comes with the electrum and http servers by default.
If you're only interested in the Electrum server, you can install with `BWT_VARIANT=electrum_only npm install bwt-daemon`.
This reduces the download size by ~1.6MB.

> Note: `bwt-daemon` uses [`ffi-napi`](https://github.com/node-ffi-napi/node-ffi-napi), which requires
> a recent nodejs version. If you're running into errors during installation or segmentation faults,
> try updating to a newer version.

### Use

Below is the minimally viable configuration. If bitcoind is running at the default
location, on the default ports and with cookie auth enabled, this should Just Work™ \o/

```js
import BwtDaemon from 'bwt-daemon'

let bwt = await BwtDaemon({
  xpubs: [ [ 'xpub66...', 'now' ] ],
  electrum: true,
})

console.log('bwt electrum server ready on', bwt.electrum_addr)
```

With some more advanced options:

```js
let bwt = await BwtDaemon({
  // Network and Bitcoin Core RPC settings
  network: 'regtest',
  bitcoind_dir: '/home/satoshi/.bitcoin',
  bitcoind_url: 'http://127.0.0.1:9008/',
  bitcoind_wallet: 'bwt',

  // Descriptors or xpubs to track as an array of (desc_or_xpub, rescan_since) tuples
  // Use 'now' to look for new transactions only, or the unix timestamp to begin rescanning from.
  descriptors: [ [ 'wpkh(tpub61.../0/*)', 'now' ] ],
  xpubs: [ [ 'tpub66...', 'now' ] ],

  // Enable HTTP and Electrum servers
  http: true,
  electrum: true,

  // Bind on port 0 to use any available port (the default)
  electrum_addr: '127.0.0.1:0',
  http_addr: '127.0.0.1:0',

  // Set the gap limit of watched unused addresses
  gap_limit: 100,

  // Progress notifications for history scanning (a full rescan from genesis can take 20-30 minutes)
  progress_fn: progress => console.log('bwt progress %f%%', progress*100),
})

// Get the assigned address/port for the Electrum/HTTP servers
console.log('bwt electrum server ready on', bwt.electrum_addr)
console.log('bwt http server ready on', bwt.http_addr)

// Shutdown
bwt.shutdown()
```

See [`example.js`](https://github.com/shesek/bwt/blob/master/contrib/nodejs-bwt-daemon/example.js) for an even more complete
example, including connecting to the HTTP API.

### Options

#### Network and Bitcoin Core RPC
- `network`
- `bitcoind_dir`
- `bitcoind_wallet`
- `bitcoind_url`
- `bitcoind_auth`
- `bitcoind_cookie`

#### Address tracking
- `descriptors`
- `xpubs`
- `bare_xpubs`

#### General settings
- `verbose`
- `gap_limit`
- `initial_import_size`
- `poll_interval`
- `tx_broadcast_cmd`

#### Electrum
- `electrum`
- `electrum_addr`
- `electrum_skip_merkle`

#### HTTP
- `http`
- `http_addr`
- `http_cors`

#### Web Hooks
- `webhooks_urls`

#### UNIX only
- `unix_listener_path`

### License
MIT