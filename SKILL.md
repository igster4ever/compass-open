# compass-open

OODA-open sub-skill for `/compass`, invoked by the router when `open_session = false`
(extracted from it 2026-08-29; see `docs/compass-open-extraction-sop.md` in the `compass`
skill). Owns Step 0 (impulse capture) through Step 4.6 (goal-conditioned retrieval).
**Input:** `namespace`.

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

Capture what the user wants to do **before** OBSERVE fetches anything.

- If the router passed `impulse="<text>"`, use it verbatim as `raw_impulse` and do not ask: it
  was typed before anything was fetched.
- Otherwise ask one optional question:
  ```
  Before I check anything — what do you want to do right now, in one sentence? (or enter to skip)
  ```
  Capture the answer verbatim (`""` if skipped). Never build one from conversation that came
  after the router (it may already reflect what compass showed).

Hold `raw_impulse`: it goes to `open` at Step 4, or into the `acknowledge-cooldown-violation`
payload at Step 2f if that path fires instead.

**Rule:** this is instrumentation, not a decision point. Do not interpret, tag or reconcile
it; never surface it again this session or use it to steer ORIENT's synthesis or DECIDE's goals;
and ask every time, even when the answer seems predictable. Its value depends on being
genuinely prior to mediation.

### Step 1 — OBSERVE

```bash
~/.claude/skills/compass/scripts/compass.sh generate-orient-brief <namespace>
```
Returns `{context, gitlog, carry_forward, global_cross_project, artefacts_matched,
watch_signals, brief_markdown}`; `context` is what `read` returns. The output is often large
enough that the harness persists it to a file. **Do NOT `Read` the whole file** (mostly raw JSON
that `brief_markdown` re-expresses); print only what a step needs:
```bash
python3 -c "import json; d=json.load(open('<persisted-file>')); print(d['brief_markdown'])"
```
`brief_markdown` is the pre-rendered mechanical part of Step 2's brief (reality completeness,
zone-grouped top-learnings, cross-namespace signals, tactical backlog): use it as the base and
prepend the P48 synthesis blockquote plus the judgement-driven sections (drift/gaps,
advisories). `global_cross_project` (≤3) and `artefacts_matched` (≤2) are already matched;
`watch_signals` is `null` without `config.watches`, else check `watch_signals.empty`.

**Gate flags come from a separate ~2KB call, not from this output:**
```bash
~/.claude/skills/compass/scripts/compass.sh orient-flags <namespace>
```
It carries every `*_due` flag with its `*_status`, the defer counts, `suggested_goal_count`,
`last_close`, `intent_changed`, `goal_completion_trend`, `stale_bullet_count`,
`pending_validations` and `assumption_audit_candidate_count`. Steps 2b onward, 2f, 3 and 3f read
their flags from it. A due audit with no candidates arrives as `assumption_audit_due: false`
(`open` resets its counter). For any other `context.*` field, pull it from the persisted file
one at a time; key paths: `compass.py schema generate-orient-brief`.

Still fetch two things separately — both need per-session context this command can't supply:

- **`session_index`** (in `context`): the last 20 sessions, one line each (P28). Scan it for
  sessions relevant to carry-forward, hypothesis validation or prior decisions, then
  `compass.sh expand-session <namespace> <session-id>` for those only.
- **BM25 goal-relevance query (R9):** if `session_index` has planned actions for the most
  recent session, run `compass.sh query-learnings <namespace> '{"query": "<last session planned
  actions joined>", "limit": 5}'` and use its `results` as `top_learnings` (re-render that
  section of the brief if the ranking changes). Fall back to `context.top_learnings` if it
  returns fewer than 2 results; skip if no prior planned actions exist.

### Step 2 — ORIENT

**P48 — Synthesis brief.** Before the structured brief, write a 2–4 sentence "situation frame"
across intent, reality and the top learnings, as a blockquote **before** the `## 🧭 Compass`
header (not inside the detail list):

