# compass-open

OODA-open sub-skill for the compass loop (Track 2 Phase 1, 2026-08-29 —
`docs/compass-open-extraction-sop.md` in the `compass` skill). Mirrors the
`compass-close` extraction (P25) exactly: content-preserving move, no behaviour
change.

Invoked from `/compass`'s router when `open_session = false`. Owns the full OODA
half — Step 0 through Step 4.6 — from the pre-mediation impulse capture through
the goal-conditioned supplemental retrieval that runs right after session
confirmation.

**Inputs (from invocation context):**
- `namespace` — the compass namespace opening

<!-- SLOW_UPDATE_START -->
## Durable guidance — distilled from loop history

*This section encodes what has proven reliably true across many sessions. It is only modified during deliberate periodic skill reviews (P-SkillOpt) — never during normal sessions.*

- **SKILL.md wiring is prerequisite.** Any Python feature that isn't referenced in a SKILL.md step is invisible to the loop. Ship the SKILL.md change alongside (or before) the Python change — not after.
- **Exact-text deduplication is the dedup contract.** Learning deduplication uses exact-text match, not semantic similarity. Near-duplicates are signalled but not auto-merged — the dream pass handles structural cleanup at cadence.
- **The script is the only safe write path for reality.md — never edit the file directly.** Direct edits orphan SHA-256 hashes and trigger false "Never verified" alerts. Use `update-reality` for a full rewrite (rewording, restructuring); use `append-reality-bullet`/`remove-reality-bullet` for a single addition or removal — O(edit), not O(document size) (2026-08-28 audit finding #4).
- **One advisory block per DECIDE step.** Batch all hits from a given check (P13, P21, P_ARCH) into a single prompt — multiple prompts per step cause decision fatigue and are ignored.
- **Focused sessions outperform sprawling ones.** Namespaces with 3–4 tightly-scoped goals consistently produce more learnings per goal than sessions with 6–8 broad goals.
<!-- SLOW_UPDATE_END -->

---

## OODA half — session open

Run when `open_session = false`.

### Step 0 — Pre-mediation impulse capture (P74 Phase 1)

Before running OBSERVE — before any git signal, learning, or reality bullet is fetched —
capture what the user wants to do.

**If the router passed `impulse="<text>"`**, use that text verbatim as `raw_impulse` and do
not ask. The user typed it before anything was fetched, so it is already pre-mediation.

**Otherwise** (no `impulse`, e.g. the invocation was only a namespace), ask one optional
question:

```
Before I check anything — what do you want to do right now, in one sentence? (or enter to skip)
```

Capture the response verbatim as `raw_impulse` (empty string if skipped). Do not
interpret, tag, or reconcile it against anything yet — that would defeat the purpose.
Hold the value; it gets passed to `open` at Step 4 (or folded into the
`acknowledge-cooldown-violation` payload at Step 2f, if that path fires instead).

**Rule:** this is instrumentation, not a decision point. Never surface it back to the
user later in the same session, never use it to steer ORIENT's synthesis brief or DECIDE's
goal proposal, and never skip asking just because the answer seems likely to match
whatever OBSERVE will find anyway — the entire value of the capture depends on it being
genuinely prior to mediation, every time, not just on sessions where drift seems likely.
Only the router's `impulse` replaces the question. Never build one from conversation that
came after the router (it may already reflect what compass showed).

### Step 1 — OBSERVE

Run the consolidated command (one call replaces the old seven):
```bash
/opt/homebrew/bin/python3 ~/.claude/skills/compass/scripts/compass.py generate-orient-brief <namespace>
```

**The output is often large enough that the harness persists it to a file. When that
happens, do NOT `Read` the whole file** (most of it is raw JSON that `brief_markdown`
already re-expresses). Extract only what a step needs, e.g.:
```bash
/opt/homebrew/bin/python3 -c "import json; d=json.load(open('<persisted-file>')); print(d['brief_markdown'])"
```
**Gate flags come from a separate small call, not from this output:**
```bash
/opt/homebrew/bin/python3 ~/.claude/skills/compass/scripts/compass.py orient-flags <namespace>
```
About 2KB: every `*_due` flag with its `*_status` block, the defer counts,
`suggested_goal_count`, `last_close`, `intent_changed`, `goal_completion_trend`, and counts
of stale bullets and pending validations. Steps 2b onward, 2f, 3 and 3f read their flags
from it. Pull other `context.*` fields (e.g. `session_index`) from the persisted file one
field at a time; for key paths run `compass.py schema generate-orient-brief` (cadence
counters are nested: `context.skill_opt_status.sessions_since_skill_opt`).

Returns `{context, gitlog, carry_forward, global_cross_project, artefacts_matched,
watch_signals, brief_markdown}`. `context` is what `read` returns. `brief_markdown` is the
pre-rendered mechanical part of Step 2's brief (reality completeness, zone-grouped
top-learnings, cross-namespace signals, tactical backlog): use it as the base and prepend
the P48 synthesis blockquote plus the judgement-driven sections (drift/gaps, advisories).
`global_cross_project` (≤3) and `artefacts_matched` (≤2) are already matched; `watch_signals`
is `null` without `config.watches`, else check `watch_signals.empty` before rendering.

