# trustin-ai

A small marketplace of Claude Code plugins focused on identity & access management automation.

## Plugins

- [`onelogin`](./plugins/onelogin/) — 14 agents for OneLogin operations (helpdesk, JML, admin, audit). Powered by the OneLogin MCP server.
- [`sailpoint`](./plugins/sailpoint/) — SailPoint ISC transform authoring skills, reporting agents, and maintenance of the OSS [`terraform-provider-sailpoint-isc-community`](https://github.com/AnasSahel/terraform-provider-sailpoint-isc-community) provider.

Each plugin has its own `README.md` with a detailed breakdown.

## Install

```
/plugin marketplace add AnasSahel/trustin-ai
/plugin install onelogin@trustin-ai
/plugin install sailpoint@trustin-ai
```

## Versioning — per-plugin monorepo tags

Each plugin evolves at its own cadence. Tags follow the Go-modules monorepo convention:

```
<plugin>/v<MAJOR>.<MINOR>.<PATCH>
```

Examples: `onelogin/v0.1.0`, `sailpoint/v0.2.0`.

This lets `onelogin` and `sailpoint` release independently without noisy cross-bumps.

## Releasing

For a plugin named `<plugin>` moving to version `<X.Y.Z>`:

1. Bump `plugins/<plugin>/.claude-plugin/plugin.json` → `"version": "<X.Y.Z>"`.
2. Commit with a conventional message scoped to the plugin:
   ```
   git commit -m "feat(<plugin>): <what's new>"
   ```
3. Tag and push:
   ```
   git tag -a <plugin>/v<X.Y.Z> -m "<plugin>/v<X.Y.Z>"
   git push origin main <plugin>/v<X.Y.Z>
   ```
4. Create the GitHub release:
   ```
   gh release create <plugin>/v<X.Y.Z> \
     --title "<plugin> v<X.Y.Z>" \
     --notes "..."
   ```

## Anonymization gate — this repo is public

Before any commit, every file must be free of:

- Real client or company names
- Tenant IDs, source IDs, UUIDs, workflow IDs
- Internal hostnames, DNs, IPs (non-RFC5737), emails (non-example.com)
- Service accounts, secrets, tokens — even redacted snippets
- Private GitHub org references
- Personal memory references (`feedback_*`)
- Hardcoded paths leaking a project or client name

Use placeholders: `<client>`, `<tenant>`, `<source-id>`, `<email>`, IPs from RFC-5737 (`192.0.2.0/24`, `198.51.100.0/24`, `203.0.113.0/24`), `example.com` domains.

If sensitive content ever lands on `main`, editing the file does **not** remove it from git history — force-push a cleaned history or delete and recreate the repo.
