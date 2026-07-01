# Financial Services Plugins

Cowork plugins and Claude Managed Agent (CMA) templates for financial services.
The organizing idea: **each named agent ships two ways from one source**. A
single canonical system prompt in `plugins/agent-plugins/<slug>/agents/<slug>.md`
is wrapped both as a Cowork plugin (interactive, in the marketplace) and as a
Managed Agent cookbook (`POST /v1/agents`, headless/orchestrated). Skills have
one source of truth in `vertical-plugins/` and are copied into the agent bundles.

## Repository Structure

```
├── .claude-plugin/
│   └── marketplace.json             # marketplace manifest — registers every plugin + its source path
├── plugins/
│   ├── agent-plugins/               # named agents — one self-contained plugin each (10 agents)
│   │   └── <slug>/
│   │       ├── .claude-plugin/plugin.json
│   │       ├── agents/<slug>.md     # ← canonical system prompt (one source, two wrappers)
│   │       └── skills/              # ← bundled copies, synced FROM vertical-plugins/
│   ├── vertical-plugins/            # FSI verticals — skill SOURCES, commands, MCP connectors (7 verticals)
│   │   └── <vertical>/
│   │       ├── .claude-plugin/plugin.json
│   │       ├── commands/*.md        # /<plugin>:<command>
│   │       ├── skills/<name>/SKILL.md
│   │       ├── hooks/hooks.json     # optional
│   │       └── .mcp.json            # optional — HTTP MCP data connectors
│   └── partner-built/               # partner plugins (lseg, spglobal)
├── managed-agent-cookbooks/         # CMA cookbooks — one dir per named agent (mirrors agent-plugins/)
│   └── <slug>/
│       ├── agent.yaml               # orchestrator manifest; system + skills reference ../../plugins/agent-plugins/<slug>/…
│       ├── subagents/*.yaml         # depth-1 leaf workers (reader / critic / resolver, etc.)
│       ├── steering-examples.json   # sample steering events to kick a session
│       └── README.md                # overview, deploy steps, security tier + handoff notes
├── claude-for-msft-365-install/     # admin tooling for the Claude Microsoft 365 add-in (separate from FSI plugins)
├── scripts/                         # check.py, validate.py, sync-agent-skills.py, deploy-managed-agent.sh, orchestrate.py, version_bump.py, test-cookbooks.sh
├── .githooks/pre-commit             # version-bump hook (git config core.hooksPath .githooks)
└── .github/workflows/               # plugin-validate.yml, version-bump.yml, secret-scan.yml
```

## Plugin Inventory

**Verticals** (`plugins/vertical-plugins/`) — skill sources, slash commands, MCP connectors:
`financial-analysis`, `investment-banking`, `equity-research`, `private-equity`,
`wealth-management`, `fund-admin`, `operations`.

**Named agents** (`plugins/agent-plugins/`) — each has a canonical `agents/<slug>.md` prompt and a matching cookbook:
`pitch-agent`, `market-researcher`, `earnings-reviewer`, `meeting-prep-agent`,
`model-builder`, `gl-reconciler`, `kyc-screener`, `valuation-reviewer`,
`month-end-closer`, `statement-auditor`.

**Partner-built** (`plugins/partner-built/`): `lseg`, `spglobal` (S&P Global / Kensho).

Every plugin above (plus `claude-for-msft-365-install`) is registered in
`.claude-plugin/marketplace.json` with `name`, `displayName`, `source`, and `description`.

## The Two-Wrapper Model

The same agent exists as both a Cowork plugin and a Managed Agent:

- **Cowork plugin** (`plugins/agent-plugins/<slug>/`): interactive. `agents/<slug>.md`
  carries YAML frontmatter (`name`, `description`, `tools`) plus the system prompt.
  `skills/` holds bundled copies of the skills this agent needs.
- **Managed Agent cookbook** (`managed-agent-cookbooks/<slug>/`): headless, for
  `POST /v1/agents`. `agent.yaml` is the orchestrator manifest; it *references* the
  same `agents/<slug>.md` via `system.file` and pulls skills via `from_plugin`, so
  there is no second copy of the prompt. `deploy-managed-agent.sh` resolves these
  conveniences (`system.file` → inline string, `skills.path`/`from_plugin` → uploaded
  skill_id, `callable_agents.manifest` → created sub-agent id) before posting.

### Managed-agent security tiers

Cookbooks that ingest outsider-authored documents (counterparty statements, KYC
docs, GP packages) are structured so an injected instruction can never reach a
shell, a write tool, or a firm system. The pattern, documented per-cookbook in
each `README.md`:

- **reader** subagent — the *only* tier that touches untrusted docs. Read/Grep
  only, no MCP, no bash, no write. Emits schema-validated, length-capped,
  character-class-restricted JSON (`output_schema` in the subagent yaml, enforced
  by `scripts/validate.py`) so injected text cannot survive intact.
