---
name: pakt-agent
description: Use when a user wants to draft, review, activate, disable, inspect, or authorize trading, payments, services, or vault moves through the Pakt MCP server or HTTP API. Explain Pakt policies in plain language, preserve human owner-wallet approval, and treat proof refusals as final.
---

# Pakt agent

Pakt lets a user set rules, approve the exact policy in a browser with their
user-owned wallet, and ask Pakt to authorize requests that satisfy those rules.
Pakt can return a signed, submission-ready request, but it never submits an
order or moves funds itself.

## Choose a transport

Pakt exposes the same account-scoped operations and enforcement through two
peer transports. Reuse whichever authenticated transport the agent already has:

- MCP-compatible clients connect to `https://api.usepakt.ai/mcp` and complete
  browser OAuth.
- Code-based agents use the HTTPS API at `https://api.usepakt.ai/api/v1` with
  `Authorization: Bearer <Pakt API key>`. Read the live OpenAPI 3.1 document at
  `https://api.usepakt.ai/api/openapi.json`; resolve operations by `operationId`
  and follow its current method, path, query, and JSON-body schemas.

If an API key is needed, the human opens `https://usepakt.ai/api-keys` with the
intended Pakt account, enters a **Key name**, selects **Create key**, and copies
the secret when it appears; Pakt will not show the full key again. Keep it in the
client's secret store or process environment. Never ask the user to paste it into
a prompt, and never put it in source code, logs, query strings, or requests to
any host other than `api.usepakt.ai`. Deleting the key on that page revokes it
immediately. API keys do not authenticate MCP. MCP tokens do not authenticate API
operations; browser sessions reach only the web app's explicit operation list.

## Non-negotiable behavior

- Never activate or disable a Pakt on the user's behalf. Preparation only stages
  a six-hour browser challenge; the human approves it with their owner wallet.
- Never send, request, generate, or replace a private key or seed.
- Never route around a refusal. Report the rule and the concrete value that
  crossed it. Retry only an infrastructure failure for which no signature was
  released.
- Asset identity is venue-native. Pakt commits the exact chain or venue
  namespace, its native asset identifier, and the oracle identity used to value
  it; there is no second Pakt asset-id namespace. For hosted Hyperliquid, use
  `list_hyperliquid_perp_markets` to resolve live default or HIP-3 names to exact
  mainnet global perp asset ids, and use `get_execution_context` to read the
  exact ids an active Pakt allows. Never retain or invent a symbol-to-id mapping.
- Obey explicit scope such as “draft only” or “do not activate.” Do not end such
  a response with an activation sales pitch.

## Draft and review

1. Browse with `list_pakt_templates()` when the user wants starting points. For
   specifically named Hyperliquid perps, call `list_hyperliquid_perp_markets()`
   once and copy the requested markets' returned `asset_ref` objects into the
   draft; use `query` only when resolving one name without browsing the
   catalogue. An `any` asset selection does not require enumerating the whole
   catalogue. Inspect each `pakt_support` and follow `next_cursor` only when it
   is non-null. The current policy program supports cross and isolated margin,
   and treats isolated margin the same whether the venue requires it or the
   user selected it on a cross-capable market. Each order uses the margin mode
   selected for its market on Hyperliquid, and the Pakt's margin-mode rule
   applies to every order, including reduce-only closes. By
   `policy_program_generation` from `get_execution_context`: `current`,
   `account_leverage_v6`, and `mixed_margin_mode_v5` evaluate isolated orders
   and positions normally;
   `single_margin_mode_v4` allows one margin mode per order and refuses an
   order mixing cross and isolated markets; `cross_only_v3` refuses every order
   while the account holds isolated margin on a market Pakt can trade or the
   order uses isolated margin (`policy_program_upgrade_required`); an isolated
   position on a delisted or other untradable market does not trigger it. A cross position on a market that
   Pakt can trade and Hyperliquid has since made isolated-only blocks every
   order until the user closes it on Hyperliquid; one on a delisted or otherwise
   untradable market does not block. Never route around Pakt on the user's behalf, and
   do not imply that an unsupported market can trade. To replace an older
   Pakt, the human disables it, waits the 30-second cooldown, and activates a
   newly drafted Pakt.
2. Follow the returned `authoring_intake`. Unless the user already supplied the
   answers or explicitly requested unchanged example limits, ask in one concise
   prompt:
   - how much money they intend to allocate to this Pakt; and
   - whether they want cautious, moderate, high, or exact custom risk limits.
   Treat the amount as authoring context, never as a claim about their balance,
   a required deposit, or a maximum portfolio rule.
3. Translate the answers into a visible proposal. State the exact leverage,
   maximum trade size, collateral-outflow cap, liquid-reserve floor, and
   liquidation-health buffer. Relate dollar caps to the intended allocation and
   flag values that could consume most of it. Never invent an allocation or call
   preset values personalized recommendations. If a disclosed allocation cannot
   satisfy an equity floor, revise the proposal or explain the incompatibility.
4. Choose the closest preset as an editable starting point. Compile its
   `preset_id`, edit the returned `source.draft_json`, and compile that edited
   `draft_json`. Present only the final customized root. If the user explicitly
   asks for the unchanged example, compile the preset directly and label its
   limits as examples.
