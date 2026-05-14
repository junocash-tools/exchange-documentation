# Juno Cash Exchange Documentation

This repo collects the exchange-facing answers exposed by the local Juno Cash toolchain under `junocash-tools`.

It is a documentation layer over the implementation repos, not a separate SDK or service. The source tools are:

- [`juno-exchange-kit`](../juno-exchange-kit/README.md): working demo/example harness only. It shows one possible exchange integration shape for accounts, deposit addresses, deposits, balances, withdrawals, hot/cold sweeps, and local regtest/testnet/mainnet dependency stacks; it is not the production integration contract.
- [`juno-addrgen`](../juno-addrgen/README.md): offline UFVK + index address derivation.
- [`juno-address-validators`](../juno-address-validators/README.md): Java and JavaScript/TypeScript address validators.
- [`juno-txbuild`](../juno-txbuild/README.md): online `TxPlan` builder for offline signing.
- [`juno-txsign`](../juno-txsign/README.md): offline signer that returns raw transaction hex and txid.
- [`juno-broadcast`](../juno-broadcast/README.md): signed transaction submit/status HTTP API and CLI.
- [`juno-scan`](../juno-scan/README.md): watch-only scanner/indexer for Orchard deposits, spends, notes, confirmations, and reorg events.
- [`juno-sdk-go`](../juno-sdk-go/README.md): Go client/types package for `junocashd`, `juno-scan`, and `juno-broadcast`.

## Short answers

| Area | Current answer |
| --- | --- |
| Wallet and address management | Supported through UFVK-based deterministic Orchard unified address derivation. `juno-exchange-kit` demonstrates one sample internal-account mapping; production exchanges should treat it as an example, not as a required account model. |
| Address validation | Supported in Java and JavaScript/TypeScript. Validators support mainnet `j`, testnet `jtest`, and regtest `jregtest` unified address HRPs. |
| Memo/tag/payment-id | Orchard memo is supported as optional `memo_hex`, up to 512 bytes. There is no destination tag or payment-id model in these tools. |
| HD wallet support | The key library derives UFVKs from seed, coin type, and account using ZIP32-style Orchard key derivation. The exchange-facing address derivation API is UFVK + scope + index. |
| Account or UTXO model | Chain state is shielded note/nullifier based, closer to a UTXO model than an account model. `juno-exchange-kit` adds a demo internal account ledger on top to illustrate exchange accounting. |
| Balances | Native JUNO only. Amounts are represented in zatoshis (`zat`), with 100,000,000 zat per JUNO. User liabilities are confirmed balances; pending deposits are shown separately. Hot/cold wallet balances are separate liquidity views. |
| Assets/tokens/NFTs | Not covered by the current toolchain. The docs and schemas are native-asset Juno Cash only. |
| Transaction lifecycle | Build `TxPlan` online with `juno-txbuild`, sign offline with `juno-txsign`, then submit/status with `juno-broadcast`. The signer returns txid before broadcast. |
| Nonce/sequence/replacement | No account nonce flow. Spending is by Orchard notes/nullifiers plus `expiry_height`. Current docs state no replacement/RBF and no CPFP fee bump for Orchard spends. |
| Deposit tracking | `juno-scan` scans blocks by UFVK trial decryption, supports backfill, exposes HTTP events/notes, emits confirmation/reorg lifecycle events, and can publish to Kafka/NATS/RabbitMQ. |
| WebSocket/subscription | No WebSocket API is documented. Real-time-ish flows use polling, ZMQ block notifications into the scanner, and optional broker publishing. |
| Transaction lookup | `juno-broadcast` exposes `/v1/tx/{txid}` for mempool/confirmation status. `juno-scan` exposes wallet events and notes by txid. `juno-sdk-go` wraps `getrawtransaction`, block, and broadcast RPCs. |
| API/SDK | REST-like HTTP APIs exist for `juno-scan` and `juno-broadcast`, both versioned under `/v1` with OpenAPI files. Go SDK exists. JavaScript/TypeScript support exists for address validation only, not full transaction orchestration. |

## Documentation set

- [Exchange capability map](docs/exchange-capability-map.md)
- [Operational flows](docs/operational-flows.md)
- [API and SDK surface](docs/api-sdk-surface.md)

## Source-of-truth rule

When this repo disagrees with a tool repo, the implementation repo wins. Update this documentation from the current local tool README, OpenAPI, schema, and code before using it for integration signoff.
