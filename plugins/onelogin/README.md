# onelogin

OneLogin operations toolkit for Claude Code.

## Contents

14 agents, grouped by function:

**Helpdesk (N1)** — read-only or narrow-scope safe actions
- `onelogin-user-lookup` — consolidated user profile (status, MFA, apps, roles, privileges, custom attrs)
- `onelogin-password-invite` — send or generate an invite / password-reset link
- `onelogin-account-unlock` — unlock a locked account after reviewing the lock event context
- `onelogin-mfa-reset` — remove factors, re-enroll, generate temporary MFA tokens

**JML (lifecycle)** — joiner / mover / leaver
- `onelogin-joiner` — create a new user, assign roles/groups, send invite
- `onelogin-mover` — compute delta between old and new profile, apply role/attribute changes after diff confirmation
- `onelogin-leaver` — snapshot accesses, disable, revoke tokens/sessions, remove roles; hard delete behind DOUBLE confirmation

**Admin**
- `onelogin-app-admin` — SAML/OIDC app CRUD, app rules, mappings; all writes diff before/after + confirm
- `onelogin-smart-hooks-dev` — Smart Hook dev lifecycle (create, update, env vars, log inspection)

**Audit / reporting** — read-only
- `onelogin-access-review` — explode app/role/privilege → effective users list (SOC2/ISO reviews)
- `onelogin-mfa-coverage-report` — users without MFA, users with weak-only factors (SMS/email), factor stats
- `onelogin-app-usage-stats` — login stats per app over a period (totals, unique users, last used)
- `onelogin-incident-investigator` — timeline of events (logins, MFA, IPs, risk) for a user or window
- `onelogin-directory-sync-troubleshooter` — diagnose AD/LDAP/Google/SCIM sync failures

## Requirements

- **OneLogin MCP server** configured (tools prefixed `mcp__onelogin-prd__*`).
- `gh` CLI and git are NOT required — all agents call the OneLogin API via MCP.

## Install (from the trustin-ai marketplace)

```
/plugin install onelogin@trustin-ai
```

## Usage

Invoke an agent by describing the task. Claude Code routes automatically when the description matches:

> "Can you look up the OneLogin profile for <email>?" → `onelogin-user-lookup`
>
> "Reset MFA for <user>, they lost their phone" → `onelogin-mfa-reset`
>
> "Generate an MFA coverage report across the tenant" → `onelogin-mfa-coverage-report`

## Safety notes

- All **write** agents (JML, admin, MFA reset, account unlock) present a plan/diff and wait for explicit confirmation.
- `onelogin-leaver` requires DOUBLE confirmation for hard delete.
- Read-only agents (lookup, audit, stats) are safe to delegate freely.