```
> <1: where we are relative to intent — the primary gap or confirmation.>
> <2: the dominant pattern in the top learnings — what the corpus is signalling now.>
> <3 (optional): what this means for today's goals.>
> <4 (optional): an anomaly or risk — elevated complexity, exploration gap, stale reality.>
```

Write it from the data, not the user's words; one clause per sentence, no hedging. Omit it
silently when the corpus has fewer than 3 learnings or on a first-ever session.

Present a concise brief. Use Step 1's `brief_markdown` as the base: it already renders the
header, intent, reality completeness, signals since last session, top learnings (P56
zone-grouping), cross-project and cross-namespace signals, carry-forward, tactical backlog and
relevant artefacts. Flag every carry-forward and tactical backlog item as a goal candidate. Add
the sections it does not render:

```
**Reality (last known):**
<2–4 bullet summary of reality.md>

**External research signals:** *(only if `external_signals` is non-empty — P11)*
<up to 3 most recent, each prefixed [research]>

**Drift / gaps detected:**
<1–3 specific gaps between intent and reality — be direct>

**Goal hit-rate (P0.2):**
<last_session_hit_rate or "no prior session">

**Session complexity (P24):** *(omit if session not open or avg_last_5 is null)*
<session_complexity.current> compass calls (avg: <session_complexity.avg_last_5> over last 5 sessions)
```

Only when `brief_markdown` is missing, or Step 1's BM25 query changed the top-learnings ranking,
load the full format with
`~/.claude/skills/compass/scripts/compass.sh prompt orient-brief-format --section "Step 2 — ORIENT brief format (P56 zone-grouping included)"`.

**Conditional advisories:** check these fields in the brief's context — `session_complexity.elevated` ·
`goal_stats.total_goals > 0 AND goal_stats.hit_rate < 70` · `reality_completeness.regression` ·
`exploration_ratio.low` · `reality_structure_warnings` non-empty · `outcome_rate < 0.3` ·
`contract_coverage < 0.4` · `quality_trend == "declining"` · `quality_plateau.plateaued` with a
pulled-forward cadence · `quality_trend == "declining"` AND `quality_plateau.trend ==
"improving"` (P75: takes priority over the plain declining message). If **any** hold, load
`~/.claude/skills/compass/scripts/compass.sh prompt orient-brief-advisories --section "ORIENT brief — conditional advisory blocks"`
and append the matching block(s) in the order listed there. If **none** hold, do not load it.

**Record surfacing (P32/P44):** right after presenting the brief, run silently
`~/.claude/skills/compass/scripts/compass.sh surface-learnings <namespace>` (and
`surface-learnings global` if the global cross-project block was shown). This bumps
`times_surfaced`/`last_surfaced_at` for the learnings actually shown; `read` is a pure query and
does not (`docs/cmd-read-purity-design.md`). Call it from no other step.

### Step 2b — Skill opportunity detection

After the brief, run the **Skill opportunity detection** pass (the router's
`## Skill opportunity detection` section; only its trigger point lives here). If signals exist,
surface the `💡 Tooling opportunities` block as a standalone prompt, **separate from the
brief**, and wait for the response (Y / S / D) before Step 2b.1. No signals: continue silently.

**Cadence-gate convention (2b.1, 2b.3b, 2b.4b):** each checks a `*_due` flag (from
`orient-flags`). `false` → skip entirely, no prompt. `true` → surface once as part of ORIENT,
never again at close or mid-session. `code_review_due`, `research_due` and `dream_due` follow
the same rule but fire at Step 3f, not here.

**Checklist (2026-09-24):** a long side-quest (e.g. batch-verifying many stale bullets at Step
2c) makes it easy to jump from ORIENT straight to Step 3 and skip a 2b.x sub-step. Before
leaving ORIENT confirm each was looked at this session (skip is a valid outcome):
☐ 2b.1 (assumption audit) · ☐ 2b.3b (CLAUDE.md hygiene) · ☐ 2b.4b (SkillOpt) ·
☐ 2b.5 (code context) · ☐ 2b.6 / 3f-item-3 (dream pass)

