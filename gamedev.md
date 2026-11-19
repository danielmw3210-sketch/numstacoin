# Game integration: Numista testnet RPC

This is a developer reference for connecting a game backend to a
self-operated Numista testnet node. Numista Coins is experimental; testnet
balances and assets have no production value. There is no public Numista RPC
host or seed-peer service configured by this repository.

## Network and ports

| Service | Numista testnet port | Use |
| --- | ---: | --- |
| P2P | `48080` | Node-to-node traffic; not a game API |
| Daemon HTTP/JSON RPC | `48081` | Chain, transaction-pool, and node methods |
| Daemon ZMQ RPC | `48082` | Optional ZMQ interface |
| Wallet RPC | Chosen by operator | Wallet methods; pass `--rpc-bind-port` |

The daemon and wallet tools select Numista testnet by default, but explicitly
pass `--numista-testnet` in service scripts to make the intended network
clear. Do not pass Monero's `--testnet` or `--stagenet`; `--mainnet` selects
Monero mainnet. The Numista testnet has no Monero seed peers: connect nodes
explicitly with `--add-peer`.

For local development, start a daemon and send HTTP requests to loopback:

```sh
monerod --numista-testnet --rpc-bind-ip 127.0.0.1
curl --fail-with-body --silent --show-error \
  http://127.0.0.1:48081/get_info
```

JSON-RPC methods use `POST /json_rpc`:

```sh
curl --fail-with-body --silent --show-error \
  -H 'Content-Type: application/json' \
  --data '{"jsonrpc":"2.0","id":"0","method":"get_info"}' \
  http://127.0.0.1:48081/json_rpc
```

Run wallet RPC separately when the game needs wallet operations. Choose and
document its bind port (the wallet RPC port is not a Numista network
parameter), and pass `--numista-testnet` and a daemon address such as
`http://127.0.0.1:48081`. Wallet RPC authentication is enabled by default; if
it generates credentials, it writes them to
`monero-wallet-rpc.<port>.login`. Keep wallet RPC loopback-only for local
development. Never expose a wallet or unrestricted daemon RPC directly to the
internet; use an authenticated, encrypted, access-controlled service boundary
for remote clients. Do not use `--disable-rpc-login` on an exposed service.

## Daemon HTTP endpoints

These are daemon paths on `http://<node>:48081`. The binary `.bin` endpoints
use Monero's binary serialization; the other listed paths use JSON. The names
in parentheses are compatibility aliases.

| Path | Purpose |
| --- | --- |
| `/get_height` (`/getheight`) | Current chain height |
| `/get_info` (`/getinfo`) | Node/network status |
| `/get_blocks.bin` (`/getblocks.bin`) | Retrieve chain blocks and their transactions |
| `/get_blocks_by_height.bin` (`/getblocks_by_height.bin`) | Retrieve blocks by height |
| `/get_hashes.bin` (`/gethashes.bin`) | Retrieve block hashes |
| `/get_transactions` (`/gettransactions`) | Retrieve transactions by hash |
| `/get_alt_blocks_hashes` | Alternate-chain block hashes |
| `/is_key_image_spent` | Check key-image spend status |
| `/send_raw_transaction` (`/sendrawtransaction`) | Submit a serialized transaction |
| `/get_o_indexes.bin` | Transaction output indexes |
| `/get_outs.bin`, `/get_outs` | Output lookup |
| `/get_public_nodes` | Public-node list reported by this node |
| `/get_transaction_pool_hashes.bin` | Transaction-pool hashes (binary response) |
| `/get_transaction_pool_hashes` | Transaction-pool hashes (JSON response) |
| `/get_transaction_pool_stats` | Transaction-pool statistics |
| `/get_output_distribution.bin` | Output distribution (binary response) |
| `/get_limit` | Current network limits |
| `/start_mining`, `/stop_mining`, `/mining_status` | Node mining controls/status |
| `/get_peer_list`, `/get_net_stats` | Peer and network statistics |
| `/save_bc` | Save blockchain state |
| `/set_log_hash_rate`, `/set_log_level`, `/set_log_categories` | Logging controls |
| `/get_transaction_pool` | Pending transactions (unrestricted RPC only) |
| `/stop_daemon` | Stop the daemon |
| `/set_limit`, `/out_peers`, `/in_peers` | Change node limits/connections |
| `/update` | Daemon update check |
| `/pop_blocks` | Remove blocks from the local chain |