- **orchestrator** (`agent.yaml`) — dispatches and aggregates; read-only tools +
  read-only MCP connectors; never reads outsider files directly.
- **critic** — independently re-verifies findings against trusted sources.
- **resolver** (Write-holder) — writes outputs to `./out/`; never opens an outsider file.

Cross-agent handoffs are emitted as an allowlisted, schema-validated `handoff_request`
in final output and routed by `scripts/orchestrate.py` (a *reference* event loop —
replace with your firm's Temporal/Airflow/event-bus in production). Managed agents
are **depth-1**: the orchestrator has `callable_agents`, subagents must not.

## Scripts

| Script | What it does |
|---|---|
| `scripts/check.py` | Lint all manifests; verify every `system.file` / `skills.path` / `callable_agents.manifest` reference resolves; fail if any `agent-plugins/<slug>/skills/` copy has drifted from its `vertical-plugins/` source. Also self-installs the git hook. **Run before committing.** Requires `pyyaml`. |
| `scripts/sync-agent-skills.py` | Copy skill sources from `vertical-plugins/` into the `agent-plugins/<slug>/skills/` bundles. Run after editing any skill. |
| `scripts/deploy-managed-agent.sh <slug> [--dry-run]` | Resolve a cookbook manifest and `POST /v1/agents`. Needs `ANTHROPIC_API_KEY`, `jq`, `pyyaml`, plus any `${…_MCP_URL}` env vars the cookbook references. |
| `scripts/test-cookbooks.sh` | Dry-run every cookbook and assert the resolved API bodies are well-formed (valid JSON, depth-1, non-empty system, no leaked `output_schema`). |
| `scripts/validate.py` | Validate a worker's JSON output against its `output_schema` (used by the deploy harness to wrap reader subagents). |
| `scripts/orchestrate.py` | Reference cross-agent handoff event loop (allowlist + payload validation). Not production. |
| `scripts/version_bump.py` | Patch-bump plugin `version`s; invoked by the pre-commit hook and the CI backstop. |

## Conventions & Guardrails

- **Edit skills in `vertical-plugins/`, never in `agent-plugins/<slug>/skills/`.**
  Those are generated copies. After editing, run `python3 scripts/sync-agent-skills.py`,
  then `python3 scripts/check.py` (which fails on drift).
- **Version bumps gate update delivery.** A plugin's `.claude-plugin/plugin.json`
  `version` controls whether already-installed users receive an update. The
  `.githooks/pre-commit` hook patch-bumps any changed plugin's `version` so a branch
  ends up exactly **one patch ahead of `main`** (bumped once for the branch, not
  per commit). The `version-bump` GitHub Action enforces the same rule on PRs as a
  backstop. `check.py` wires the hook via `git config core.hooksPath .githooks`
  (native git — no Husky/Node). Bypass a single commit with `git commit --no-verify`.
- **MCP connectors** live in each vertical's `.mcp.json` as HTTP MCP servers
  (Daloopa, Morningstar, S&P Global/Kensho, FactSet, Moody's, LSEG, PitchBook,
  Chronograph, Egnyte, Box, etc.). Managed-agent cookbooks reference MCP servers by
  URL env var (e.g. `${GL_MCP_URL}`).
- **CI**: `plugin-validate.yml` runs the linter on PRs; `version-bump.yml` enforces
  the patch-bump rule; `secret-scan.yml` scans for leaked secrets.
- `*.local.md`, `TASKS.md`, `MEMORY.md`, and `out/` are gitignored.

## Key Files

- `.claude-plugin/marketplace.json` — registers every plugin with its source path and displayName.
- `<plugin>/.claude-plugin/plugin.json` — plugin metadata: `name`, `version`, `description`, `author`.
- `agents/<slug>.md` — canonical named-agent system prompt (frontmatter: `name`, `description`, `tools`).
- `commands/*.md` — slash commands, invoked as `/<plugin>:<command>`.
- `skills/<name>/SKILL.md` — skill knowledge/workflows; may carry `scripts/`, `references/`, `requirements.txt`.
- `managed-agent-cookbooks/<slug>/agent.yaml` + `subagents/*.yaml` — CMA manifests.

## Development Workflow

1. Edit markdown/skill files directly — changes take effect immediately in Cowork.
2. If you changed a skill, run `python3 scripts/sync-agent-skills.py` to propagate to agent bundles.
3. Run `python3 scripts/check.py` — must print `OK` before committing.
4. For managed agents, dry-run with `scripts/deploy-managed-agent.sh <slug> --dry-run`
   or the whole set with `scripts/test-cookbooks.sh`.
5. Commit — the pre-commit hook handles the version bump. Test Cowork commands with
   `/<plugin>:<command>`; skills fire automatically when their triggers match.
