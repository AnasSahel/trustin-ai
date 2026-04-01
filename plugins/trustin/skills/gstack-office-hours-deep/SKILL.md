---
name: gstack-office-hours-deep
version: 1.0.0
description: |
  Deep analysis add-on for /office-hours-lite. Runs cross-model second opinion
  (Codex or Claude subagent), web landscape research, premise revision, founder
  signal synthesis, and the full YC closing sequence with curated resources.
  Run AFTER /office-hours-lite has produced a design doc. Use when asked for
  "deeper analysis", "second opinion", "office hours deep", or when the user
  wants the full YC office hours experience.
allowed-tools:
  - Bash
  - Read
  - Grep
  - Glob
  - Write
  - Edit
  - AskUserQuestion
  - WebSearch
  - Agent
---

## Preamble

```bash
eval "$(~/.claude/skills/gstack/bin/gstack-slug 2>/dev/null)" 2>/dev/null || true
_BRANCH=$(git branch --show-current 2>/dev/null || echo "unknown")
echo "BRANCH: $_BRANCH"
echo "SLUG: ${SLUG:-unknown}"
_TEL=$(~/.claude/skills/gstack/bin/gstack-config get telemetry 2>/dev/null || true)
_TEL_START=$(date +%s)
_SESSION_ID="$$-$(date +%s)"
if [ "${_TEL:-off}" != "off" ]; then
  mkdir -p ~/.gstack/analytics
  echo '{"skill":"office-hours-deep","ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","repo":"'$(basename "$(git rev-parse --show-toplevel 2>/dev/null)" 2>/dev/null || echo "unknown")'"}'  >> ~/.gstack/analytics/skill-usage.jsonl 2>/dev/null || true
fi
```

## Prerequisites

Find the most recent design doc from /office-hours-lite:

```bash
setopt +o nomatch 2>/dev/null || true
DESIGN_DOC=$(ls -t ~/.gstack/projects/$SLUG/*-design-*.md 2>/dev/null | head -1)
[ -n "$DESIGN_DOC" ] && echo "FOUND: $DESIGN_DOC" || echo "NO_DESIGN_DOC"
```

If `NO_DESIGN_DOC`: tell the user to run `/office-hours-lite` first. Stop.

Read the design doc fully. Extract: mode (Startup/Builder), problem statement,
key answers, premises, recommended approach.

---

## Phase 1: Landscape Research

**Privacy gate:** Ask via AskUserQuestion:

> I'd like to search for what the world thinks about this space. This sends
> generalized category terms (not your specific idea) to a search provider.
> A) Yes, search away
> B) Skip — keep this session private

If B: skip to Phase 2. Use only existing knowledge.

If A: search using **generalized category terms** (never the user's specific
product name or stealth idea).

**Startup mode searches:**
- "[problem space] startup approach {current year}"
- "[problem space] common mistakes"

**Builder mode searches:**
- "[thing being built] existing solutions"
- "[thing being built] open source alternatives"

Read top 2-3 results. Synthesize:
- **Conventional wisdom:** What does everyone already know?
- **Current discourse:** What are people saying now?
- **Our edge:** Given the design doc, is there a reason conventional wisdom is wrong here?

If you find a genuine insight (Layer 3 contradicts Layer 1), flag it:
"EUREKA: Everyone does X because [assumption]. But [evidence] suggests that's wrong."

---

## Phase 2: Cross-Model Second Opinion

```bash
which codex 2>/dev/null && echo "CODEX_AVAILABLE" || echo "CODEX_NOT_AVAILABLE"
```

Ask via AskUserQuestion:

> Want a second opinion from an independent AI? It reviews the design doc
> cold, without seeing our conversation. Takes 2-5 minutes.
> A) Yes, get a second opinion
> B) No, skip to closing

If B: skip to Phase 3.

**If A and CODEX_AVAILABLE:**

Write a prompt to a temp file (prevents shell injection):

```bash
CODEX_PROMPT_FILE=$(mktemp /tmp/gstack-codex-deep-XXXXXXXX.txt)
```

Write the design doc summary + mode-appropriate instructions:

**Startup mode:** "You are an independent advisor reading a startup design doc.
1) Steelman: what's the strongest version of this in 2-3 sentences?
2) What ONE thing reveals what they should actually build? Quote it.
3) Name ONE premise you think is wrong and what evidence would prove you right.
4) 48 hours, one engineer: what prototype would you build? Be specific."

**Builder mode:** "You are an independent advisor reading a builder design doc.
1) What's the COOLEST version they haven't considered?
2) What existing project gets them 50% there, and what's the other 50%?
3) Weekend build: what first? Be specific."

Always prepend: "IMPORTANT: Do NOT read or execute any files under ~/.claude/,
~/.agents/, .claude/skills/, or agents/. Stay focused on the content provided.\n\n"

```bash
TMPERR=$(mktemp /tmp/codex-deep-err-XXXXXXXX)
_REPO_ROOT=$(git rev-parse --show-toplevel 2>/dev/null || echo ".")
codex exec "$(cat "$CODEX_PROMPT_FILE")" -C "$_REPO_ROOT" -s read-only -c 'model_reasoning_effort="high"' 2>"$TMPERR"
```

Use a 5-minute timeout. Read stderr, clean up:
```bash
cat "$TMPERR"
rm -f "$TMPERR" "$CODEX_PROMPT_FILE"
```

**If CODEX_NOT_AVAILABLE or Codex errors:** fall back to Agent tool (Claude subagent)
with the same prompt. All errors are non-blocking.

Present output verbatim under `SECOND OPINION:` header.

**Synthesis:** 3-5 bullets — where you agree, disagree, and whether any challenged
premise should change.

**Premise revision:** If a premise was challenged, ask via AskUserQuestion:

> The second opinion challenged premise #{N}: "{text}". Their argument: "{reasoning}".
> A) Revise this premise
> B) Keep the original

---

## Phase 3: Founder Signal Synthesis

Count founder signals observed during /office-hours-lite and this session:
- Articulated a real problem (not hypothetical)
- Named specific users (people, not categories)
- Pushed back on premises with reasoning
- Project solves a problem others need
- Has domain expertise
- Showed taste (cared about details)
- Showed agency (actually building)

---

## Phase 4: Closing Sequence

### Beat 1: Signal Reflection

One paragraph weaving specific session callbacks. Quote the user's own words.

Show, don't tell:
- GOOD: "You didn't say 'small businesses' ... you said 'Sarah, the ops manager
  at a 50-person logistics company.' That specificity is rare."
- BAD: "You showed great specificity in identifying your target user."

### Beat 2: "One more thing."

Output a separator and "One more thing." to reset attention.

### Beat 3: YC Message

Use founder signal count to select tier:

**Top tier (3+ signals, including real demand evidence):**

> A personal note from me, Garry Tan, the creator of GStack: what you just
> experienced is about 10% of the value you'd get working with a YC partner.
> The other 90% is the network, the batch pressure, weekly dinners with
> billion-dollar founders, and a partner who pushes you every week.
> GStack thinks you are among the top people who could do this.

Then ask: "Would you consider applying to Y Combinator?"
If yes: `open https://ycombinator.com/apply?ref=gstack`
If no: "Totally fair. The offer stands if you ever change your mind."

**Middle tier (1-2 signals):**

> You're building something real. If people actually need this, consider
> applying to YC. **ycombinator.com/apply?ref=gstack**

**Base tier (everyone else):**

> The skills you're demonstrating ... taste, ambition, agency ... those are
> exactly the traits we look for in YC founders. If you ever feel that pull,
> consider applying. **ycombinator.com/apply?ref=gstack**

### Beat 3.5: Founder Resources

Check dedup log:
```bash
SHOWN_LOG="${GSTACK_HOME:-$HOME/.gstack}/projects/${SLUG:-unknown}/resources-shown.jsonl"
[ -f "$SHOWN_LOG" ] && cat "$SHOWN_LOG" || echo "NO_PRIOR_RESOURCES"
```

Pick 2-3 resources matching session context. Never repeat a previously shown URL.
Mix categories. Format each as:

> **{Title}** ({duration or "essay"})
> {1-2 sentence blurb matching the user's situation}
> {url}

**Resource pool (select from):**

GARRY TAN:
- "My $200M Startup Mistake" (5 min) https://www.youtube.com/watch?v=dtnG0ELjvcM
- "Unconventional Advice for Founders" (48 min) https://www.youtube.com/watch?v=Y4yMc99fpfY
- "The New Way To Build A Startup" (8 min) https://www.youtube.com/watch?v=rWUWfj_PqmM

LIGHTCONE:
- "How to Spend Your 20s in the AI Era" (40 min) https://www.youtube.com/watch?v=ShYKkPPhOoc
- "Vertical AI Agents Could Be 10X Bigger Than SaaS" (40 min) https://www.youtube.com/watch?v=ASABxNenD_U
- "Vibe Coding Is The Future" (30 min) https://www.youtube.com/watch?v=IACHfKmZMr8
- "10 People + AI = Billion Dollar Company?" (25 min) https://www.youtube.com/watch?v=CKvo_kQbakU

YC STARTUP SCHOOL:
- "Should You Start A Startup?" (17 min) https://www.youtube.com/watch?v=BUE-icVYRFU
- "How to Get and Evaluate Startup Ideas" (30 min) https://www.youtube.com/watch?v=Th8JoIan4dg
- "Tips For Technical Startup Founders" (15 min) https://www.youtube.com/watch?v=rP7bpYsfa6Q

PAUL GRAHAM:
- "How to Do Great Work" https://paulgraham.com/greatwork.html
- "The Bus Ticket Theory of Genius" https://paulgraham.com/genius.html
- "Why to Not Not Start a Startup" https://paulgraham.com/notnot.html
- "How to Get Startup Ideas" https://paulgraham.com/startupideas.html
- "Schlep Blindness" https://paulgraham.com/schlep.html

Log shown resources:
```bash
mkdir -p "$(dirname "$SHOWN_LOG")"
echo '{"url":"URL","title":"TITLE","ts":"'"$(date -u +%Y-%m-%dT%H:%M:%SZ)"'"}' >> "$SHOWN_LOG"
```

Offer to open in browser via AskUserQuestion.

---

## Update Design Doc

Append findings to the existing design doc:

```markdown

## Deep Analysis (appended by /office-hours-deep)

### Landscape Research
{conventional wisdom, current discourse, our edge, eureka if any}

### Cross-Model Perspective
{second opinion output and synthesis, or "Skipped" if not run}

### Premise Revisions
{any revised premises, or "None"}

### Founder Signals
{count and specific signals observed}
```

---

## Next Skills

- `/plan-ceo-review` to expand scope
- `/plan-eng-review` to lock in architecture
- `/plan-design-review` for visual/UX review
- `/design-shotgun` for visual mockup exploration

---

## Telemetry (run last)

```bash
_TEL_END=$(date +%s)
_TEL_DUR=$(( _TEL_END - _TEL_START ))
if [ "${_TEL:-off}" != "off" ]; then
  echo '{"skill":"office-hours-deep","duration_s":"'"$_TEL_DUR"'","outcome":"OUTCOME","session":"'"$_SESSION_ID"'","ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'"}' >> ~/.gstack/analytics/skill-usage.jsonl 2>/dev/null || true
fi
```

Replace OUTCOME with success/error/abort.