5. Compile exactly one source per `draft_pakt(preset_id)` or
   `draft_pakt(draft_json)`. Both calls are pure: no storage, activation,
   signature, wallet action, or execution occurs. Pass `view: "review"` for an
   ordinary demo; it omits `technical_details` and `dual_readback` but keeps
   the root, `canonical_ssz`, and `pakt.review`. The default `view: "full"`
   returns the advanced audit fields.
6. Present `pakt.review`, not a dump of `dual_readback`. It is already arranged
   for an ordinary user: lead with `headline` and `important_note`, then show each
   returned group's `title` and `summary`. Do not turn it back into a dense table.
7. The grouped summaries explain the effective user rules without exposing
   machine slugs. Do not add opaque asset ids, raw units, or inferred venue
   mappings to them.
8. If the user explicitly asks for an advanced trust audit, explain the two
   labels returned in `pakt.technical_details` without introducing internal jargon:
   - `proof_checked`: the proof applies this rule to signed venue/account data.
   - `signed_input`: Pakt pins the accepted data signer; this is an input source,
     not another user-configurable trading limit.
9. Do not present `pakt.technical_details` in an ordinary demo. Include it only
   when the user explicitly asks for every committed metadata/configuration term
   or an advanced trust audit. Keep raw public keys, numeric scope codes,
   canonical SSZ, and `dual_readback` out unless specifically requested. Do not
   reconstruct those internals from the beginner review.
10. In examples, say “blocked” in ordinary prose rather than exposing internal
   refusal enum names. Do not infer a different boundary from rounded display
   text: “exceeds” means strictly above, so the exact cap remains allowed when
   all other rules pass.
11. State the root once and say plainly that the draft caused no side effects.

A good refusal explanation is: “A $101 trade is blocked because this Pakt allows
at most $100 per trade.” A poor explanation is a raw code or a table of
micro-unit fields with no dollar or percentage translation.

`dual_readback` is still the exhaustive technical audit over the same committed
bytes. Use it to verify completeness or answer a deep technical question, not as
the default response format.

## Browser lifecycle

Call `prepare_pakt_activation(canonical_ssz)` only when the user asks to start
activation. Give the returned dashboard URL to the human. Preparation does not
activate anything. After the human says they approved, confirm with
`list_pakts(cursor?, page_size?)` rather than assuming success.

Call `prepare_pakt_disable(pakt_root)` only when the user asks to disable a
root. Explain that disable stops new authorizations, requires a separate browser
owner-wallet approval, and can be undone only by the owner approving a fresh
activation of the same root with `prepare_pakt_activation`.
The activation and disable challenges each last six hours; preparation alone
changes nothing.

## Hosted authorization

Use `get_execution_context(cursor?, page_size?)` to obtain the authenticated
account's revalidated active roots, adapters, network, and venue-native
Hyperliquid asset selection. `asset_selection` is either `"any"` or an exact
`{"only":[...]}` list of asset indices. Choose exactly one returned root.

Call `propose_execution(pakt_root, intent)` with one exact request accepted by
that root. A prepared Hyperliquid order comes from the existing venue client;
Pakt validates it rather than planning or repairing it, then synchronously
returns the signed, refused, or recovery-required result.
When a rebalance needs several IOC orders, send them as one Hyperliquid order
batch (at most 10 legs). Put risk-reducing
and risk-increasing legs in that same batch: Pakt conservatively assumes every
risk-increasing leg fills and every risk-reducing leg does not when checking the
post-state limits.
For Hyperliquid orders, one rule governs retries: a nonce together with the
order legs identifies one proposal, and the deadline is not part of it. If the
outcome is unknown (lost
response, timeout, internal error, or `account_proof_in_progress`), resend the
same order unchanged: Pakt returns the stored result, including its original
deadline, or decides it once. If the outcome is known and you want another
attempt (refused, `invalid_argument`, or the signed order expired), use a fresh
nonce: the current Unix time in milliseconds. A refused nonce returns the same
refusal. After `recovery_required`, do not resend or replace the order until it
is reconciled. Hyperliquid rejects nonces more than two days old or one day
ahead, and executes at most one action per nonce. Omit `expires_after_ms`: Pakt
sets the venue deadline to the latest instant the selected root allows, counted
from its own attested decision time, so time spent preparing the call does not
consume the window. Submit a signed result promptly; its `expiresAfter` is the
deadline Pakt chose. A
separate batch whose every leg is venue-enforced reduce-only may instead supply
its own `expires_after_ms` within the bounded five-minute emergency-exit window;
it must be an absolute future Unix-millisecond timestamp, oppose an
authenticated position, and stay within that position's aggregate size. It
never becomes an unbounded signature.
The client remains responsible for submission; the call does not submit or move
funds. After the client observes a supported terminal Hyperliquid or Ethereum
outcome, call `record_execution_completion(pakt_root, receipt_id, completion)`
to attach that typed fact to the exact signed receipt. Supported Hyperliquid
outcomes include fill, definite no-fill, cancellation, and expired-unreconciled;
Ethereum includes confirmed success, confirmed revert, and finalized nonce
supersession. The annotation cannot authorize, sign, submit, or change replay
eligibility. Do not treat an unbroadcast ERC-20 approval as generically expired;
it remains usable until its Ethereum nonce is consumed.