Still fetch two things separately — both need per-session context this command can't
supply:

- `session_index` (inside `context`) — compact index of the last 20 sessions (P28: one
  line each — id, date, goals, top learning, tags). Scan this to identify sessions
  relevant to carry-forward, hypothesis validation, or prior decisions, then call
  `expand-session <id>` for those sessions only:
  ```bash
  /opt/homebrew/bin/python3 ~/.claude/skills/compass/scripts/compass.py expand-session <namespace> <session-id>
  ```
- **BM25 goal-relevance query (R9):** if `session_index` contains planned actions for
  the most recent session, run:
  ```bash
  /opt/homebrew/bin/python3 ~/.claude/skills/compass/scripts/compass.py query-learnings <namespace> \
    '{"query": "<last session planned actions joined>", "limit": 5}'
  ```
  Use the returned `results` as `top_learnings` in the orient brief instead of the
  weight-ranked list (re-render the top-learnings section of the brief with these
  instead of `context.top_learnings` if it changes the ranking). Fall back to
  `context.top_learnings` if the query returns fewer than 2 results (sparse corpus).
  Skip if no prior planned actions exist.

### Step 2 — ORIENT

**P48 — Synthesis brief (generate before rendering the detail block):**

Before presenting the structured brief below, generate a 2–4 sentence "situation frame" that synthesises across intent, reality, and the top learnings. Format as a blockquote:

```
> <Sentence 1: where we are relative to intent — the primary gap or confirmation.>
> <Sentence 2: the dominant pattern from top learnings — what the corpus is signalling right now.>
> <Sentence 3 (optional): implication for this session — what this means for today's goals.>
> <Sentence 4 (optional): any anomaly or risk worth flagging — elevated complexity, exploration gap, stale reality, etc.>
```

Rules:
- Write this from the data — not from the user's words. If the corpus has no signal (fewer than 3 learnings), omit it silently.
- Keep each sentence to one clause. No hedging, no "it seems". Direct frame.
- This appears **before** the `## 🧭 Compass` header, not embedded in the detail list.
- Omit on first-ever session (no history to synthesise from).

Present a concise orient brief to the user. Use Step 1's `brief_markdown` as the base —
it already renders the header, intent, reality completeness, signals since last session,
top learnings (with the P56 zone-grouping rule), cross-project and cross-namespace signals,
carry-forward, tactical backlog and relevant artefacts. Flag every carry-forward and
tactical backlog item as a goal candidate. Add the sections it does not render:

```
**Reality (last known):**
<2–4 bullet summary of reality.md>

**External research signals:** *(only if `external_signals` array is non-empty — P11)*
<up to 3 most recent signals from external_signals.jsonl — prefix each with [research] so provenance is visible>

**Drift / gaps detected:**
<1–3 specific gaps between intent and reality — be direct>

**Goal hit-rate (P0.2):**
<last_session_hit_rate or "no prior session">

**Session complexity (P24):** *(omit if session not open or avg_last_5 is null)*
<session_complexity.current> compass calls (avg: <session_complexity.avg_last_5> over last 5 sessions)
```

