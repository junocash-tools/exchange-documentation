# Operational Flows

These flows show how the current tools answer exchange integration workflows.

## Local stack

`juno-exchange-kit` can run the required local dependencies:

```sh
make up
make up regtest
make up testnet
make up mainnet
```

Default services:

- `junocashd` RPC at `http://127.0.0.1:${JUNO_RPC_PORT_HOST:-28232}`
- `juno-scan` API at `http://127.0.0.1:${JUNO_SCAN_PORT_HOST:-18080}`
- `juno-broadcast` API at `http://127.0.0.1:${JUNO_BROADCAST_PORT_HOST:-18081}`

Use a separate `JUNO_EXCHANGE_KIT_DATA_DIR` per network. Reusing a regtest data dir on mainnet can produce regtest addresses while connected to mainnet.

## Deposit address provisioning

1. Initialize hot/cold wallets and scanner registration:

```sh
bin/juno-exchange-kit init --json
```

2. Create an internal account:

```sh
bin/juno-exchange-kit account create --json
```

3. Allocate the account deposit address:

```sh
bin/juno-exchange-kit account deposit-address <account_id> --json
```

4. Store the returned `account_id` and `address` in the exchange system. The on-chain data is shielded, so the exchange must preserve its own address-to-account mapping.

## Deposit ingestion

1. Run `juno-scan` with the hot wallet UFVK registered.
2. Consume scanner events:

```sh
curl -sS "http://127.0.0.1:18080/v1/wallets/hot/events?cursor=0&limit=100"
```

3. Let `juno-exchange-kit` sync scanner events into local exchange accounting:

```sh
bin/juno-exchange-kit sync
bin/juno-exchange-kit daemon start
```

4. Treat `DepositEvent` as detected/pending.
5. Credit the user only on `DepositConfirmed`.
6. Handle `DepositOrphaned` and `DepositUnconfirmed` as debit/reversal signals when they affect a previously credited deposit.

`juno-exchange-kit` already follows this model: pending deposits are recorded immediately, but account credits increase only on confirmed deposits whose recovered `recipient_address` matches a known external deposit address.

## Balance reconciliation

User balance:

```sh
bin/juno-exchange-kit account balance <account_id> --json
```

Exchange balance:

```sh
bin/juno-exchange-kit balances
```

Wallet liquidity:

```sh
bin/juno-exchange-kit wallet balance hot
bin/juno-exchange-kit wallet balance cold
```

Interpretation:

- Confirmed user balances are liabilities.
- Pending deposits are visible but not credited.
- Hot/cold wallet balances are asset/liquidity views.
- Equity is assets minus liabilities.

## Withdrawal

1. Validate the destination address with the JavaScript/TypeScript or Java validator.
2. Build a transaction plan:

```sh
juno-txbuild send \
  --rpc-url "$JUNO_RPC_URL" \
  --rpc-user "$JUNO_RPC_USER" \
  --rpc-pass "$JUNO_RPC_PASS" \
  --scan-url "$JUNO_SCAN_URL" \
  --wallet-id hot \
  --coin-type <coin_type> \
  --account 0 \
  --to <j...> \
  --amount-zat <amount> \
  --change-address <internal_change_address> \
  --out txplan.json \
  --json
```

3. Sign offline:

```sh
juno-txsign sign --txplan ./txplan.json --seed-file ./hot.seed --out ./rawtx.hex --json
```

4. Store `txid` before broadcast.
5. Broadcast:

```sh
juno-broadcast submit --rpc-url "$JUNO_RPC_URL" --rpc-user "$JUNO_RPC_USER" --rpc-pass "$JUNO_RPC_PASS" --raw-tx-hex "$(cat rawtx.hex)" --json
```

6. Track status:

```sh
juno-broadcast status --rpc-url "$JUNO_RPC_URL" --rpc-user "$JUNO_RPC_USER" --rpc-pass "$JUNO_RPC_PASS" --txid <txid> --json
```

`juno-exchange-kit` wraps this flow for exchange users:

```sh
bin/juno-exchange-kit withdraw --account <id> --to <j...> --amount-zat <n> --wait-confirmations 1 --json
bin/juno-exchange-kit withdrawals list --account <id> --json
```

## Hot and cold operations

Hot to cold sweep:

```sh
bin/juno-exchange-kit sweep to-cold
```

Hot note consolidation:

```sh
bin/juno-exchange-kit sweep consolidate
```

Cold to hot top-up:

```sh
bin/juno-exchange-kit cold-to-hot plan
bin/juno-exchange-kit cold-to-hot sign
bin/juno-exchange-kit cold-to-hot broadcast
```

Cold-to-hot is modeled as an offline signing flow suitable for HSM or air-gapped operation.

## Backfill

If a UFVK is registered after the scanner has already advanced, call backfill:

```sh
curl -sS -X POST http://127.0.0.1:18080/v1/wallets/hot/backfill \
  -H 'content-type: application/json' \
  -d '{"from_height":0,"batch_size":10000}'
```

Repeat with the returned `next_height` until the desired scanned range is complete.

## Reorg handling

Do not key deposit finality only on first block inclusion. Consume scanner lifecycle events:

- `DepositConfirmed`
- `DepositOrphaned`
- `DepositUnconfirmed`
- `SpendConfirmed`
- `SpendOrphaned`
- `SpendUnconfirmed`
- `OutgoingOutputConfirmed`
- `OutgoingOutputOrphaned`
- `OutgoingOutputUnconfirmed`
- `OutgoingOutputExpired`

Use cursor-based consumption. Filtering by `block_height` is documented for debug/audit use only because reorgs can move events across heights.

