# Security review response (2026-10-22)

DigitalOcean Security review notes for the Grok Bot plugin (`digitalocean-internal`).
Each item maps to implementation in this repository and notes anything that still
requires action outside the repo.

## 1. Enforce tiers with a plugin hook

**Done.**

- Cursor: [`plugins/digitalocean-internal/hooks/hooks.json`](plugins/digitalocean-internal/hooks/hooks.json)
  registers `beforeMCPExecution` → `scripts/before-mcp-hook.mjs` with
  `failClosed: true`.
- Claude Code: same file registers `PreToolUse` with matcher
  `mcp__plugin_digitalocean-internal_digitalocean-.*` and
  `${CLAUDE_PLUGIN_ROOT}`-quoted command path.
- Logic: [`plugins/digitalocean-internal/lib/mcp-tier-policy.mjs`](plugins/digitalocean-internal/lib/mcp-tier-policy.mjs)
  loads `policy/approval-tiers-*.json` from the plugin root; Tier 1 → `allow`,
  Tier 2–4 → `ask` with target summary from `tool_input`, unmapped writes →
  `deny`.
- Unit tests: [`tests/test_mcp_tier_policy.mjs`](tests/test_mcp_tier_policy.mjs).
- README documents verification steps and the Cursor **2.4.x `ask` ignored** caveat.

**Not fully resolvable here:** Grok Bot may not load plugin hooks yet; skills remain
mandatory defense in depth.

## 2. Fail closed on unmapped tools + tier maps

**Done.**

- Skill rule: unmapped writes are Tier 3 in
  [`skills/digitalocean-safety/SKILL.md`](skills/digitalocean-safety/SKILL.md).
- Hook denies unmapped writes (allows read-shaped tool names).
- New maps (upstream `8ce21db`):
  - [`approval-tiers-accounts.json`](plugins/digitalocean-internal/policy/approval-tiers-accounts.json)
    (`key-create` elevated to Tier 3)
  - [`approval-tiers-networking.json`](plugins/digitalocean-internal/policy/approval-tiers-networking.json)
  - [`approval-tiers-databases.json`](plugins/digitalocean-internal/policy/approval-tiers-databases.json)
  - [`approval-tiers-apps.json`](plugins/digitalocean-internal/policy/approval-tiers-apps.json)
  - [`approval-tiers-insights.json`](plugins/digitalocean-internal/policy/approval-tiers-insights.json)
  - [`approval-tiers-docs.json`](plugins/digitalocean-internal/policy/approval-tiers-docs.json)
    (all tools upstream read-only, Tier 1; hook denies unlisted docs tools)
- Generator: [`scripts/bootstrap-tier-maps.mjs`](scripts/bootstrap-tier-maps.mjs)

## 3. Policy from plugin root; maintainer sync out of runtime skills

**Done.**

- Skills reference `${CLAUDE_PLUGIN_ROOT}/policy/...` and plugin install root for
  Cursor; Tier 3/4 lists inlined in **digitalocean-safety** for Cursor.
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

## 9. Claude Code MCP discovery

**Done.**

- [`plugins/digitalocean-internal/.mcp.json`](plugins/digitalocean-internal/.mcp.json)
  (copy of `mcp.json` for dotfile discovery).
- [`plugins/digitalocean-internal/.claude-plugin/plugin.json`](plugins/digitalocean-internal/.claude-plugin/plugin.json)
  sets `"mcpServers": "./mcp.json"`.
