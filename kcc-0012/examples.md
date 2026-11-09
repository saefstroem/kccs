# KCC-12 examples

Non-normative code illustrating [KCC-12](../kcc-0012.md). Where this file
and the specification disagree, the specification is authoritative. Types
refer to [interfaces.ts](interfaces.ts).

## 1. Wallet side: announcing a provider

Run in the page context once the provider object is ready. Register the request listener first, then perform the announcement. This is done so that an app that requests between the two steps is still answered.

```typescript
import type { KaspaProvider, KaspaProviderDetail, KaspaProviderInfo } from "./interfaces";

export function announceProvider(provider: KaspaProvider): void {
  const info: KaspaProviderInfo = Object.freeze({
    uuid: crypto.randomUUID(), // fresh per page load
    name: "Example Wallet",
    icon: "data:image/svg+xml;base64,PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciLz4=",
    rdns: "com.example.wallet", // stable across page loads
  });
  const detail: KaspaProviderDetail = Object.freeze({ info, provider });

  const announce = () =>
    window.dispatchEvent(new CustomEvent("kaspa:announceProvider", { detail }));

  window.addEventListener("kaspa:requestProvider", announce);
  announce();
}
```

## 2. App side: discovering providers

```typescript
import type { KaspaProviderDetail } from "./interfaces";

const providers = new Map<string, KaspaProviderDetail>(); // keyed by uuid

export function discoverProviders(onChange: (list: KaspaProviderDetail[]) => void): void {
  window.addEventListener("kaspa:announceProvider", (event) => {
    const detail = event.detail;
    if (!detail?.info?.uuid || typeof detail.provider?.request !== "function") return;
    if (!detail.info.icon.startsWith("data:")) return; // Section 3.6
    const seen = providers.get(detail.info.uuid);
    if (seen && (seen.provider !== detail.provider || seen.info.rdns !== detail.info.rdns)) {
      console.warn("conflicting announcement for uuid", detail.info.uuid);
      return;  // Section 3.5
    }
    providers.set(detail.info.uuid, detail);
    onChange([...providers.values()]);
  });
  // Dispatch only after the listener is registered
  window.dispatchEvent(new Event("kaspa:requestProvider"));
}
```

## 3. App side: connecting and reacting to changes

```typescript
import type { Address, KaspaProvider, NetworkId, ProviderRpcError } from "./interfaces";

export async function connect(provider: KaspaProvider) {
  // returns [] until the user has authorized this origin.
  let accounts = await provider.request({ method: "kaspa_accounts" });
  if (accounts.length === 0) {
    try {
      accounts = await provider.request({ method: "kaspa_requestAccounts" });
    } catch (e) {
      const err = e as ProviderRpcError;
      if (err.code === 4001) return null; // user declined
      throw err;
    }
  }
  const networkId = await provider.request({ method: "kaspa_networkId" });

  provider.on("accountsChanged", (next: Address[]) => {
    if (next.length === 0) console.log("disconnected or locked");
    else console.log("active disclosed address is now", next[0]);
  });
  provider.on("networkChanged", (id: NetworkId) => {
    console.log("network is now", id); // discard cached UTXOs, balances, etc.
  });

  return { accounts, networkId };
}

export async function disconnect(provider: KaspaProvider) {
  await provider.request({
    method: "wallet_revokePermissions",
    params: [{ kaspa_accounts: {} }],
  });
}
```

## 4. App side: signing a message

```typescript
const signature = await provider.request({
  method: "kaspa_signMessage",
  params: ["Sign in to example.com\nNonce: 7f3a…\nExpires: 2026-09-10T12:00:00Z", accounts[0]],
});
// signature is 128 hex characters; verify per KIP-5 with the x-only key taken
// from the address payload.
```

## 5. App side: signing a transaction with a covenant input

The app builds the transaction with the Kaspa WASM SDK, including inputs
whose signature scripts are already in place (a covenant input), serializes
it, and asks the wallet to sign only the user's inputs.

```typescript
import type { SerializedTransaction } from "./interfaces";

// tx: kaspa.Transaction built by the app; input 0 is a covenant input whose
// signatureScript is already set; input 1 spends the user's UTXO.
const unsigned: SerializedTransaction = tx.serializeToSafeJSON();

const signed = await provider.request({
  method: "kaspa_signTransaction",
  params: [{ transaction: unsigned, signInputs: [{ index: 1, sighashType: 1 }] }],
});
// Input 1 may sit at an address kaspa_accounts did not return, if that
// address belongs to an authorized account. The call does not add the
// address to kaspa_accounts. An account the user did not authorize is 4100.

const txid = await provider.request({
  method: "kaspa_sendRawTransaction",
  params: [signed],
});
```

When the covenant itself requires the owner's transaction signature, as the
key-based owner schemes of KCC-20 do, the wallet cannot write the input's
script and the app cannot make the signature. `kaspa_signInputs` returns the
bare signature; the app places it and then has the wallet sign its fee
input as above.