On `account_proof_in_progress`, do not submit another concurrent request; wait
for the active operation to finish, then retry by the rule for that kind of
request: the nonce rule above for a Hyperliquid order, the same `operation_id`
for a yield step. On refusal or `invalid_argument`, report the reason and stop.
If the response says a signature was released or recovery is required, do not
resubmit; report the receipt id. For a yield step whose outcome is unknown,
resend the exact original step with the same `operation_id`, never a new nonce
or ID; a new `operation_id` is only for a genuinely new permitted step, not for
a retry or a workaround (the one exception, a later attempt after an
`Unavailable` refusal, is under Vault moves).
On `wallet_nonce_held` (HTTP 409), nothing was signed or recorded: Pakt already
signed another transaction from the same wallet at that nonce or a later one.
`blocked_by {receipt_id, venue}` names that operation when Pakt knows it. If it
is a transaction you were given but did not broadcast, broadcast it and record
its completion; wait until it is mined, then resend. A transfer, Permit2
approval or yield step is resent unchanged with the same `operation_id`; a
prepared Gateway transaction is rebuilt on the wallet's current nonce.
If the call returns `unsupported_market_metadata`, no mandate verdict or signature
was produced: do not retry unchanged until `list_hyperliquid_perp_markets`
reports that exact asset as supported.
If the call returns `unsupported_margin_mode`, no mandate verdict or signature
was produced: the named asset has a cross margin setting (its selected margin
mode or an open position's margin mode) on a market that
`list_hyperliquid_perp_markets` reports as `venue_margin_mode` `isolated_only`,
so Pakt cannot evaluate orders on this account. Ask the user to close that cross
position on Hyperliquid (or set the flat market to isolated), then retry with a
fresh nonce. Isolated margin on a `cross_capable` market is supported and does
not cause this refusal.

### Paying for a service (x402 / MPP)

When a merchant answers HTTP 402 and the selected root is a services Pakt with a
`service-account` rule, call `propose_execution` with the `service_request`
intent and add `payment_challenge`: `{"protocol":"x402","payment_required":
<PAYMENT-REQUIRED header or 402 JSON body>}` or `{"protocol":"mpp",
"www_authenticate":<WWW-Authenticate value>}`, unchanged. Set
`max_charge_usd_micro` to the challenge amount. Supported: USDC on Base,
Arbitrum One and Ethereum; USDT0 on Arbitrum One; USDT on Ethereum through x402
Permit2 only (no MPP); USDC.e on Tempo through MPP `tempo` (below). Send the
whole challenge: Pakt pays the first offered option the Pakt's rules admit, in
the merchant's order (`pay_with: {chain_id, token}` narrows the options). A
paying services Pakt always needs the challenge; never pay the merchant
yourself, or the request is paid twice.
An allow returns `SIGNED_NOT_SUBMITTED` with `submission.header_name` and
`submission.header_value`: retry the exact original request with that header
and the same `operation_id` as the provider idempotency key. Pakt never
contacts the merchant. If the outcome is unknown, resend the same request and
challenge unchanged: Pakt returns the recorded outcome and never pays twice.
`invalid_argument` naming the payment window, the payable options or a missing
challenge recorded nothing: fix the request and resend. A refusal (including a
`signer-refused` one from the enclave) is recorded under that `operation_id`;
make any new attempt with a fresh challenge and a new `operation_id`. After the
merchant settles, call `record_execution_completion` with `service_reported`
and the settlement transaction hash from its payment receipt. To draft a paying
services Pakt, add `service-account` (the account's payments wallet address,
lowercase hex, from `get_portfolio().balances.payments_wallet.address`),
`payment-assets` and `payment-payees`. `draft_pakt` adds `state-source-config`
(state source 3, the deployed service-payment attestor key) automatically and
refuses a different key, so leave it out.
Tokens must be the pinned ones, with their exact EIP-712 `name`/`version`
(`"USD Coin"`/`"2"` for USDC, `"USD₮0"`/`"1"` for USDT0, `""`/`""` for USDT);
`draft_pakt` names the pinned values when one is wrong.

To pay USDT through Permit2, the service account must allow `x402-permit2` and
the Pakt needs `service-network-fees` (a per-chain cap in wei on the gas the
payments wallet may spend on its own transactions), with a cap on every chain
that lists a token without an EIP-712 domain. Before the first Permit2
payment on a token, call `propose_execution` with the `service_permit2_approval`
intent: `{"operation_id", "chain_id", "token", "amount": "max"}`. The spender
is always Permit2 and Pakt sets the nonce and fees. Broadcast the returned
`submission.raw_transaction` unchanged; the payments wallet needs a little of the
chain's native token for gas. Until it is mined, a Permit2 payment is refused
before any proof with an invalid-argument error naming the too-low allowance;
nothing is recorded, so resend the same request once the approval is mined.
A nonzero approval is accepted only while the allowance is zero and
`"amount": "0"` only while it is nonzero. If the cap cannot pay the current
base fee with headroom, the approval is refused before any proof and nothing is
recorded; retry later. Resending the same `operation_id`, even after the Pakt
is replaced, returns the same transaction or refusal (including a
`signer-refused` one); use a new `operation_id` for a different approval.

### Paying on Tempo (MPP `tempo`)

The `service-account` must enable `mpp-tempo-charge` and/or `mpp-tempo-session`,
the Pakt pins USDC.e on Tempo (`0x20c000000000000000000000b9537d11c60e8b50`,
`"Bridged USDC (Stargate)"`/`"1"`), and `service-network-fees` needs a cap for
chain 4217. On Tempo the cap is in attodollars (`10^18` per USD):
`"50000000000000000"` caps each transaction's fee at $0.05, paid in USDC.e.

- **Charge.** Pass the `WWW-Authenticate` value as `{"protocol":"mpp",...}`.
  Pakt pays the first `evm` charge it finds, otherwise the first `tempo`
  charge; add `"pay_with":{"chain_id":4217,"token":"0x20c0…8b50"}` to choose the
  Tempo option. Retry the request with `submission.header_name`/`header_value`;
  the merchant broadcasts the transaction.
- **Session.** Generate a session key yourself and keep it. Add
  `"tempo_session":{"amount_atomic":<deposit>,"session_key":<its 0x address>}`
  and set `max_charge_usd_micro` to the deposit. The result's
  `credential_json_template` has two placeholders: sign the EIP-712 voucher in
  `submission.voucher` (`{channelId, cumulativeAmount}`) with the session key,
  replace the placeholders with the cumulative amount and the signature, and send
  `Authorization: Payment <base64url(JSON), no padding>`. Keep
  `submission.channel`: later vouchers and the cooperative close are yours to
  sign with the session key and never go through Pakt. To add funds, send
  `tempo_session` again with `amount_atomic` and `top_up:{salt,
  expiring_nonce_hash}` from that channel; the result is a ready header.
- **Direct transfer or payer-side close.** Use the `tempo_operation` intent
  (`transfer` to a listed payee, `request_close`, then `withdraw` once the
  900 s grace period has passed; an earlier `withdraw` is refused) and
  broadcast `submission.transaction.raw_transaction` yourself before its
  `valid_before`. Any Pakt that enables a Tempo method may close and withdraw
  the payments wallet's channels. Its `operation_id` shares the account-wide
  namespace with service requests, Permit2 approvals and transfers: resending
  it returns the same transaction, even after the Pakt is replaced.

A challenge with `methodDetails.feePayer: true` is refused (`invalid_argument`):
Pakt never signs sponsored Tempo transactions. A challenge with its own
`methodDetails.memo` is refused too: Pakt binds a charge to the challenge's
attribution memo only. Each signed
transaction uses the payments wallet's next nonce, so a second payment signed
before the first lands will fail on chain; wait for one to settle, or for its
`valid_before`, before retrying with a new `operation_id`.

Budgets and recurrences retain a Tempo payment as pending until a safe block
shows its account nonce consumed, or safe chain time reaches its `valid_before`.
Latest inclusion alone does not release the entry; a used payment counts once
from transfer history while awaiting safe clearance. A Pakt whose budgets or
recurrences cover a Tempo token cannot
enable `mpp-tempo-session` (`SessionUnderSpendLimit`): a channel deposit sits in
the reserve, where the history cannot keep counting it against its payee.

### Paying from the owner's master (payments Pakt)

A payments Pakt lets you send USDC, USDT0 or USDT from the owner's master wallet
to listed payees on Base (8453), Arbitrum (42161) and Ethereum (1). The master
approves the payments wallet with `approve(payments wallet, amount)` for a
bounded amount the owner chooses (the dashboard's payments setup builds it). The
Pakt limits what you send; the owner's own login to the payments wallet can also
spend the allowance, which is why it stays bounded. Read
`get_execution_context().active_pakts[].payments_policy` for the
chains and masters, tokens and payees (with labels) before you propose.

- **Pay:** `propose_execution` with `{"kind":"transfer","transfer":{"operation_id":
  "<invoice id>","chain_id":8453,"token":"0x…","payee":"0x…","amount":"1000000"}}`.
  Amounts are atomic units as a decimal string (6 decimals: `"1000000"` is 1).
  Use `payee_label` instead of `payee` to name a listed payee by its exact label.
- **Refill the service account:** `"payee_label":"refill"` sends from the master
  to the payments wallet, which then pays x402 and MPP charges.
- **Return to the master:** `"return_to_master":true` sends the payments
  wallet's own balance of the token back to the bound master. It is always
  allowed and does not use the allowance.

Never supply a nonce, fees, gas or calldata: Pakt builds the transaction. An
allow returns `SIGNED_NOT_SUBMITTED` with `submission.raw_transaction` and
`submission.transaction_hash`. Broadcast it yourself with
`eth_sendRawTransaction` on that chain, and wait until it is mined before the
next transfer on the same chain (each transfer uses the next nonce; a second
request before then is answered `wallet_nonce_held`, and nothing is recorded).
Then call
`record_execution_completion` with `ethereum_transaction_confirmed_success` (or
`…_revert`), the transaction hash and the response's `deciding_pakt_root` (the
Pakt that decided the transfer, also after a replacement). If the outcome is unknown, resend the same
request unchanged: Pakt returns the same signed transaction (or refusal) and
never signs a second one for that `operation_id`, even after the owner replaces
the payments Pakt. The same `operation_id` with other terms is refused with
`invalid_argument` naming the original receipt; use a new `operation_id` for a
new transfer, distinct from the account's service request and Permit2 approval
operation_ids. `invalid_argument` naming the allowance or balance means
Pakt refused before proving: ask the owner to raise the allowance (or fund the
master) and resend. The payments wallet pays network fees in ETH, so it needs
ETH on each chain.

To draft a payments Pakt, add `payments-wallet` (`agent_wallet`: the payments
wallet address, lowercase hex, from `get_portfolio().balances.payments_wallet.address`;
and per chain the owner's `master`, `gas_limit` of at least 80,000 and
`max_fee_per_gas_wei`), `payment-assets` (pinned stablecoins only),
optionally `payment-payees` (listed payees only:
`unlisted_max_per_payment_atomic` is `"0"`, no `service`, and labels are
names, never addresses), all at venue `payments`, instance 1, and
`state-source-config` (state source 4, pubkey
`033b87504b90819970ef2ca34d02f7a337d4a42b999e5294e4452eacbb34605925`, the
staging payments attestor, which services Pakts also pin) at venue `config`.
Pakt adds the payments wallet as the `refill` payee, tagged `refill`, on every
bound chain.

### Vault moves (yield)

A yield Pakt lets the agent move one stablecoin on one chain between the owner's
listed ERC-4626 vaults. The agent wallet holds the stablecoin and shares; this is
the account's own yield wallet, separate from the payments wallet, the owner's
trading/Pakt signer and the payments master's balance. It needs no ETH: each
step is one ERC-4337 user operation whose network fee is paid in USDC through
Circle's paymaster, at most the Pakt's per-move fee ceiling.
Pakt permits bounded approvals and deposits, and withdrawals or redemptions back
to that same wallet. It never submits a user operation or promises a return.

**Draft.** Ask how much the owner wants to put to work and how careful they want
to be, unless they already said. Propose limits in dollars: at most how much per
move, in one vault, in a group of vaults that count as one exposure, and in all
vaults together; also how much stablecoin to keep ready in the wallet. Show the
chain, token, listed vaults, their families and groups, and maximum network fee.
Groups are fixed when the owner approves, not a live look-through of allocations.
Exposure is read at the attested block, not guaranteed against later price
changes or a vault reporting false values. There is no rolling budget.

Find candidate vaults with `list_yield_vaults` (Base USDC; only for accounts with
Pakt Yield on). `eligible` marks the vaults the Pakt app offers; copy a vault's
`address` and `family` into the draft, and call it with `vault` for that vault's
risk report. The list is informational: its rates and ratings never authorize,
and a `YIELD_VAULT_CATALOGUE_BUSY` answer means retry after `retry_after_secs`.

Offer the optional health checks in plain words:

- Before a deposit, simulate that deposit followed by withdrawing the existing
  position plus the new money, at the chosen minimum return and within the gas
  limit. A 9990 bps minimum asks for 99.9%; this does not reserve liquidity.
- Compare the share price with a chosen earlier window and refuse a larger fall
  or faster rise than accepted. The example allows no fall and at most 1% growth
  over seven days, not a 1% annual yield.
- For Morpho vaults, stop new money on warning signs: a funded market with its
  cap cut to zero or removal pending, near-full borrowing, or reported lost
  assets. A stable share price alone does not prove the vault is healthy.
- Limit the Morpho performance fee, the share of earnings the vault takes.
  The example's 2500 bps is 25%, not a fee on the deposit and not an all-in
  ceiling on management or underlying-market fees.

These checks gate nonzero approvals and deposits, never withdrawals or
redemptions. Only deposits request the simulation. If an enabled check cannot
be read, new money is refused; missing price or health facts never block an
otherwise valid exit. Omit `yield-health` to disable all its checks. Within it,
zero disables the round trip or price window, `warning_signs: false` disables
warnings, and `max_performance_fee_bps: 10000` removes the fee ceiling. A zero
fee ceiling requires no performance fee. Zero `min_idle_atomic` disables the
idle reserve; group and total limits must be nonzero.

Use the account's `get_portfolio().balances.yield_wallet.address` for
`agent_wallet`. Compile this draft shape with `draft_pakt(draft_json)` after
replacing the example limits and vault addresses with the owner's choices.
Stablecoin amounts are decimal atomic-unit strings with six decimals:
`"500000000"` is $500. List each vault once, in ascending address-byte order; if
`draft_pakt` refuses the list, it names the most vaults a Pakt may list.
Pakt pins its deployed yield attestor automatically when `state-source-config`
is omitted.

```json
{ "version": 1, "valid_until": 18446744073709551615, "constraints": [
  { "kind": "yield-account", "venue": "yield", "instance": 1, "params": {
      "agent_wallet": "0x…", "chain_id": 8453,
      "max_fee_per_step_atomic": "500000" } },
  { "kind": "yield-vaults", "venue": "yield", "instance": 1, "params": {
      "token": "0x833589fcd6edb6e08f4c7c32d4f71b54bda02913",
      "max_per_move_atomic": "300000000", "max_per_vault_atomic": "600000000",
      "max_per_group_atomic": "600000000", "max_total_atomic": "1000000000",
      "min_idle_atomic": "200000000",
      "vaults": [ { "address": "0x…", "family": "morpho-v1", "group": 0 },
                  { "address": "0x…", "family": "morpho-v2", "group": 1 },
                  { "address": "0x…", "family": "erc4626", "group": 2 } ] } },
  { "kind": "yield-health", "venue": "yield", "instance": 1, "params": {
      "round_trip_min_bps": 9990, "price_window_secs": 604800,
      "price_max_drop_bps": 0, "price_max_growth_bps": 100,
      "warning_signs": true, "max_performance_fee_bps": 2500 } } ] }
```

Keep `"valid_until": 18446744073709551615`: the Pakt never expires, as the app
drafts it. A yield Pakt with an earlier `valid_until` stops every step once that
time has passed, withdrawals, redemptions and allowance revocations included, so
money still in a vault stays there until the owner approves guardrails that list
it again. Set one only when the owner wants exits to stop too.

Present the returned review in plain words. When asked to activate, call
`prepare_pakt_activation(canonical_ssz)` and give the owner its browser link.
Only the owner approves the staged root with a visible wallet prompt; a yield
Pakt is never a silent default.

**Act.** Read `get_execution_context().active_pakts[].yield_policy` for the exact
wallet, chain, token, vault objects (address, family and group), all five limits,
optional health checks and the USDC fee ceiling. Pakt does not build steps. Encode
the accepted call yourself, or use a builder the owner chose; set up its
credentials outside the chat and never ask for them in it. Request the step for
the Pakt's wallet address. From a builder's answer take only `to`, `data` and
`value`; Pakt supplies the nonce, gas and fees. The builder is untrusted: a
quote or displayed rate does not authorize its calldata.

Distinguish a first allocation from switching an existing position. Before
choosing a vault, establish the owner's comparison priorities and what risk
information they accept; the listed vaults are permission, not a ranking. For a
switch, also establish the owner's minimum net advantage in delivered yield and
how long it must have lasted. Compare what each vault lends against and the
estimated network cost of leaving and entering, and say what cannot be
established instead of substituting the displayed rate or a third-party rating.
Missing risk data is not a clean bill of health, and different vault names need
not mean different exposures. Do not churn between near ties.

To measure a lasting advantage, use a named return source and an agreed elapsed
time. If you derive share-price growth on chain, find the earlier block by its
timestamp, verify the elapsed time, and compare the same raw amount of shares at
both blocks. Read the vault's share decimals; never infer them from its family.
Share-price growth leaves out external rewards, so it is not by itself delivered
yield. If a builder's step is not one of the accepted calls, Pakt refuses it; do
not propose that vault.
Propose each step in order:

```json
{ "kind": "yield_step",
  "step": { "operation_id": "vault-deposit-1", "chain_id": 8453,
            "to": "0x…", "data": "0x…", "value": "0" } }
```

Use a fresh account-wide `operation_id` for each new step, distinct from service,
payments and Permit2 operation IDs. `value` may be omitted for zero. Never
supply nonce, gas or fees. If the answer is `yield_account_setup_required`
(HTTP 409, nothing recorded), the yield account first needs the owner's
one-time smart-account delegation and/or a USDC fee permit, as its
`delegation` and `fee_permit` name: ask the owner to open the Pakt app, which
signs them, and resend the same step with them as `step.account_setup`. On
`yield_paymaster_unavailable`, retry later. On `SIGNED_NOT_SUBMITTED`, send
`submission.user_operation` unchanged (adding `submission.eip7702_auth` as
`eip7702Auth` when it is not null) with `eth_sendUserOperation` to an ERC-4337
bundler for `submission.entry_point` on `submission.chain_id`. Wait until its
`UserOperationEvent` lands, then call `record_execution_completion` with
`submission.user_op_hash` as `tx_hash`,
`ethereum_transaction_confirmed_success` or `ethereum_transaction_confirmed_revert`
as the event reports, the `receipt_id` and the response's `deciding_pakt_root`.
Only then propose the next step: an approval must land before the deposit can
use its allowance. The yield account has its own EntryPoint sequence, so
payments never wait on a yield step. A signed user operation has no deadline
and its vault checks held only when it was signed, so send it at once. The
owner can turn the Pakt off at any time, signed steps or not, and the owner's
app never sends a deposit or nonzero approval signed more than a few seconds
earlier: it lets the owner cancel it instead, with an empty user operation at
its sequence, and records it as superseded.
If the response is uncertain, resend the exact original step unchanged with
the same ID, including after a Pakt replacement; never invent a second step to
recover a lost signature.

**Always refused.** Unlisted vaults or another token or chain; native ETH;
unbounded or over-limit approvals and deposits; shares or withdrawals sent
anywhere but the agent wallet; `mint`, multicalls, batches and non-ERC-4626
calls; extra or malformed calldata; caller-selected nonce, gas or fees.
Withdraw and redeem have no deposit cap, but must use the listed vault and the
wallet as both owner and receiver and cannot exceed its balance. `withdraw`
takes the stablecoin's atomic units; `redeem` takes raw vault-share units, whose
decimals may differ. To withdraw everything, read `balanceOf(wallet)` on every
listed vault and redeem that exact share count from each, because a rounded
asset amount can leave dust. `get_portfolio()` does not report vault shares.
Permission to exit is not a promise that the vault has cash when the call lands.

Explain the proved yield refusals in plain words:

| Refusal | Meaning |
|---|---|
| `YieldCallNotAllowed` | The step is not one of the accepted calls on the Pakt's chain. |
| `YieldVaultNotAllowed` | The vault is not on the Pakt's list, or it does not hold the Pakt's stablecoin. |
| `YieldReceiverNotWallet` | The step would send shares or money to anyone but the agent wallet. |
| `YieldMoveExceeded` | An approval or deposit is above the per-move limit. |
| `YieldVaultCapExceeded` | The vault would hold more than the per-vault limit after the deposit. |
| `YieldGroupExceeded` | Vaults counted as one exposure would exceed the group's limit. |
| `YieldTotalExceeded` | All vaults together would exceed the total limit. |
| `YieldReserveBreached` | The wallet would keep less stablecoin than the amount to keep ready. |
| `YieldHealthUnavailable` | The Morpho market checks could not be read, so this approval or deposit was not signed. |
| `YieldFeeTooHigh` | The vault takes a larger share of earnings than accepted. |
| `YieldVaultFlagged` | The vault shows an on-chain warning sign. |
| `YieldPriceHistoryUnavailable` | The earlier share price could not be read, so this approval or deposit was not signed. |
| `YieldPriceAnomaly` | The share price fell or rose faster than accepted. |
| `YieldExitUnavailable` | A simulated deposit followed by a withdrawal failed, returned too little, or exceeded the gas limit, so the deposit was not signed. |

If a request plainly breaks the Pakt's visible rules (an unlisted vault, an
approval or deposit above the per-move limit, a receiver or owner other than the
wallet), decline before proposing: name the limit or the unlisted destination
and say that nothing was proposed. That is not a recorded Pakt refusal. If Pakt
returns `REFUSED_NOT_SIGNED`, report its receipt and reason and stop: the
refusal is final for that `operation_id`. Never split a move or change vaults,
identifiers or Pakts to get around it; only the two `Unavailable` refusals
below allow a later attempt.