**Moved steps:** 2b.3 (code review), 2b.4 (research) and 2b.6 (dream pass) are now Step 3f items
1–3; 2g (git vs reality) is Step 3b item 1; 3d and 3e are Step 3b items 3 and 4. Their flags are
still read at Step 1.

### Step 2b.1 — Strategic Assumption Audit (P47)

If `assumption_audit_due` is `false`, skip entirely (it is also `false` when there is nothing to
audit). If `true`, load
`~/.claude/skills/compass/scripts/compass.sh prompt assumption-audit --section "Step 2b.1 — Strategic Assumption Audit (P47)"`
and follow it: candidate display, the P50/P59 add-ons, the V/C/D/S mapping and the counter reset.

### Step 2b.3b — Periodic CLAUDE.md hygiene review

If `claude_review_due` is `false`, skip entirely. If `true` and the namespace has no `CLAUDE.md`
(`repo_path/CLAUDE.md`, or `~/.claude/skills/<namespace>/CLAUDE.md` when `repo_path` is unset),
run `record-claude-review <namespace>` and skip silently. Otherwise load
`~/.claude/skills/compass/scripts/compass.sh prompt claude-md-hygiene-review --section "Step 2b.3b — Periodic CLAUDE.md hygiene review"`
and follow it.

### Step 2b.4b — Periodic SKILL.md optimisation pass (P-SkillOpt)

If `skill_opt_due` is `false`, skip entirely. If `true`, load
`~/.claude/skills/compass/scripts/compass.sh prompt skillopt-protocol --section "Gate prompt (Step 2b.4b)"`
(the P61c friction-gate variants). On Y, load
`~/.claude/skills/compass/scripts/compass.sh prompt skillopt-protocol --section "Evidence collection + reflect pass (on Y)"`;
n/later defers (counter not reset).

### Step 2b.5 — Code context (cold-start orientation)

Run `cat ~/.claude/loop/<namespace>/code_context.md 2>/dev/null`. If it is non-empty, show it
as-is (no summarising) under `📁 Code context (last session):`; otherwise skip silently. Read-only,
no confirmation; continue to Step 2c.

### Step 2c — Reality validation protocol (P0.1)

If `stale_bullet_count` is 0, skip to Step 2d. Otherwise load
`~/.claude/skills/compass/scripts/compass.sh prompt reality-and-hypothesis-validation --section "Step 2c — Reality validation protocol (P0.1)"`
and follow it: one `auto-verify-reality` call (filesystem/git evidence, stamped in Python), then
a manual confirmation prompt for anything left in `manual`. **Gating:** do not proceed to
DECIDE until it resolves.

### Step 2d — Hypothesis validation (P1.1)

If `pending_validations` is empty, skip to Step 2f. Otherwise load the Step 2d section of the
same `reality-and-hypothesis-validation` prompt (the Confirmed/Disproven/Untested prompt per
expired hypothesis).

### Step 2f — Session hygiene precondition check (P4.2)

Take `last_close` from `orient-flags` (no second `read`: it rebuilds the ~100KB context) and
compute the hours elapsed. If under `session_cooldown_hours` (default 4), load
`~/.claude/skills/compass/scripts/compass.sh prompt orient-gates --section "Step 2f — Session hygiene"`
and follow it: a **blocking** reason prompt, then `acknowledge-cooldown-violation`, which opens
the session itself. Otherwise (or never closed), skip silently.

### Step 2h — Intent drift check (P1.2)

If `intent_changed` is `false`, skip silently. If `true`, load
`~/.claude/skills/compass/scripts/compass.sh prompt orient-gates --section "Step 2h"`
and follow it: a gating Y/N drift prompt recorded with `set-intent` now, never re-prompted at close.

### Step 3 — DECIDE

Propose up to `suggested_goal_count.count` session goals (P-DC2; default 4 with no history) as a
numbered list. Ground them in the gaps found, including the carry-forward and `backlog_tactical`
items flagged as candidates at ORIENT. Keep each specific and completable in one session.