The full section-by-section brief format and the P56 zone-grouping rule are in
`/opt/homebrew/bin/python3 ~/.claude/skills/compass/scripts/compass.py prompt orient-brief-format --section "Step 2 — ORIENT brief format (P56 zone-grouping included)"`.
Load it only when you must render a section yourself: `brief_markdown` is missing, or Step 1's BM25 query
changed the top-learnings ranking.

**Conditional advisories:** check these fields from the current `read` output —
`session_complexity.elevated` · `goal_stats.total_goals > 0 AND goal_stats.hit_rate < 70` ·
`reality_completeness.regression` · `exploration_ratio.low` ·
`reality_structure_warnings` non-empty · `outcome_rate < 0.3` ·
`contract_coverage < 0.4` · `quality_trend == "declining"` ·
`quality_plateau.plateaued` with a pulled-forward cadence ·
`quality_trend == "declining"` AND `quality_plateau.trend == "improving"` (P75 —
takes priority over the plain declining message when both hold).
If **any** hold, load `/opt/homebrew/bin/python3 ~/.claude/skills/compass/scripts/compass.py prompt orient-brief-advisories --section "ORIENT brief — conditional advisory blocks"`
and append the matching block(s), in the order listed there. If **none** hold, skip entirely —
don't load it for a brief with nothing to add.

**Record surfacing (P32/P44):** immediately after presenting the brief above (and the
global cross-project block, if shown), run silently:
```bash
/opt/homebrew/bin/python3 ~/.claude/skills/compass/scripts/compass.py surface-learnings <namespace>
```
If the global cross-project block was shown, also run `surface-learnings global`. This
is what bumps `times_surfaced`/`last_surfaced_at` for the learnings actually shown to the
user — `read` itself is a pure query and does not do this (see
`docs/cmd-read-purity-design.md`). Do not call `surface-learnings` from any other step —
only here, right after the top_learnings a namespace's `read` returned have genuinely
been displayed.

### Step 2b — Skill opportunity detection

After presenting the orient brief, run the **Skill opportunity detection** pass — see
`~/.claude/skills/compass/SKILL.md`'s "## Skill opportunity detection" section (that
top-level section stayed in the parent router; only this step's trigger point moved
here). If signals exist, surface the `💡 Tooling opportunities`
block as a standalone prompt **separate from the brief** and **wait for the user's response
before proceeding to Step 2b.1**. If no signals, skip silently and continue.

**Rule:** do not begin Step 2b.1 until the user has responded to the tooling opportunities
prompt (Y / S / D). The prompt and the orient brief must not be presented as one wall of
text — the user must have a clear opportunity to respond to the opportunities before the
ORIENT checks continue. If no signals: proceed immediately and silently.

**Cadence-gate convention (applies to 2b.1, 2b.3b, 2b.4b):** each of these checks a
`*_due` flag from `read`. `false` → skip entirely, no prompt. `true` → surface once as
part of ORIENT, never re-surface at close or mid-session. This is assumed below; only
deviations from it are called out per step.

**`code_review_due` / `research_due` / `dream_due`** follow the same false→skip rule, but
their prompts no longer fire here — see the v1 consolidation note above. Nothing to do
at this point in ORIENT; the flags are already sitting in `context` from Step 1.

**Checklist (2026-09-24, skill-feedback weight-2):** a side-quest that consumes a lot of
turn attention (e.g. batch-verifying a large stale-bullet list at Step 2c) makes it easy
to jump straight from ORIENT into Step 3 DECIDE, silently skipping the remaining 2b.x
sub-steps. Before proceeding past ORIENT, confirm each of the following has actually been
checked this session (skip is a valid outcome for any of them — the point is not to forget
to look):
☐ 2b.1 (assumption audit) · ☐ 2b.3b (CLAUDE.md hygiene) · ☐ 2b.4b (SkillOpt) ·
☐ 2b.5 (code context) · ☐ 2b.6 / 3f-item-3 (dream pass)

### Step 2b.1 — Strategic Assumption Audit (P47)

