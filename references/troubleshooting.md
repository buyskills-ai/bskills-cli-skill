# Troubleshooting bskills-cli + ows errors

When a command errors mid-flow, look up the message here for the cause and
the exact next action. If the message you see isn't listed, fall back to
the human and surface the raw error rather than guessing.

## Settle exit codes — recovery contract

The settle call (`bskills-cli pay <slug> --signature-hex <hex>`) communicates
the recovery path through its **exit code**. This table is authoritative and
matches the settle table in `SKILL.md` and `references/commands.md` exactly —
an agent reading any of the three must take the same action.

| Exit | Outcome | Cache | What to do |
|---|---|---|---|
| `0` | `done` / `owned` | cleared | Success. Surface the Solana Explorer link from `purchase.txSignature` + `purchase.network`. |
| `2` | network error | kept | Transient. Re-run the **same** settle. |
| `3` | `auth` (HTTP 401) | **kept** | Re-authenticate — ask the human to run `bskills-cli login` — then re-run settle with the **same** `--signature-hex`. Do **not** re-initiate, do **not** re-sign. |
| `4` | `timeout` (HTTP 504 — broadcast but not yet confirmed) | **kept** | Re-run the **same** `--signature-hex` to retry confirmation. Do **not** re-initiate, do **not** re-sign. |
| `5` | `reinitiate` (HTTP 400 / expired blockhash / soft-expiry) | **cleared** | Go back to step 2: re-initiate with `--wallet`, re-sign, settle. |

**The exit-4 vs exit-5 distinction is the crux.** Exit `4` keeps the cache and
you retry the **identical** `--signature-hex` (the transaction was broadcast,
just not yet confirmed). Exit `5` clears the cache and you start over from
initiate (the transaction is dead — expired blockhash or soft-expiry). Treating
a `4` as a `5` throws away a transaction that may still confirm; treating a `5`
as a `4` loops forever on a dead one.

## Error messages

| Error / message | Cause | Action |
|---|---|---|
| Auth, exit code `3` / "Not logged in" | No token in `~/.bskills-cli/config.json`. | Ask the human to run `bskills-cli login`. **Never run it silently** — it opens a browser. |
| `Pass exactly one of --wallet <pubkey> (to initiate) or --signature-hex <hex> (to settle).` | Passed both `--wallet` and `--signature-hex`, or neither, to `pay`. | Pass exactly one: `--wallet` to initiate, `--signature-hex` to settle. |
| `--wallet must be a base58 Solana pubkey.` | The `$PAYER` value isn't a base58 pubkey — usually a failed `ows wallet list` parse. | Re-resolve the pubkey from `ows wallet list` (or `bskills-cli doctor --wallet <name>`). |
| `Plugin is not available for purchase` | Plugin status is not `published`. | Tell the user; nothing to do on the agent side. |
| `This plugin is free. Use the /acquire endpoint instead.` | `priceCents === 0`. | Use `bskills-cli acquire <slug>` instead of `pay`. |
| `Plugin creator has not configured Solana payments` | Creator has no `solanaWalletAddress` in their profile. | Cannot pay this plugin today; surface to user. |
| `You have already purchased this plugin` (409 on initiate) | Purchase row exists already (initiate returns `outcome: "owned"`). | Skip to `bskills-cli install <slug>`. |
| ``No pending payment found for "<slug>". Run `bskills-cli pay <id> --wallet <pubkey>` first.`` (settle) | The CLI has no cached unsigned for this plugin (never initiated, or already settled/cleared). | Re-run step 2 of the workflow (`pay --wallet`). |
| ``Cached payment for "<slug>" expired — re-initiate with `--wallet`.`` (exit `5`) | The cached entry soft-expired past the server `expiresAt`. Settle fails locally before hitting the network. | Cache is cleared. Re-run step 2 (`pay --wallet`) for a fresh transaction, then re-sign and settle. |
| ``Cached unsigned transaction has unexpected layout. Re-initiate with `--wallet`.`` | Local state was corrupted or hand-edited; the splice asserts the layout. | Re-run step 2. Don't try to repair the cache. |
| `--signature-hex must be 128 hex chars (64 bytes).` | Signature length wrong; `ows` likely failed silently. | Re-run step 3 and verify `ows` exited cleanly. |
| `Transaction signer does not match the initiating payer` | The OWS wallet you signed with is not the pubkey you initiated for. | Re-run from step 2 with the wallet that owns `$PAYER`. |
| `Transaction does not match the prescribed payment` | Signed bytes differ from what the server prescribed. Almost always: cache reused after the backend rotated the prescription (e.g. you ran `pay --wallet` twice). | Re-run step 2 to get a fresh prescription, then sign that one. |
| Settle exits `4` (timeout, HTTP 504) | Broadcast not yet confirmed on-chain. | **Keep the cache** and re-run step 4 with the **same** `--signature-hex`. |
| Settle exits `5` (reinitiate, HTTP 400) | Expired blockhash / on-chain reject / soft-expiry. | **Cache is cleared.** Re-initiate from step 2 for a fresh transaction. |
| Backend reports broadcast/confirm error (insufficient lamports, no USDC ATA, etc.) | The signed tx reached the backend but Solana rejected it. The `message` field of the settle `--json` carries the reason. | Surface the message to the user; if a funding issue, ask them to top up SOL/USDC, then re-run from step 2. |

## Recovery patterns

**Cache went stale during user confirmation.** Common when OWS policy gating
asks the user to approve out-of-band. If the cache soft-expires (settle exit
`5`, ``Cached payment for "<slug>" expired …``), restart from step 2 — the
unsigned bytes embed a blockhash Solana has since dropped from its
recent-blockhashes window. The settle window is short; stay well under
`expiresAt` between initiate and settle.

**Broadcast confirmed-but-slow vs dead transaction.** A `504`/exit `4` means
the backend broadcast your transaction but Solana hasn't confirmed it yet —
**keep the cache and re-run the same `--signature-hex`.** A `400`/exit `5`
means the transaction can never land (expired/rejected) — **the cache is
cleared; re-initiate.** Never re-sign on an exit `4`, and never retry the same
signature on an exit `5`.

**Wrong wallet signed.** If the user has multiple OWS wallets, the
`Transaction signer does not match the initiating payer` error means the wallet
you resolved `$PAYER` from (step 2, `ows wallet list`) and the wallet you signed
with (step 3, `ows sign tx --wallet …`) differ. Re-run with a consistent
`--wallet` name.

**Re-running step 2 without finishing step 4.** This silently overwrites the
cached unsigned tx for the plugin (the cache holds one pending payment per
plugin). That's fine — you'll just get a fresh prescription. But if you sign the
OLD hex from the first run and try to settle, the splice rejects it because the
cache now holds different bytes (`Transaction does not match the prescribed
payment`). Sign whatever the **latest** `pay --wallet` produced.

**Environment not ready.** Before initiating, run `bskills-cli doctor [--wallet
<name>]` — it checks auth, `ows` install, and a Solana wallet read-only. If a
check fails, follow its `remediation` (e.g. `bskills-cli login`, or
`npm install -g @open-wallet-standard/core`) before touching `pay`.
