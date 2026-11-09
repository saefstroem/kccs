# Account model

Non-normative companion to [KCC-12](../kcc-0012.md). Where this file and the
specification disagree, the specification is authoritative.

<table>
<thead>
<tr><th>Normative</th><th>Why</th></tr>
</thead>
<tbody>
<tr>
<td>
<b>Account.</b> The set of addresses a wallet spends as one balance. The set may contain one address. A second balance is a second account. The provider must not return an extended public key, a derivation path, or any other value from which an app can derive an address.
<br><br>
<a href="../kcc-0012.md#1-definitions">Section 1</a>
</td>
<td>
One balance is often spent from many addresses, including change the app is not shown. Calling each address an account forces the app to learn every address before the balance is knowable from what it holds. A single address is the same rule with a set of one, and <code>kaspa_accounts</code> is still an array of address strings.
</td>
</tr>
<tr>
<td>
<b>Disclosed address.</b> An address <code>kaspa_accounts</code> returns. The first element is the active disclosed address and names the account used when the app does not name one. Every disclosed address is authorized. An app must not require a switch, and must not treat a missing entry as proof that the account has no further funds.
<br><br>
<a href="../kcc-0012.md#1-definitions">Section 1</a>,
<a href="../kcc-0012.md#61-kaspa_requestaccounts">Section 6.1</a>
</td>
<td>
The array is what the user agreed to show, not the account. Keeping that split lets a wallet show one receive address and still spend the rest of the same balance, without handing the origin a path.
</td>
</tr>
<tr>
<td>
<b><code>kaspa_getBalance</code>.</b> Optional, restricted, no prompt, permission <code>kaspa_accounts</code>. Result <code>{ kas: Amount, tokens?: ... }</code>. <code>kas</code> is the sompi sum of the balance set of the account that contains the active disclosed address. Amounts are <code>Uint64</code> strings. Locked or unauthorized is <code>4100</code> with no prompt. Unimplemented is <code>4200</code>. The call works while disconnected and must not return <code>4900</code>.
<br><br>
<code>kaspa_getBalance: { params: []; result: AccountBalance }</code>
<br><br>
<a href="../kcc-0012.md#614-kaspa_getbalance">Section 6.14</a>,
<a href="../kcc-0012.md#72-unsigned-decimal-strings-and-amounts">Section 7.2</a>
</td>
<td>
An app that sums disclosed addresses on a public node misses undisclosed change. This call returns the account total without listing those addresses and without forwarding node RPC. It does not prompt because it reveals an amount, not a new address.
</td>
</tr>
<tr>
<td>
<b>Balance set.</b> An output is in the set only when its creating transaction is accepted on the selected chain, no accepted transaction spends it, the account can move it with its own keys alone, and consensus currently allows the spend. Mempool creations are out. An accepted output spent only by an unaccepted transaction stays in. A time lock that has not expired is out, as is an immature coinbase. A position that needs another party's key is out. A reorganisation changes the set only by changing acceptance. The wallet answers from its current view. A change of balance does not emit an event.
<br><br>
<a href="../kcc-0012.md#614-kaspa_getbalance">Section 6.14</a>,
<a href="../kcc-0012.md#44-events">Section 4.4</a>
</td>
<td>
The figure is the accepted sum the account can move now, so two wallets that share a view agree, and a public node that knew every address would compute the same sompi. Unaccepted funds, locked outputs, and shared positions are not that sum. An event for the change would publish the moment the funds moved.
</td>
</tr>
<tr>
<td>
<b>Change.</b> Change is paid to the same account the inputs came from. A fresh change address is added to <code>kaspa_accounts</code> only when the provider does not implement <code>kaspa_getBalance</code> (it would return <code>4200</code>). Code <code>4100</code> does not mean the method is unimplemented. When the method exists, the new output enters <code>kas</code> only once its creating transaction is accepted.
<br><br>
<a href="../kcc-0012.md#65-kaspa_sendtransaction">Section 6.5</a>
</td>
<td>
Disclosing every new change address is what made an address-walking total complete, and it publishes the cluster. A wallet that serves the total does not need to publish the address. A wallet that does not serve the total keeps the old disclosure, so existing apps do not lose the change.
</td>
</tr>
<tr>
<td>
<b>Send.</b> When <code>from</code> is absent, inputs come from the selectable set of the whole account of the active disclosed address. When <code>from</code> is present, inputs come only from that one address, which need not be disclosed. The selectable set is the balance set minus this wallet's outstanding submissions, minus outputs that carry a covenant id. If that set cannot cover the payment, the error is <code>-32000</code> and it has no <code>submitted</code> member. A one-address account selects the same outputs either way.
<br><br>
<a href="../kcc-0012.md#65-kaspa_sendtransaction">Section 6.5</a>
</td>
<td>
Omitting <code>from</code> is how a payment spends change the app has not been shown. Passing <code>from</code> keeps the old one-address selection, which is what an app that already passes the active address asked for. Covenant outputs stay out of a plain send because this method does not build a continuation.
</td>
</tr>
<tr>
<td>
<b>Spendability.</b> An input is spendable by an authorized account when a key of that account matches the script, disclosed or not. A key of an account the user did not authorize is <code>4100</code>. A script that needs another party's key is outside the balance set and is still spendable for a partial signature. <code>kaspa_signMessage</code> still requires a disclosed address, and it does not prompt for any other address. The origin does not receive the key.
<br><br>
<a href="../kcc-0012.md#77-signature-scripts">Section 7.7</a>,
<a href="../kcc-0012.md#64-kaspa_signmessage">Section 6.4</a>
</td>
<td>
Section 7.7 already decided spendability from the key, not from the position of an address in a list. Once the account includes undisclosed addresses, those keys are part of the authorized account. The wallet uses them privately. Message signing stays on the disclosed list so a signature is not a probe for hidden addresses.
</td>
</tr>
<tr>
<td>
<b><code>kaspa_getNotes</code> and <code>kaspa_requestReceiveAddress</code>.</b> Both are optional and apply only to the account of the active disclosed address. Notes are the balance set. <code>kaspa_getNotes</code> is the permission of that name. The first call prompts once. The prompt says the origin will receive every note of the account, including undisclosed addresses. The grant is per origin and lasts until <code>wallet_revokePermissions</code>. Later calls do not prompt. The call does not add addresses to <code>kaspa_accounts</code>. From the same view, the sompi sum equals <code>kas</code>. A receive address becomes disclosed, does not become the first element merely because it was requested, and the call does not prompt.
<br><br>
<a href="../kcc-0012.md#615-kaspa_getnotes">Section 6.15</a>,
<a href="../kcc-0012.md#616-kaspa_requestreceiveaddress">Section 6.16</a>,
<a href="../kcc-0012.md#613-permissions">Section 6.13</a>
</td>
<td>
The note list is the cluster, so it needs consent, once. A prompt on every call would ask the user to approve the same disclosure again. The total stays a different method so that balance does not mean every outpoint. One new receive address is a disclosure the user already authorized the origin to ask for.
</td>
</tr>
<tr>
<td>
<b>Decoded note.</b> When the wallet has decoded the covenant state, the note includes <code>state</code> (<code>Hex</code>, <code>EncodeState</code>). When that state is KCC-20, the note also includes <code>tokenAmount</code> (<code>Uint64</code>, <code>KCC20State.amount</code>). When the output is P2SH and the wallet can reconstruct the redeem script, the note includes <code>redeemScript</code> (<code>Hex</code>). These fields are public data: a public key or a key hash at most, never a private key, an extended key, or a path.
<br><br>
<a href="../kcc-0012.md#615-kaspa_getnotes">Section 6.15</a>
</td>
<td>
The token amount and the state live in the redeem script. The chain does not reveal that script until the output is spent. Without these fields an app cannot select the token note or build the transaction it passes to <code>kaspa_signInputs</code> or <code>kaspa_signTransaction</code>.
</td>
</tr>
<tr>
<td>
<b>What an origin may learn.</b> Disclosed addresses; a further address when the user discloses it, when it calls <code>kaspa_requestReceiveAddress</code>, or in the notes once it holds <code>kaspa_getNotes</code>; the aggregate <code>kas</code> and any reported <code>tokens</code>, with no breakdown; and, after submit, what the chain makes public. A decoded note may include a public key or a key hash. It must not learn a private key, an extended public key, a path, an address of an unauthorized account, or how many accounts exist. There is no balance event. <code>kas</code> is exact. Repeated balance reads and probes should be answered with <code>-32005</code>, with no substitute figure.
<br><br>
<a href="../kcc-0012.md#security-considerations">Security Considerations</a>
</td>
<td>
The total is useful only if it is the real sum, so the difference from a public-node read of the disclosed addresses is visible. That difference is an amount. It is not a location, and it is not a key. Rate limiting is the control on quiet re-reading and on address probing, not noise in the figure.
</td>
</tr>
</tbody>
</table>

