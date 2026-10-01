# Linkora Mini App Bridge API

The Bridge API is the contract between a mini app (a web page) and the Linkora
native host. When a mini app loads inside the host, `window.LinkoraSDK` is
injected with all methods the app is permitted to call.

Source: [`apps/mobile/mini-apps/bridge.ts`](../../apps/mobile/mini-apps/bridge.ts)

---

## How the bridge works

```
┌─────────────────────────────────┐
│  Mini App  (WebView / iframe)   │
│                                 │
│  const SDK = window.LinkoraSDK  │
│  SDK.wallet.signTransaction(…)  │
└──────────────┬──────────────────┘
               │  JS bridge call
               ▼
┌─────────────────────────────────┐
│  Linkora Native Host            │
│                                 │
│  1. Check manifest permissions  │
│  2. Show approval sheet (if     │
│     required)                   │
│  3. Call native handler         │
│  4. Return result / error       │
└─────────────────────────────────┘
```

1. The mini app calls a method on `window.LinkoraSDK`.
2. The bridge checks that the method's required permission is declared in the
   manifest. If not, a `PermissionDenied` error is thrown immediately.
3. For sensitive methods (signing, post creation, profile updates), a native
   confirmation sheet is shown to the user. If the user cancels, a
   `UserRejected` error is thrown.
4. If approved, the native handler runs and the result is returned to the
   calling page.

---

## Namespace: `wallet`

### `wallet.getAddress()`

```ts
SDK.wallet.getAddress(): Promise<string>
```

Returns the Stellar G-address of the currently connected wallet.

**Permission required:** none (the address is public).

**Returns:** `Promise<string>` — the Stellar address, e.g.
`"GAAZI4TCR3TY5OJHCTJC2A4QSY6CJWJH5IAJTGKIN2ER7LBNVKOCCWN"`.

**Example:**

```js
const address = await SDK.wallet.getAddress();
document.getElementById("addr").textContent = address;
```

---

### `wallet.signTransaction(xdr)`

```ts
SDK.wallet.signTransaction(xdr: string): Promise<{ signedXdr: string }>
```

Shows the native transaction approval sheet. The host decodes the XDR and
presents a human-readable summary. On user approval, the transaction is signed
with the user's key and the signed XDR is returned.

**Permission required:** `wallet.sign` (manifest alias: `wallet.signTransaction`).

**Approval required:** yes — always shows a native confirmation sheet.

**Parameters:**

| Name  | Type     | Description                             |
| ----- | -------- | --------------------------------------- |
| `xdr` | `string` | Base64-encoded Stellar transaction XDR. |

**Returns:** `Promise<{ signedXdr: string }>` — the signed transaction XDR.

**Throws:**

| Error code         | Condition                                    |
| ------------------ | -------------------------------------------- |
| `PermissionDenied` | `wallet.sign` not in manifest `permissions`. |
| `UserRejected`     | User dismissed the approval sheet.           |

**Example:**

```js
try {
  const { signedXdr } = await SDK.wallet.signTransaction(unsignedXdr);
  // submit to Horizon / Soroban RPC
} catch (err) {
  if (err.code === "UserRejected") {
    showStatus("Cancelled.", "info");
  }
}
```

---

## Namespace: `post`

### `post.create(content)`

```ts
SDK.post.create(content: string): Promise<number | null>
```

Opens a native post confirmation sheet pre-filled with `content`. The user can
edit the text before confirming. On confirmation the post is submitted to the
Linkora smart contract and the new post ID is returned.

**Permission required:** `post.create`.

**Approval required:** yes — always shows a native confirmation sheet.

**Parameters:**

| Name      | Type     | Description                                    |
| --------- | -------- | ---------------------------------------------- |
| `content` | `string` | Initial text to pre-fill in the post composer. |

**Returns:** `Promise<number | null>` — the new `postId` on success, or `null`
if the user cancelled.

**Throws:**

| Error code         | Condition                                    |
| ------------------ | -------------------------------------------- |
| `PermissionDenied` | `post.create` not in manifest `permissions`. |