```typescript
// tx: input 0 spends a token UTXO whose covenant checks the owner's
// signature; input 1 pays the fee from the user's account.
const [ownerSig] = await provider.request({
  method: "kaspa_signInputs",
  params: [{ transaction: tx.serializeToSafeJSON(), signInputs: [{ index: 0, address: ownerAddress, sighashType: 1 }] }],
});
// Compose input 0's signature script as the covenant defines it, appending
// the sighash type byte "01" to ownerSig where the covenant expects a
// 65-byte transaction signature, then continue with kaspa_signTransaction
// for input 1 and kaspa_sendRawTransaction.
```

## 6. App side: co-signing a bundle (multi-signature)

For flows that gather signatures from several parties, the app exchanges a
`Pskb` string and asks each wallet to add its partial signatures without
finalizing.

```typescript
import type { Pskb, TransactionId } from "./interfaces";

// pskb: a "PSKB..."-prefixed bundle the app built with the SDK's PSKB class.
const cosigned = await provider.request({
  method: "kaspa_signPskb",
  params: [pskb],
});

// After combining the co-signed bundles (combined: Pskb), any connected
// wallet can finalize, extract, and submit the result in bundle order.
const txids = await provider.request({
  method: "kaspa_sendRawPskb",
  params: [combined],
});

// Single-signer flows sign and submit under one prompt.
const txids2 = await provider.request({
  method: "kaspa_sendPskb",
  params: [pskb],
});
```

A submission error carries `data.submitted`, the identifiers the node has
already accepted, so a dependent chain can be resumed rather than resent.

## 7. App side: feature detection

```typescript
import type {
  KaspaProvider,
  KaspaRpcMethod,
  RequestArguments,
  ProviderRpcError,
} from "./interfaces";

async function supports<M extends KaspaRpcMethod>(
  provider: KaspaProvider,
  args: RequestArguments<M>,
): Promise<boolean> {
  try {
    await provider.request(args);
    return true;
  } catch (e) {
    return (e as ProviderRpcError).code !== 4200; // any other error means "supported"
  }
}

const hasBundles = await supports(provider, {
  method: "kaspa_signPskb",
  params: ["PSKB00"],
});
```

A vendor method is detected the same way once the app declares it, which
also gives the call its types:

```typescript
declare module "./interfaces" {
  interface KaspaRpcSchema {
    "com.example.wallet_getVaults": { params: []; result: string[] };
  }
}

const hasVaults = await supports(provider, {
  method: "com.example.wallet_getVaults",
});
```

## 8. Wallet side: a page-to-extension transport

KCC-12 does not specify how the provider reaches the extension. The example
below is one common design: an in-page script which exposes the provider and
relays each request through `window.postMessage` to a content script. This is then
forwarded to the extension background over a runtime port. The content
script validates the origin of every message where the background then attributes
every request to that origin.

A transport relays opaque messages, so it is written untyped inside and
exposed through the typed interface at the boundary.

The transport gives the app no protection from scripts in its own page. Any
page script can read a request's `id` from the `message` event and post a
matching response before the extension does, just as it can wrap or replace
the provider object itself (KCC-12, Security Considerations). Responses are
matched to their request by `id`, never to the oldest pending request, so an
unrelated reply cannot settle a pending send; but the page remains untrusted,
and the wallet's own interface is the only place a request is confirmed.

```typescript
// inpage.ts (runs in the page's JavaScript realm)
type Pending = { resolve: (v: unknown) => void; reject: (e: unknown) => void };
const pending = new Map<string, Pending>();
const listeners = new Map<string, Set<(...args: unknown[]) => void>>();
const CHANNEL = "com.example.wallet";

const relay = {
  request(args: { method: string; params?: unknown }): Promise<unknown> {
    return new Promise((resolve, reject) => {
      const id = crypto.randomUUID();
      pending.set(id, { resolve, reject });
      window.postMessage({ channel: CHANNEL, kind: "request", id, ...args }, window.location.origin);
    });
  },
  on(event: string, listener: (...args: unknown[]) => void) {
    (listeners.get(event) ?? listeners.set(event, new Set()).get(event)!).add(listener);
    return this;
  },
  removeListener(event: string, listener: (...args: unknown[]) => void) {
    listeners.get(event)?.delete(listener);
    return this;
  },
};

// The relay cannot prove to the compiler that each method returns that
// method's result type; the announcement exposes it as a KaspaProvider.
const provider = relay as unknown as KaspaProvider;

window.addEventListener("message", (event) => {
  if (event.source !== window || event.origin !== window.location.origin) return;
  const msg = event.data;
  if (msg?.channel !== CHANNEL) return;
  if (msg.kind === "response") {
    const p = pending.get(msg.id);
    if (!p) return;
    pending.delete(msg.id);
    if (msg.error) {
      const err = new Error(msg.error.message) as ProviderRpcError;
      err.code = msg.error.code;
      err.data = msg.error.data;
      p.reject(err);
    } else {
      p.resolve(msg.result);
    }
  } else if (msg.kind === "event") {
    listeners.get(msg.event)?.forEach((l) => l(msg.payload));
  }
});

announceProvider(provider);
```

