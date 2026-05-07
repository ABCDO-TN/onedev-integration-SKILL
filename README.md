# OneDev Integration Skill

A skill that teaches an AI agent how to authenticate to and operate a self-hosted [OneDev](https://onedev.io) server — the all-in-one DevOps platform combining Git hosting, issue tracking, kanban, CI/CD pipelines, and a package registry.

Supports OneDev **13+** (MCP-first) with full fallback to the REST API for any version.

---

## What this skill enables

Once loaded, the AI agent can:

- Authenticate via OneDev personal access tokens (Bearer / Basic / `tod` config).
- Talk to OneDev through the **`tod mcp`** server (~25 typed tools + 4 guided prompts) on OneDev 13+.
- Talk to OneDev's **REST API** at `/~api/...` on any version.
- Create, query, edit, transition, link, and comment on **issues**.
- Create, review, approve, merge, and discard **pull requests**.
- Trigger, inspect, and debug **CI/CD builds** — including diff-since-last-green-build investigation.
- Edit and validate `.onedev-buildspec.yml` against the live schema.
- Compose precise queries in OneDev's ANTLR-built **query language** (issues, PRs, builds, commits, projects).
- Run CI jobs against **uncommitted local changes** via `tod run-local`.

---

## Installation

### Option A — drop-in install (Claude.ai / Claude Desktop / generic skill loader)

1. Download and unzip `onedev-integration.zip`.
2. Copy the entire `onedev-integration/` folder into your skills directory.
   - **Claude.ai user skills**: upload via the Skills settings UI.
   - **Local installs** (Claude Desktop, Claude Code, custom agents): drop into `~/.claude/skills/` (or whatever path your runner reads).
3. Restart the AI client. The skill triggers automatically on OneDev-related prompts.

### Option B — install via skill-creator's packager

```bash
python -m scripts.package_skill ./onedev-integration
# install the resulting .skill file through your client
```

---

## Prerequisites the human user must have

Before the agent can do anything useful, the user needs:

1. **A running OneDev server** they can reach (cloud, on-prem, or local Docker).
   - Quick local install:
     ```bash
     docker run -it --rm \
       -v /var/run/docker.sock:/var/run/docker.sock \
       -v $(pwd)/onedev:/opt/onedev \
       -p 6610:6610 -p 6611:6611 \
       1dev/server
     ```
2. **A personal access token** generated from the OneDev web UI (avatar → *Access Tokens* → *New Access Token*). Scope it to only what the agent needs (read code, manage issues, run jobs, etc.).
3. **For MCP mode (OneDev 13+):** the `tod` binary on `$PATH`. Download from `https://code.onedev.io/onedev/tod/~builds`.
4. **`~/.todconfig`** file:
   ```ini
   server-url=https://onedev.example.com
   access-token=<your-token>
   ```

---

## Wiring `tod mcp` into your AI client

Add this to the client's MCP server config:

```json
{
  "mcpServers": {
    "OneDev": {
      "command": "tod",
      "args": ["mcp", "--log-file", "/tmp/tod.log"]
    }
  }
}
```

Restart the client. You should see the OneDev tools and four prompt templates (`change-issue-state`, `edit-build-spec`, `investigate-build-problems`, `review-pull-request`) appear in the available-tools list.

---

## Folder contents

```
onedev-integration/
├── SKILL.md                  Main skill instructions (always loaded when triggered)
├── README.md                 This file
└── references/
    ├── mcp-tools.md          Every `tod mcp` tool, its parameters, and when to use it
    ├── rest-api.md           Practical REST API recipes (auth, projects, issues, PRs, builds)
    └── query-language.md     Full OneDev query-language grammar with field tables per entity
```

The reference files use **progressive disclosure** — the agent loads them on demand, not on every turn, so they don't bloat context.

---

## Example prompts that trigger this skill

- "Connect to my OneDev server and show me all open bugs assigned to me."
- "Create a feature request for Docker support on the `my-app` project, assign it to alice, and link it as a sub-issue of #14."
- "Build #137 just failed — investigate and tell me what changed since the last green build."
- "Review pull request #42 and approve it if the changes look safe."
- "Help me write a `.onedev-buildspec.yml` for a Node.js project with a Postgres test step."
- "Run the `ci` job against my uncommitted local changes."

---

## Safety defaults the skill enforces

- **Reads before writing.** The agent fetches an entity's current state before transitioning, merging, or editing it.
- **Confirms destructive actions.** Merges, force-pushes, mass deletions, and irreversible state transitions trigger a confirmation step unless pre-authorised.
- **Respects confidentiality.** Issues containing secrets, credentials, or PII are flagged `confidential: true` on creation.
- **Treats the access token as secret.** Never echoed in chat, never written outside `~/.todconfig`, never put into URLs over plain HTTP.
- **Honours the working directory.** Calls `getCurrentProject` before write operations so the user knows which repo will be touched.

---

## Troubleshooting

| Symptom | Most likely cause | Fix |
| --- | --- | --- |
| `401 Unauthorized` | Bad / expired / wrong-scope token | Regenerate token; verify scopes |
| `tod mcp` exits immediately | Missing or malformed `~/.todconfig` | Recreate it; ensure `https://` scheme, no trailing slash |
| MCP tools say "no current project" | Working dir isn't a clone of a OneDev repo | `cd` into the clone, or call `setWorkingDir` |
| `tod run-local` logs never stream | Nginx is buffering | Add `proxy_buffering off;` for `location /~api/streaming` |
| `trigger-job` returns `403` | Token lacks `Run Job` permission | Update token scopes |
| MCP server doesn't appear in client | Client wasn't restarted after config edit | Quit and relaunch |

A fuller troubleshooting table lives in `SKILL.md` §9.

---

## Compatibility

| Component | Required version |
| --- | --- |
| OneDev server | **13.0+** for the MCP path; **any version** for the REST path |
| `tod` CLI | latest release from `code.onedev.io/onedev/tod` |
| MCP-aware AI client | Cursor, Claude Desktop, Claude Code, Continue, or any client supporting the MCP protocol |

---

## Sources

This skill was built from the official OneDev documentation, the `theonedev/tod` repository, and the OneDev MCP tutorial:

- OneDev docs: <https://docs.onedev.io>
- MCP tutorial: <https://docs.onedev.io/tutorials/ai/working-with-mcp>
- `tod` CLI + MCP tool reference: <https://code.onedev.io/onedev/tod>
- REST API (live, version-accurate): `https://<your-onedev-host>/~help/api`

---

## License

The skill itself is provided as-is for use with any OneDev installation. OneDev and `tod` are licensed by their respective authors — see the OneDev project for details.