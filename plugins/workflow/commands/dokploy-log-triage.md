---
description: Fetch Dokploy deployment logs for an app and diagnose known failure patterns (OOM, IPv6 healthcheck, port conflict, missing env var, build cache). Proposes a concrete fix.
allowed-tools: Bash, Read, mcp__dokploy-mcp__application-one, mcp__dokploy-mcp__application-readAppMonitoring, mcp__dokploy-mcp__project-all
---

Triage a failed / flapping Dokploy deployment. Read logs, match against known failure signatures, propose a fix.

## When to use

- User says "the deploy failed", "Dokploy healthcheck is red", "why did it OOM", "build is stuck", or opens a session from a repo with a Dokploy deployment issue.
- Applicable to any app deployed on a Dokploy VPS. Opinionated for stacks like Next.js + Dokploy on a self-hosted VPS.

## Steps

1. **Identify the app**
   - If the user didn't name it, list projects via `mcp__dokploy-mcp__project-all` and ask which. Otherwise fetch directly with `mcp__dokploy-mcp__application-one`.
2. **Pull recent monitoring + logs**
   - `mcp__dokploy-mcp__application-readAppMonitoring` for the application ID.
   - Fetch the last deployment's build + run logs (Dokploy UI paste from user if MCP can't fetch them directly — ask for last 200 lines).
3. **Pattern match** (tier 1 = cheap regex, tier 2 = context-aware reading)

   **Tier 1 signatures (match first)**
   | Pattern in logs | Diagnosis | Fix |
   |---|---|---|
   | `killed` + memory > 90% or `Out of memory` / `OOMKilled` | Container OOMed | Bump `memory` in Dokploy app config (e.g., 512M → 1G); check for a memory leak if it recurs every N hours |
   | `wget: bad address 'localhost'` or healthcheck timing out on Alpine | IPv6 localhost resolution — Alpine busybox resolves `localhost` to `::1` first | Change healthcheck to `127.0.0.1` in the Dockerfile `HEALTHCHECK` line |
   | `bind: address already in use` / `Port X is already in use` | Port conflict with existing container | Stop / remove the orphan container, or let Dokploy assign the port via its reverse proxy (don't hardcode host ports) |
   | `environment variable ... is not defined` / `undefined` at runtime | Missing env var | List expected vars from the repo's `.env.example`, compare to Dokploy app `Environment` tab, add the missing ones |
   | `Module not found` / `Cannot find module` right after `npm ci` | Lockfile drift or missing dep | Run `pnpm install` locally, commit updated lockfile, redeploy |
   | `no space left on device` | VPS disk full | `docker system prune -af` on the VPS; consider a cron to prune weekly |
   | `413 Request Entity Too Large` in Traefik/Nginx access logs | Reverse proxy body limit | Raise `client-max-body-size` / set `traefik.http.middlewares` annotation |
   | `fetch failed` to a service on the same VPS | Wrong inter-service URL | Use the Docker service name (e.g., `http://db:5432`), not `localhost`, inside the compose network |
   | `SSL_ERROR` / `no valid certification path` on first start | Cert not yet issued or stapling race | Wait 60s for Traefik/Let's Encrypt; if persists, check DNS A record points to the VPS |

   **Tier 2 (context-aware)**
   - If no tier-1 match: read the last 100 lines, quote the first 3 error-like lines, and form a specific hypothesis (not a generic "check logs").
4. **Propose the fix**
   - Be concrete: exact file path + line if it's a code change, exact Dokploy UI step if it's a config change.
   - For Next.js 16 apps, common misconfigurations to check: missing `output: "standalone"` in `next.config.ts` when using a minimal Docker runtime, RSC boundaries polluting the client bundle, ORM column-case mismatches between env `DATABASE_URL` and schema.
5. **Offer to apply**
   - If the fix is a code edit, offer to open a branch + PR (don't commit directly to main).
   - If it's a Dokploy config change, list the exact clicks / API call (`mcp__dokploy-mcp__application-saveEnvironment`, etc.) but require explicit confirmation before writing.

## Output format

```
# Dokploy triage — <app name>

**Signal** — <one line: OOM / healthcheck / port / env / build / net / cert / other>

**Evidence**
```
<3-5 most informative log lines>
```

**Diagnosis** — <1-2 sentences, specific>

**Fix**
- <concrete step 1>
- <concrete step 2>

**To apply**
- (a) I can open a PR on branch `fix/<slug>` with the code change.
- (b) You apply the Dokploy config change in UI → [path].
```

## Guardrails

- **Smoke test after fix**: if the app has a UI, include a UI-driven smoke test step in the PR body — don't declare a deploy fixed until the user has exercised the golden path.
- Never propose migrating off the user's chosen hosting stack (e.g., Dokploy → Vercel) unless they ask.
- Never print secrets, tokens, or DSNs from logs. Redact `*****` in any quoted line containing `=` after a key that looks sensitive (`DATABASE_URL`, `*_TOKEN`, `*_SECRET`, `*_KEY`).