Some node-management paths and methods are disabled when daemon
`--restricted-rpc` is enabled. Keep the unrestricted RPC private; it includes
administrative operations such as mining controls, peer controls, daemon
shutdown, pool mutation, and blockchain pruning.

## Daemon JSON-RPC methods

Call these method names with JSON-RPC 2.0 at `POST /json_rpc`. Compatibility
aliases are shown in parentheses.

| Method | Purpose |
| --- | --- |
| `get_info` | Node/network status |
| `get_numista_asset` | Look up asset type, cap, issued supply, and metadata hash |
| `get_numista_asset_balance` | Look up a username's balance for an asset |
| `get_numista_asset_username_key` | Look up the public key registered to a username |
| `get_version` | RPC and software version |
| `get_block_count` (`getblockcount`) | Chain height |
| `on_get_block_hash` (`on_getblockhash`) | Block hash lookup |
| `get_last_block_header` (`getlastblockheader`) | Tip header |
| `get_block_header_by_hash` (`getblockheaderbyhash`) | Header by hash |
| `get_block_header_by_height` (`getblockheaderbyheight`) | Header by height |
| `get_block_headers_range` (`getblockheadersrange`) | Header range |
| `get_block` (`getblock`) | Block by hash/height |
| `get_alternate_chains` | Alternate-chain information |
| `hard_fork_info` | Hard-fork status |
| `get_transactions` | Retrieve transactions |
| `get_txids_loose` | Transaction IDs matching the request |
| `get_miner_data` | Data used to construct a mining block |
| `get_block_template` (`getblocktemplate`) | Mining block template |
| `calc_pow` | Calculate proof of work |
| `add_aux_pow` | Add auxiliary proof-of-work data |
| `submit_block` (`submitblock`) | Submit a block |
| `generateblocks` | Generate blocks for local testing |
| `get_output_histogram` | Output histogram |
| `get_output_distribution` | Output distribution |
| `get_coinbase_tx_sum` | Coinbase totals over a height range |
| `get_fee_estimate` | Fee estimate |
| `get_txpool_backlog` | Fee/weight backlog |
| `get_connections` | Connected peers |
| `get_bans`, `set_bans`, `banned` | Peer ban inspection and controls |
| `relay_tx` | Relay a transaction |
| `sync_info` | Synchronization details |
| `flush_txpool` | Remove transactions from the pool |
| `prune_blockchain` | Prune the local blockchain |
| `flush_cache` | Flush daemon caches |

## Wallet JSON-RPC methods

Wallet RPC uses JSON-RPC at `POST /json_rpc` on the operator-selected wallet
RPC port. It requires a running wallet and talks to the configured daemon.
These methods are implemented by this codebase:

```text
get_wallet_info, get_balance, get_address, get_address_index,
set_subaddress_lookahead, getbalance, getaddress, create_address,
label_address, get_accounts, create_account, label_account,
get_account_tags, tag_accounts, untag_accounts,
set_account_tag_description, get_height, getheight, freeze, thaw, frozen,
transfer, numista_asset_operation, transfer_split, sign_transfer, describe_transfer, submit_transfer,
sweep_dust, sweep_unmixable, sweep_all, sweep_single, relay_tx, store,
get_payments, get_bulk_payments, incoming_transfers, query_key,
make_integrated_address, split_integrated_address, stop_wallet,
rescan_blockchain, set_tx_notes, get_tx_notes, set_attribute, get_attribute,
get_tx_key, check_tx_key, get_tx_proof, check_tx_proof, get_spend_proof,
check_spend_proof, get_reserve_proof, check_reserve_proof, get_transfers,
get_transfer_by_txid, sign, verify, export_outputs, import_outputs,
export_key_images, import_key_images, make_uri, parse_uri,
get_address_book, add_address_book, edit_address_book,
delete_address_book, refresh, auto_refresh, scan_tx, rescan_spent,
start_mining, stop_mining, get_languages, create_wallet, open_wallet,
wallet_exists, close_wallet, change_wallet_password, generate_from_keys,
restore_deterministic_wallet, is_multisig, prepare_multisig, make_multisig,
export_multisig_info, import_multisig_info, exchange_multisig_keys,
get_multisig_key_exchange_booster, sign_multisig, submit_multisig,
validate_address, set_daemon, set_log_level, set_log_categories,
estimate_tx_size_and_weight, get_default_fee_priority, get_version,
setup_background_sync, start_background_sync, stop_background_sync
```

These are general wallet-management and coin-transfer APIs; request fields
and response formats are defined in
[`src/wallet/wallet_rpc_server_commands_defs.h`](src/wallet/wallet_rpc_server_commands_defs.h).
Protect wallet RPC credentials and wallet secrets. Use restricted mode when
the application does not need wallet administration or spending operations.

### Numista asset calls

The daemon's three read-only JSON-RPC methods are available at
`POST /json_rpc`. A `found`/`registered` value of `false` means the requested
record does not exist; an invalid asset ID returns a parameter error.

```sh
# Asset record
curl --fail-with-body --silent --show-error \
  -H 'Content-Type: application/json' \
  --data '{"jsonrpc":"2.0","id":"0","method":"get_numista_asset","params":{"asset_id":"<64-hex-character-id>"}}' \
  http://127.0.0.1:48081/json_rpc

# Username balance (integer asset quantity, not coin atomic units)
curl --fail-with-body --silent --show-error \
  -H 'Content-Type: application/json' \
  --data '{"jsonrpc":"2.0","id":"0","method":"get_numista_asset_balance","params":{"asset_id":"<64-hex-character-id>","username":"player_01"}}' \
  http://127.0.0.1:48081/json_rpc

# Username signing key
curl --fail-with-body --silent --show-error \
  -H 'Content-Type: application/json' \
  --data '{"jsonrpc":"2.0","id":"0","method":"get_numista_asset_username_key","params":{"username":"player_01"}}' \
  http://127.0.0.1:48081/json_rpc
```

Wallet RPC adds one transaction-building method,
`numista_asset_operation`. It signs and submits (or returns, if
`do_not_relay` is set) exactly one transaction containing the selected
operation. The wallet also spends native coins for the normal transaction
fee. Amount and supply are integer asset quantities.

| `action` | Required operation fields |
| --- | --- |
| `register_username` | `username`, `asset_key` (32-byte public key, hex) |
| `create_asset` | `asset_id`, `asset_type` (`fungible` or `collectible`), `max_supply`, `metadata_hash` |
| `mint` | `asset_id`, `asset_type`, `recipient_username`, `amount` |
| `transfer` | `asset_id`, `asset_type`, `username`, `recipient_username`, `amount` |

`asset_id` is a unique 32-byte identifier chosen by the game/backend for the
creation operation. `metadata_hash` is a 32-byte hash of metadata stored
elsewhere; do not put private or mutable game data in transaction extra.
Collectibles must have a cap of one and mint/transfer quantity of one.
Usernames are case-sensitive and limited to ASCII letters, digits, `_`, `-`,
and `.`.

Example wallet JSON-RPC call (the authenticated wallet endpoint must remain
private to the game backend):

```json
{
  "jsonrpc": "2.0",
  "id": "0",
  "method": "numista_asset_operation",
  "params": {
    "action": "mint",
    "asset_id": "<64-hex-character-id>",
    "asset_type": "fungible",
    "recipient_username": "player_01",
    "amount": 25,
    "do_not_relay": false,
    "get_tx_key": false,
    "get_tx_hex": true
  }
}
```