Check `assumption_audit_due` from read output. If `false`, skip entirely. If `true`, load
`/opt/homebrew/bin/python3 ~/.claude/skills/compass/scripts/compass.py prompt assumption-audit --section "Step 2b.1 — Strategic Assumption Audit (P47)"` and follow it — it covers
candidate selection, the P50/P59 advisory add-ons, the V/C/D/S response mapping, and the
counter reset.

### Step 2b.3 — Periodic code quality review (moved)

Moved to **Step 3f**, item 1. The flag is still read at Step 1 OBSERVE.

---

### Step 2b.3b — Periodic CLAUDE.md hygiene review

Check `claude_review_due` from read output. If `false`, skip entirely. If `true`, check
whether a `CLAUDE.md` exists for this namespace (`repo_path/CLAUDE.md`, or
`~/.claude/skills/<namespace>/CLAUDE.md` if `repo_path` is unset) — if none, reset the
counter (`record-claude-review <namespace>`) and skip silently. Otherwise load
`/opt/homebrew/bin/python3 ~/.claude/skills/compass/scripts/compass.py prompt claude-md-hygiene-review --section "Step 2b.3b — Periodic CLAUDE.md hygiene review"` and follow it.

---

### Step 2b.4 — Periodic external research (P11) — compass-research-scope (moved)

Moved to **Step 3f**, item 2. The flag is still read at Step 1 OBSERVE.

---

### Step 2b.4b — Periodic SKILL.md optimisation pass (P-SkillOpt)

Check `skill_opt_due` from read output. If `false`, skip entirely. If `true`, load
`/opt/homebrew/bin/python3 ~/.claude/skills/compass/scripts/compass.py prompt skillopt-protocol --section "Gate prompt (Step 2b.4b)"` and follow it — the P61c friction-gate
prompt variants. On Y, load `/opt/homebrew/bin/python3 ~/.claude/skills/compass/scripts/compass.py prompt skillopt-protocol --section "Evidence collection + reflect pass (on Y)"`;
n/later defers (counter not reset).

---

### Step 2b.5 — Code context (cold-start orientation)

After skill opportunity detection, check whether a `code_context.md` file exists
for this namespace:

```bash
cat ~/.claude/loop/<namespace>/code_context.md 2>/dev/null
```

If the file exists and is non-empty, surface it as a compact block:

```
📁 Code context (last session):
<content of code_context.md — show as-is, no summarisation>
```

If the file does not exist or is empty, skip silently. This is a read-only display step —
no confirmation needed. Continue immediately to Step 2c.

### Step 2b.6 — Inter-loop dream pass (P12.1) (moved)

Moved to **Step 3f**, item 3. The flag is still read at Step 1 OBSERVE.

### Step 2c — Reality validation protocol (P0.1)

Check `reality_stale_bullets` from the `read` output. If empty, skip to Step 2d's check
below. If non-empty, load
`/opt/homebrew/bin/python3 ~/.claude/skills/compass/scripts/compass.py prompt reality-and-hypothesis-validation --section "Step 2c — Reality validation protocol (P0.1)"`
and follow it — one `auto-verify-reality` call (filesystem/git evidence,
stamped in Python) and the manual confirmation prompt for anything left in `manual`. **This is a gating
step** — do not proceed to DECIDE until it resolves.

### Step 2d — Hypothesis validation (P1.1)

Check `pending_validations` from the `read` output. If empty, skip to Step 2f. If
non-empty, read the same `reality-and-hypothesis-validation.md` file's Step 2d section —
it covers the Confirmed/Disproven/Untested prompt per expired hypothesis.

### Step 2f — Session hygiene precondition check (P4.2)

Before proceeding to DECIDE, check if this namespace was last closed less than 4 hours ago
(or less than the configured cooldown, if customised):

Use `last_close` from Step 1's `orient-flags` output — do not run a
second `read` for it (it rebuilds the same ~80KB context). Calculate hours elapsed. If elapsed time is less than the
configured `session_cooldown_hours` (default: 4), load the Step 2f section with
`/opt/homebrew/bin/python3 ~/.claude/skills/compass/scripts/compass.py prompt orient-gates --section "Step 2f — Session hygiene"` and follow it — a
**blocking** reason prompt, then `acknowledge-cooldown-violation`, which opens the session itself.

