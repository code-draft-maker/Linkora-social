# linkora-sdk

Typed TypeScript client for `LinkoraContract` on Stellar.

## Transaction retries: exponential backoff with jitter

`TransactionQueue` submits transactions through the Soroban RPC with a retry
policy designed to survive network congestion without making it worse. Instead
of fixed-interval retries — which cause every client to hit the network in
lockstep (a "thundering herd") — failed submissions are retried with
**exponential backoff and jitter**, bounded by a **circuit breaker**.

### Behavior

- **Exponential backoff** — the delay between retries grows as
  `baseDelayMs * 2^attempt`, capped at `maxDelayMs`.
- **Jitter** — a random `0 … jitterFactor * delay` is added to each delay so
  concurrent retriers spread out instead of retrying in sync.
- **Retry-After** — `429` (rate-limited) responses that carry a `Retry-After`
  header (delta-seconds or HTTP-date) are honored; that delay takes precedence
  over the computed backoff (still bounded by `maxDelayMs`).
- **Circuit breaker** — after `circuitBreakerThreshold` consecutive retryable
  failures the breaker opens, the queue is paused, and a `CircuitBreakerError`
  is thrown so callers can report the endpoint as unhealthy.
- **Permanent failures are not retried** — a submission the RPC rejects outright
  (`ERROR` status) fails fast rather than consuming retry budget.
- **Structured logging** — every retry decision is reported to an optional
  `logger` with the attempt number, delay, and reason
  (`error` | `rate-limited` | `circuit-open` | `exhausted`).

### Configuration

Retry tunables default to the environment, and can be overridden per queue.

| Environment variable                 | Default | Meaning                                        |
| ------------------------------------ | ------- | ---------------------------------------------- |
| `TX_RETRY_MAX_ATTEMPTS`              | `5`     | Max attempts (including the first) per submit  |
| `TX_RETRY_BASE_DELAY_MS`             | `1000`  | Base delay for the exponential term            |
| `TX_RETRY_MAX_DELAY_MS`              | `30000` | Upper bound on a single delay                  |
| `TX_RETRY_JITTER_FACTOR`             | `0.5`   | Jitter window as a fraction of the delay (0–1) |
| `TX_RETRY_CIRCUIT_BREAKER_THRESHOLD` | `5`     | Consecutive failures before the breaker opens  |

### Usage

```ts
import { TransactionQueue, ConnectionHealthMonitor } from "linkora-sdk";

const health = new ConnectionHealthMonitor(rpcUrl);

const queue = new TransactionQueue({
  signer,
  rpc,
  // Per-queue overrides (any omitted field falls back to the env defaults):
  retry: { maxAttempts: 5, baseDelayMs: 1000, maxDelayMs: 30000, jitterFactor: 0.5 },
  // Surface retry telemetry + circuit-breaker health through the monitor:
  logger: (info) => health.recordRetry(info),
});

queue.enqueue(xdr1).enqueue(xdr2);

try {
  await queue.run();
} catch (err) {
  if (queue.isCircuitOpen) {
    // Endpoint is unhealthy — back off before submitting more work.
  }
  throw err;
}
```

The lower-level primitives (`backoffWithJitter`, `CircuitBreaker`,
`parseRetryAfter`, `withRetry`) are exported directly for reuse.

### Dry-run status events

`TransactionQueue` emits status events via `queue.on("status", ...)`. When run
in **dry-run** mode (`run({ dryRun: true })` or a queue-level `dryRun: true`),
each step is simulated but never submitted. The queue still emits a `confirmed`
event at the end of each step to mark it complete, but that event:

- carries `dryRun: true`, and
- has **no** `hash` (nothing was broadcast).

Consumers listening for on-chain confirmations should check the `dryRun` flag to
avoid mistaking a simulated flow for a real submission:

```ts
queue.on("status", (e) => {
  if (e.status === "confirmed") {
    if (e.dryRun) {
      // Simulated — do not show as "submitted to network".
    } else {
      // Real confirmation — e.hash is present.
    }
  }
});
```

## API Semantics

The SDK exposes two distinct paths for mutative (write) operations:

### 1. `prepare*Tx` (Submittable)

Methods like `prepareCreatePostTx`, `prepareFollowTx`, and `prepareDmKeyTx` are the **intended path for client-side applications**.
They fetch the actual account sequence from Horizon, simulate the transaction to discover footprint/fees, and return a base64-encoded `TransactionEnvelope` XDR that is fully ready to be signed (e.g. by Freighter) and submitted to the network.

```ts
const txXdr = await client.prepareCreatePostTx("GBFOY...", "Hello!");
// txXdr is ready to be passed to wallet for signing
```

