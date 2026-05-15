# Exchange Capability Map

This map answers the common exchange due-diligence questions using only currently represented Juno Cash tools.

Important: `juno-exchange-kit` is solely a working demo/example harness. It is useful for showing how the lower-level tools can be composed, but it is not itself the production integration contract or a required exchange architecture.

## Wallet and address management

### Address and account generation

In the demo, `juno-exchange-kit` creates internal exchange accounts:

```sh
bin/juno-exchange-kit account create --json
```

It allocates one stable external deposit address per account from the hot wallet:

```sh
bin/juno-exchange-kit account deposit-address <account_id> --json
```

For standalone/offline address generation, `juno-addrgen` derives one or many unified addresses from a UFVK:

```sh
juno-addrgen derive --ufvk <jview*1...> --index 0 --json
juno-addrgen batch --ufvk <jview*1...> --start 0 --count 10 --json
```

### Deterministic generation

Address generation is deterministic:

- `juno-addrgen`: UFVK + index gives the same address.
- `juno-exchange-kit`: demonstrates one way to assign account deposit addresses by wallet, scope, and address index, then store the mapping so the same demo account receives the same address.
- Because Juno Cash recipient/value data is encrypted on-chain, exchanges must persist their deposit address or index to account mapping. They cannot rely on Bitcoin-style plaintext address matching.

### Address validation

`juno-address-validators` provides:

- Java validator.
- JavaScript/TypeScript validator.
- Format-only validation and full validation modes.
- Network detection for `j`, `jtest`, and `jregtest`.

Full validation checks unified-address receiver structure and requires an Orchard receiver. Transparent and Sapling receivers are rejected by the documented validator behavior.

### Address format

The exchange-facing address format is Juno Cash Orchard unified address:

| Network | Address HRP | UFVK HRP |
| --- | --- | --- |
| Mainnet | `j` | `jview` |
| Testnet | `jtest` | `jviewtest` |
| Regtest | `jregtest` | `jviewregtest` |

### Memo, tag, and payment-id support

Supported:

- Optional Orchard memo via `memo_hex`.
- Maximum memo size: 512 bytes.
- Encoding: hex-encoded bytes in APIs and schemas.
- Incoming and outgoing scanner records expose `memo_hex`; absent memo is returned as `null` on notes.

Not part of the current model:

- Destination tags.
- Payment IDs.
- Mandatory memo for deposits.

### HD wallet support

The key library can derive UFVKs from seed material, coin type, and account using ZIP32-style Orchard derivation. `juno-exchange-kit init` demonstrates this by creating local hot/cold seed files, deriving hot/cold UFVKs, and storing wallet metadata.

The exchange-facing deposit derivation surface is still UFVK + scope + index. Treat seed files as signing material and keep them out of tracked content.

### Account model or UTXO model

Juno Cash chain state is shielded note/nullifier based:

- `juno-scan` tracks notes and nullifiers per `wallet_id`.
- Spendable state is based on unspent Orchard notes.
- Pending spend state is exposed through fields such as `pending_spent_txid`, `pending_spent_at`, and `pending_spent_expiry_height`.

`juno-exchange-kit` adds a demo internal account ledger for exchange users:

- `accounts`
- `deposit_addresses`
- `deposits`
- `account_balances`
- `withdrawals`

So the demonstrated model is: chain notes for assets/liquidity, internal account balances for exchange liabilities. A production exchange can use a different internal ledger as long as it preserves the same chain-facing requirements.

## Balance management

### Native balance

Native JUNO is the only asset covered by the current exchange tools.

Amounts are expressed in monetas (`zat`). User-facing formatting in the demo uses:

```text
1 JUNO = 100,000,000 zat
```

Account balance:

```sh
bin/juno-exchange-kit account balance <account_id>
bin/juno-exchange-kit account balance <account_id> --json
```

Demo exchange aggregate view:

```sh
bin/juno-exchange-kit balances
```

Hot/cold wallet liquidity:

