# OneDev Integration Skill

A skill that teaches an AI agent how to authenticate to and operate a self-hosted [OneDev](https://onedev.io) server — the all-in-one DevOps platform combining Git hosting, issue tracking, kanban, CI/CD pipelines, and a package registry.

Supports OneDev **13+** (MCP-first) with full fallback to the REST API for any version.

> **Single-file skill.** Everything the agent needs — main instructions plus the full MCP tool catalogue, REST API reference, and query-language grammar — is inlined into one self-contained `SKILL.md`. This guarantees the skill works on any platform, including loaders that only ingest the root `SKILL.md` and ignore subdirectories (Hechicha / Paperclip, some agent platforms, etc.).

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

### Hechicha / Paperclip (or any GitHub-URL skill loader)

Just paste the repo URL into the loader's "Add skill" field:

```
https://github.com/ABCDO-TN/onedev-integration-SKILL
```

The loader will fetch `SKILL.md` from the repo root. Because everything is inlined, no other files need to be present for the skill to work fully.

### Claude.ai / Claude Desktop / Claude Code

1. Download `SKILL.md` from this repo (or clone the whole repo).
2. Place it in the appropriate skills directory:
   - **Claude.ai user skills**: upload via the Skills settings UI.
   - **Local installs**: drop the file into `~/.claude/skills/onedev-integration/SKILL.md` (or whatever path your runner reads).
3. Restart the AI client. The skill triggers automatically on OneDev-related prompts.

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

## What's inside `SKILL.md`

A single file, structured as:

| Section | Contents |
| --- | --- |
| §1 — Authentication | Token creation, scopes, how to send it |
| §2 — MCP setup | Installing `tod`, wiring it into AI clients |
| §3 — REST API basics | Bearer auth, common endpoints |
| §4 — Query language overview | Syntax skeleton + worked examples |
| §5 — Reference formats | `#42`, `project#42`, `KEY-42` |
| §6 — Recipes | Triage, ship-a-fix, debug build, run-local CI |
| §7 — Guided prompt templates | The four built-in `tod mcp` prompts |
| §8 — Safety & good manners | Read-before-write, blast-radius rules |
| §9 — Troubleshooting | Common errors and fixes |
| **Appendix A** | Full MCP tool catalogue (~25 tools, every parameter) |
| **Appendix B** | REST API reference (auth, projects, issues, PRs, builds, webhooks, errors) |
| **Appendix C** | Query language grammar with field tables for every entity type |

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

## Compatibility

| Component | Required version |
| --- | --- |
| OneDev server | **13.0+** for the MCP path; **any version** for the REST path |
| `tod` CLI | latest release from `code.onedev.io/onedev/tod` |
| MCP-aware AI client | Cursor, Claude Desktop, Claude Code, Continue, or any client supporting the MCP protocol |
| Skill loaders | Any platform that ingests Markdown skills — Claude.ai, Hechicha/Paperclip, custom agent runners. The single-file design means subdirectory support is **not** required. |

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