If no violation (last_close is more than cooldown_hours ago, or never closed), skip silently.

### Step 2g — Git vs reality reconciliation (moved)

Moved to **Step 3b**, item 1 (the scan and its prompt). `gitlog` is still fetched at Step 1.

### Step 2h — Intent drift check (P1.2)

Check `intent_changed` from the read output. If `false`: skip silently.

If `true`, load the Step 2h section with
`/opt/homebrew/bin/python3 ~/.claude/skills/compass/scripts/compass.py prompt orient-gates --section "Step 2h"` and follow it — a gating Y/N drift prompt recorded with `set-intent` now, at ORIENT; never re-prompted at close.

### Step 3 — DECIDE

Propose up to `suggested_goal_count.count` session goals as a numbered list (P-DC2 — default 4 if no history).
Ground them in the gaps identified — this explicitly includes carry-forward items and
`backlog_tactical` entries flagged as goal candidates at ORIENT, not just newly-noticed
drift. Keep them specific and completable in one session.

**Order goals by natural execution sequence**, not by priority alone. Consider:
- **Dependencies first** — if goal B requires the codebase state left by goal A, list A before B
- **Structural before additive** — refactors and cleanup that make subsequent goals cleaner go earlier
- **Exploit before explore** — implementation tasks before design/research goals, so exploratory work
  benefits from concrete session context rather than abstract planning
- **Riskier goals earlier** — goals likely to surface blockers should come first while there is still
  time to adapt scope

After the numbered list, add one line stating the ordering rationale:
```
*Order: <brief reason — e.g. "CR-04 cleans the learnings path before P33 touches it; exploratory design last"*>
```

Ask: *"Does this match your intent for today? Add, remove, or reorder before I lock it in."*

Wait for confirmation. Accept additions, removals, rephrasing. If the user just says
"yes" or "go", treat the proposal as confirmed.

### Step 3b — Consolidated pre-lock advisory batch (v2 consolidation, 2026-09-16)

Run **after** Step 3 (DECIDE goal list confirmed), **before** Step 3f.

Build one numbered list, one line per item, **omitting any line whose trigger is false**
(never renumber around a gap):

1. **`[GIT/REALITY]`** — only if old Step 2g's scan (stale "missing"/Backlog bullets vs
   `gitlog`; new files not in reality's "What exists") finds at least one match.
   `gitlog` covers only this namespace's own repo. For each Backlog bullet that names
   another repo (a sibling skill such as `compass-research-scope`, or a namespace such as
   `agentic-loopkit`), also run `git -C <that repo> log --oneline -20` — the repo is
   `~/.claude/skills/<name>` or that namespace's `repo_path`. A commit there that does
   what the bullet asks is a match too; preview it as `<repo>@<hash>`.
2. **`[VAGUE GOALS]`** — only if one or more confirmed goals match old Step 3b's vagueness
   signals (*"work on", "look at", "investigate", "sort out", "pick up", "think about",
   "explore", "continue with", "review", "have a look"*; fewer than 5 words with no
   concrete output verb; no observable deliverable implied).
3. **`[RESEARCH FORK]`** — only if a confirmed goal contains a decision verb (*implement,
   migrate, replace, integrate, adopt, switch, introduce, rewrite, refactor, extract,
   redesign*).
4. **`[DOMAIN NOVELTY]`** — only if, per old Step 3e's token-query check, a confirmed goal
   touches a domain with no results or only >90-day-old results, and at least 3 prior
   sessions exist.

If **none** of 1–4 trigger, skip this step silently and render nothing. Otherwise load
`/opt/homebrew/bin/python3 ~/.claude/skills/compass/scripts/compass.py prompt pre-lock-advisory-batch --section "Step 3b — Consolidated pre-lock advisory batch (v2 consolidation, 2026-09-16)"` and follow it — the
one-screen render, override parsing, per-item routing and rules.

### Step 3f — Consolidated DECIDE-tail batch (v1 consolidation, 2026-09-01)

Run **after** Step 3b (the consolidated pre-lock advisory batch), **before** Step 3c
(architecture check).