```typescript
// content.ts
const port = chrome.runtime.connect({ name: "provider" });
window.addEventListener("message", (event) => {
  if (event.source !== window || event.origin !== window.location.origin) return;
  const msg = event.data;
  if (msg?.channel !== "com.example.wallet" || msg.kind !== "request") return;
  port.postMessage({ id: msg.id, method: msg.method, params: msg.params, origin: event.origin });
});
port.onMessage.addListener((msg) => {
  window.postMessage({ channel: "com.example.wallet", ...msg }, window.location.origin);
});
```

## 9. App side: reading the account balance

`kaspa_getBalance` is optional and does not prompt. Code `4200` means this wallet has no account total. The app then has the disclosed addresses only. Code `4100` means the origin is not authorized, or the wallet is locked. Neither code is a prompt.

```typescript
import type { AccountBalance, KaspaProvider, ProviderRpcError } from "./interfaces";

async function readBalance(provider: KaspaProvider): Promise<AccountBalance | null> {
  try {
    return await provider.request({ method: "kaspa_getBalance" });
  } catch (e) {
    const err = e as ProviderRpcError;
    if (err.code === 4200) return null; // sum disclosed addresses on a public node
    if (err.code === 4100) return null; // unauthorized or locked; no prompt
    if (err.code === -32005) return null; // the wallet rate-limited this origin
    throw err;
  }
}
```

`kas` is sompi as a decimal string. It follows acceptance on the selected chain. A change of balance does not emit an event. An app that displays it calls the method again. The wallet may answer with `-32005`. `tokens`, when absent, is not a report of zero.

A wallet that implements `kaspa_getBalance` does not add a fresh change address to `kaspa_accounts` merely because the address received change. A wallet that returns `4200` does add it, and emits `accountsChanged`. An app that received `null` above, for code `4200`, keeps summing disclosed addresses, including addresses that arrive through that event.

## 10. App side: asking the wallet to send

Omit `from` to spend the account of the active disclosed address. Pass `from` to spend only that address. A one-address account spends the same outputs either way. The result is the transaction id.

```typescript
const accountSend = await provider.request({
  method: "kaspa_sendTransaction",
  params: [{ outputs: [{ address: payTo, amount: "100000000" }] }],
});

const oneAddress = await provider.request({
  method: "kaspa_sendTransaction",
  params: [{ from: accounts[0], outputs: [{ address: payTo, amount: "100000000" }] }],
});
```

`oneAddress` does not spend an undisclosed output of the same account. A covenant output is not an input of either call. The app builds that transaction and uses `kaspa_signInputs` or `kaspa_signTransaction`. If the wallet cannot cover the payment from the outputs it is willing to select, the error is `-32000` and it has no `submitted` member.

## 11. App side: notes and a receive address

Both methods are optional. `kaspa_getNotes` is the permission `kaspa_getNotes`. The first call prompts. The prompt says that the origin will receive every note of the account, including addresses that were not disclosed. If the user approves, later calls do not prompt, until `wallet_revokePermissions` names that permission. The call does not add those addresses to `kaspa_accounts`. `kaspa_requestReceiveAddress` does not prompt. The address it returns becomes disclosed, and it does not become the first element only because it was requested.

```typescript
import type { Address, Note, ProviderRpcError } from "./interfaces";

let notes: Note[] | null = null;
try {
  notes = await provider.request({ method: "kaspa_getNotes" });
} catch (e) {
  const err = e as ProviderRpcError;
  if (err.code === 4200) notes = null; // this wallet has no note list
  else if (err.code === 4001) notes = null; // user declined the permission
  else throw err;
}

const receive: Address = await provider.request({ method: "kaspa_requestReceiveAddress" });
```

Code `4200` on either call means that method is absent. The other method is unaffected. An app that only needed a total uses `kaspa_getBalance` and does not call `kaspa_getNotes`. A later `kaspa_getNotes` from this origin does not prompt while the permission stands.

On a note whose covenant state the wallet decoded, `tokenAmount` is the token amount and `state` is the decoded state. `redeemScript`, when present, is the P2SH script. The chain does not reveal that script until the note is spent. The app uses these fields to choose the note and to build the transaction it passes to `kaspa_signInputs` or `kaspa_signTransaction`. The fields hold public data: a public key or a key hash at most.

```typescript
const tokenNote = notes?.find((note) => note.tokenAmount !== undefined);
// tokenNote?.state is EncodeState. tokenNote?.redeemScript is omitted when
// the output is not P2SH or the wallet cannot reconstruct the script.
```