**Example:**

```js
const postId = await SDK.post.create("Launched my new mini app! 🚀");
if (postId !== null) {
  showStatus("Posted as #" + postId, "success");
}
```

---

## Namespace: `profile`

### `profile.get()`

```ts
SDK.profile.get(): Promise<Profile>
```

Returns the current user's Linkora profile.

**Permission required:** `profile.read` (manifest alias: `profile.get`).

**Returns:**

```ts
type Profile = {
  address: string; // Stellar G-address
  username: string | null; // handle, e.g. "maya"
  creatorToken: CreatorToken | string | null; // see below
};

type CreatorToken = {
  code: string; // e.g. "MAYA"
  issuer: string; // Stellar account of the token issuer
};
```

**Throws:**

| Error code         | Condition                                     |
| ------------------ | --------------------------------------------- |
| `PermissionDenied` | `profile.read` not in manifest `permissions`. |

**Example:**

```js
const profile = await SDK.profile.get();
console.log(profile.username); // "maya"
console.log(profile.creatorToken.code); // "MAYA"
```

---

## Error reference

All bridge errors are plain JavaScript objects with a `code` string and a
`message` string. They are instances of `BridgeError`.

```ts
class BridgeError extends Error {
  code: "PermissionDenied" | "UserRejected" | "MethodUnavailable";
}
```

| `code`              | When thrown                                                       |
| ------------------- | ----------------------------------------------------------------- |
| `PermissionDenied`  | The manifest did not declare the required permission scope.       |
| `UserRejected`      | The user dismissed a native approval sheet (signing, post, etc.). |
| `MethodUnavailable` | The host has no registered handler for the requested method.      |

### Handling errors

```js
try {
  await SDK.wallet.signTransaction(xdr);
} catch (err) {
  switch (err.code) {
    case "UserRejected":
      // Normal user action — don't show an error alert
      break;
    case "PermissionDenied":
      console.error("Check your manifest permissions:", err.message);
      break;
    case "MethodUnavailable":
      console.error("Host does not support this method:", err.message);
      break;
    default:
      console.error("Unexpected error:", err);
  }
}
```

---

## Dev fallback pattern

Because the host is only available inside the Linkora mobile app, always provide
a dev fallback for local testing. The pattern used in the canonical examples:

```js
const SDK = window.LinkoraSDK || {
  wallet: {
    getAddress: async () => "GAAZI4TCR3TY5OJHCTJC2A4QSY6CJWJH5IAJTGKIN2ER7LBNVKOCCWN",
    signTransaction: async (xdr) => {
      console.log("[mock] signTransaction", xdr);
      return { signedXdr: xdr };
    },
  },
  post: {
    create: async (content) => {
      console.log("[mock] post.create", content);
      return 1;
    },
  },
  profile: {
    get: async () => ({
      address: "GAAZI4TCR3TY5OJHCTJC2A4QSY6CJWJH5IAJTGKIN2ER7LBNVKOCCWN",
      username: "dev-user",
      creatorToken: {
        code: "DEV",
        issuer: "GAAZI4TCR3TY5OJHCTJC2A4QSY6CJWJH5IAJTGKIN2ER7LBNVKOCCWN",
      },
    }),
  },
};
```

The mock never shows any native UI, so all flows complete instantly in the
browser. Swap to the real host by loading the page inside the Linkora app.

---

## Method quick-reference

| Method                 | Namespace | Permission     | Approval sheet |
| ---------------------- | --------- | -------------- | -------------- |
| `getAddress()`         | `wallet`  | none           | no             |
| `signTransaction(xdr)` | `wallet`  | `wallet.sign`  | ✅ yes         |
| `create(content)`      | `post`    | `post.create`  | ✅ yes         |
| `get()`                | `profile` | `profile.read` | no             |

---

See [DEVELOPER_GUIDE.md](./DEVELOPER_GUIDE.md) for the full quickstart and
manifest schema documentation.
