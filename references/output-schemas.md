# JSON output schemas

Use `--json` and parse. Below are the (trimmed) shapes for the commands that
agents most commonly need to introspect.

## `bskills search --json`

```jsonc
{
  "plugins": [
    {
      "id": "...",
      "slug": "...",
      "displayName": "...",
      "priceCents": 799,                  // 0 means free → use `acquire` not `pay`
      "type": "skill",                    // or "plugin"
      "status": "published",              // only purchasable when "published"
      "mainCategory": "developer-tools",  // nullable
      "averageRating": 4.7,
      "totalRatings": 23,
      "creator": { "githubUsername": "alice" }
    }
  ],
  "total": 42,
  "page": 1,
  "limit": 20
}
```

## `bskills pay <slug> --wallet <pubkey> --json` (initiate)

The hex you feed to `ows sign tx` is the **flat, top-level `transactionHex`**
field. Extract it inline: `... --json | python3 -c "import sys,json;
print(json.load(sys.stdin)['transactionHex'])"`.

```jsonc
{
  "phase": "initiate",
  "purchaseId": "...",                          // correlation token for settle (flows on the wire, not the pluginId)
  "transactionBase64": "AQAA...==",             // the bytes the server prescribed
  "transactionHex": "0100...",                  // same bytes, hex — feed this to `ows sign tx`
  "network": "devnet",                          // or "mainnet-beta"
  "mint": "<USDC mint>",                        // devnet: 4zMMC9srt5Ri5X14GAgXhaHii3GnPAEERYPJgZJDncDU
  "expiresAt": "2026-04-25T12:05:00.000Z",      // soft-expiry; past this, settle routes to reinitiate (exit 5)
  "lastValidBlockHeight": 351234567,
  "amounts": {
    "totalBaseUnits": 5000000,                  // base units; 5 USDC = 5_000_000
    "creatorBaseUnits": 4750000,
    "feeBaseUnits": 250000                       // 0 ⇒ round-to-zero single transfer (treasuryAta omitted)
  },
  "recipients": {
    "creatorAta": "<creator USDC ATA>",
    "treasuryAta": "<treasury USDC ATA>"         // omitted when feeBaseUnits === 0
  }
}
// or, if the user already owns the skill (409 already own):
// { "phase": "initiate", "outcome": "owned" }
```

## `bskills pay <slug> --signature-hex <hex> --json` (settle)

`outcome` plus the exit code drives recovery (see `references/commands.md` /
`references/troubleshooting.md`): `done`/`owned` ⇒ exit 0; `auth` ⇒ exit 3
(cache kept); `timeout` ⇒ exit 4 (cache kept, retry same sig); `reinitiate` ⇒
exit 5 (cache cleared, start over). The `purchase` object is present only on
`done`.

```jsonc
{
  "phase": "settle",
  "outcome": "done",                            // or "owned" | "timeout" | "reinitiate" | "auth"
  "purchase": {                                 // present only when outcome === "done"
    "purchaseId": "...",
    "pluginId": "...",
    "txSignature": "...",                       // base58, from the on-chain broadcast — build the explorer link from THIS
    "amountBaseUnits": 5000000,
    "network": "devnet",                        // drives ?cluster=… on the explorer URL
    "grantedAt": "2026-04-25T12:00:00.000Z"
  }
  // non-"done" outcomes carry a "message" field instead of "purchase":
  // { "phase": "settle", "outcome": "reinitiate", "message": "Cached payment ... expired ..." }
}
```

## `bskills doctor --json`

```jsonc
{
  "ok": true,                                   // false ⇒ exit code 1
  "checks": [
    { "name": "bskills-cli auth", "ok": true, "detail": "logged in as you@example.com" },  // literal check name — the CLI still labels it "bskills-cli auth"
    { "name": "ows installed", "ok": true, "detail": "version 1.2.3" },
    { "name": "ows solana wallet", "ok": true, "detail": "agent-treasury → HF85…o5s" }
    // a failing check adds "remediation": "<the exact fix command/url>"
  ]
}
```

## `bskills install --json`

```jsonc
{
  "installed": [
    {
      "success": true,
      "skillName": "code-review",
      "agentId": "claude-code",
      "targetPath": "/Users/.../.claude/skills/code-review"
    }
  ],
  "failed": []
}
```

## `bskills init --json`

One-shot bootstrap report. `ok` is `false` (and exit code `1`) when install
succeeded on no agent. In `--json` mode a missing session is a hard error
(exit `3`), not a field here.

```jsonc
{
  "ok": true,                                   // false ⇒ exit 1 (no agent installed)
  "skill": { "slug": "...", "displayName": "...", "id": "..." },
  "auth": {
    "status": "existing",                       // "existing" (token reused) | "logged-in" (fresh browser login)
    "user": "alice"                             // optional
  },
  "acquire": {
    "status": "already-owned",                  // "already-owned" | "acquired"
    "purchaseId": "..."                         // present only when status === "acquired"
  },
  "install": {
    "installed": [
      { "success": true, "skillName": "...", "agentId": "claude-code", "targetPath": "..." }
    ],
    "failed": []                                // failed entries carry an "error" field
  }
}
```

## `bskills installed --remote --json`

```jsonc
{
  "installations": [
    {
      "skillName": "code-review",
      "agentId": "claude-code",
      "scope": "global",
      "installMode": "copy",
      "installedAt": "2026-04-24T…",
      "version": "abc1234"
    }
  ],
  "remote": [
    {
      "slug": "code-review",
      "displayName": "Code Reviewer",
      "status": "completed",                    // server-supplied free-form string — treat any non-pending status as owned rather than hardcoding "completed"
      "amount": 0
    }
  ]
}
```

To find owned-but-not-installed skills, filter `remote[*]` where
`status === "completed"` and no entry in `installations[*]` has a matching
`skillName` for the slug.
