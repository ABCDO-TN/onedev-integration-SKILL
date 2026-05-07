# OneDev MCP Tools — Full Reference

Every tool the `tod mcp` server exposes, organised by domain. Tool names are case-sensitive. All "reference" parameters accept the three forms documented in `SKILL.md` §5: `#42`, `project#42`, or `KEY-42`.

---

## Issue Management

### `queryIssues`
Query issues. Returns a list.
- `query` (optional) — OneDev issue query syntax (see `query-language.md`).
- `project` (optional) — defaults to current project.
- `count` (optional) — default 25, max 100.
- `offset` (optional) — for pagination.

### `getIssue`
Full detail of one issue, including custom fields.
- `issueReference` (required).

### `getIssueComments`
List comments on an issue.
- `issueReference` (required).

### `createIssue`
Create a new issue.
- `title` (required).
- `description` (optional, Markdown).
- `project` (optional) — defaults to current.
- `iterations` (optional) — array of iteration names to schedule into.
- `confidential` (optional, boolean).
- Any number of **custom fields** the project defines (Type, Priority, Assignees, etc. — names depend on the workflow).

### `editIssue`
Update an existing issue. Same shape as `createIssue` but with `issueReference` instead of `project`. Only fields you provide are changed.

### `changeIssueState`
Transition an issue. Honours state-transition side effects defined in the workflow.
- `issueReference` (required).
- `state` (required) — exact state name (e.g. `"In Progress"`, `"Closed"`).
- `comment` (optional).

### `linkIssues`
Create a typed link between two issues.
- `sourceIssueReference` (required).
- `targetIssueReference` (required).
- `linkName` (required) — one of: `Sub Issues`, `Parent Issue`, `Related` (custom link types may exist per project).

### `addIssueComment`
Plain comment. **For state changes use `changeIssueState`, not a comment.**
- `issueReference` (required).
- `commentContent` (required).

### `logWork`
Time tracking entry against an issue.
- `issueReference` (required).
- `spentHours` (required, number).
- `comment` (optional).

---

## Pull Request Management

### `queryPullRequests`
Same shape as `queryIssues`.
- `query`, `project`, `count`, `offset`.

### `getPullRequest`
Detail for one PR.
- `pullRequestReference` (required).

### `getPullRequestComments`
General PR comments (the conversation thread).
- `pullRequestReference` (required).

### `getPullRequestCodeComments`
Inline code review comments only.
- `pullRequestReference` (required).

### `getPullRequestFileChanges`
Patch-format diff of the PR.
- `pullRequestReference` (required).
- `sinceLastReview` (required, boolean) — when true, only the delta since the previous review.

### `getPullRequestFileContent`
Full file content as it appears in the PR.
- `pullRequestReference` (required).
- `filePath` (required) — repo-root-relative.
- `revision` (required) — `initial`, `latest`, or `lastReviewed`.

### `createPullRequest`
- `sourceBranch` (required).
- `title` (required).
- `description` (optional).
- `sourceProject` (optional) — defaults to current.
- `targetProject` (optional) — defaults to original parent if source is a fork.
- `targetBranch` (optional) — defaults to repo default branch.
- `assignees` (optional) — array of login names.
- `reviewers` (optional) — array of login names.
- `mergeStrategy` (optional) — one of: `CREATE_MERGE_COMMIT`, `CREATE_MERGE_COMMIT_IF_NECESSARY`, `SQUASH_SOURCE_BRANCH_COMMITS`, `REBASE_SOURCE_BRANCH_COMMITS`.

### `editPullRequest`
- `pullRequestReference` (required).
- Optional: `title`, `description`, `assignees`, `addReviewers`, `removeReviewers`, `mergeStrategy`, `autoMerge` (boolean), `autoMergeCommitMessage`.

### `processPullRequest`
The action verb. Use this for state changes.
- `pullRequestReference` (required).
- `operation` (required) — one of: `approve`, `requestChanges`, `merge`, `discard`, `reopen`, `deleteSourceBranch`, `restoreSourceBranch`.
- `comment` (optional).

### `addPullRequestComment`
- `pullRequestReference` (required).
- `commentContent` (required).

### `checkoutPullRequest`
Check out the PR branch in the working directory (local Git operation under the hood).
- `pullRequestReference` (required).

---

## Build Management

### `queryBuilds`
- `query`, `project`, `count`, `offset` — same shape as the others.

### `getBuild`
- `buildReference` (required).

### `getBuildLog`
Full text log of a build run.
- `buildReference` (required).

### `getBuildFileContent`
Read a file from the build's checkout.
- `buildReference` (required).
- `filePath` (required).

### `getFileChangesSincePreviousSuccessfulSimilarBuild`
Highest-signal triage tool — diff of the source between the failing build and the last green build of the same job.
- `buildReference` (required).

### `runJob`
Trigger a job against a branch or tag.
- `jobName` (required).
- `branch` (optional) — exclusive with `tag`.
- `tag` (optional) — exclusive with `branch`.
- `params` (optional) — array of `key=value` strings.

### `runLocalJob`
Trigger a job against the current working directory's uncommitted changes (the `tod run-local` flow).
- `jobName` (required).
- `params` (optional).

---

## Build Spec Management

### `getBuildSpecSchema`
Returns the JSON schema describing valid `.onedev-buildspec.yml` shape for the connected server's version. Always call this before generating a build spec from scratch — schemas evolve between major versions.

### `checkBuildSpec`
Validates and (if needed) auto-migrates the build spec in the working directory.

---

## Project & System

### `getCurrentProject`
Returns the OneDev project the working directory is associated with.

### `getCurrentRemote`
Returns the Git remote URL pointing at OneDev.

### `getWorkingDir`
Returns the current working directory.

### `setWorkingDir`
- `workingDir` (required) — absolute path.

### `getLoginName`
- `userName` (optional) — if omitted, returns the calling user's login.

### `getUnixTimestamp`
Convert natural-language dates to epoch ms — useful for query construction.
- `dateTimeDescription` (required) — e.g. `"today"`, `"next month"`, `"2026-01-01"`.

---

## Prompts (high-level workflows)

These appear under `/` in MCP-aware chat clients. Prefer them over hand-rolling tool sequences when your task fits.

### `change-issue-state`
- `issueReference`, `instruction`.
Parses the instruction, transitions state, runs any state-description side effects (branch creation, reviewer assignment, etc.).

### `edit-build-spec`
- `instruction`.
Fetches schema → reads current spec → applies instruction → validates → saves.

### `investigate-build-problems`
- `buildReference`, optional `instruction`.
Pulls log + spec + diff-since-green, analyses, recommends.

### `review-pull-request`
- `pullRequestReference`, `sinceLastReview` (true/false), optional `instruction`.
Reads the diff and key files, decides approve/requestChanges/comment.