`YieldHealthUnavailable` and `YieldPriceHistoryUnavailable` mean a fact could
not be read, not that a check failed; do not call the vault flagged. The step
stays refused for its `operation_id`. Tell the owner, and you may propose the
same action again later as a new step with a fresh `operation_id`.

An `invalid_argument` naming insufficient balance, allowance or excessive network
fees is an unrecorded pre-proof rejection: resend unchanged only after that fact
changes. A wallet nonce another accepted operation still holds is answered
`wallet_nonce_held` (see above), also unrecorded; resend the unchanged step once
the holder is mined. On
recovery-required or any response saying authorization was released, stop and
report the receipt instead of re-proposing.

### Budgets and payee limits

A paying services Pakt or a payments Pakt may add `payment-budgets` (up to 16
budgets: a `group` of `{chain_id, token}` from `payment-assets`, a `scope` of
`{"kind":"all"}`, `{"kind":"tag","tag":…}` or `{"kind":"payee","chain_id":…,
"address":…}`, a `window` of `{"kind":"trailing","secs":…}` (3600 to 2678400)
or `{"kind":"calendar","period":"day"|"week"|"month"}` in UTC, `cap_atomic`
and optional `max_count`) and `payee-recurrence` (per listed payee:
`max_amount_atomic`, `period`, `max_payments`). In a payments Pakt every
group's tokens are on one chain. Payees take optional `tags` for tag scopes.
In a services Pakt, `returns_to` lists the owner's master per chain so a
return to it does not count; it may not be a listed payee. Explain the selected
scope: an all-outflow budget counts every transfer from the master (including
the owner's own), or from the payments wallet except an eligible plain return;
a payee or tag scope counts only payments to those payees. For an account using
Yield, propose tagging every intended service payee `services` and scoping the
service budget to that tag so vault deposits do not consume it; preserve the
owner's review and approval, never silently rewrite their active spending Pakt.

The pending list holds at most 16 outstanding entries across the Pakt, not 16
payments per day or over its lifetime. Entries leave it when the pinned
attestor's RPC-reported `safe` head establishes use, nonce consumption or
expiry. For unused EIP-3009 and Permit2 authorizations, safe chain time and
attested authorization status establish expiry; wall-clock expiry is not
enough. Missing safe evidence fails closed: there is no `latest` or
fixed-confirmation fallback. On Base on 2026-11-24, measured head ages were
latest 0 s, safe 19 s and finalized 1,096 s; these are observations, not a
clearance-time guarantee.

Explain the separate safe-head reorg risk when reviewing these limits. Cleared
entries are not retained. A reorg below safe can remove transfers between
finalized and safe (1,077 s, about 18 minutes in that Base observation) and
revive still-valid forgotten authorizations. An expired-unused authorization
revives only if chain time rolls back before its expiry. The amount at risk is
the sum of affected forgotten authorizations, each within its per-payment cap;
repeated clearance cycles can forget more than 16. Window and count limits
constrain each signing decision but do not generally bound that forgotten sum
or count: repeated unused-expiry clearances can recycle headroom even within
one window, and windows can roll over. Removed history also lets new signatures
spend headroom again.
Reincluded transfers count again in canonical history but do not revoke those
new signatures. The separate enclave-restart bound remains 16 live pending
authorizations, and direct transfers' sequential-nonce protection is unchanged.

These facts and pending entries remain in the existing proved evidence/state
path, not a new store. Safe semantics use new Services/Payments programs and
v3 signed spend-bearing schemas; request-only Services remains v2. Gateway
has its own new program, which authorizes USDC offramps, bounds their swap
independently of the price feeds and refuses any order while a stablecoin
price is more than 2% from one dollar (`GatewayPegOutOfBand`): a Gateway Pakt approved before it refuses with `policy_program_upgrade_required` until the owner turns it off and
approves freshly drafted rules that list each input token (Ethereum USDT or
USDC, as its contract with its Chainlink dollar feed) next to the Bitcoin
entry. `list_gateway_routes` returns each supported token's entry as
`pakt_support.allowed_asset`, ready to copy into the draft. After deployment, old active spending/payments roots refuse
payments until the owner turns off the old set and approves freshly authored
rules. Their exact old executable and decoder remain for historical evidence
only. Explain this cutover and the trust change, then use the existing
owner-approved browser replacement flow. The app replaces only the sign-up
spending defaults on the retired program, at the owner's next sign-in; never
silently upgrade or approve anything for the user.
Request-only Your model roots on the previous Services program keep working;
nothing needs re-approval for them.

A proved refusal is final for that exact request:
- `BudgetExceeded` or `RecurrenceExceeded`: never retry the same payment
  blindly, split it into smaller ones, or switch to another token, payee or Pakt
  to get around the limit. Tell the user which budget or payee limit was
  reached. Headroom can free as older transfers leave a trailing window, the UTC
  calendar period rolls over, unused pending authorizations expire at safe, or
  the owner approves a larger limit. Do not promise that a new request passes.
- `SpendHistoryIncomplete`: the account's transfer history was over the
  attestor's log cap (or read too late); nothing was paid. Do not retry in a
  loop; tell the user, who may approve a shorter window or a different budget.
- `SpendHistoryUnavailable`: the attested history did not match the Pakt;
  report it as an infrastructure problem rather than a spending decision.
- `PolicyStateCapacityExceeded`: 16 outstanding payments these limits count
  still await safe-head clearance of use, nonce consumption or expiry. Submit
  an unsubmitted original only while valid, then wait for safe clearance before
  another payment. Already-cleared payments do not occupy the list. Do not
  duplicate a payment or treat a missing safe head as permission to bypass Pakt.

## Account balances

Call `get_portfolio()` for a read-only snapshot of the authenticated
account in independent sections: `execution_wallet` carries the Privy execution
wallet's ETH and USDT on Ethereum mainnet whenever its durable address exists;
`funding_recovery_wallet` (the Hyperliquid master address) and
`hyperliquid_master` (unified USDC balances and current perpetual positions) are
`null` until a Hyperliquid master is linked. Each position carries its venue
`coin` and exact signed `szi` (`> 0` long, `< 0` short). A section that cannot be
read carries its own `error` while the others still answer. Amounts are exact decimal strings.
`payments_wallet` carries the payments wallet's Base USDC (`null` until the
owner creates one in the dashboard); it is informational, and payments are
checked against the attested balance. `yield_wallet` carries the yield wallet's
Base USDC and `eth_wei` for gas (`null` until the owner creates one).
Addresses are resolved server-side; the call takes no arguments and never
provisions, proves, signs, or submits anything.

