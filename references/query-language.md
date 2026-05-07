# OneDev Query Language Reference

OneDev exposes a single, ANTLR-built query language used for issues, pull requests, builds, projects, commits, and packages. Same grammar, different field sets per entity type. This file is the reference Claude consults when composing non-trivial queries for `queryIssues`, `queryPullRequests`, `queryBuilds`, REST `?query=...` parameters, or saved-query strings in the OneDev UI.

---

## General grammar

```
<criterion> [<and|or> <criterion>]* [order by <Field> <asc|desc> [, <Field> <asc|desc>]*]
```

A `<criterion>` is one of:

```
"<Field>" <operator> "<value>"
"<Field>" <operator> #<reference>
"<Field>" <operator>                            # for unary operators like "is empty"
<built-in clause>                               # like `fixed between build "v1.0" and build "v1.1"`
( <criterion> [and|or <criterion>]* )           # parentheses for grouping
```

### Quoting rules
- **Field names** are always wrapped in `"double quotes"` — even single-word ones like `"State"`.
- **String values** are also `"double quotes"`.
- **Issue/PR/build references** use `#NUMBER` and are **not** quoted.
- **Numeric values** for sizes/counts are not quoted.
- Boolean operators `and`, `or`, `is`, `is not` are lowercase, unquoted.

### Operator vocabulary

| Operator | Meaning | Works on |
| --- | --- | --- |
| `is` | equals | enums, refs, users |
| `is not` | not equal | same |
| `contains` | substring match | text fields (Title, Description, Comment) |
| `is greater than` | `>` | dates, numbers |
| `is less than` | `<` | dates, numbers |
| `is greater than or equal to` | `>=` | dates, numbers |
| `is less than or equal to` | `<=` | dates, numbers |
| `is empty` | unary; field has no value | optional fields |
| `is not empty` | unary | optional fields |
| `is current` | references "this project" | `Project` field |
| `is me` | references the calling user | user fields |
| `is previous` | references the previous successful similar build | `Build` field |

### Date & time values
String dates accept absolute or relative forms:
- Absolute: `"2026-05-07"`, `"2026-05-07 10:30"`.
- Relative: `"1 day ago"`, `"2 weeks ago"`, `"yesterday"`, `"today"`, `"last month"`.
- `getUnixTimestamp` MCP tool can resolve any of these to epoch ms if you need to do arithmetic.

### Sorting
Append `order by "<Field>" asc|desc` once at the end. Chain with commas for tie-breakers:

```
"State" is "Open" order by "Priority" desc, "Submit Date" asc
```

---

## Issue fields

Common built-in fields (custom workflows add more):

| Field | Type | Example value |
| --- | --- | --- |
| `State` | enum | `"Open"`, `"In Progress"`, `"Closed"`, `"Released"` |
| `Type` | enum | `"Bug"`, `"New Feature"`, `"Improvement"`, `"Task"` |
| `Priority` | enum | `"Critical"`, `"Major"`, `"Normal"`, `"Minor"` |
| `Project` | project | quoted name, or `is current` |
| `Submitter` | user | login name, or `is me` |
| `Assignees` | user list | login name, or `is me` |
| `Title` | text | use with `contains` |
| `Description` | text | use with `contains` |
| `Comment` | text | use with `contains` |
| `Number` | numeric | `is greater than 100` |
| `Submit Date` | date | `is greater than "1 week ago"` |
| `Update Date` | date | same |
| `Last Activity Date` | date | same |
| `Iteration` | iteration name | `"Sprint 12"` |
| `Label` | label name | `"good-first-issue"` |
| `Parent Issue` | reference | `is #4` |
| `Sub Issues` | reference | `has any` style — see link clauses below |
| `Estimated Time` | duration | `is greater than "2 hours"` |
| `Spent Time` | duration | same |

### Issue-link clauses

OneDev exposes link relationships as queryable clauses:

```
"Sub Issues" is #4                       # this issue is a sub-issue of #4
"Parent Issue" is #4                     # parent is #4
"Related" is #5                          # related to #5
"any sub issue" is "Open"                # has at least one open sub-issue
"all sub issues" is "Closed"             # every sub-issue is closed
```

