# Linkora Mini App Developer Guide

Mini apps are self-contained web pages that run inside a native Linkora host
(the mobile app). The host injects a `window.LinkoraSDK` object at load time,
giving the page access to wallet signing, profile data, and post creation — all
gated by permissions the user approves on install.

---

## Table of contents

1. [Quickstart — scaffold in < 5 minutes](#quickstart)
2. [Manifest schema](#manifest-schema)
3. [Manifest validation](#manifest-validation)
4. [Bridge API reference](#bridge-api-reference)
5. [Canonical example: Tip Jar](#canonical-example-tip-jar)
6. [Submitting your mini app](#submitting-your-mini-app)

---

## Quickstart

You only need a static HTML file and a manifest. No bundler required.

### 1. Create your project directory

```bash
mkdir my-mini-app && cd my-mini-app
```

### 2. Write the manifest

```json
{
  "name": "My Mini App",
  "version": "1.0.0",
  "description": "Does something useful.",
  "entryPoint": "index.html",
  "permissions": ["wallet.getAddress"]
}
```

Save it as `linkora-manifest.json`.

### 3. Write the HTML entry point

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>My Mini App</title>
  </head>
  <body>
    <p id="address">Loading wallet…</p>

    <script>
      // The host injects window.LinkoraSDK when running inside Linkora.
      // Provide a mock for local development.
      const SDK = window.LinkoraSDK || {
        wallet: {
          getAddress: async () => "GAAZI4TCR3TY5OJHCTJC2A4QSY6CJWJH5IAJTGKIN2ER7LBNVKOCCWN",
        },
      };

      SDK.wallet.getAddress().then((address) => {
        document.getElementById("address").textContent = address;
      });
    </script>
  </body>
</html>
```

### 4. Test locally

Open `index.html` directly in a browser. The dev-fallback mock returns a static
Stellar address so you can develop without the Linkora app. When the host loads
your page it replaces `window.LinkoraSDK` with the real bridge.

### 5. Host and register

Deploy your files to any static host (GitHub Pages, Vercel, IPFS, etc.) and
submit your manifest URL through the Linkora developer portal.

That's it — five steps, no build toolchain required.

---

## Manifest schema

The schema lives at [`docs/mini-apps/manifest.schema.json`](./manifest.schema.json)
(JSON Schema draft-07). Every field is described below.

| Field         | Type                     | Required | Description                                                         |
| ------------- | ------------------------ | -------- | ------------------------------------------------------------------- |
| `name`        | `string` (1–50 chars)    | ✅       | Display name shown in the app store.                                |
| `version`     | `string` (semver)        | ✅       | Semantic version, e.g. `"1.2.0"`.                                   |
| `description` | `string` (max 200 chars) | —        | Short description shown under the app name.                         |
| `entryPoint`  | `string`                 | ✅       | Relative or absolute URL of the HTML entry point.                   |
| `icon`        | `string`                 | —        | URL (or data-URI) for the app icon. Recommended size: 256 × 256 px. |
| `permissions` | `string[]`               | ✅       | Scopes the app needs (see [permissions table](#permissions) below). |

`additionalProperties` is `false` — unknown fields cause validation to fail.

### Permissions

| Permission     | Description                                                        |
| -------------- | ------------------------------------------------------------------ |
| `wallet.read`  | Read-only access to the connected wallet address.                  |
| `wallet.sign`  | Request that the user sign arbitrary data.                         |
| `profile.read` | Read the user's public profile (address, username, creator token). |
| `post.create`  | Open a native confirmation sheet to publish a post.                |

> **Note:** The bridge also exposes the convenience aliases `wallet.getAddress`
> and `wallet.signTransaction` used in existing examples. These map
> internally to `wallet.read` and `wallet.sign` respectively. Prefer the
> canonical names above in new manifests.

### Example manifest with all fields

```json
{
  "name": "Tip Jar",
  "version": "1.0.0",
  "description": "Tip any Linkora post with XLM using your connected wallet.",
  "entryPoint": "index.html",
  "icon": "https://example.com/tip-jar-icon.png",
  "permissions": ["wallet.read", "wallet.sign"]
}
```

---

## Manifest validation

Validate your manifest against the schema before submitting. The easiest way is
with [`ajv-cli`](https://github.com/ajv-validator/ajv-cli):

```bash
npx ajv-cli validate \
  -s docs/mini-apps/manifest.schema.json \
  -d path/to/your/linkora-manifest.json
```

A valid manifest prints `path/to/your/linkora-manifest.json valid`. Any schema
violation prints a human-readable error with the failing field path.

You can also add validation to CI:

```yaml
# .github/workflows/ci.yml (excerpt)
- name: Validate mini-app manifest
  run: |
    npx ajv-cli validate \
      -s docs/mini-apps/manifest.schema.json \
      -d linkora-manifest.json
```

### Common validation errors

| Error                                                         | Fix                                            |
| ------------------------------------------------------------- | ---------------------------------------------- |
| `"name" must NOT have more than 50 characters`                | Shorten the display name.                      |
| `"version" must match pattern "^\d+\.\d+\.\d+$"`              | Use semver: `"1.0.0"` not `"v1"`.              |
| `"permissions[n]" must be equal to one of the allowed values` | Use only the [permitted scopes](#permissions). |
| `must NOT have additional properties`                         | Remove any fields not in the schema.           |

---

## Bridge API reference

When your page loads inside the Linkora host, `window.LinkoraSDK` is injected
automatically. Always guard with a dev fallback:

```js
const SDK = window.LinkoraSDK || {/* your mock */};
```

The bridge has two namespaces: `wallet` and `post`.

---

### `SDK.wallet`

#### `wallet.getAddress() → Promise<string>`

Returns the currently connected Stellar address (G-address).

**Requires:** no special permission (address is public).

```js
const address = await SDK.wallet.getAddress();
// "GAAZI4TCR3TY5OJHCTJC2A4QSY6CJWJH5IAJTGKIN2ER7LBNVKOCCWN"
```

---

#### `wallet.signTransaction(xdr: string) → Promise<{ signedXdr: string }>`

Shows the native signing confirmation sheet with a human-readable summary of the
transaction. Returns the signed XDR on approval.

**Requires:** `wallet.sign` (or legacy alias `wallet.signTransaction`) in
`permissions`.

**Throws:** `BridgeError` with `code: "UserRejected"` if the user dismisses the
sheet.

```js
try {
  const { signedXdr } = await SDK.wallet.signTransaction(unsignedXdr);
  // submit signedXdr to Horizon / Soroban RPC
} catch (err) {
  if (err.code === "UserRejected") {
    console.log("User cancelled.");
  }
}
```

---

### `SDK.post`

#### `post.create(content: string) → Promise<number | null>`

Opens a native post confirmation sheet pre-filled with `content`. The user can
edit the text before confirming. On confirmation the post is submitted to the
contract.

**Requires:** `post.create` in `permissions`.

**Returns:** the new `postId` (integer) on success, or `null` if the user
cancelled.

```js
const postId = await SDK.post.create("Just joined Linkora! 🚀");
if (postId !== null) {
  console.log("Published as post #" + postId);
}
```

---

### `SDK.profile`

#### `profile.get() → Promise<Profile>`

Returns the current user's profile object.

**Requires:** `profile.read` in `permissions`.

```ts
type Profile = {
  address: string; // Stellar G-address
  username: string | null; // display handle, e.g. "maya"
  creatorToken: CreatorToken | string | null;
};
```

```js
const profile = await SDK.profile.get();
console.log(profile.username); // "maya"
```

---

### Error codes

All bridge errors are instances of `BridgeError`:

| `code`                | When thrown                                                  |
| --------------------- | ------------------------------------------------------------ |
| `"PermissionDenied"`  | The manifest did not declare the required permission.        |
| `"UserRejected"`      | The user dismissed a signing or post confirmation sheet.     |
| `"MethodUnavailable"` | The host has no handler registered for the requested method. |

```js
try {
  await SDK.wallet.signTransaction(xdr);
} catch (err) {
  switch (err.code) {
    case "UserRejected":
      // user tapped Cancel — this is normal, no need to alert
      break;
    case "PermissionDenied":
      console.error("Add wallet.sign to your manifest permissions.");
      break;
    default:
      console.error("Bridge error:", err.message);
  }
}
```

---

## Canonical example: Tip Jar

The Tip Jar lives at [`examples/mini-apps/tip-jar/`](../../examples/mini-apps/tip-jar/).
It is the reference implementation and demonstrates every concept in this guide.

```
examples/mini-apps/tip-jar/
├── index.html             ← single-file app, no build step
└── linkora-manifest.json  ← manifest with wallet permissions
```

### What it shows

- **Dev fallback mock** — `window.LinkoraSDK` is read with an `||` fallback, so
  `index.html` works in any browser without the Linkora host.
- **Wallet connection display** — calls `SDK.wallet.getAddress()` on init and
  shows a connected/disconnected indicator.
- **Transaction signing flow** — builds a transaction XDR, calls
  `SDK.wallet.signTransaction`, handles `UserRejected`, and shows status
  feedback at each stage (building → waiting for signature → submitting).
- **Input validation** — guards against empty post IDs and non-positive amounts
  before touching the bridge.

### Run it locally

```bash
# No install needed — just open the file
open examples/mini-apps/tip-jar/index.html
# or: python3 -m http.server 8080
```

### Tip Jar manifest

```json
{
  "name": "Tip Jar",
  "version": "1.0.0",
  "description": "Tip any Linkora post with XLM using your connected wallet.",
  "entryPoint": "index.html",
  "icon": "...",
  "permissions": ["wallet.getAddress", "wallet.signTransaction"]
}
```

> The `wallet.getAddress` and `wallet.signTransaction` aliases used here are
> supported for backwards compatibility. New apps should prefer `wallet.read`
> and `wallet.sign`.

---

## Submitting your mini app

1. Host your files on a publicly reachable URL.
2. Validate the manifest (see [Manifest validation](#manifest-validation)).
3. Open a pull request adding your app entry to
   `apps/mobile/mini-apps/store.ts` following the existing shape:

   ```ts
   {
     id: "my-mini-app",
     name: "My Mini App",
     description: "Does something useful.",
     icon: "https://example.com/icon.png",
     entry: "https://example.com/my-mini-app/index.html",
     permissions: ["wallet.read"],
   }
   ```

4. Link to your hosted `linkora-manifest.json` in the PR description.

The bridge API reference is also published separately in
[`BRIDGE_API.md`](./BRIDGE_API.md).
