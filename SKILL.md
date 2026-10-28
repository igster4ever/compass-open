# compass-open

OODA-open sub-skill for `/compass`, invoked by the router when `open_session = false`. Owns
Step 0 (impulse capture) through Step 4.6 (goal-conditioned retrieval). **Input:** `namespace`.
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

### Step 0 — Pre-mediation impulse capture

Capture what the user wants to do **before** OBSERVE fetches anything.

- If the router passed `impulse="<text>"`, use it verbatim as `raw_impulse` and do not ask: it
  was typed before anything was fetched.
- Otherwise ask one optional question:
  ```
  Before I check anything — what do you want to do right now, in one sentence? (or enter to skip)
  ```
  Capture the answer verbatim (`""` if skipped). Never build one from conversation after the
  router.

Hold `raw_impulse` for `open` (Step 4) or `acknowledge-cooldown-violation` (Step 2f).

**Rule:** this is instrumentation, not a decision point. Do not interpret, tag or reconcile
it; never surface it again or use it to steer ORIENT or DECIDE; and ask every time, even when the
answer seems predictable.

### Step 1 — OBSERVE

One read-only call returns everything OBSERVE needs, as plain text:
```bash
~/.claude/skills/compass/scripts/compass.sh brief <namespace> --with-flags
```
In order: `brief_markdown` (header, intent, reality completeness, signals, top learnings,
cross-namespace signals, carry-forward, tactical backlog, artefacts), a reality summary,
**Recent sessions** (S-1 to S-5), the **R9 goal-relevance block**, an **Advisory inputs** JSON
line, `code_context.md` when present, and the **Gate flags** JSON line.
`brief_markdown` is the base of Step 2's brief.

- **Gate flags** (`orient-flags`): every `*_due` flag, the `*_status` block of each due gate,
  defer counts and the other ORIENT inputs. Steps 2b onward, 2f, 3 and 3f read their flags here.
  `due` can list `corpus_summary`, a close-time gate: open takes no action on it.
- **R9 block:** ranked by S-1's planned goals. When it lists learnings,
  render them as the brief's top learnings instead of the weight-ranked ones; when it says
  "fewer than 2 matches" or "no planned goals", keep the top learnings as they are.
- **Recent sessions:** for those relevant to carry-forward, hypotheses or prior decisions, run `~/.claude/skills/compass/scripts/compass.sh expand-session <namespace> <session-id>` for those only.
- **Advisory inputs:** the fields Step 2's conditional advisories, goal hit-rate and external
  research signals read.

Only for a `context.*` field none of these carry, run `~/.claude/skills/compass/scripts/compass.sh generate-orient-brief
<namespace>` (large: print single keys from it, never `Read` it whole; key paths: `~/.claude/skills/compass/scripts/compass.sh
schema generate-orient-brief`).

### Step 2 — ORIENT

**Synthesis brief.** Before the structured brief, write a 2–4 sentence "situation frame"
across intent, reality and the top learnings, as a blockquote **before** the `## 🧭 Compass`
header (not inside the detail list):

```
> <1: where we are relative to intent — the primary gap or confirmation.>
> <2: the dominant pattern in the top learnings — what the corpus is signalling now.>
> <3 (optional): what this means for today's goals.>
> <4 (optional): an anomaly or risk — elevated complexity, exploration gap, stale reality.>
```

Write it from the data, not the user's words; no hedging. Omit it when the corpus has fewer
than 3 learnings or on a first-ever session.

Present a concise brief: Step 1's `brief_markdown`, with every carry-forward and tactical
backlog item flagged as a goal candidate, plus the sections it does not render:

```
**Reality (last known):**
<2–4 bullet summary of reality.md>

**External research signals:** *(only if `external_signals` is non-empty)*
<up to 3 most recent, each prefixed [research]>

**Drift / gaps detected:**
<1–3 specific gaps between intent and reality — be direct>

**Goal hit-rate:**
<last_session_hit_rate or "no prior session">

**Session complexity:** *(omit if session not open or avg_last_5 is null)*
<session_complexity.current> compass calls (avg: <session_complexity.avg_last_5> over last 5 sessions)
```

Only when `brief_markdown` is missing, load the full format with
`~/.claude/skills/compass/scripts/compass.sh prompt orient-brief-format --section "Step 2 — ORIENT brief format (P56 zone-grouping included)"`.