Before another Hyperliquid perpetual trade, inspect fresh
`balances.hyperliquid_master.perpetuals.positions`; if unavailable, stop that
Hyperliquid flow. This read does not authorize a proposal or submission. A yield
move does not depend on it: use the yield policy's wallet and token with current
chain balances and listed-vault share balances. An account without a Hyperliquid
master must not be blocked from a yield move.

## Operation index

| Operation | Purpose |
|---|---|
| `list_pakt_templates()` | List shipped starting points; pure |
| `list_hyperliquid_perp_markets(query?, cursor?, page_size?)` | Resolve one live market snapshot; pure |
| `draft_pakt(preset_id)` / `draft_pakt(draft_json)` (`view?`) | Compile one review artifact; pure |
| `prepare_pakt_activation(canonical_ssz)` | Stage browser owner-wallet activation; does not activate |
| `prepare_pakt_disable(pakt_root)` | Stage browser disable; does not disable |
| `list_pakts(cursor?, page_size?)` | List account-scoped lifecycle summaries |
| `get_execution_context(cursor?, page_size?)` | Read revalidated active roots and asset selections |
| `propose_execution(pakt_root, intent)` | Authorize a strict intent, including `yield_step` for vault moves; never submits |
| `record_execution_completion(pakt_root, receipt_id, completion)` | Record a typed client-observed terminal venue result |
| `get_portfolio()` | Informational balances and deployed Pakts; PnL unavailable until reconciled |

Use the connected server's current MCP schemas or live OpenAPI document for exact
wire fields. This skill explains the workflow and presentation; it grants no
capability and cannot widen Pakt's enforcement.
