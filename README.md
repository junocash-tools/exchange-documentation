# Juno Cash Exchange Documentation

This repo collects the exchange-facing answers exposed by the local Juno Cash toolchain under `junocash-tools`.

It is a documentation layer over the implementation repos, not a separate SDK or service. The source tools are:

- [`juno-exchange-kit`](https://github.com/Abdullah1738/juno-exchange-kit): working demo/example harness only. It shows one possible exchange integration shape for accounts, deposit addresses, deposits, balances, withdrawals, hot/cold sweeps, and local regtest/testnet/mainnet dependency stacks; it is not the production integration contract.
- [`juno-addrgen`](https://github.com/Abdullah1738/juno-addrgen): offline UFVK + index address derivation.
- [`juno-address-validators`](https://github.com/junocash-tools/juno-address-validators): Java and JavaScript/TypeScript address validators.
- [`juno-txbuild`](https://github.com/junocash-tools/juno-txbuild): online `TxPlan` builder for offline signing.
- [`juno-txsign`](https://github.com/junocash-tools/juno-txsign): offline signer that returns raw transaction hex and txid.
- [`juno-broadcast`](https://github.com/Abdullah1738/juno-broadcast): signed transaction submit/status HTTP API and CLI.
- [`juno-scan`](https://github.com/junocash-tools/juno-scan): watch-only scanner/indexer for Orchard deposits, spends, notes, confirmations, and reorg events.
- [`juno-sdk-go`](https://github.com/Abdullah1738/juno-sdk-go): Go client/types package for `junocashd`, `juno-scan`, and `juno-broadcast`.

## OpenAPI location

The machine-readable API files are kept with the implementation repos, not copied into this documentation repo:

- [`juno-scan/api/openapi.yaml`](https://github.com/junocash-tools/juno-scan/blob/main/api/openapi.yaml)
- [`juno-broadcast/api/openapi.yaml`](https://github.com/Abdullah1738/juno-broadcast/blob/main/api/openapi.yaml)

If an older checkout or integration note refers to `juno-scan/api/openapi.yaml`, that path is still the scanner repo path. In this docs repo, use the links above or the API summary below.

## Short answers

| Area | Current answer |
| --- | --- |
| Wallet and address management | Supported through UFVK-based deterministic Orchard unified address derivation. `juno-exchange-kit` demonstrates one sample internal-account mapping; production exchanges should treat it as an example, not as a required account model. |
| Address validation | Supported in Java and JavaScript/TypeScript. Validators support mainnet `j`, testnet `jtest`, and regtest `jregtest` unified address HRPs. |
| Memo/tag/payment-id | Orchard memo is supported as optional `memo_hex`, up to 512 bytes. There is no destination tag or payment-id model in these tools. |
| HD wallet support | The key library derives UFVKs from seed, coin type, and account using ZIP32-style Orchard key derivation. The exchange-facing address derivation API is UFVK + scope + index. |
| Account or UTXO model | Chain state is shielded note/nullifier based, closer to a UTXO model than an account model. `juno-exchange-kit` adds a demo internal account ledger on top to illustrate exchange accounting. |
| Balances | Native JUNO only. Amounts are represented in monetas (`zat`), with 100,000,000 zat per JUNO. User liabilities are confirmed balances; pending deposits are shown separately. Hot/cold wallet balances are separate liquidity views. |
| Assets/tokens/NFTs | Not covered by the current toolchain. The docs and schemas are native-asset Juno Cash only. |
| Transaction lifecycle | Build `TxPlan` online with `juno-txbuild`, sign offline with `juno-txsign`, then submit/status with `juno-broadcast`. The signer returns txid before broadcast. |
| Nonce/sequence/replacement | No account nonce flow. Spending is by Orchard notes/nullifiers plus `expiry_height`. Current docs state no replacement/RBF and no CPFP fee bump for Orchard spends. |
| Deposit tracking | `juno-scan` scans blocks by UFVK trial decryption, supports backfill, exposes HTTP events/notes, emits confirmation/reorg lifecycle events, and can publish to Kafka/NATS/RabbitMQ. |
| WebSocket/subscription | No WebSocket API is documented. Real-time-ish flows use polling, ZMQ block notifications into the scanner, and optional broker publishing. |
| Transaction lookup | `juno-broadcast` exposes `/v1/tx/{txid}` for mempool/confirmation status. `juno-scan` exposes wallet events and notes by txid. `juno-sdk-go` wraps `getrawtransaction`, block, and broadcast RPCs. |
| API/SDK | REST-like HTTP APIs exist for `juno-scan` and `juno-broadcast`, both versioned under `/v1` with OpenAPI files. Go SDK exists. JavaScript/TypeScript support exists for address validation only, not full transaction orchestration. |

## Integration examples

These examples assume the exchange has already provisioned a wallet UFVK for scanning, keeps its own address-to-user mapping, and runs the services on private infrastructure:

- `JUNO_SCAN_URL=http://127.0.0.1:18080`
- `JUNO_BROADCAST_URL=http://127.0.0.1:18081`
- `JUNO_RPC_URL=http://127.0.0.1:28232`
- `JUNO_RPC_USER=rpcuser`
- `JUNO_RPC_PASS=rpcpass`

Juno Cash deposits are shielded. You cannot query an arbitrary public address balance from chain data alone. The exchange must register its UFVK with `juno-scan`, derive deposit addresses from that UFVK, store the address/index mapping in its own system, and consume scanner notes/events.

### Node.js

The current JavaScript surface is HTTP APIs plus CLI JSON surfaces. There is address validation for JavaScript/TypeScript, but no full JavaScript transaction SDK yet.

```js
import { spawn } from "node:child_process";

const scanURL = process.env.JUNO_SCAN_URL ?? "http://127.0.0.1:18080";
const broadcastURL = process.env.JUNO_BROADCAST_URL ?? "http://127.0.0.1:18081";
const rpcURL = process.env.JUNO_RPC_URL ?? "http://127.0.0.1:28232";
const rpcUser = process.env.JUNO_RPC_USER ?? "rpcuser";
const rpcPass = process.env.JUNO_RPC_PASS ?? "rpcpass";

async function jsonFetch(baseURL, path, options = {}) {
  const res = await fetch(`${baseURL}${path}`, {
    ...options,
    headers: {
      accept: "application/json",
      ...(options.body ? { "content-type": "application/json" } : {}),
      ...(options.headers ?? {})
    }
  });
  if (!res.ok) {
    throw new Error(`${res.status} ${await res.text()}`);
  }
  return res.json();
}

async function cliJSON(command, args, stdin) {
  const stdout = await new Promise((resolve, reject) => {
    const child = spawn(command, args, { stdio: ["pipe", "pipe", "pipe"] });
    const out = [];
    const err = [];

    child.stdout.on("data", (chunk) => out.push(chunk));
    child.stderr.on("data", (chunk) => err.push(chunk));
    child.on("error", reject);
    child.on("close", (code) => {
      if (code !== 0) {
        reject(new Error(Buffer.concat(err).toString("utf8") || `${command} exited ${code}`));
        return;
      }
      resolve(Buffer.concat(out).toString("utf8"));
    });

    child.stdin.end(stdin ?? "");
  });

  const parsed = JSON.parse(stdout);
  if (parsed.status && parsed.status !== "ok") {
    throw new Error(parsed.error?.message ?? parsed.error ?? `${command} failed`);
  }
  return parsed;
}

// 1. Create a deposit address from a UFVK and deterministic address index.
export async function createAddress(ufvk, index) {
  const res = await cliJSON("juno-addrgen", [
    "derive",
    "--ufvk",
    ufvk,
    "--index",
    String(index),
    "--json"
  ]);
  return res.address;
}

// 2. Check the balance for an exchange-owned deposit address.
export async function balanceForAddress(walletID, address) {
  let cursor = "";
  let total = 0n;

  do {
    const page = await jsonFetch(
      scanURL,
      `/v1/wallets/${encodeURIComponent(walletID)}/notes?spent=false&direction=incoming&limit=1000${cursor ? `&cursor=${encodeURIComponent(cursor)}` : ""}`
    );

    for (const note of page.notes) {
      if (note.recipient_address === address && !note.spent_txid) {
        total += BigInt(note.value_zat);
      }
    }
    cursor = page.next_cursor ?? "";
  } while (cursor);

  return total;
}

// 3. Find a transaction by txid.
export async function findTransaction(walletID, txid) {
  const [broadcastStatus, scanEvents] = await Promise.all([
    jsonFetch(broadcastURL, `/v1/tx/${encodeURIComponent(txid)}`).catch((err) => ({
      error: err.message
    })),
    jsonFetch(
      scanURL,
      `/v1/wallets/${encodeURIComponent(walletID)}/events?txid=${encodeURIComponent(txid)}&limit=1000`
    )
  ]);
  return { broadcastStatus, scanEvents: scanEvents.events };
}

// 4. Build online, sign offline, then broadcast.
export async function signOfflineAndBroadcast({
  walletID,
  coinType,
  account,
  toAddress,
  changeAddress,
  amountZat,
  seedFile
}) {
  const plan = await cliJSON("juno-txbuild", [
    "send",
    "--rpc-url",
    rpcURL,
    "--rpc-user",
    rpcUser,
    "--rpc-pass",
    rpcPass,
    "--scan-url",
    scanURL,
    "--wallet-id",
    walletID,
    "--coin-type",
    String(coinType),
    "--account",
    String(account),
    "--to",
    toAddress,
    "--amount-zat",
    String(amountZat),
    "--change-address",
    changeAddress,
    "--json"
  ]);

  const signed = await cliJSON(
    "juno-txsign",
    ["sign", "--txplan", "-", "--seed-file", seedFile, "--json"],
    JSON.stringify(plan.data)
  );

  const submitted = await jsonFetch(broadcastURL, "/v1/tx/submit", {
    method: "POST",
    body: JSON.stringify({ raw_tx_hex: signed.data.raw_tx_hex, wait_confirmations: 1 })
  });

  return { txid: signed.data.txid, fee_zat: signed.data.fee_zat, submitted };
}

// 5. Register and track incoming deposits.
export async function registerAndTrackDeposits(walletID, ufvk, cursor = 0) {
  await jsonFetch(scanURL, "/v1/wallets", {
    method: "POST",
    body: JSON.stringify({ wallet_id: walletID, ufvk })
  });

  const page = await jsonFetch(
    scanURL,
    `/v1/wallets/${encodeURIComponent(walletID)}/events?cursor=${cursor}&limit=1000`
  );

  const depositEvents = page.events.filter((event) =>
    ["DepositEvent", "DepositConfirmed", "DepositOrphaned", "DepositUnconfirmed"].includes(
      event.kind
    )
  );

  return { nextCursor: page.next_cursor, depositEvents };
}
```

### Go

Use `juno-sdk-go` for `juno-scan`, `juno-broadcast`, and `junocashd` RPC clients. Address derivation and transaction build/sign are currently exposed as stable CLI JSON surfaces, so the example calls those binaries from Go.

```go
package main

import (
	"bytes"
	"context"
	"encoding/json"
	"fmt"
	"net/http"
	"net/url"
	"os"
	"os/exec"
	"strings"

	"github.com/Abdullah1738/juno-sdk-go/junobroadcast"
	"github.com/Abdullah1738/juno-sdk-go/junoscan"
)

type cliEnvelope struct {
	Status  string          `json:"status"`
	Address string          `json:"address,omitempty"`
	Data    json.RawMessage `json:"data,omitempty"`
	Error   any             `json:"error,omitempty"`
}

func runCLIJSON(ctx context.Context, stdin []byte, command string, args ...string) (cliEnvelope, error) {
	cmd := exec.CommandContext(ctx, command, args...)
	if stdin != nil {
		cmd.Stdin = bytes.NewReader(stdin)
	}
	out, err := cmd.Output()
	if err != nil {
		return cliEnvelope{}, err
	}
	var env cliEnvelope
	if err := json.Unmarshal(out, &env); err != nil {
		return cliEnvelope{}, err
	}
	if env.Status != "" && env.Status != "ok" {
		return cliEnvelope{}, fmt.Errorf("%s failed: %v", command, env.Error)
	}
	return env, nil
}

// 1. Create a deposit address from a UFVK and deterministic address index.
func createAddress(ctx context.Context, ufvk string, index uint32) (string, error) {
	res, err := runCLIJSON(ctx, nil, "juno-addrgen",
		"derive", "--ufvk", ufvk, "--index", fmt.Sprint(index), "--json")
	if err != nil {
		return "", err
	}
	return res.Address, nil
}

// 2. Check the balance for an exchange-owned deposit address.
func balanceForAddress(ctx context.Context, scan *junoscan.Client, walletID, address string) (int64, error) {
	var total int64
	var cursor string
	for {
		page, err := scan.ListWalletNotesPage(ctx, walletID, junoscan.ListWalletNotesOptions{
			OnlyUnspent: true,
			Direction:   "incoming",
			Limit:       1000,
			Cursor:      cursor,
		})
		if err != nil {
			return 0, err
		}
		for _, note := range page.Notes {
			if note.RecipientAddress == address && note.SpentTxID == nil {
				total += note.ValueZat
			}
		}
		if page.NextCursor == "" {
			return total, nil
		}
		cursor = page.NextCursor
	}
}

// 3. Find a transaction by txid.
func findTransaction(ctx context.Context, scanURL string, bc *junobroadcast.Client, walletID, txid string) error {
	status, found, err := bc.Status(ctx, txid)
	if err != nil {
		return err
	}
	if found {
		fmt.Printf("broadcast status: confirmations=%d mempool=%v\n", status.Confirmations, status.InMempool)
	}

	endpoint := strings.TrimRight(scanURL, "/") +
		"/v1/wallets/" + url.PathEscape(walletID) +
		"/events?txid=" + url.QueryEscape(txid) + "&limit=1000"
	req, err := http.NewRequestWithContext(ctx, http.MethodGet, endpoint, nil)
	if err != nil {
		return err
	}
	resp, err := http.DefaultClient.Do(req)
	if err != nil {
		return err
	}
	defer resp.Body.Close()
	if resp.StatusCode < 200 || resp.StatusCode > 299 {
		return fmt.Errorf("scanner events http %d", resp.StatusCode)
	}
	var events junoscan.WalletEventsPage
	if err := json.NewDecoder(resp.Body).Decode(&events); err != nil {
		return err
	}
	fmt.Printf("scanner tx events: %d, next_cursor=%d\n", len(events.Events), events.NextCursor)
	return nil
}

// 4. Build online, sign offline, then broadcast.
func signOfflineAndBroadcast(ctx context.Context, bc *junobroadcast.Client, seedFile string) (junobroadcast.SubmitResponse, error) {
	plan, err := runCLIJSON(ctx, nil, "juno-txbuild",
		"send",
		"--rpc-url", os.Getenv("JUNO_RPC_URL"),
		"--rpc-user", os.Getenv("JUNO_RPC_USER"),
		"--rpc-pass", os.Getenv("JUNO_RPC_PASS"),
		"--scan-url", os.Getenv("JUNO_SCAN_URL"),
		"--wallet-id", "hot",
		"--coin-type", "133",
		"--account", "0",
		"--to", "<recipient-j-address>",
		"--amount-zat", "100000",
		"--change-address", "<internal-change-j-address>",
		"--json")
	if err != nil {
		return junobroadcast.SubmitResponse{}, err
	}

	signed, err := runCLIJSON(ctx, plan.Data, "juno-txsign",
		"sign", "--txplan", "-", "--seed-file", seedFile, "--json")
	if err != nil {
		return junobroadcast.SubmitResponse{}, err
	}

	var signedData struct {
		TxID     string `json:"txid"`
		RawTxHex string `json:"raw_tx_hex"`
		FeeZat   string `json:"fee_zat"`
	}
	if err := json.Unmarshal(signed.Data, &signedData); err != nil {
		return junobroadcast.SubmitResponse{}, err
	}

	wait := int64(1)
	return bc.Submit(ctx, signedData.RawTxHex, &wait)
}

// 5. Register and track incoming deposits.
func registerAndTrackDeposits(ctx context.Context, scan *junoscan.Client, walletID, ufvk string, cursor int64) (int64, error) {
	if err := scan.UpsertWallet(ctx, walletID, ufvk); err != nil {
		return cursor, err
	}
	page, err := scan.ListWalletEvents(ctx, walletID, cursor, 1000)
	if err != nil {
		return cursor, err
	}
	for _, event := range page.Events {
		switch event.Kind {
		case "DepositEvent", "DepositConfirmed", "DepositOrphaned", "DepositUnconfirmed":
			fmt.Printf("deposit lifecycle event kind=%s payload=%s\n", event.Kind, string(event.Payload))
		}
	}
	return page.NextCursor, nil
}
```

For production, persist `next_cursor` after every processed scanner page, make event processing idempotent by `(wallet_id, txid, action_index, kind)`, credit users only on `DepositConfirmed`, and handle `DepositOrphaned` / `DepositUnconfirmed` as reversal signals when they affect a credited deposit.

## Documentation set

- [Exchange capability map](docs/exchange-capability-map.md)
- [Operational flows](docs/operational-flows.md)
- [API and SDK surface](docs/api-sdk-surface.md)

## Source-of-truth rule

When this repo disagrees with a tool repo, the implementation repo wins. Update this documentation from the current local tool README, OpenAPI, schema, and code before using it for integration signoff.