**Conditional advisories:** check these fields in Step 1's Advisory inputs line — `session_complexity.elevated` ·
`goal_stats.total_goals > 0 AND goal_stats.hit_rate < 70` · `reality_completeness.regression` ·
`exploration_ratio.low` · `reality_structure_warnings` non-empty · `outcome_rate < 0.3` ·
`contract_coverage < 0.4` · `quality_trend == "declining"` · `quality_plateau.plateaued` with a
pulled-forward cadence · `quality_trend == "declining"` AND `quality_plateau.trend ==
"improving"` (takes priority over the plain declining message). If **any** hold, load
`~/.claude/skills/compass/scripts/compass.sh prompt orient-brief-advisories --section "ORIENT brief — conditional advisory blocks"`
and append the matching block(s) in the order listed there. If **none** hold, do not load it.

**Record surfacing:** right after presenting the brief, run silently `~/.claude/skills/compass/scripts/compass.sh
surface-learnings <namespace>` (and `surface-learnings global` if the global cross-project block
was shown). It bumps `times_surfaced` for the learnings shown. Call it from no other step.

### Step 2b — Skill opportunity detection

After the brief, run the router's **Skill opportunity detection** pass. If signals exist, show
the `💡 Tooling opportunities` block as its own prompt, **separate from the brief**, and wait for
Y / S / D before Step 2b.1. No signals: continue silently.

**Cadence-gate convention (2b.1, 2b.3b, 2b.4b):** each checks a `*_due` flag (from
`orient-flags`). `false` → skip entirely, no prompt. `true` → surface once as part of ORIENT,
never again at close or mid-session. `code_review_due`, `research_due` and `dream_due` follow
the same rule but fire at Step 3f, not here.

**Checklist:** a long side-quest (e.g. many stale bullets at Step 2c) makes it easy to skip a
2b.x sub-step. Before leaving ORIENT confirm each was looked at (skip is a valid outcome):
☐ 2b.1 (assumption audit) · ☐ 2b.3b (CLAUDE.md hygiene) · ☐ 2b.4b (SkillOpt) ·
☐ 2b.5 (code context) · ☐ 2b.6 / 3f-item-3 (dream pass)

### Step 2b.1 — Strategic Assumption Audit

If `assumption_audit_due` is `false`, skip entirely (it is also `false` when there is nothing to
audit). If `true`, load
`~/.claude/skills/compass/scripts/compass.sh prompt assumption-audit --section "Step 2b.1 — Strategic Assumption Audit (P47)"`
and follow it.

### Step 2b.3b — Periodic CLAUDE.md hygiene review

If `claude_review_due` is `false`, skip entirely. If `true` and the namespace has no `CLAUDE.md`
(`repo_path/CLAUDE.md`, or `~/.claude/skills/<namespace>/CLAUDE.md` when `repo_path` is unset),
run `record-claude-review <namespace>` and skip silently. Otherwise load
`~/.claude/skills/compass/scripts/compass.sh prompt claude-md-hygiene-review --section "Step 2b.3b — Periodic CLAUDE.md hygiene review"`
and follow it.

### Step 2b.4b — Periodic SKILL.md optimisation pass

If `skill_opt_due` is `false`, skip entirely. If `true`, load
`~/.claude/skills/compass/scripts/compass.sh prompt skillopt-protocol --section "Gate prompt (Step 2b.4b)"`
On Y, load
`~/.claude/skills/compass/scripts/compass.sh prompt skillopt-protocol --section "Evidence collection + reflect pass (on Y)"`;
n/later defers (counter not reset).

### Step 2b.5 — Code context (cold-start orientation)

Step 1's brief already includes `code_context.md` when it is non-empty. Show it
as-is (no summarising) under `📁 Code context (last session):`; otherwise skip silently. No
confirmation.

### Step 2c — Reality validation protocol

If `stale_bullet_count` is 0, skip to Step 2d. Otherwise load
`~/.claude/skills/compass/scripts/compass.sh prompt reality-and-hypothesis-validation --section "Step 2c — Reality validation protocol (P0.1)"`
and follow it (one `auto-verify-reality` call, then a prompt for anything left in `manual`).
**Gating:** do not proceed to DECIDE until it resolves.

### Step 2d — Hypothesis validation

If `pending_validations` is empty, skip to Step 2f. Otherwise load the Step 2d section of the
same `reality-and-hypothesis-validation` prompt (the Confirmed/Disproven/Untested prompt per
expired hypothesis).

### Step 2f — Session hygiene precondition check

Take `last_close` from `orient-flags` (never a second `read`) and compute the hours elapsed. If under `session_cooldown_hours` (default 4), load
`~/.claude/skills/compass/scripts/compass.sh prompt orient-gates --section "Step 2f — Session hygiene"`
and follow it: a **blocking** reason prompt, then `acknowledge-cooldown-violation`, which opens
the session itself. Otherwise (or never closed), skip silently.