```sh
bin/juno-exchange-kit wallet balance hot
bin/juno-exchange-kit wallet balance cold
```

### Available, pending, and locked balances

The demo harness separates:

- `balance_zat`: confirmed credited account liability.
- `pending_deposits_zat`: detected but not yet credited deposit amount.
- `liabilities_zat`: total confirmed user balances.
- `liabilities_pending_zat`: total pending deposits.
- `equity_zat`: assets minus confirmed liabilities.
- `equity_total_zat`: assets minus confirmed and pending liabilities.

For withdrawal debits, `juno-exchange-kit` demonstrates checking account balance before building/broadcasting and recording withdrawals after broadcast.

The current docs do not define staking, frozen balances, or exchange-side locked balance buckets beyond pending deposits and withdrawal debiting.

### Asset balances

Token balances, NFT balances, issuer IDs, contract IDs, and frozen token states are not represented in the current toolchain.

## Transaction lifecycle

### Create raw transaction plan

`juno-txbuild` builds a versioned `TxPlan` from online chain state:

```sh
juno-txbuild send \
  --rpc-url <url> --rpc-user <user> --rpc-pass <pass> \
  --wallet-id hot \
  --coin-type <n> \
  --account 0 \
  --to <j*1...> \
  --amount-zat <zat> \
  --change-address <j*1...> \
  --out txplan.json \
  --json
```

Supported plan kinds include withdrawal, sweep, consolidate, and rebalance flows.

### Estimate fee

`juno-txbuild` sets the default fee with ZIP-317-style conventional fee logic:

```text
fee_zat = 5000 * max(2, max(spends, outputs))
```

Supported fee controls:

- `--fee-multiplier <n>`
- `--fee-add-zat <zat>`
- `--min-change-zat <zat>`
- `--min-note-zat <zat>`

`juno-txsign` validates that the plan fee is at least the conventional minimum.

### Sign offline

`juno-txsign` signs `TxPlan` packages and emits raw transaction hex:

```sh
juno-txsign sign --txplan ./txplan.json --seed-file ./seed.b64 --json
```

It also supports external Orchard spend-auth signing:

```sh
juno-txsign ext-prepare --txplan ./txplan.json --ufvk <jview...> --out-prepared ./prepared.json --out-requests ./requests.json
juno-txsign ext-finalize --prepared-tx ./prepared.json --sigs ./sigs.json --out ./rawtx.hex --json
```

### Broadcast

`juno-broadcast` submits signed raw transactions:

```sh
juno-broadcast submit --rpc-url <url> --rpc-user <user> --rpc-pass <pass> --raw-tx-hex <hex> --json
```

HTTP equivalent:

```http
POST /v1/tx/submit
Content-Type: application/json

{"raw_tx_hex":"...","wait_confirmations":1}
```

### Get transaction hash before broadcast

`juno-txsign --json` returns:

- `txid`
- `raw_tx_hex`
- `fee_zat`

So an exchange can store the txid before calling `juno-broadcast`.

### Deterministic serialization

The signed transaction output is a raw transaction hex blob from a concrete `TxPlan`. The plan schema is versioned as `txplan.version = "v0"`, and CLI JSON wrappers are versioned separately.

### Nonce, sequence, replacement, and rebroadcast

There is no account nonce or sequence model in the exchange tools. Spend conflict handling is note/nullifier based.

`TxPlan` includes `expiry_height`, computed from the current chain tip and `--expiry-offset`. Current docs state:

- `junocashd` rejects conflicting mempool transactions.
- Replacement/RBF is not supported.
- Orchard spends cannot be fee-bumped by CPFP.
- Set the intended fee before broadcasting.

Rebroadcast can be attempted by submitting the same raw transaction again, but the tool docs do not define a special rebroadcast policy beyond node RPC behavior and status checks.

## Deposit tracking

### Block access

`juno-sdk-go` wraps `junocashd` RPC helpers for:

- `getblockchaininfo`
- `getblockcount`
- `getbestblockhash`
- `getblockhash`
- `getblockheader`
- `getblock`

