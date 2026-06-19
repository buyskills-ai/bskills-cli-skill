# Troubleshooting bskills + ows errors

When a command errors mid-flow, look up the message here for the cause and
the exact next action. If the message you see isn't listed, fall back to
the human and surface the raw error rather than guessing.

## Settle exit codes — recovery contract

Settle (`bskills pay <slug> --signature-hex <hex>`) signals recovery through its
**exit code**:

| Exit | Outcome | Cache | What to do |
|---|---|---|---|
| `0` | `done` / `owned` | cleared | Success. Surface the Solana Explorer link from `purchase.txSignature` + `purchase.network`. |
| `2` | network error | kept | Transient. Re-run the **same** settle. |
| `3` | `auth` (HTTP 401) | **kept** | Re-authenticate — ask the human to run `bskills login` — then re-run settle with the **same** `--signature-hex`. Do **not** re-initiate, do **not** re-sign. |
| `4` | `timeout` (HTTP 504 — broadcast but not yet confirmed) | **kept** | Re-run the **same** `--signature-hex` to retry confirmation. Do **not** re-initiate, do **not** re-sign. |
| `5` | `reinitiate` (HTTP 400 / expired blockhash / soft-expiry) | **cleared** | Go back to step 2: re-initiate with `--wallet`, re-sign, settle. |

**The exit-4 vs exit-5 distinction is the crux.** Exit `4` keeps the cache and
you retry the **identical** `--signature-hex` (the transaction was broadcast,
just not yet confirmed). Exit `5` clears the cache and you start over from
initiate (the transaction is dead — expired blockhash or soft-expiry). Treating
a `4` as a `5` throws away a transaction that may still confirm; treating a `5`
as a `4` loops forever on a dead one.

## Error messages

Rows tagged **(server)** are messages emitted by the BuySkills backend and
surfaced verbatim by the CLI — their exact wording can change server-side, so
branch on the **exit code / `outcome`**, not on the prose. Untagged rows are
produced locally by the CLI and are stable.

| Error / message | Cause | Action |
|---|---|---|
| Auth, exit code `3` / "Not logged in" | No token in `~/.bskills-cli/config.json`. | Ask the human to run `bskills login`. **Never run it silently** — it opens a browser. |
| ``Not logged in. Run `bskills login` before `init --json` …`` (exit `3`) | `bskills init --json` invoked with no valid session; interactive login can't run under `--json`. | Run `bskills login` once interactively, then re-run `init --json` — or run `init` without `--json` to allow the browser flow. |
| `Pass exactly one of --wallet <pubkey> (to initiate) or --signature-hex <hex> (to settle).` | Passed both `--wallet` and `--signature-hex`, or neither, to `pay`. | Pass exactly one: `--wallet` to initiate, `--signature-hex` to settle. |
| `--wallet must be a base58 Solana pubkey.` | The `$PAYER` value isn't a base58 pubkey — usually a failed `ows wallet list` parse. | Re-resolve the pubkey from `ows wallet list` (or `bskills doctor --wallet <name>`). |
| `Plugin is not available for purchase` **(server)** | Plugin status is not `published`. | Tell the user; nothing to do on the agent side. |
| `This plugin is free. Use the /acquire endpoint instead.` **(server)** | `priceCents === 0`. | Use `bskills acquire <slug>` instead of `pay`. |
| `Plugin creator has not configured Solana payments` **(server)** | Creator has no `solanaWalletAddress` in their profile. | Cannot pay this plugin today; surface to user. |
| `You have already purchased this plugin` **(server)** (409 on initiate) | Purchase row exists already (initiate returns `outcome: "owned"`). | Skip to `bskills install <slug>`. |
| ``No pending payment found for "<slug>". Run `bskills pay <id> --wallet <pubkey>` first.`` (settle) | The CLI has no cached unsigned for this plugin (never initiated, or already settled/cleared). | Re-run step 2 of the workflow (`pay --wallet`). |
| ``Cached payment for "<slug>" expired — re-initiate with `--wallet`.`` (exit `5`) | The cached entry soft-expired past the server `expiresAt`. Settle fails locally before hitting the network. | Cache is cleared. Re-run step 2 (`pay --wallet`) for a fresh transaction, then re-sign and settle. |
| ``Cached unsigned transaction has unexpected layout. Re-initiate with `--wallet`.`` | Local state was corrupted or hand-edited; the splice asserts the layout. | Re-run step 2. Don't try to repair the cache. |
| `--signature-hex must be 128 hex chars (64 bytes).` | Signature length wrong; `ows` likely failed silently. | Re-run step 3 and verify `ows` exited cleanly. |
| `Transaction signer does not match the initiating payer` **(server)** | The OWS wallet you signed with is not the pubkey you initiated for. | Re-run from step 2 with the wallet that owns `$PAYER`. |
| `Transaction does not match the prescribed payment` **(server)** | Signed bytes differ from what the server prescribed. Almost always: cache reused after the backend rotated the prescription (e.g. you ran `pay --wallet` twice). | Re-run step 2 to get a fresh prescription, then sign that one. |
| Settle exits `4` (timeout, HTTP 504) | Broadcast not yet confirmed on-chain. | **Keep the cache** and re-run step 4 with the **same** `--signature-hex`. |
| Settle exits `5` (reinitiate, HTTP 400) | Expired blockhash / on-chain reject / soft-expiry. | **Cache is cleared.** Re-initiate from step 2 for a fresh transaction. |
| Backend reports broadcast/confirm error (insufficient lamports, no USDC ATA, etc.) | The signed tx reached the backend but Solana rejected it. The `message` field of the settle `--json` carries the reason. | Surface the message to the user; if a funding issue, ask them to top up SOL/USDC, then re-run from step 2. |