### Examples

```
"State" is "Open" and "Assignees" is me
"Type" is "Bug" and "Priority" is "Critical" and "State" is not "Closed"
"Submit Date" is greater than "7 days ago" and "Submitter" is "alice"
"Title" contains "login" or "Description" contains "login"
"Iteration" is "Sprint 12" and "all sub issues" is "Closed"
"State" is "Open" order by "Priority" desc, "Submit Date" asc
```

---

## Pull-request fields

| Field | Type | Notes |
| --- | --- | --- |
| `State` | enum | `"Open"`, `"Merged"`, `"Discarded"` |
| `Number` | numeric | |
| `Title` | text | |
| `Description` | text | |
| `Source Branch` | string | |
| `Target Branch` | string | |
| `Source Project` | project | |
| `Target Project` | project | |
| `Submitter` | user | `is me` |
| `Assignees` | user list | |
| `Reviewers` | user list | |
| `To Be Reviewed By` | user | who still owes a review; `is me` is the canonical "my queue" filter |
| `Submit Date` | date | |
| `Update Date` | date | |
| `Merge Strategy` | enum | as in MCP `createPullRequest` |

```
"State" is "Open" and "To Be Reviewed By" is me
"State" is "Open" and "Target Branch" is "main"
"Submitter" is me and "State" is "Open"
"Update Date" is greater than "1 day ago" order by "Update Date" desc
```

---

## Build fields

| Field | Type | Notes |
| --- | --- | --- |
| `Status` | enum | `"Successful"`, `"Failed"`, `"Cancelled"`, `"In Progress"`, `"Waiting"`, `"Timed Out"` |
| `Job` | string | job name from build spec |
| `Number` | numeric | |
| `Branch` | string | |
| `Tag` | string | |
| `Commit` | hash | |
| `Submitter` | user | `is me` |
| `Submit Date` | date | |
| `Pending Date` | date | |
| `Running Date` | date | |
| `Finish Date` | date | |
| `Version` | string | the `buildVersion` produced by the spec |

```
"Status" is "Failed" and "Branch" is "main"
"Job" is "ci" and "Submit Date" is greater than "today"
"Tag" is "v1.3.0" and "Status" is "Successful"
```

### Cross-build clauses
Two especially useful built-in clauses:

```
fixed in build #137                                    # issues fixed by build #137
fixed between build "v1.2.0" and build "v1.3.0"        # delta between two releases
```

---

## Commit query (used in the code-search UI and `~api/commits`)

| Field | Notes |
| --- | --- |
| `Author` | name or `is me` |
| `Committer` | name or `is me` |
| `Message` | `contains` |
| `Path` | files touched, glob OK: `"src/**/*.java"` |
| `Branch` | |
| `Tag` | |
| `Submit Date` | |

```
"Author" is "alice" and "Path" is "src/auth/**" and "Submit Date" is greater than "30 days ago"
"Message" contains "fix" and "Branch" is "main"
```

---

## Project query

| Field | Notes |
| --- | --- |
| `Name` | use with `contains` |
| `Path` | full slash-separated path for nested projects |
| `Service Desk Email` | for ticketing projects |
| `Last Activity Date` | |

---

## Common pitfalls

- **Forgetting the quotes around field names.** `State is "Open"` is invalid — must be `"State" is "Open"`.
- **Using `=` instead of `is`.** OneDev's grammar is word-based, not symbol-based.
- **Quoting issue references.** `#42` not `"#42"`.
- **Case sensitivity in enum values.** `"in progress"` ≠ `"In Progress"`. Match the workflow exactly.
- **Spaces in field names** (e.g. `"Submit Date"`) — keep them inside the quotes.
- **URL-encoding.** When sending queries through REST `?query=...`, encode quotes as `%22`, spaces as `+` or `%20`.

---

## Tip: let the UI build it for you

When unsure about field names or legal values, the OneDev web UI's query bar autocompletes everything. Build the query interactively, then copy the resulting string verbatim into the API or MCP call. The string the UI shows is the exact string the parser accepts.
