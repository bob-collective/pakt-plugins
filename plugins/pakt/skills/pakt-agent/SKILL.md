---
name: pakt-agent
description: Use when a user wants to draft, review, activate, disable, inspect, or authorize a trade through the Pakt MCP server. Explain Pakt policies in plain language, preserve human owner-wallet approval, and treat proof refusals as final.
---

# Pakt agent

Pakt lets a user set trading rules, approve the exact policy in a browser with
their user-owned wallet, and ask Pakt to authorize orders that satisfy those rules. Pakt
can return a signed, submission-ready request, but it never submits an order or
moves funds itself.

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
   is non-null. The current policy program supports cross mode on cross-capable
   markets and isolated mode only where the venue requires it; Pakts committed
   to the retained legacy program remain cross-only and must be replaced before
   using a venue-forced isolated market. User-selected isolated mode on a
   cross-capable market is unsupported. Every open position on a cross-capable
   market must use cross mode, even when that market is absent from the order.
   Each cross-capable market named by the order must also have its selected mode
   set to cross. Closing an isolated position removes the account-wide
   open-position blocker but does not reset that market's selected mode. Never
   route around Pakt on the user's behalf, and do not imply that an unsupported
   market can trade. `get_execution_context` identifies each active root's
   policy-program generation. To replace a legacy Pakt, the human permanently
   disables it, waits the five-minute cooldown, and activates a newly drafted Pakt.
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
   signature, wallet action, or execution occurs.
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

Call `prepare_pakt_disable(pakt_root)` only when the user asks to permanently
disable a root. Explain that disable is terminal and requires a separate browser
owner-wallet approval. The activation and disable challenges each last six hours;
preparation alone changes nothing.

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
batch (at most 10 legs) with one 30-second `expiresAfter`. Put risk-reducing
and risk-increasing legs in that same batch: Pakt conservatively assumes every
risk-increasing leg fills and every risk-reducing leg does not when checking the
post-state limits. A separate batch whose every leg is venue-enforced
reduce-only may use the bounded five-minute emergency-exit expiry window; it
must still carry a future deadline, oppose an authenticated position, and stay
within that position's aggregate size. It never becomes an unbounded signature.
The client remains responsible for submission; the call does not submit or move
funds. After the client observes a supported terminal Hyperliquid or Ethereum
outcome, call `record_execution_completion(pakt_root, receipt_id, completion)`
to attach that typed fact to the exact signed receipt. Supported Hyperliquid
outcomes include fill, definite no-fill, cancellation, and expired-unreconciled;
Ethereum includes confirmed success, confirmed revert, and finalized nonce
supersession. The annotation cannot authorize, sign, submit, or change replay
eligibility. Do not treat an unbroadcast ERC-20 approval as generically expired;
it remains usable until its Ethereum nonce is consumed.

On `account_proof_in_progress`, do not submit another concurrent request. Wait
for the active operation to finish. If an earlier response is uncertain, retry
that exact request unchanged so durable replay returns its result; otherwise
refresh `get_execution_context` before building a new proposal. This call
started no additional proof or signature. On refusal, report the named
constraint and stop. On failure, do not resubmit if the response says a
signature was released; report the receipt id.
If the call returns `unsupported_market_metadata`, no mandate verdict or signature
was produced: do not retry unchanged until `list_hyperliquid_perp_markets`
reports that exact asset as supported.

## Account balances

Call `get_portfolio()` for a read-only snapshot of the authenticated
account in independent sections: `execution_wallet` carries the Privy execution
wallet's ETH and USDT on Ethereum mainnet whenever its durable address exists;
`funding_recovery_wallet` (the Hyperliquid master address) and
`hyperliquid_master` (unified USDC balances and current perpetual positions) are
`null` until a Hyperliquid master is linked. Each position carries its venue
`coin` and exact signed `szi` (`> 0` long, `< 0` short). A section that cannot be
read carries its own `error` while the others still answer. Amounts are exact decimal strings.
Addresses are resolved server-side; the call takes no arguments and never
provisions, proves, signs, or submits anything.

Inspect fresh `balances.hyperliquid_master.perpetuals.positions` before deciding
another trade. If unavailable, stop before proposal or submission; this read
does not authorize either.

## Tool index

| Tool | Purpose |
|---|---|
| `list_pakt_templates()` | List shipped starting points; pure |
| `list_hyperliquid_perp_markets(query?, cursor?, page_size?)` | Resolve one live market snapshot; pure |
| `draft_pakt(preset_id)` / `draft_pakt(draft_json)` | Compile one review artifact; pure |
| `prepare_pakt_activation(canonical_ssz)` | Stage browser owner-wallet activation; does not activate |
| `prepare_pakt_disable(pakt_root)` | Stage permanent browser disable; does not disable |
| `list_pakts(cursor?, page_size?)` | List account-scoped lifecycle summaries |
| `get_execution_context(cursor?, page_size?)` | Read revalidated active roots and asset selections |
| `propose_execution(pakt_root, intent)` | Return proof-gated sign-only authorization; never submits |
| `record_execution_completion(pakt_root, receipt_id, completion)` | Record a typed client-observed terminal venue result |
| `get_portfolio()` | Informational balances and deployed Pakts; PnL unavailable until reconciled |

Use the server's current tool schemas for exact wire fields. This skill explains
the workflow and presentation; it grants no capability and cannot widen Pakt's
enforcement.
