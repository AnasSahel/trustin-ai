---
name: gstack-office-hours-preamble
version: 1.0.0
description: |
  One-time setup for gstack office hours. Handles first-run prompts that the
  original skill embedded in every invocation: Boil the Lake intro, telemetry
  opt-in, proactive behavior opt-in, CLAUDE.md routing rules, and browse binary
  setup. Run this once per machine. After setup, /office-hours-lite and
  /office-hours-deep skip all first-time prompts.
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - AskUserQuestion
---

## Check Current State

```bash
_LAKE_SEEN=$([ -f ~/.gstack/.completeness-intro-seen ] && echo "yes" || echo "no")
_TEL_PROMPTED=$([ -f ~/.gstack/.telemetry-prompted ] && echo "yes" || echo "no")
_PROACTIVE_PROMPTED=$([ -f ~/.gstack/.proactive-prompted ] && echo "yes" || echo "no")
_TEL=$(~/.claude/skills/gstack/bin/gstack-config get telemetry 2>/dev/null || true)
_PROACTIVE=$(~/.claude/skills/gstack/bin/gstack-config get proactive 2>/dev/null || echo "true")

# Check CLAUDE.md routing
_HAS_ROUTING="no"
if [ -f CLAUDE.md ] && grep -q "## Skill routing" CLAUDE.md 2>/dev/null; then
  _HAS_ROUTING="yes"
fi
_ROUTING_DECLINED=$(~/.claude/skills/gstack/bin/gstack-config get routing_declined 2>/dev/null || echo "false")

# Browse binary
_ROOT=$(git rev-parse --show-toplevel 2>/dev/null)
B=""
[ -n "$_ROOT" ] && [ -x "$_ROOT/.claude/skills/gstack/browse/dist/browse" ] && B="$_ROOT/.claude/skills/gstack/browse/dist/browse"
[ -z "$B" ] && B=~/.claude/skills/gstack/browse/dist/browse
[ -x "$B" ] && _BROWSE="ready" || _BROWSE="needs_setup"

echo "LAKE_INTRO: $_LAKE_SEEN"
echo "TEL_PROMPTED: $_TEL_PROMPTED"
echo "TELEMETRY: ${_TEL:-off}"
echo "PROACTIVE_PROMPTED: $_PROACTIVE_PROMPTED"
echo "PROACTIVE: $_PROACTIVE"
echo "HAS_ROUTING: $_HAS_ROUTING"
echo "ROUTING_DECLINED: $_ROUTING_DECLINED"
echo "BROWSE: $_BROWSE"
```

If everything is already configured (all `yes`/`true`/`ready`), tell the user:
"Setup already complete. Run `/office-hours-lite` to start." and stop.

---

## Step 1: Boil the Lake Intro

If `LAKE_INTRO` is `no`:

Tell the user: "gstack follows the **Boil the Lake** principle ... always do the
complete thing when AI makes the marginal cost near-zero."

Ask: "Want to read the essay?" If yes:
```bash
open https://garryslist.org/posts/boil-the-ocean
```

Always mark as seen:
```bash
mkdir -p ~/.gstack
touch ~/.gstack/.completeness-intro-seen
```

---

## Step 2: Telemetry

If `TEL_PROMPTED` is `no`:

Ask via AskUserQuestion:

> Help gstack get better! Community mode shares usage data (which skills you use,
> how long they take, crash info) with a stable device ID. No code, file paths,
> or repo names are ever sent. Change anytime with `gstack-config set telemetry off`.
>
> A) Help gstack get better! (recommended)
> B) No thanks

If A: `~/.claude/skills/gstack/bin/gstack-config set telemetry community`

If B: follow-up AskUserQuestion:

> How about anonymous mode? Just a counter that helps us know someone used gstack.
> No unique ID, no way to connect sessions.
>
> A) Sure, anonymous is fine
> B) No thanks, fully off

If B→A: `~/.claude/skills/gstack/bin/gstack-config set telemetry anonymous`
If B→B: `~/.claude/skills/gstack/bin/gstack-config set telemetry off`

```bash
touch ~/.gstack/.telemetry-prompted
```

---

## Step 3: Proactive Behavior

If `PROACTIVE_PROMPTED` is `no`:

Ask via AskUserQuestion:

> gstack can proactively suggest skills while you work ... like suggesting /qa
> when you say "does this work?" or /investigate when you hit a bug.
>
> A) Keep it on (recommended)
> B) Turn it off — I'll type /commands myself

If A: `~/.claude/skills/gstack/bin/gstack-config set proactive true`
If B: `~/.claude/skills/gstack/bin/gstack-config set proactive false`

```bash
touch ~/.gstack/.proactive-prompted
```

---

## Step 4: CLAUDE.md Routing Rules

If `HAS_ROUTING` is `no` AND `ROUTING_DECLINED` is `false`:

Ask via AskUserQuestion:

> gstack works best when your project's CLAUDE.md includes skill routing rules.
> This tells Claude to use specialized workflows instead of answering directly.
> One-time addition, about 15 lines.
>
> A) Add routing rules to CLAUDE.md (recommended)
> B) No thanks, I'll invoke skills manually

If A: append to CLAUDE.md:

```markdown

## Skill routing

When the user's request matches an available skill, ALWAYS invoke it using the Skill
tool as your FIRST action. Do NOT answer directly.

Key routing rules:
- Product ideas, brainstorming → invoke office-hours-lite
- Bugs, errors, "why is this broken" → invoke investigate
- Ship, deploy, push, create PR → invoke ship
- QA, test the site, find bugs → invoke qa
- Code review, check my diff → invoke review
- Update docs after shipping → invoke document-release
- Design system, brand → invoke design-consultation
- Visual audit, design polish → invoke design-review
- Architecture review → invoke plan-eng-review
```

Then: `git add CLAUDE.md && git commit -m "chore: add gstack skill routing rules"`

If B: `~/.claude/skills/gstack/bin/gstack-config set routing_declined true`

---

## Step 5: Browse Binary

If `BROWSE` is `needs_setup`:

Ask: "gstack browse needs a one-time build (~10 seconds). OK to proceed?"

If yes:
```bash
cd ~/.claude/skills/gstack/browse && ./setup
```

If `bun` is missing, install it first:
```bash
if ! command -v bun >/dev/null 2>&1; then
  BUN_VERSION="1.3.10"
  BUN_INSTALL_SHA="bab8acfb046aac8c72407bdcce903957665d655d7acaa3e11c7c4616beae68dd"
  tmpfile=$(mktemp)
  curl -fsSL "https://bun.sh/install" -o "$tmpfile"
  actual_sha=$(shasum -a 256 "$tmpfile" | awk '{print $1}')
  if [ "$actual_sha" != "$BUN_INSTALL_SHA" ]; then
    echo "ERROR: bun install script checksum mismatch" >&2
    rm "$tmpfile"; exit 1
  fi
  BUN_VERSION="$BUN_VERSION" bash "$tmpfile"
  rm "$tmpfile"
fi
```

---

## Done

Tell the user: "Setup complete. Run `/office-hours-lite` to start brainstorming."
