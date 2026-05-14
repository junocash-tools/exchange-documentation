# API and SDK Surface

## REST/RPC/HTTP APIs

### `juno-scan`

OpenAPI:

- [`../juno-scan/api/openapi.yaml`](../../juno-scan/api/openapi.yaml)

Important endpoints:

```http
GET /v1/health
GET /v1/wallets
POST /v1/wallets
GET /v1/wallets/{wallet_id}/events?cursor=<id>&limit=<n>&kind=<kind>&txid=<txid>
POST /v1/wallets/{wallet_id}/backfill
GET /v1/wallets/{wallet_id}/notes?spent=true&direction=incoming|outgoing|all&min_value_zat=<zat>&limit=<n>&cursor=<c>
POST /v1/orchard/witness
```

Authentication:

- Optional bearer token via `Authorization: Bearer <token>`.
- The scanner docs treat bearer auth as an internal convenience and recommend private networking plus mTLS at the proxy/load balancer for primary service-to-service protection.

Error shape:

- Current scanner errors are `text/plain` from `http.Error`.
- Treat non-2xx as failure and do not depend on a JSON error body.

Pagination and filters:

- Events default limit is `100`, max `1000`.
- Notes max limit is `1000`.
- Event filters include `block_height`, `kind`, and `txid`.
- `block_height` is for debug/audit only due to reorg risk.

### `juno-broadcast`

OpenAPI:

- [`../juno-broadcast/api/openapi.yaml`](../../juno-broadcast/api/openapi.yaml)

Important endpoints:

```http
GET /healthz
POST /v1/tx/submit
GET /v1/tx/{txid}
```

Submit request:

```json
{
  "raw_tx_hex": "...",
  "wait_confirmations": 1
}
```

Submit response:

```json
{
  "txid": "...",
  "status": {
    "txid": "...",
    "in_mempool": false,
    "confirmations": 1,
    "blockhash": "..."
  }
}
```

Error shape:

```json
{
  "error": {
    "code": "invalid_request",
    "message": "..."
  }
}
```

Authentication:

- No built-in auth model is documented for the broadcast service.
- Deploy behind private network, gateway auth, or equivalent infrastructure controls.

### `junocashd` RPC

The tools use `junocashd` JSON-RPC with URL/user/password configuration.

`juno-sdk-go` wraps:

- `GetBlockchainInfo`
- `GetBlockCount`
- `GetBestBlockHash`
- `GetBlockHash`
- `GetBlockHeader`
- `GetBlockVerbose`
- `GetRawTransactionHex`
- `SendRawTransaction`

## CLI JSON APIs

### `juno-addrgen`

Stable automation surface: `--json`.

```sh
juno-addrgen derive --ufvk <jview*1...> --index 0 --json
```

Success:

```json
{ "version": "v1", "status": "ok", "address": "j1..." }
```

### `juno-txbuild`

Stable automation surfaces:

- `--out` for raw `TxPlan`.
- `--json` envelope for command automation.
- `api/txplan.v0.schema.json`
- `api/txoutputs.schema.json`

Success envelope:

```json
{ "version": "v1", "status": "ok", "data": { "version": "v0" } }
```

Common error codes:

- `invalid_request`
- `insufficient_balance`
- `no_liquidity_in_hot`
- `not_found`

### `juno-txsign`

Stable automation surfaces:

- `--json`
- `--out`
- `api/txplan.v0.schema.json`
- `api/prepared_tx.v0.schema.json`
- `api/signing_requests.v0.schema.json`
- `api/spend_auth_sigs.v0.schema.json`

Success:

```json
{
  "version": "v1",
  "status": "ok",
  "data": {
    "txid": "...",
    "raw_tx_hex": "...",
    "fee_zat": "..."
  }
}
```

With `--action-indices`, `data` also includes Orchard output/change action indices for reconciling transaction outputs.

## Event payloads

Scanner event payload docs:

- [`../juno-scan/docs/events.md`](../../juno-scan/docs/events.md)

Common status state values:

- `mempool`
- `confirmed`
- `orphaned`
- `expired`

Deposit event kinds:

- `DepositEvent`
- `DepositConfirmed`
- `DepositOrphaned`
- `DepositUnconfirmed`

Spend event kinds:

- `SpendEvent`
- `SpendConfirmed`
- `SpendOrphaned`
- `SpendUnconfirmed`

Outgoing output event kinds:

- `OutgoingOutputEvent`
- `OutgoingOutputConfirmed`
- `OutgoingOutputOrphaned`
- `OutgoingOutputUnconfirmed`
- `OutgoingOutputExpired`

## SDKs and supported languages

### Go

`juno-sdk-go` is the current general SDK surface. It includes:

- Typed `junocashd` RPC helpers.
- `juno-scan` client and types.
- `juno-broadcast` client and types.
- Shared transaction plan, event, note, and error types.

Status is documented as work in progress.

### JavaScript/TypeScript

Supported today:

- Address validation through `juno-address-validators/js-validators`.
- ES module usage.
- TypeScript definitions.
- Optional `blakejs` dependency for full validation mode.

Not currently provided:

- Full JavaScript transaction build/sign/broadcast SDK.
- JavaScript scanner or broadcast clients.

### Java

Supported today:

- Address validation through `juno-address-validators/java-validation`.

## Rate limits

No built-in rate limits are documented for `juno-scan` or `juno-broadcast`. If these services are exposed outside a trusted private environment, rate limiting should be enforced by the deployment layer.

## Network availability

The local exchange stack supports:

- Regtest.
- Testnet.
- Mainnet.

Address tooling supports:

- Mainnet HRP `j`.
- Testnet HRP `jtest`.
- Regtest HRP `jregtest`.

Use separate data directories per network.