## Anticipated questions

### Does a single-address wallet, and an app written for one, still work?

Yes. The account is that address ([Section 1](../kcc-0012.md#1-definitions)). `kaspa_accounts` returns it. When `from` is absent and when `from` is that address, the selected outputs are the same ([Section 6.5](../kcc-0012.md#65-kaspa_sendtransaction)). A wallet that rejects `kaspa_getBalance`, `kaspa_getNotes`, or `kaspa_requestReceiveAddress` with `4200` still conforms, and it still discloses a fresh change address ([Backwards Compatibility](../kcc-0012.md#backwards-compatibility), [Section 6.5](../kcc-0012.md#65-kaspa_sendtransaction)). An app that passes the active disclosed address as `from` still spends only that address.

### Why not return an extended public key?

An extended public key lets the origin derive every address of the account, present and future, and watch them on a public node. It also identifies the account across sites while no funds move. [Section 1](../kcc-0012.md#1-definitions) forbids returning an extended public key or a derivation path. [Section 6.9](../kcc-0012.md#69-kaspa_signpskb) forbids the wallet from adding `bip32Derivations` or `xpubs` entries. The bundle format still has those fields, because a bundle the app already built may carry them. This provider does not add them.

### Why is there no account-switch method?

The active account is the wallet's selection, expressed as the first disclosed address ([Section 1](../kcc-0012.md#1-definitions)). The user changes it in the wallet. The provider emits `accountsChanged` ([Section 4.4](../kcc-0012.md#44-events)). An app must not require a switch in order to use a disclosed address ([Section 6.1](../kcc-0012.md#61-kaspa_requestaccounts)). A method that let the page change the selection would take that choice out of the wallet. An app that needs one disclosed address passes it to a method that takes an address: `from`, `kaspa_signMessage`, or `kaspa_signInputs`. `kaspa_getBalance`, `kaspa_getNotes`, and `kaspa_requestReceiveAddress` take no account argument ([Section 6.14](../kcc-0012.md#614-kaspa_getbalance), [Section 6.15](../kcc-0012.md#615-kaspa_getnotes), [Section 6.16](../kcc-0012.md#616-kaspa_requestreceiveaddress)).

### How does this relate to KCC-3 and to BIP-44?

[KCC-3](https://github.com/kaspanet/kccs/pull/38) (pull 38, open, not a dependency of this document) defines

```text
m / purpose' / coin_type' / sig_scheme' / account' / change / address_index
```

with receive chain `0` and change chain `1` under one hardened account index. Its text says it does not claim compatibility with BIP-44. BIP-44's shape is `m / 44' / coin_type' / account' / change / address_index`. In both, one account index is a receive chain plus a change chain. [Section 1](../kcc-0012.md#1-definitions) uses "account" for the set of addresses a wallet spends as one balance. That is the same set when the wallet uses either path, and it is one address when the wallet uses neither. This document does not require a path, and it does not wait on KCC-3's purpose registry.

### What does `kas` do under a reorganisation, and what about the mempool?

[Section 6.14](../kcc-0012.md#614-kaspa_getbalance) defines it. A transaction is accepted when the consensus rules of the current network accept it on the selected chain, in the view the wallet holds when it answers. `kas` sums outputs whose creating transaction is accepted, that no accepted transaction spends, that the account can move with its own keys alone, and that consensus currently allows to be spent. A coinbase output that has not matured is out. An output whose time lock has not expired is out. An output that exists only in the mempool is out. An accepted output that an unaccepted transaction spends, including one this wallet has submitted, stays in. After a reorganisation the wallet follows the new selected chain. It does not keep the previous figure, and it does not average.

`kaspa_sendTransaction` does not select an output spent by an outstanding submission of its own, and it does not select an output that carries a covenant id ([Section 6.5](../kcc-0012.md#65-kaspa_sendtransaction)). `kas` can therefore be larger than the next send can pay. The refusal is `-32000`, before submit, with no `submitted` member. While disconnected, the call still succeeds and returns the last view, which may be stale. It must not return `4900`.

### Why is a time-locked output outside the balance?

[Section 6.14](../kcc-0012.md#614-kaspa_getbalance) includes an output only when consensus currently allows it to be spent. A coinbase that has not matured is the case that section names. A time lock that has not expired fails the same test: the account cannot move the output now, so its sompi is not part of `kas`. [Section 6.5](../kcc-0012.md#65-kaspa_sendtransaction) then also skips this wallet's outstanding spends. A future KCC may report outputs that are not yet spendable. This document does not.

### Which covenant outputs count, and how are token totals keyed?

An output is in the balance set when the account can move it alone ([Section 6.14](../kcc-0012.md#614-kaspa_getbalance)): the version 0 or 1 script of one of its keys; a version 8 script whose redeem script those keys alone satisfy; or a covenant the wallet can decode and whose owner authorization those keys alone can produce. For KCC-20 the schemes are the ones KCC-20 Section 3 lists, matched as [KCC-2](../kcc-0002.md) defines. `p2pk-schnorr/v1` matches a public key of the account. `p2pkh-schnorr/v1` and `p2pkh-ecdsa/v1` match the scheme's hash of such a key. `p2sh/v1` counts when the account's keys alone satisfy the redeem script. `covenant-id/v1` counts only when the wallet can produce that covenant's authorization from keys of the account. If it cannot decide, the output is out. A state the wallet cannot decode is out. The wallet does not guess.

Sompi on an output in the set is part of `kas`, including sompi on a covenant output. The token amount is not sompi and is not part of `kas`. `tokens`, when present, sums `KCC20State.amount` by `covenantId` and `commitment`. `covenantId` is 64 lowercase hex characters, the id [Section 7.5](../kcc-0012.md#75-serialized-transactions) already uses. `commitment` is KCC-20 `extension_commitment`, also 64 lowercase hex characters, and it is present for a KCC-20 total even when every byte is zero. Different commitments are different elements because KCC-20 Section 4 says those states are not fungible. An omitted `tokens` member is not a report of zero. `"tokens": []` means every total the wallet reports is zero, not that every covenant was decoded. Borrowed receive keeps the owner (KCC-20 Section 5), so the output stays in the set when the owner test still holds.

A script that needs another party's key is not in the set. It can still take a partial signature under [Section 7.7](../kcc-0012.md#77-signature-scripts). `kaspa_sendTransaction` does not spend an output that carries a covenant id. The app builds that transaction and uses `kaspa_signInputs` or `kaspa_signTransaction`. When the wallet decoded the state, `kaspa_getNotes` returns the token amount, the state, and, for P2SH, the redeem script the app needs for that transaction ([Section 6.15](../kcc-0012.md#615-kaspa_getnotes)).

### Why is a multi-party position outside the balance?

[Section 6.14](../kcc-0012.md#614-kaspa_getbalance) counts an output only when keys of the account alone can move it. An escrow, a pool share, or any other position that needs another party's key fails that test, so its sompi is not in `kas`. [Section 7.7](../kcc-0012.md#77-signature-scripts) still allows a partial signature on that input, through `kaspa_signInputs` or `kaspa_signPskb`. A method that lists positions held with other parties is a separate concern. This document does not define one.

### What about a watch-only account?

[Section 1](../kcc-0012.md#1-definitions) and [Section 5.4](../kcc-0012.md#54-watch-only-accounts): the wallet can derive addresses and cannot produce a signature. It may be authorized. `kaspa_getBalance` must not reject it for being watch-only, and neither the address nor `kas` says that it is watch-only ([Section 6.14](../kcc-0012.md#614-kaspa_getbalance)). A method that needs a signature returns `4100` ([Section 5.4](../kcc-0012.md#54-watch-only-accounts)).

### What if the wallet has several accounts?

It may authorize more than one ([Section 1](../kcc-0012.md#1-definitions)). `kaspa_accounts` may list addresses from several of them ([Section 6.1](../kcc-0012.md#61-kaspa_requestaccounts)). The balance, the notes, a new receive address, and input selection when `from` is absent all use only the account that contains the first element ([Section 6.5](../kcc-0012.md#65-kaspa_sendtransaction), [Sections 6.14 to 6.16](../kcc-0012.md#614-kaspa_getbalance)). Another authorized account is excluded, even when one of its addresses appears later in the array. The user makes a different account active in the wallet. The app does not pass an account id. One address is never placed in two accounts ([Section 1](../kcc-0012.md#1-definitions)).

### Can a site infer funds it was not shown?

Yes, the amount. Not the location. If `kas` is larger than the sum of the disclosed addresses read from a public node, the difference sits at undisclosed addresses of the same account ([Security Considerations](../kcc-0012.md#security-considerations)). `kaspa_getBalance` does not reveal those addresses, their outpoints, or how many there are. `kaspa_getNotes` lists them after the origin holds that permission ([Section 6.15](../kcc-0012.md#615-kaspa_getnotes)). If `kas` is smaller than that public sum, the cause may be another disclosed account, a stale view, or an output outside the balance set. That inequality is not proof of a second account.

### Does an origin ever receive a key?

No. The wallet decides spendability with keys it holds, and it does not return them ([Section 6.14](../kcc-0012.md#614-kaspa_getbalance), [Section 7.7](../kcc-0012.md#77-signature-scripts), [Security Considerations](../kcc-0012.md#security-considerations)). The origin receives a signature only from a signing method it called and the user approved ([Section 5.3](../kcc-0012.md#53-consent)). The other results are public data: disclosed addresses, amounts, scripts, and the notes the origin was granted. A decoded note may carry a public key or a key hash in `state` or `redeemScript` ([Section 6.15](../kcc-0012.md#615-kaspa_getnotes)). That is public data. [Section 1](../kcc-0012.md#1-definitions) forbids an extended public key, a derivation path, and any value from which an app can derive an address. A result must not contain a private key.

### What about fingerprinting and rate limiting?

`kas` is exact. The wallet must not add noise to it. Two origins the user has authorized can correlate the same total until the funds move, and `kaspa_getBalance` does not prompt, so an authorized origin can re-read it ([Security Considerations](../kcc-0012.md#security-considerations)). A wallet should rate-limit that call per origin, and may rate-limit the others. The code is `-32005`. There is no prompt, and there is no substitute figure ([Section 6.14](../kcc-0012.md#614-kaspa_getbalance), Denial of service). Repeated probes of signing methods should get the same code. `4100` is a locked or unauthorized wallet. `kas` of `"0"` is an authorized empty balance set. Those are different results on purpose.

### How does an app learn that the balance changed?

It calls `kaspa_getBalance` again ([Section 6.14](../kcc-0012.md#614-kaspa_getbalance)). [Section 4.4](../kcc-0012.md#44-events) defines no balance event, and a change of funds must not emit `accountsChanged`. An event with no payload would still tell the origin when funds moved. The wallet should rate-limit the call per origin and reject the excess with `-32005`, and it must not return a substitute figure ([Section 6.14](../kcc-0012.md#614-kaspa_getbalance), Denial of service). The app chooses when to read. The wallet chooses whether to answer.

### Which error codes apply?

| Situation | Code | Where |
| --- | --- | --- |
| The optional method is not implemented | `4200`, before any prompt and before parameter checks | [Section 4.1](../kcc-0012.md#41-request), Sections 6.14 to 6.16 |
| Unauthorized, locked, or not an authorized account | `4100`, and no prompt when `kaspa_accounts` would be empty | Sections 5.4, 6.4, 6.5, and 6.14 to 6.16 |
| The user rejects a prompt | `4001` | [Section 5.3](../kcc-0012.md#53-consent) |
| Bad parameters, including a non-canonical `Uint64` or a wrong address prefix | `-32602` | [Section 7.2](../kcc-0012.md#72-unsigned-decimal-strings-and-amounts), Section 6.5 |
| The selectable outputs cannot cover the payment, or the wallet cannot finish the script | `-32000`, before submit, no `submitted` member on a send | Sections 6.5 and 6.6 |
| The origin is over a wallet limit, including a rate limit | `-32005`, no prompt | [Section 5.3](../kcc-0012.md#53-consent), Sections 6.14 and 6.15 |
| The remote rejects a submitted transaction | `-32003` | Section 6.5, [Section 8](../kcc-0012.md#8-error-codes) |
| Disconnected, and only for the four submit methods | `4900` | [Section 6](../kcc-0012.md#6-rpc-methods) |
| A total does not fit in a `Uint64`, or any other failure | `-32603`, and the wallet must not truncate | Section 6.14 |

`kaspa_signMessage` uses `4100` both for an undisclosed address and for an address of no authorized account, and the message must not say which ([Section 6.4](../kcc-0012.md#64-kaspa_signmessage)). Balance, notes, and a receive address must not return `4900`. The user declining the notes prompt is `4001`, and the permission is not granted ([Section 6.15](../kcc-0012.md#615-kaspa_getnotes)).

### Why are the new methods optional?

The method table marks them Required: No. A wallet that only returns addresses would otherwise fail conformance. `4200` is how an app detects any method that is not required ([Section 4.1](../kcc-0012.md#41-request)). The change rule in [Section 6.5](../kcc-0012.md#65-kaspa_sendtransaction) is what keeps an address-walking total complete when `kaspa_getBalance` is absent. `kaspa_getNotes` is a separate permission so a wallet can serve the total and refuse the cluster ([Section 6.13](../kcc-0012.md#613-permissions), [Section 6.15](../kcc-0012.md#615-kaspa_getnotes)).

### What should an app do when the wallet returns `4200`?

`4200` means that method is not implemented. It is not `4100` and it is not `4001`. For `kaspa_getBalance`, sum the disclosed addresses on a public node and keep the `accountsChanged` listener: fresh change is disclosed, so the sum stays complete ([Section 6.5](../kcc-0012.md#65-kaspa_sendtransaction)). Against a wallet that implements the method, that same sum is only the disclosed subset. For `kaspa_getNotes`, the app has no note list. The total is a different method and may still exist. For `kaspa_requestReceiveAddress`, use a disclosed address, normally the first element ([Section 6.1](../kcc-0012.md#61-kaspa_requestaccounts)).
