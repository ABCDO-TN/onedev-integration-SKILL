# OneDev REST API — Practical Reference

OneDev exposes a full REST API at `https://<host>/~api/...`. Every install ships interactive, version-accurate documentation at:

```
https://<host>/~help/api
```

When uncertain about an endpoint shape on a specific server, fetch that page first — it reflects the exact version installed, including any custom plugins.

This file covers the endpoints you'll need most often.

---

## Authentication

Personal access token, sent as Bearer or Basic auth. The username is irrelevant for Basic — only the token value matters.

```bash
TOKEN="your-token-here"
HOST="https://onedev.example.com"

# Bearer (preferred)
curl -H "Authorization: Bearer $TOKEN" "$HOST/~api/users/me"

# Basic — equivalent
curl -u anything:$TOKEN "$HOST/~api/users/me"
```

A `200` from `/~api/users/me` proves the token is valid. A `401` means it isn't (or it's been revoked).

---

## Discovery

| What | How |
| --- | --- |
| Live API docs | `GET /~help/api` (HTML) |
| Server version | `GET /~api/server/version` |
| Current user | `GET /~api/users/me` |
| List projects you can see | `GET /~api/projects` |
| Find a project by name | `GET /~api/projects?name=my-app` |

---

## Projects

```bash
# List
curl -H "Authorization: Bearer $TOKEN" "$HOST/~api/projects?count=100"

# Get one (by ID)
curl -H "Authorization: Bearer $TOKEN" "$HOST/~api/projects/123"

# Create
curl -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"name":"new-project","description":"..."}' \
  "$HOST/~api/projects"
```

---

## Issues

```bash
# Query (use OneDev's query language in the `query` param, URL-encoded)
curl -H "Authorization: Bearer $TOKEN" \
  "$HOST/~api/issues?query=%22State%22+is+%22Open%22&count=25"

# Get by ID
curl -H "Authorization: Bearer $TOKEN" "$HOST/~api/issues/42"

# Create
curl -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{
    "projectId": 1,
    "title": "Login button broken on Safari",
    "description": "Reproduces on macOS 14, Safari 17.x ...",
    "fieldValues": {
      "Type": "Bug",
      "Priority": "Major"
    }
  }' \
  "$HOST/~api/issues"

# Add comment
curl -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"content":"Confirmed, working on a fix."}' \
  "$HOST/~api/issues/42/comments"

# Transition state
curl -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"state":"In Progress","comment":"Picked up"}' \
  "$HOST/~api/issues/42/state-transitions"
```

`fieldValues` keys are the **custom field names** as defined in the project's issue workflow. Names — and the legal value sets for enum-typed fields like `Priority` — are server-configurable. To discover what a project supports, GET an existing issue and inspect the `fields` array.

---

## Pull Requests

```bash
# Query
curl -H "Authorization: Bearer $TOKEN" \
  "$HOST/~api/pull-requests?query=%22State%22+is+%22Open%22"

# Create
curl -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{
    "targetProjectId": 1,
    "sourceProjectId": 1,
    "targetBranch": "main",
    "sourceBranch": "feature/login-fix",
    "title": "Fix Safari login regression",
    "description": "Closes #42"
  }' \
  "$HOST/~api/pull-requests"

# Approve
curl -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"comment":"LGTM"}' \
  "$HOST/~api/pull-requests/7/approve"

# Merge
curl -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"commitMessage":"Fix Safari login regression (#42)"}' \
  "$HOST/~api/pull-requests/7/merge"
```

---

## Builds & CI/CD

The lightweight, scriptable way to kick off a job:

```bash
# trigger-job: token is sent in the URL — only safe over https
curl -X POST "$HOST/~api/trigger-job?project=my-app&job=ci&branch=main&access-token=$TOKEN"

# Job name with a space must be URL-encoded:
#   "Push to GitHub"  →  Push%20to%20GitHub
```

Heavier-weight build inspection:

```bash
curl -H "Authorization: Bearer $TOKEN" "$HOST/~api/builds/137"
curl -H "Authorization: Bearer $TOKEN" "$HOST/~api/builds/137/log"
```

---

## Access tokens (administrative)

Since OneDev 8.5+ access tokens are first-class resources under the user endpoint. Admins can list/create/delete them on behalf of users via the REST API; non-admins can manage their own.

```bash
# List your own tokens
curl -H "Authorization: Bearer $TOKEN" "$HOST/~api/users/me/access-tokens"
```

Exact paths and payload shapes vary by version — confirm at `/~help/api`.

---

## Webhooks

Webhooks are configured **per project** under `Settings → Web Hooks` in the UI, but can also be managed via REST under `/~api/projects/{id}/web-hooks`. Set the target URL, choose event types (push, pull request, issue, build, etc.), and OneDev will POST a JSON payload on each event.

---

## Rate limits & pagination

OneDev does not impose hard rate limits on authenticated REST traffic by default, but list endpoints cap responses at **100 items per call**. For large result sets, paginate with `count` and `offset`:

```
GET /~api/issues?query=...&count=100&offset=0
GET /~api/issues?query=...&count=100&offset=100
...
```

Stop when a page returns fewer items than `count`.

---

## Error shapes

OneDev returns standard HTTP codes plus a JSON body on errors:

```json
{ "errorMessage": "Unauthorized access to user profiles" }
```

Common ones:
- `400` — validation error (bad field name, wrong enum value).
- `401` — missing/invalid token.
- `403` — token is valid but lacks the permission for this resource.
- `404` — resource doesn't exist (or token can't see it — OneDev sometimes returns 404 instead of 403 to avoid leaking existence).
- `409` — conflict (e.g. trying to create something with a name already in use).