Build one numbered list, one line per item, **omitting any line whose trigger is false**
(never renumber around a gap — a line's number is stable across a session):

1. **`[CODE REVIEW]`** — only if `code_review_due` is true. Cite `code_review_status.sessions_since_review`/
   `interval_sessions` (or the complexity-pull-forward basis from `code_review_status`,
   same framing rule the old Step 2b.3 used) and `code_review_defer_count`. Default: **yes**.
2. **`[RESEARCH]`** — only if `research_due` is true. Cite `research_status.sessions_since_research`/
   `interval_sessions` (or the pull-forward basis) and `research_defer_count`. Default: **yes**.
3. **`[DREAM PASS]`** — only if `dream_due` is true. Cite `dream_status.sessions_since_dream`/
   `interval_sessions`. Default: **yes**.
4. **`[STRETCH GOAL]`** — only if either old-3b.5 trigger holds (all confirmed goals are
   execution tasks with none exploratory/design, OR the last 3 `goal_completion_trend`
   values are all `100.0`). Name which trigger fired. Default: **no**.
5. **`[GOAL TYPES]`** — always present. Default: **all exploit** (`E` for every goal) —
   this is the one default that changes behaviour from the old bare-skip (which recorded
   no types at all): a rendered default line needs an actual value, not an absence, so
   accepting it now means "everything type-tagged E," and exploration-ratio tracking
   includes the session as 100% exploit rather than excluding it.
6. **`[CONTRACTS]`** — always present. Default: **no** (skip; `goal_contracts` stays empty).
7. **`[HYPOTHESES]`** — always present. Default: **no**.

If item 1, 2 or 3 renders, load
`/opt/homebrew/bin/python3 ~/.claude/skills/compass/scripts/compass.py prompt decide-tail-cadence-items --section "Rendering lines 1/2"` and
`/opt/homebrew/bin/python3 ~/.claude/skills/compass/scripts/compass.py prompt decide-tail-cadence-items --section "Routing items 1–3"`
first — they hold the `<escalation hint>` rule for lines 1/2, the routing for items 1–3 and
the escalation-after-lock note. Skip it when only items 4–7 render.

Render:
```
1. [CODE REVIEW] due (<sessions_since_review>/<interval_sessions> sessions, deferred <N>×) → run? (default: yes)<escalation hint>
2. [RESEARCH] due (<sessions_since_research>/<interval_sessions> sessions, deferred <N>×) → scope Q&A? (default: yes)<escalation hint>
3. [DREAM PASS] due (<sessions_since_dream>/<interval_sessions> sessions) → run? (default: yes)
4. [STRETCH GOAL] <trigger: all-execution-tasks | 3× 100% completion> → add one? (default: no)
5. [GOAL TYPES] tag each goal E/X, e.g. "EEXEE" for 5 goals (default: all E)
6. [CONTRACTS] pre-specify success criteria for any goal? (default: no)
7. [HYPOTHESES] log assumptions to validate at close? (default: no)

Accept all defaults, or override by number? [Enter to accept all]
```

If none of items 1–4 are triggered this session, the list only shows 5/6/7 — never fully
empty (5/6/7 are always present, matching the old always-offered behaviour of 3b.6/3b.8
and the always-optional-but-always-asked 4.5 gate).

**Enter** accepts every default: items 1–3 run their yes-path (routing already loaded with the lines), item 4 no, item 5 all `E` (`goal_types` = `exploit` for the confirmed goal count), items 6 and 7 no (`goal_contracts` stays empty; Step 4.5 is skipped). Any other response → load `/opt/homebrew/bin/python3 ~/.claude/skills/compass/scripts/compass.py prompt decide-tail-cadence-items --section "Step 3f — Override parsing and routing (items 4–7)"` and follow it.

### Step 3c — Architecture constraint check (P_ARCH)

Run **only if** the namespace has a `repo_path` configured. If no `repo_path`, skip
silently — do not even read the prompts file. If configured, read the same
`contract-and-architecture-checks.md` file's Step 3c section — it covers the
service/module-name detection and the pom.xml dependency-declaration check.

### Steps 3d, 3e (moved)

3d (research fork, P13) and 3e (domain novelty, P21) now live in **Step 3b**, items 3 and 4.

---

### Step 4 — Lock and open

Once confirmed, open the session:
```bash
/opt/homebrew/bin/python3 ~/.claude/skills/compass/scripts/compass.py open <namespace> '<json_array_of_goals>' '<raw_impulse from Step 0, or omit the third arg entirely if it was skipped>'
```
The third argument is `raw_impulse` from Step 0 (P74): omit it if skipped, or if
`acknowledge-cooldown-violation` (Step 2f) already opened the session with it.

Then populate the todo list with the confirmed goals, if this runtime has a
TodoWrite-equivalent tool (use it). **Either way**, as each confirmed goal actually
completes during the session, also call:
```bash
/opt/homebrew/bin/python3 ~/.claude/skills/compass/scripts/compass.py set-goal-status <namespace> '{"index": <0-based goal index>, "done": true}'
```
This is what the Drift nudge check (and any future mid-session goal-progress check)
reads when no host todo tool is available — `open` already initialises
`goal_progress` to all-`false` for the confirmed goal count, so there's nothing to
set up here beyond calling this as goals land. Skip only if you've confirmed this
runtime's TodoWrite-equivalent tool exists AND is the thing Drift nudge is already
configured to read instead (2026-09-02 — a real session found `ToolSearch` returning
nothing for TodoWrite, exactly the gap this closes).

**P67 cold-start seeding:** if the response's `seeded_count` is greater than 0 (only
possible on this namespace's genuinely first-ever open), surface once, no action
required:
```
🌱 Seeded <seeded_count> learning(s) from `<seed_source>` — cold-start context, not yet
   validated in this namespace.
```
`seed_source` is `null`/`seeded_count` is `0` on every other open — skip silently.

### Step 4.5 — Pre-session hypothesis elicitation (P15)

The Y/n gate ("Log any to validate at close?") is now item 7 of Step 3f — this step
only runs the free-text follow-up loop, and only if item 7's answer was **Y**:

- **Y** (from Step 3f item 7) → load
  `/opt/homebrew/bin/python3 ~/.claude/skills/compass/scripts/compass.py prompt hypothesis-elicitation --section "Step 4.5 — Pre-session hypothesis elicitation (P15)"` and follow it — one
  `log-learning` hypothesis per assumption, until the user says n.

Confirm to the user:
```
✓ Session open. Todo list set. Let's go.
```

### Step 4.6 — Goal-conditioned supplemental retrieval (P43b)

After the confirm, run silently:
```bash
/opt/homebrew/bin/python3 ~/.claude/skills/compass/scripts/compass.py query-learnings <namespace> \
  '{"query": "<all confirmed goals joined by space>", "limit": 3}'
```

If the query returns **fewer than 2 results** or the corpus has **fewer than 5 learnings**: skip entirely — the corpus is too sparse for goal-conditioned retrieval to add signal.

If **2 or more results** are returned, surface once as a compact block immediately after the confirm:

```
📚 Session-relevant learnings:
  · "<learning text, first 90 chars>" [w:<weight>, <age in days>d] ← <tag1>, <tag2>
  · "<learning text, first 90 chars>" [w:<weight>, <age in days>d] ← <tag1>
  · ...
```

No confirmation needed. No action required. This is a read-only orientation supplement — the learnings that BM25 judges most relevant to today's goals, surfaced so they are in working context from the start.

**Rule:** one pass only, immediately after Step 4.5 confirm. Skip silently if the corpus is sparse or the query returns fewer than 2 results. Never re-run mid-session.

---

## Return

Once Step 4.5/4.6 confirms and the todo list is set, this sub-skill is done — the
conversation simply continues into the session's work. There is nothing to hand
back to the parent router.

**Prompt-count tally:** each time Steps 0–4.6 show the user an interactive screen, run
```bash
/opt/homebrew/bin/python3 ~/.claude/skills/compass/scripts/compass.py tally-prompt <namespace> open
```
An "interactive screen" is one round trip that waited for a response, not one script
call or one bullet within a screen (Step 3f's batch counts as **one**, however many of
its 7 items rendered). The script keeps the count across `open`, and `close` records it
as `open_prompt_count` — don't keep or estimate the number yourself.
