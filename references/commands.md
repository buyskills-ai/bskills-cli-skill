# bskills command reference

Full flag list and behavioral nuance for each command. Consult this when
SKILL.md's command summary isn't enough — for example, when you need to
know default values, what a flag actually controls, or which exit codes
mean what.

All commands exit with:
- `0` — success.
- `1` — user/validation error.
- `2` — network error.
- `3` — not logged in (run `bskills login` first).

`pay --signature-hex` (settle) additionally uses exit codes `4` and `5` — see
`references/troubleshooting.md` for the settle recovery contract.

Every read/list command accepts `--json` and prints a structured payload on
stdout. Prefer `--json` when you need to parse results programmatically.

## Auth

```bash
bskills login                       # opens a browser for GitHub OAuth, persists a token
bskills login --timeout <seconds>   # bound the browser-callback wait (default 300)
bskills logout                      # clears the stored token
bskills whoami                      # shows the current user
```

Browser-only — there is no headless / API-key flag. Log in interactively once;
the persisted token is reused afterward.

Auth state lives in `~/.bskills-cli/config.json`. If the user has no token,
`whoami`, `acquire`, `pay`, `install`, and `installed --remote` will fail
with exit code `3`. Suggest `bskills login` in that case — never run it
silently.

## Init (one-shot bootstrap)

```bash
bskills init
  --slug <slug>                # default: buyskills-ai-bskills-cli-skill
  -a, --agent <id>             # repeatable; default: all detected agents
  -s, --scope <global|project> # default: config defaultScope (factory: global)
  -m, --mode <copy|symlink>    # default: config defaultInstallMode (factory: copy)
  --timeout <seconds>          # browser-login timeout; default 300
  --json
```

Free-skill bootstrap in one shot: ensure a session (browser login if none),
acquire `--slug` if unowned, then install to detected agents.

- Treat it like `login` — it may open a browser, so don't fire it autonomously.
- `--json` can't run interactive login: a missing session is a hard error
  (exit `3`). Report shape in `references/output-schemas.md`.
- Exit `1` if the slug is paid, no agents are detected, or all installs fail.

## Doctor (read-only preflight)

```bash
bskills doctor [--wallet <name>] [--json]
```

Diagnoses whether the environment is ready for the paid-purchase flow. Runs
three checks and exits `0` only when **all** pass, `1` otherwise. **Strictly
read-only** — it never signs, never touches keys, and never logs you in.

The three checks:
1. **`bskills-cli auth`** — a stored token in `~/.bskills-cli/config.json`.
   This check is offline (token presence only); `whoami` does the online
   validity probe. Remediation when missing: `bskills login`.
