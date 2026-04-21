---
description: Create multiple GitHub issues from a template + item list. Previews all issues before any creation, supports labels / milestone / assignee / epic linking.
allowed-tools: Bash, Read, Write
---

Create several GitHub issues in one go, with a strict preview-before-create flow.

## When to use

- User says "ouvre 5 issues pour X", "crée ces issues", "batch issues", "file these as GitHub issues", or provides a list of items to track.

## Input contract

The user provides:
- **Repo** — `owner/name`. Required. Confirm with `gh repo view <owner/name>` if unfamiliar.
- **Items** — a list. Each item minimally has a `title`; may have `body`, `labels`, `assignee`, `milestone`, `epic`.
- **Template** (optional) — body template with `{{placeholders}}` substituted per item. If omitted, use the user's one-line description as the body.
- **Common labels** (optional) — applied to every issue on top of per-item labels.

If anything is missing, ask **one** consolidated question before proceeding — not one question per field.

## Steps

1. **Pre-flight**
   - `gh auth status` — confirm authenticated on the right account. Abort if not.
   - `gh repo view <repo> --json name,visibility,hasIssuesEnabled` — confirm issues are enabled. Abort if not.
   - If `visibility = PUBLIC`: **hard gate**. Read the rule `projects/.claude/rules/product-issue-tracking.md` if present, and scan every title/body for: real names, client names, tenant IDs, resource UUIDs, internal hostnames/IPs, emails, service accounts, secrets, private file paths. Replace every hit with a neutral placeholder (`<client>`, `<source-id>`, `<email>`, etc.). If you cannot safely anonymize, stop and tell the user what to redact.
   - If repo has any existing labels relevant to epic/priority/effort (`priority:P*`, `effort:*`, `epic:*`), reuse them; do not auto-create new labels.
2. **Preview table**
   - Render a Markdown table: `#`, `Title`, `Labels`, `Assignee`, `Milestone`, `Body preview (first 80 chars)`.
   - Show the preview in chat. Do **not** create anything yet.
3. **Ask for confirmation** — wait for explicit "go" / "ok" / "ship it". Any correction from the user means re-render the preview.
4. **Create**
   - Loop items. For each, run `gh issue create --repo <repo> --title "…" --body-file <tmp> --label "…" --assignee "…" --milestone "…"`.
   - Use `--body-file` (not `--body`) to avoid shell escaping issues. Write to a tmp file, delete after.
   - On any failure, stop the loop and report: how many were created, which item failed, the error.
5. **Report**
   - Markdown list of created issues with URLs returned by `gh` (`https://github.com/<repo>/issues/<n>`).
   - If an epic slug was provided: post a single comment on the epic issue linking all new issues (`- #<n> <title>`). Find the epic issue by title or `epic:<slug>` label search — ask if ambiguous.

## Guardrails

- **Public repos**: anonymization is mandatory. When in doubt, redact. Per `product-issue-tracking.md`: if sensitive data is accidentally published, delete the issue (`gh issue delete <n> --repo <repo> --yes`) and recreate anonymized — edits do not remove content from GitHub's history.
- Never use `gh issue create` inside a `for` loop without the preview step, even for 2 issues.
- Never pass user-supplied strings directly to the shell — always via `--body-file`.
- If the user asks to create issues without titles, refuse and ask for at least a one-line title per item.

## Example preview (render like this)

```
Repo: example-org/app · 4 issues to create · common labels: [triage, good-first-issue]

| # | Title                                  | Labels                 | Assignee | Body (80ch)                     |
|---|----------------------------------------|------------------------|----------|---------------------------------|
| 1 | Add source feed to scanner targets     | feature, priority:P2   | <user>   | As a user I want to scan…       |
| 2 | Fix panic on empty input               | bug, priority:P1       | <user>   | Reproduction: run the CLI…      |
| 3 | Document deployment pipeline           | docs                   | —        | Update README with deployment…  |
| 4 | Switch healthcheck to 127.0.0.1        | bug, infra             | <user>   | Alpine wget resolves localhost… |
```