### Step 2h — Intent drift check

If `intent_changed` is `false`, skip silently. If `true`, load
`~/.claude/skills/compass/scripts/compass.sh prompt orient-gates --section "Step 2h"`
and follow it: a gating Y/N drift prompt recorded with `set-intent` now, never re-prompted at close.

### Step 3 — DECIDE

Propose up to `suggested_goal_count.count` session goals (default 4 with no history) as a
numbered list. Ground them in the gaps found, including the carry-forward and `backlog_tactical`
items flagged as candidates at ORIENT. Keep each specific and completable in one session.

**Order by natural execution sequence**, not priority alone:
- **Dependencies first** — if B needs the state A leaves, list A first.
- **Structural before additive** — refactors that make later goals cleaner go earlier.
- **Exploit before explore** — so exploration has concrete session context.
- **Riskier first** — while scope can still adapt.

Add one line: `*Order: <brief reason>*`. Then ask: *"Does this match your intent for today? Add,
remove, or reorder before I lock it in."* Accept additions, removals and rephrasing; "yes" or
"go" confirms the proposal.

### Step 3b — Consolidated pre-lock advisory batch

Run after Step 3 is confirmed, before Step 3f. Build one numbered list, one line per item,
**omitting any line whose trigger is false** (never renumber around a gap):

1. **`[GIT/REALITY]`** — only if a scan (stale "missing"/Backlog bullets vs
   `gitlog`; new files not in reality's "What exists") finds a match. For each Backlog bullet
   that names another repo (a sibling skill or another namespace), also run `git -C <that repo>
   log --oneline -20`; a commit there that does what the bullet asks is a match too (preview:
   `<repo>@<hash>`).
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
and follow it for the parsing, routing and rules. It computes `b1` and `b3` follow-ups before the
screen and shows small ones (≤ 3) inline, so Enter answers them.

**One screen with Step 3f (default):** do not show 3b alone. Render its lines as `b1`–`b4` at the top
of Step 3f's screen, under a `Before locking:` header, and take one response for both (tokens
`b2Y`, `3n`, `5:EEXX`). Tally one `batch` screen. If a `b2`/`b4` yes changes the goal list, re-ask
only the `[GOAL TYPES]` line afterwards. Show 3b alone only when the user asks. When Step 3b renders no lines, put Step 3's goal list on this
screen too and take one response for goals and overrides; tally `goals` and `batch`.

### Step 3f — Consolidated DECIDE-tail batch