2. **`ows` installed** — the `ows` binary on PATH. Remediation when missing:
   `npm install -g @open-wallet-standard/core` (the standalone Open Wallet
   Standard CLI — NOT the unrelated `ows` npm squatter; see https://openwallet.sh).
3. **`ows` solana wallet** — at least one Solana wallet in `ows`. With
   `--wallet <name>` the check is scoped to that named wallet and, on success,
   reports `<name> → <pubkey>`. Resolution parses `ows wallet list`; there is
   no per-wallet address command.

`doctor` does **not** check on-chain USDC/SOL balance — that needs a network
call the CLIs don't expose uniformly. If `pay` later fails with a funding
error, ask the user to top up.

`--json` emits `{ ok: boolean, checks: [{ name, ok, detail?, remediation? }] }`
and still sets exit `1` when `ok === false`.

## Search

```bash
bskills search [query]
  -t, --type <skill|plugin>   # omit to show both
  -c, --category <slug>
  --min-price <cents>         # integer, 0 means free
  --max-price <cents>
  --featured
  --sort <newest|trending>    # default: newest
  --page <n>                  # default: 1
  --limit <n>                 # default: 20, max 100
  --json
```

- Prices are in **cents**. $7.99 → `799`.
- Free-text query matches `displayName` and `description` on the server.
- Results include `slug`, `id`, `displayName`, `description`, `priceCents`,
  `averageRating`, `totalRatings`, `type`, `mainCategory`, `creator`.

## Acquire (free skills only)

```bash
bskills acquire <slug-or-uuid> [--json]
```

Only works for skills where `priceCents === 0`. The CLI resolves slug → UUID
automatically. On success the user's purchase record is created and the
skill becomes installable.

## Pay (direct-USDC-split, server-side broadcast)

```bash
bskills pay <slug-or-uuid> --wallet <solana-pubkey>      [--json]   # 1. initiate
bskills pay <slug-or-uuid> --signature-hex <hex>         [--json]   # 2. settle
```

The CLI never holds keys and never signs. The agent only orchestrates `ows`
(for signing) between the two `bskills pay` calls. Both calls are plain
JSON over HTTPS, with no special payment headers.

- **Initiate** (`--wallet <pubkey>`): hits `POST /api/pay/initiate` with body
  `{pluginId, payerWallet}`. The server replies with `purchaseId`, the unsigned
  USDC-split transaction (`transactionBase64`), `network`, `expiresAt`, `mint`,
  `lastValidBlockHeight`, an `amounts` object (`totalBaseUnits`,
  `creatorBaseUnits`, `feeBaseUnits`), and a `recipients` object (`creatorAta`,
  plus `treasuryAta` when `feeBaseUnits > 0`). The CLI converts the unsigned tx
  to hex, caches it in `~/.bskills-cli/state.json` keyed by **pluginId** (one
  pending payment per plugin), and in `--json` mode surfaces a **flat,
  top-level `transactionHex`** field — that is exactly what `ows sign tx`
  consumes. A `409 already own` returns `{phase: "initiate", outcome: "owned"}`.
- **Settle** (`--signature-hex <hex>`): the agent passes the bare 64-byte
  ed25519 signature (128 hex chars) returned by `ows`. The CLI loads the cached
  unsigned tx for the plugin, splices the signature into the wire format,
  base64-encodes it, and POSTs `/api/pay/settle` with body
  `{purchaseId, signedTransactionBase64}`. The backend broadcasts, confirms,
  and settles in a single round trip, replying with a `purchase` object
  (`purchaseId`, `pluginId`, `txSignature`, `amountBaseUnits`, `network`,
  `grantedAt`). On a successful settle the cached unsigned is cleared.
- **Pass exactly one** of `--wallet` (initiate) or `--signature-hex` (settle).
  Passing both or neither is rejected: `Pass exactly one of --wallet <pubkey>
  (to initiate) or --signature-hex <hex> (to settle).`
- The price is **never** passed on the command line — the server reads it from
  the plugin record.
- Keep the **same logged-in user** across initiate → settle; the cache is keyed
  by pluginId for the current user.

### Settle exit codes

Exit `0` (done/owned), `2` (network), `3` (auth), `4` (timeout — cache kept,
retry the same `--signature-hex`), `5` (reinitiate — cache cleared, start over).
Full recovery contract incl. soft-expiry: `references/troubleshooting.md`.

## Install

```bash
bskills install <slug-or-uuid>
  -a, --agent <id>             # repeatable; default: all detected agents
  -s, --scope <global|project> # default: config defaultScope (factory: global)
  -m, --mode <copy|symlink>    # default: config defaultInstallMode (factory: copy)
  --force                      # re-pull the cached repo
  --json
```

- The user must **own** the skill first (via `acquire` or `pay`). If they
  don't, the command errors and suggests the right previous step.
- `global` writes to each agent's own skills dir — agent-specific, **not** a
  uniform `~/.{id}/skills` (e.g. Windsurf → `~/.codeium/windsurf/skills`). Read
  the real path from `bskills agents --json` (`globalSkillsPath`); don't
  construct it. `project` writes to `<cwd>/<agent-project-dir>/skills`.
- `copy` duplicates files; `symlink` links to the shared cache at
  `~/.cache/bskills-cli/repos/<repo>`.

## Update (alias: upgrade)

```bash
bskills update <slug-or-name>
  -a, --agent <id>             # repeatable; default: every agent it's installed on
  -s, --scope <global|project>
  --json
```

Pulls the **latest version** of an already-installed skill (force-refreshes the
cached repo) and re-installs it in place. Use this for "get the newest version
of X" — do not re-`pay`/`acquire`, and do not delete-then-reinstall. If the
skill isn't installed, it errors with `Skill "<name>" is not installed.`;
filters that match nothing error with `No matching installations for
"<name>" with the given filters.`. `upgrade` is an exact alias.

## Uninstall (alias: remove)

```bash
bskills uninstall <slug-or-name>
  -a, --agent <id>             # repeatable; default: every agent it's installed on
  -s, --scope <global|project>
  --json
```

Removes the skill from your AI agents' skill directories and keeps local state
in sync. **Never shell out to `rm`** — this command removes exactly the files
it installed. `remove` is an exact alias. Same not-installed / no-match error
messages as `update`.

## Check what is installed

```bash
bskills installed [-a <agent>] [-s <scope>] [--remote] [--json]
```

- Without `--remote`: reads `~/.bskills-cli/state.json`. Safe offline.
- With `--remote`: also calls `GET /api/checkout/purchases` to mark which
  owned skills are installed (● installed) vs not (○ not installed). Needs
  auth.

## Agents

```bash
bskills agents [--installed] [--json]
```

Lists the 18 supported agents. `●` means the detection path exists on this
machine (e.g. `~/.claude` for Claude Code), `○` means it does not.

Valid agent ids to pass to `-a/--agent`:

```
claude-code, cursor, windsurf, github-copilot, roo-code, opencode, cline, amp,
aider, codex, continue, void, pear, zed, trae, melty, aide, openclaw
```

## Config

```bash
bskills config list
bskills config get <key>
bskills config set <key> <value>
```

Writable keys:
- `defaultScope` — `global` | `project`
- `defaultInstallMode` — `copy` | `symlink`

API URLs are **not** user-configurable. The CLI targets production
(`https://api.buyskills.ai`) by default and has no user-facing environment
switch; respect that.

## Self-update (operator action — NOT an agent action)

```bash
bskills self-update [--json]
```

Upgrades the CLI itself via `npm install -g bskills@latest`. **Operator action —
an agent must not run this autonomously** (mutates global npm state, may need
`sudo`). Surface the CLI's "update available" notice to the human instead.
