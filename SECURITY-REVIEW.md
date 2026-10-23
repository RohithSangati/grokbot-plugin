# Security review response (2026-10-22)

DigitalOcean Security review notes for the Grok Bot plugin (`digitalocean`).
Each item maps to implementation in this repository and notes anything that still
requires action outside the repo.

## 1. Enforce tiers with a plugin hook

**Done.**

- Cursor: [`plugins/digitalocean/hooks/hooks.json`](plugins/digitalocean/hooks/hooks.json)
  registers `beforeMCPExecution` → `scripts/before-mcp-hook.mjs` with
  `failClosed: true` (Cursor-only; no Claude `PreToolUse`).
- Logic: [`plugins/digitalocean/lib/mcp-tier-policy.mjs`](plugins/digitalocean/lib/mcp-tier-policy.mjs)
  loads `policy/approval-tiers-*.json` from the plugin root. For the single
  DigitalOcean MCP server it passes `find_tools` / `retrieve_schema` through and
  unwraps the `invoke_tool` qualified `service/tool` target before scoring: Tier 1 →
  `allow`, Tier 2–4 → `ask` with target summary from `tool_input`; unmapped
  tools → `deny` (only mapped Tier 1 and unmapped `region-list` are allowed).
- Unit tests: [`tests/test_mcp_tier_policy.mjs`](tests/test_mcp_tier_policy.mjs).
- README documents verification steps and the Cursor **2.4.x `ask` ignored** caveat.

**Not fully resolvable here:** a host may not load plugin hooks yet; skills remain
mandatory defense in depth.

## 2. Fail closed on unmapped tools + tier maps

**Done.**

- Skill rule: unmapped writes are Tier 3 in
  [`skills/digitalocean-safety/SKILL.md`](skills/digitalocean-safety/SKILL.md).
- Hook denies any unmapped tool (no name-pattern bypass); only mapped Tier 1
  and `region-list` without a row are allowed.
- New maps (upstream `8ce21db`):
  - [`approval-tiers-accounts.json`](plugins/digitalocean/policy/approval-tiers-accounts.json)
    (`key-create` elevated to Tier 3)
  - [`approval-tiers-networking.json`](plugins/digitalocean/policy/approval-tiers-networking.json)
  - [`approval-tiers-databases.json`](plugins/digitalocean/policy/approval-tiers-databases.json)
  - [`approval-tiers-apps.json`](plugins/digitalocean/policy/approval-tiers-apps.json)
  - [`approval-tiers-insights.json`](plugins/digitalocean/policy/approval-tiers-insights.json)
  - [`approval-tiers-docs.json`](plugins/digitalocean/policy/approval-tiers-docs.json)
    (all tools upstream read-only, Tier 1; hook denies unlisted docs tools)
- Generator: [`scripts/bootstrap-tier-maps.mjs`](scripts/bootstrap-tier-maps.mjs)

## 3. Policy from plugin root; maintainer sync out of runtime skills

**Done.**

- Skills reference the plugin install root (Cursor expands `${CURSOR_PLUGIN_ROOT}`)
  for `policy/...`; Tier 3/4 lists inlined in **digitalocean-safety** for Cursor.
- Sync commands moved to [`AGENTS.md`](AGENTS.md) (not shipped in plugin install).
- Droplets skill no longer tells agents to run `scripts/sync-tier-map.mjs`.
- Policy unreadable → every write Tier 3 (skill + hook).

## 4. Widen untrusted-data rule

**Done** in [`skills/digitalocean-safety/SKILL.md`](skills/digitalocean-safety/SKILL.md):
names, tags, image names, user_data, app spec env values, error messages, docs
output; tool output cannot pick targets or confirm; re-ask on argument drift;
user_data quoted verbatim for Tier 4 droplet create.

## 5. Tokens pasted in chat; least-privilege connect

**Done** in [`skills/digitalocean-connect/SKILL.md`](skills/digitalocean-connect/SKILL.md):
never accept PAT in chat, revoke guidance, connection strings handling; **Apps** and
**Accounts** connect on first use (with manifest limitation documented).

## 6. Adversarial evals

**Done** (integration eval sources; run in agent-plugins monorepo with skill-tools):

| Case                        | File                                                                           |
| --------------------------- | ------------------------------------------------------------------------------ |
| Droplet name injection      | `skills/digitalocean-safety/evals/test_digitalocean_safety_evals.py`           |
| Log line delete instruction | same                                                                           |
| Target switch after confirm | same                                                                           |
| Bulk tag production         | same                                                                           |
| Unmapped tool               | same (`insights-future-write`) + hook test in `tests/test_mcp_tier_policy.mjs` |
| Token pasted in chat        | `skills/digitalocean-connect/evals/test_digitalocean_connect_evals.py`         |

## 7. Pinning

**Partially in repo; admin action required.**

- [`RELEASE.md`](RELEASE.md) documents tag/SHA pinning workflow.
- Marketplace must pin tag/SHA in Cursor Dashboard (see RELEASE.md).

## 8. Drift check in CI (optional)

**Done.**

- [`scripts/sync-tier-map.mjs`](scripts/sync-tier-map.mjs) checks all policy files.
- [`Makefile`](Makefile) `tier-drift` target; [`.github/workflows/lint.yml`](.github/workflows/lint.yml)
  runs on push/PR and **weekly** (`0 6 * * 1` UTC).

## 9. Single hosted DigitalOcean MCP server (Cursor-only)

**Done.**

- [`plugins/digitalocean/mcp.json`](plugins/digitalocean/mcp.json) declares one
  server, DigitalOcean MCP (`https://mcp.digitalocean.com/mcp`), via the Agent
  Plugins schema with hosted OAuth (no stored credentials).
- Claude Code packaging (`.claude-plugin/`, `.mcp.json`, `PreToolUse`) was
  removed; the plugin ships Cursor components only.