Run after Step 3b (its lines share this screen), before Step 3c. Build one numbered list, one line per item, **omitting any
line whose trigger is false** (a line's number is stable across the session):

1. **`[CODE REVIEW]`** — only if `code_review_due`. Cite `code_review_status.sessions_since_review`/
   `interval_sessions` (or the complexity pull-forward basis) and `code_review_defer_count`.
   Default: **yes**.
2. **`[RESEARCH]`** — only if `research_due`. Cite `research_status.sessions_since_research`/
   `interval_sessions` (or the pull-forward basis) and `research_defer_count`. Default: **yes**.
3. **`[DREAM PASS]`** — only if `dream_due`. Cite `dream_status.sessions_since_dream`/
   `interval_sessions` and its `corpus_size`, `cluster_count_estimate` and `weak_count_estimate` (all in
   `orient-flags`; the line needs no load). Default: **yes**.
4. **`[STRETCH GOAL]`** — only if all confirmed goals are execution tasks with none
   exploratory/design, OR the last 3 `goal_completion_trend` values are all `100.0`. Name the
   trigger. Default: **no**.
5. **`[GOAL TYPES]`** — always. Default: **all exploit** (`E` per goal).
6. **`[CONTRACTS]`** — always. Default: **no** (`goal_contracts` stays empty).
7. **`[HYPOTHESES]`** — always. Default: **no**.

If item 1, 2 or 3 renders, first load both sections in one call:
`~/.claude/skills/compass/scripts/compass.sh prompt decide-tail-cadence-items --section "Rendering lines 1/2" --section "Routing items 1–3"`
Skip it when only items 4–7 render.

Render:
```
1. [CODE REVIEW] due (<sessions_since_review>/<interval_sessions> sessions, deferred <N>×) → run? (default: yes)<escalation hint>
2. [RESEARCH] due (<sessions_since_research>/<interval_sessions> sessions, deferred <N>×) → scope Q&A? (default: yes)<escalation hint>
3. [DREAM PASS] due (<sessions_since_dream>/<interval_sessions> sessions; <corpus_size> learnings, ~<cluster_count_estimate> near-dup, ~<weak_count_estimate> weak) → run? (default: yes)
4. [STRETCH GOAL] <trigger: all-execution-tasks | 3× 100% completion> → add one? (default: no)
5. [GOAL TYPES] tag each goal E/X, e.g. "EEXEE" for 5 goals (default: all E)
6. [CONTRACTS] pre-specify success criteria for any goal? (default: no)
7. [HYPOTHESES] log assumptions to validate at close? (default: no)

Accept all defaults, or override by number? [Enter to accept all]
```

**Enter** accepts every default:
items 1–3 take their yes-path, item 4 no, item 5 all `E` (`goal_types` = `exploit` per confirmed
goal), items 6 and 7 no (`goal_contracts` stays empty; Step 4.5 is skipped). Any other response →
load `~/.claude/skills/compass/scripts/compass.sh prompt decide-tail-cadence-items --section "Step 3f — Override parsing and routing (items 4–7)"`
and follow it.

### Step 3c — Architecture constraint check

Only if the namespace has a `repo_path`; otherwise skip silently without loading anything. If
set, read the Step 3c section of `contract-and-architecture-checks` (service/module-name
detection and the pom.xml dependency-declaration check).

---

### Step 4 — Lock and open

Once confirmed, open the session:
```bash
~/.claude/skills/compass/scripts/compass.sh open <namespace> '{"goals": [...], "goal_types": ["E", "X", ...]}' '<raw_impulse from Step 0>'
```
`goal_types` is Step 3f item 5's tags (one `E`/`X` per goal); `open` stores them for close.
Omit the third argument if `raw_impulse` was skipped. Skip `open` if `acknowledge-cooldown-violation`
(Step 2f) already opened the session. `assumption_audit_auto_reset: true` needs no action.

Then put the goals on the host's todo list, if it has one. **Either way**, as each goal completes, call:
```bash
~/.claude/skills/compass/scripts/compass.sh set-goal-status <namespace> '{"index": <0-based goal index>, "done": true}'
```
The Drift nudge reads this when no host todo tool exists. Skip it only if you have confirmed the
TodoWrite-equivalent exists and is what Drift nudge reads.

**Cold-start seeding:** if `seeded_count` > 0 (only on a namespace's first-ever open), say
once, no action: `🌱 Seeded <seeded_count> learning(s) from <seed_source> — cold-start context,
not yet validated in this namespace.` Otherwise skip silently.

### Step 4.5 — Pre-session hypothesis elicitation

The Y/n gate is item 7 of Step 3f. Only if its answer was **Y**, load
`~/.claude/skills/compass/scripts/compass.sh prompt hypothesis-elicitation --section "Step 4.5 — Pre-session hypothesis elicitation (P15)"`
and follow it: one `log-learning` hypothesis per assumption, until the user says n.

Confirm: `✓ Session open. Todo list set. Let's go.`

### Step 4.6 — Goal-conditioned supplemental retrieval

After the confirm, run silently:
```bash
~/.claude/skills/compass/scripts/compass.sh query-learnings <namespace> \
  '{"query": "<all confirmed goals joined by space>", "limit": 3}'
```
With **fewer than 2 results** or **fewer than 5 learnings** in the corpus, skip silently.
Otherwise surface once, no action needed:

```
📚 Session-relevant learnings:
  · "<learning text, first 90 chars>" [w:<weight>, <age in days>d] ← <tag1>, <tag2>
  · ...
```
One pass only; never re-run mid-session.

---

## Return

Once Step 4.5/4.6 confirms and the todo list is set, this sub-skill is done; nothing goes back
to the router.

**Prompt-count tally:** note each interactive screen Steps 0–4.6 show and, once the todo list is
set, tally them all in one call:
```bash
~/.claude/skills/compass/scripts/compass.sh tally-prompt <namespace> open <screen>[,<screen>...]
```
Keys: `gate`, `impulse` (only if Step 0 asked), `orient`, `skill` (2b), `audit` (2b.1),
`hygiene` (2b.3b), `skillopt` (2b.4b), `hypothesis` (2d, 4.5), `cooldown` (2f), `drift` (2h),
`goals`, `advisories` (3b), `batch` (3f, only if rendered), `research`, `arch` (3c); others are
refused, and `auto-verify-reality` tallies `stale` itself. A screen is one round trip that
waited for a response (Step 3f's batch is **one**). Don't estimate the count.