**Order by natural execution sequence**, not priority alone:
- **Dependencies first** — if B needs the state A leaves, list A first.
- **Structural before additive** — refactors that make later goals cleaner go earlier.
- **Exploit before explore** — implementation before design/research, so exploration has
  concrete session context.
- **Riskier first** — goals likely to surface blockers come early, while scope can still adapt.

Add one line: `*Order: <brief reason>*`. Then ask: *"Does this match your intent for today? Add,
remove, or reorder before I lock it in."* Accept additions, removals and rephrasing; "yes" or
"go" confirms the proposal.

### Step 3b — Consolidated pre-lock advisory batch (v2 consolidation, 2026-09-16)

Run after Step 3 is confirmed, before Step 3f. Build one numbered list, one line per item,
**omitting any line whose trigger is false** (never renumber around a gap):

1. **`[GIT/REALITY]`** — only if the old Step 2g scan (stale "missing"/Backlog bullets vs
   `gitlog`; new files not in reality's "What exists") finds a match. `gitlog` covers only this
   namespace's repo: for each Backlog bullet that names another repo (a sibling skill such as
   `compass-research-scope`, or a namespace such as `agentic-loopkit`), also run
   `git -C <that repo> log --oneline -20` (`~/.claude/skills/<name>` or that namespace's
   `repo_path`). A commit there that does what the bullet asks is a match too; preview it as
   `<repo>@<hash>`.
2. **`[VAGUE GOALS]`** — only if a confirmed goal matches the vagueness signals (*"work on",
   "look at", "investigate", "sort out", "pick up", "think about", "explore", "continue with",
   "review", "have a look"*; under 5 words with no concrete output verb; no observable
   deliverable).
3. **`[RESEARCH FORK]`** — only if a confirmed goal has a decision verb (*implement, migrate,
   replace, integrate, adopt, switch, introduce, rewrite, refactor, extract, redesign*).
4. **`[DOMAIN NOVELTY]`** — only if a token-query check finds a confirmed goal in a domain with
   no results or only >90-day-old results, and at least 3 prior sessions exist.

If none trigger, skip silently and render nothing. Otherwise load
`~/.claude/skills/compass/scripts/compass.sh prompt pre-lock-advisory-batch --section "Step 3b — Consolidated pre-lock advisory batch (v2 consolidation, 2026-09-16)"`
and follow it (the one-screen render, override parsing, routing and rules).

### Step 3f — Consolidated DECIDE-tail batch (v1 consolidation, 2026-09-01)