`register_username`, `create_asset`, and `mint` can only be signed by the
software wallet whose spend public key matches the configured Numista testnet
issuer key. Keep that issuer wallet's private key offline or in a protected
signing service; never put it in the game client, backend configuration, or
repository. A `transfer` is signed only if the loaded software wallet's spend
key matches the username's currently registered key, verified through the
configured daemon. Device and watch-only wallets are not currently supported
for asset signing. A successful response means a transaction was built and
accepted for relay, not that it was mined or irrevocably confirmed. Track
confirmation and reorgs from chain data.

### Game-fund block reward

Numista testnet consensus adds `0.0005` coin (`500,000,000` atomic units) to
each mined block's normal reward and pays it to the consensus-fixed game-fund
wallet. The allocation starts with the first mined block; the genesis block
itself is not a mined reward. The testnet genesis nonce has changed, so nodes
must initialize a fresh testnet database before joining this chain. Do not
reuse existing testnet data.

The testnet fund wallet's public spend/view keys are in
[`src/cryptonote_config.h`](src/cryptonote_config.h). Keep its spend key
private and offline. Its Numista testnet address is
`SbMSgsKs938Z9uQK3S7wgxXxXw3iB3L333U5HetbvXvJhK4sjRfUfxzKmMMkxhWHpTCckdLngjygRNGZ2FJghMxp2EN79dEDV`.
The production profile contains separately generated
fund public keys, but Numista production is not configured and has no active
consensus reward or production address yet.

### Game backend and browser mining

The game backend is the trust boundary between the executable/browser and a
private node/wallet service:

1. Authenticate the game account and map it to a registered on-chain
   username. Use daemon query methods for current asset metadata, balances,
   and username keys; do not treat cached game data as chain truth.
2. Keep wallet RPC, daemon administrative RPC, issuer credentials, and
   signing keys on private backend infrastructure. Expose narrow,
   authenticated game API routes—not raw node or wallet RPC—to clients.
3. Have the backend validate game rules and authorize asset actions, then call
   `numista_asset_operation` for a wallet that is authorized to sign that
   action. Store the transaction hash and monitor chain inclusion and
   reorganizations before granting final in-game benefits.
4. Reconcile balances and ownership after reconnects and reorgs. ZMQ can
   reduce update latency, but it is a lossy notification stream and must be
   reconciled against daemon RPC.

There is no browser-miner protocol or public mining service implemented in
this repository. If the game adds one, use an explicit opt-in, a visible
on/off control, and a user-configurable CPU/time cap. The browser worker
should communicate only with the authenticated game backend over TLS; it
must never receive daemon credentials or call unrestricted daemon RPC.
The backend obtains mining templates from its private daemon, issues bounded
jobs to opted-in workers, validates returned candidates, and submits valid
blocks through the daemon's `submit_block` method. Treat workers and their
results as untrusted, expire jobs on tip changes, and report accepted/rejected
results transparently. This project does not define the game-side job/share
protocol, worker implementation, or mining economics.

## ZMQ notifications

For optional low-latency updates, the daemon's ZMQ publisher supports the
`json-(minimal|full)-chain_main` and
`json-(minimal|full)-txpool_add` topic families. ZMQ is a notification stream,
not a replacement for HTTP RPC: messages can be dropped, so reconcile chain
state with daemon RPC after reconnects and reorganizations. See
[`docs/ZMQ.md`](docs/ZMQ.md) for topic and payload behavior.

For the authoritative list of daemon routes and method names, see
[`src/rpc/core_rpc_server.h`](src/rpc/core_rpc_server.h); wallet method
registrations are in
[`src/wallet/wallet_rpc_server.h`](src/wallet/wallet_rpc_server.h). Network
ports are defined in [`src/cryptonote_config.h`](src/cryptonote_config.h).