### 2. Base Write Methods (Throwaway XDR)

Methods like `createPost`, `follow`, and `tip` **do not fetch sequence numbers** and return XDR built using a throwaway `Keypair`.
**These are not directly submittable.** They exist primarily to easily extract the Soroban `Operation` for batching (e.g., passing to `buildMultiOpTx`) or for server-side queueing where sequence management is handled by a background worker (like `TransactionQueue`).

If you attempt to sign and submit this XDR directly, the network will reject it with a `tx_bad_seq` error.

---

## TipClient

Tipping operations are exposed directly on `LinkoraClient`. The protocol charges a fee in basis points (bps) on every tip, deducted from the tip amount before the remainder reaches the post author.

### Methods

| Method                                                     | Type                  | Description                                                |
| ---------------------------------------------------------- | --------------------- | ---------------------------------------------------------- |
| `tip(tipper, postId, token, amount)`                       | Write (throwaway XDR) | Builds a `tip` operation XDR                               |
| `prepareTipTx(tipper, postId, token, amount, horizonUrl?)` | Write (submittable)   | Fetches real sequence, simulates, returns wallet-ready XDR |
| `getFeeBps()`                                              | Read                  | Returns the current protocol fee in basis points           |
| `setFee(feeBps)`                                           | Write (throwaway XDR) | Admin: set the protocol fee (e.g. `150` = 1.5%)            |
| `getTipCooldownWindow()`                                   | Read                  | Returns the tip cooldown in ledgers                        |
| `setTipCooldownWindow(cooldownLedgers)`                    | Write (throwaway XDR) | Admin: set the tip cooldown window                         |

> **Note:** The SDK does not expose a `getTipTotals` method — cumulative tip totals per post are maintained by the indexer in the `posts.tip_total` column and returned in feed and search responses. Query them via the indexer's search or feed endpoints.

### Fee split explanation

When a user tips `amount` stroops of `token`:

1. The contract reads the current `feeBps` (settable by admin via `setFee`).
2. Protocol fee = `floor(amount × feeBps / 10000)`.
3. The protocol fee is transferred to the `treasury` address.
4. The remaining `amount − fee` is credited to the post author.

```
tip amount = 1,000,000 stroops
feeBps     = 200 (2%)
fee        = 20,000 stroops → treasury
payout     = 980,000 stroops → post author
```

A `feeBps` of `0` means no fee is charged.

### getTipTotals — time-range parameters

Cumulative tip totals are available from the indexer. The `/api/search` endpoint and `/api/feed` responses include a `tip_total` field per post (sum of all tip payouts since the post was indexed).

For time-range breakdowns, query the `tips` table directly or use the indexer search endpoint with the `from` / `to` date parameters:

| Parameter | Type                 | Description                          |
| --------- | -------------------- | ------------------------------------ |
| `from`    | ISO 8601 date string | Start of the time window (inclusive) |
| `to`      | ISO 8601 date string | End of the time window (inclusive)   |

```bash
# Tips received on a post within a date range
GET /api/search?q=&from=2024-01-01&to=2024-12-31
```

### Examples

#### sendTip — throwaway XDR (server-side queue)

```ts
import { LinkoraClient } from "linkora-sdk";

const client = new LinkoraClient({
  contractId: "CXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX",
  rpcUrl: "https://soroban-testnet.stellar.org",
});

// Tip the author of post #42 with 5 XLM (50,000,000 stroops)
const opXdr = client.tip(
  "GBFOY...", // tipper
  42n, // postId
  "CTOKEN...", // token contract address (e.g. XLM wrapped SEP-41)
  50_000_000n // amount in stroops
);
console.log("Tip Op XDR:", opXdr);
```

#### prepareTipTx — wallet-ready transaction

```ts
const txXdr = await client.prepareTipTx(
  "GBFOY...", // tipper
  42n, // postId
  "CTOKEN...", // token contract address
  50_000_000n // amount in stroops
);
// Pass txXdr to Freighter or another Stellar wallet for signing
```

#### Check current fee and simulate net payout

```ts
const feeBps = await client.getFeeBps(); // e.g. 200
const amount = 50_000_000n;
const fee = (amount * BigInt(feeBps)) / 10_000n;
const payout = amount - fee;
console.log(`Fee: ${fee} stroops, Creator receives: ${payout} stroops`);
```

#### Query tip totals for a post from the indexer

```ts
const response = await fetch("https://indexer.linkora.example/api/feed?limit=1&viewer=GBFOY...");
const { posts } = await response.json();
const post = posts[0];
console.log(`Post #${post.id} has received ${post.tip_total} stroops in tips`);
```