Run after Step 3b, before Step 3c. Build one numbered list, one line per item, **omitting any
line whose trigger is false** (a line's number is stable across the session):

1. **`[CODE REVIEW]`** — only if `code_review_due`. Cite `code_review_status.sessions_since_review`/
   `interval_sessions` (or the complexity pull-forward basis) and `code_review_defer_count`.
   Default: **yes**.
2. **`[RESEARCH]`** — only if `research_due`. Cite `research_status.sessions_since_research`/
   `interval_sessions` (or the pull-forward basis) and `research_defer_count`. Default: **yes**.
3. **`[DREAM PASS]`** — only if `dream_due`. Cite `dream_status.sessions_since_dream`/
   `interval_sessions`. Default: **yes**.
4. **`[STRETCH GOAL]`** — only if all confirmed goals are execution tasks with none
   exploratory/design, OR the last 3 `goal_completion_trend` values are all `100.0`. Name the
   trigger. Default: **no**.
5. **`[GOAL TYPES]`** — always. Default: **all exploit** (`E` per goal). The rendered default
   needs a value, so accepting it records every goal as exploit; exploration-ratio tracking counts
   the session as 100% exploit instead of excluding it.
6. **`[CONTRACTS]`** — always. Default: **no** (`goal_contracts` stays empty).
7. **`[HYPOTHESES]`** — always. Default: **no**.

If item 1, 2 or 3 renders, first load
`~/.claude/skills/compass/scripts/compass.sh prompt decide-tail-cadence-items --section "Rendering lines 1/2"` and
`~/.claude/skills/compass/scripts/compass.sh prompt decide-tail-cadence-items --section "Routing items 1–3"`
(the `<escalation hint>` rule, the routing for items 1–3 and the escalation-after-lock note).
Skip both when only items 4–7 render.

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

Items 5–7 are always present, so the list is never empty. **Enter** accepts every default:
items 1–3 take their yes-path, item 4 no, item 5 all `E` (`goal_types` = `exploit` per confirmed
goal), items 6 and 7 no (`goal_contracts` stays empty; Step 4.5 is skipped). Any other response →
load `~/.claude/skills/compass/scripts/compass.sh prompt decide-tail-cadence-items --section "Step 3f — Override parsing and routing (items 4–7)"`
and follow it.

### Step 3c — Architecture constraint check (P_ARCH)

Only if the namespace has a `repo_path`; otherwise skip silently without loading anything. If
set, read the Step 3c section of `contract-and-architecture-checks` (service/module-name
detection and the pom.xml dependency-declaration check).

---

### Step 4 — Lock and open

Once confirmed, open the session:
```bash
~/.claude/skills/compass/scripts/compass.sh open <namespace> '<json_array_of_goals>' '<raw_impulse from Step 0>'
```
Omit the third argument if `raw_impulse` was skipped, or if `acknowledge-cooldown-violation`
(Step 2f) already opened the session with it. The response's `assumption_audit_auto_reset: true`
needs no action (a due audit with nothing to audit was reset).

Then populate the todo list with the confirmed goals if this runtime has a TodoWrite-equivalent
tool. **Either way**, as each goal completes, call:
```bash
~/.claude/skills/compass/scripts/compass.sh set-goal-status <namespace> '{"index": <0-based goal index>, "done": true}'
```
The Drift nudge reads this when no host todo tool exists (`open` already initialises
`goal_progress` to all-`false`). Skip it only if you have confirmed the TodoWrite-equivalent
exists and is what Drift nudge reads (a real session found `ToolSearch` returning nothing for
TodoWrite, 2026-09-02).

**P67 cold-start seeding:** if `seeded_count` > 0 (only on a namespace's first-ever open), say
once, no action: `🌱 Seeded <seeded_count> learning(s) from <seed_source> — cold-start context,
not yet validated in this namespace.` Otherwise skip silently.

### Step 4.5 — Pre-session hypothesis elicitation (P15)

The Y/n gate is item 7 of Step 3f. Only if its answer was **Y**, load
`~/.claude/skills/compass/scripts/compass.sh prompt hypothesis-elicitation --section "Step 4.5 — Pre-session hypothesis elicitation (P15)"`
and follow it: one `log-learning` hypothesis per assumption, until the user says n.

Confirm: `✓ Session open. Todo list set. Let's go.`

### Step 4.6 — Goal-conditioned supplemental retrieval (P43b)

After the confirm, run silently:
```bash
~/.claude/skills/compass/scripts/compass.sh query-learnings <namespace> \
  '{"query": "<all confirmed goals joined by space>", "limit": 3}'
```
If it returns **fewer than 2 results** or the corpus has **fewer than 5 learnings**, skip
silently. Otherwise surface once, immediately after the confirm, no action needed:

```
📚 Session-relevant learnings:
  · "<learning text, first 90 chars>" [w:<weight>, <age in days>d] ← <tag1>, <tag2>
  · ...
```
One pass only; never re-run mid-session.

---

## Return

Once Step 4.5/4.6 confirms and the todo list is set, this sub-skill is done and the
conversation continues into the session's work; there is nothing to hand back to the router.

**Prompt-count tally:** note each interactive screen Steps 0–4.6 show and, once the todo list is
set, tally them all in one call:
```bash
~/.claude/skills/compass/scripts/compass.sh tally-prompt <namespace> open <screen>[,<screen>...]
```
Each `<screen>` is a short key (`gate`, `impulse`, `orient`, `goals`, `advisories`, `batch`,
`research`); repeating a key never double counts. A screen is one round trip that waited for a
response, not one script call or bullet (Step 3f's batch is **one**). `close` records the count
as `open_prompt_count`; don't estimate it.