`juno-scan` uses `junocashd` RPC and maintains a scanned tip.

### Scan address history and blocks

Juno Cash deposits are shielded. `juno-scan` detects wallet activity by registering UFVKs and trial-decrypting scanned blocks, not by plaintext address matching.

For full history, register wallets before scanning or run backfill:

```sh
curl -sS -X POST http://127.0.0.1:8080/v1/wallets/exchange-hot-001/backfill \
  -H 'content-type: application/json' \
  -d '{"from_height":0,"batch_size":10000}'
```

The backfill response includes `next_height`; repeat from `next_height` until caught up.

### Subscription support

No WebSocket API is documented.

Supported alternatives:

- `juno-scan` HTTP polling for `/v1/wallets/{wallet_id}/events`.
- Optional ZMQ `hashblock` notifications from `junocashd` to trigger scanner work.
- Optional broker outbox publishing to Kafka, NATS, or RabbitMQ.

### Confirmation count and finality

`juno-scan` emits:

- `DepositEvent`: detected in a scanned block.
- `DepositConfirmed`: reached configured confirmation threshold.
- `DepositOrphaned`: original block orphaned by reorg.
- `DepositUnconfirmed`: previously confirmed deposit fell below threshold after rollback.

Default confirmation threshold is `100`. The `juno-exchange-kit` demo stack defaults regtest to `1` and non-regtest networks to `100`.

Finality is operationally modeled by configured confirmation threshold plus reorg event handling. Consumers should process `*Orphaned` and `*Unconfirmed` events rather than treating block inclusion as irreversible.

## Transaction lookup

### By transaction hash

`juno-broadcast`:

```http
GET /v1/tx/{txid}
```

Response includes:

- `txid`
- `in_mempool`
- `confirmations`
- `blockhash` when confirmed

`juno-scan`:

```http
GET /v1/wallets/{wallet_id}/events?txid=<txid>
GET /v1/wallets/{wallet_id}/notes
```

`juno-sdk-go`:

- `GetRawTransactionHex(ctx, txid)`

### Pending, confirmed, failed states

Current transaction state surfaces are split:

- Broadcast status: mempool vs confirmed via `in_mempool`, `confirmations`, and `blockhash`.
- Scanner event status: `mempool`, `confirmed`, `orphaned`, or `expired`.
- Exchange withdrawal history: `broadcasted` or `failed`.

The tools do not expose a single universal `pending|confirmed|failed` transaction enum across all APIs.

### Timestamps, block inclusion, and events/logs

Block headers and verbose blocks include `height` and `time` through `juno-sdk-go` RPC types.

Wallet event payloads include heights, confirmation fields, and event `created_at` timestamps where exposed by the scanner API. Internal transfers/change are represented through outgoing output events and `recipient_scope`/`ovk_scope` when recoverable by the wallet UFVK.

## API and SDK documentation

The current public machine-readable surfaces are:

- `juno-scan/api/openapi.yaml`
- `juno-broadcast/api/openapi.yaml`
- `juno-txbuild/api/txplan.v0.schema.json`
- `juno-txbuild/api/txoutputs.schema.json`
- `juno-txsign/api/*.schema.json`

Supported languages:

- Go: `juno-sdk-go` for clients and shared types.
- JavaScript/TypeScript: address validation only.
- Java: address validation only.

Authentication:

- `juno-scan`: optional bearer token, intended for internal service-to-service use; docs recommend private networking and mTLS at the proxy/load balancer rather than bearer as primary auth.
- `juno-broadcast`: no built-in auth model is documented in its API; deploy behind private network or gateway controls.
- `junocashd` RPC: RPC URL/user/pass.

Rate limits:

- No built-in API rate limits are documented for these tool services. Enforce limits at the deployment layer if exposed beyond a trusted private network.

Testnet:

- `juno-exchange-kit` demo Docker workflows support `regtest`, `testnet`, and `mainnet`.
- Address validators and address derivation support mainnet, testnet, and regtest HRPs.
