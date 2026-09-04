# compass-open

OODA-open sub-skill for the compass loop (Track 2 Phase 1, 2026-08-29 —
`docs/compass-open-extraction-sop.md` in the `compass` skill). Mirrors the
`compass-close` extraction (P25) exactly: content-preserving move, no behaviour
change.

Invoked from `/compass`'s router when `open_session = false`. Owns the full OODA
half — Step 0 through Step 4.6 — from the pre-mediation impulse capture through
the goal-conditioned supplemental retrieval that runs right after session
confirmation.

**v1 consolidation (2026-09-01, `docs/2026-08-31-consolidate-open-close-prompts-plan.md`
in the `compass` skill):** the separate code-review-due / research-due / dream-pass-due
prompts (formerly Steps 2b.3, 2b.4, 2b.6) and the separate ambition-nudge / goal-type-tagging
/ verification-contract prompts (formerly Steps 3b.5, 3b.6, 3b.8) are now one batched screen
— **Step 3f** — fired once, right after the DECIDE goal list is confirmed. The `*_due` flags
themselves are still read at Step 1 OBSERVE; only the *prompt* moved. Step 4.5's Y/n gate
is folded into Step 3f's item 7; only its free-text-per-hypothesis follow-up loop still runs
where Step 4.5 used to be. v2 (per-item confidence-gated auto-apply using P59 override
history) is scoped in the same plan doc but not built — see the Strategic backlog.

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
ask one optional question:

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

### Step 1 — OBSERVE

Run the consolidated command — one script call instead of the seven separate
`read`/`gitlog`/`carry-forward`/`read global`/`list-artefacts`/`watch-signals` calls
this step used to make:
```bash
python3 ~/.claude/skills/compass/scripts/compass.py generate-orient-brief <namespace>
```

**This output is often large enough that the harness persists it to a file instead of
inlining it (2026-09-02 session-review finding — a real session `Read` the entire
43KB persisted file into context here, most of it the raw `context`/`recent_history`/
`cycle_history` JSON already re-expressed by `brief_markdown` below). When that
happens, do NOT `Read` the whole file.** Extract only what a given step actually needs
with a targeted one-liner, e.g.:
```bash
python3 -c "import json; d=json.load(open('<persisted-file>')); print(d['brief_markdown'])"
```
and pull individual `context.*` fields (the cadence-due flags Step 2b onward check,
`session_index` for `expand-session` lookups, etc.) the same way, one field at a time,
rather than loading the full structure into context at once.

Returns `{context, gitlog, carry_forward, global_cross_project, artefacts_matched,
watch_signals, brief_markdown}`. `context` is the same dict `read` returns (`intent`,
`reality`, `top_learnings`, `planned_actions`, `session_index`, all cadence-due flags,
etc.). `brief_markdown` is a pre-rendered markdown block covering the mechanical
sections of Step 2's brief (reality completeness, zone-grouped top-learnings,
cross-namespace signals, tactical backlog) — use it as the base and prepend the P48
synthesis blockquote plus any judgement-driven sections (drift/gaps, advisories),
which stay LLM-authored rather than templated.

`global_cross_project` is already the ≤3 tag-overlap-matched entries from `global`
(empty list if this namespace *is* `global`, or if there's no overlap — nothing further
to fetch). `artefacts_matched` is already the ≤2 tag-matched artefacts. `watch_signals`
is `null` when `config.watches` is empty, otherwise the same shape `watch-signals`
returns — check `watch_signals.empty` before rendering that section.

Still fetch two things separately — both need per-session context this command can't
supply:

- `session_index` (inside `context`) — compact index of the last 20 sessions (P28: one
  line each — id, date, goals, top learning, tags). Scan this to identify sessions
  relevant to carry-forward, hypothesis validation, or prior decisions, then call
  `expand-session <id>` for those sessions only:
  ```bash
  python3 ~/.claude/skills/compass/scripts/compass.py expand-session <namespace> <session-id>
  ```
- **BM25 goal-relevance query (R9):** if `session_index` contains planned actions for
  the most recent session, run:
  ```bash
  python3 ~/.claude/skills/compass/scripts/compass.py query-learnings <namespace> \
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

Present a concise orient brief to the user. Format:

```
## 🧭 Compass — <namespace>
Last session: <last_close or "never">

**Intent:** <one-line summary of intent.md>

**Reality (last known):**
<2–4 bullet summary of reality.md>

**Signals since last session:**
<git commits if any, grouped by theme; otherwise "none">

**Top learnings in play:** *(P56 zone-grouping — see below)*
<top 3 learnings with weight indicators — use ★ for weight ≥3, · for weight 1–2>

**Cross-project signals (global):** *(only if tag overlap found — P7)*
<up to 3 global learnings with matching tags — omit section if none>

**External research signals:** *(only if `external_signals` array is non-empty — P11)*
<up to 3 most recent signals from external_signals.jsonl — prefix each with [research] so provenance is visible>

**Relevant artefacts:** *(only if tag-matched artefacts found — P41)*
<up to 2 artefacts: "title" (type, date) — description — omit section entirely if none>

**📡 Cross-namespace signals:** *(only if watch-signals returned empty: false — P54)*
For each watched namespace with signals, group as:
  `<watched_ns>`: <N> decisions, <M> learnings
  · [decision] "<decision text, first 80 chars>" [tag1, tag2]
  · [learning] "<learning text, first 80 chars>" [w:<weight>]
Omit section entirely when empty: true.

*If `bootstrapped: true` (P54 Phase 2)* — this namespace has no tagged learnings of its own yet,
so signals below are unweighted tag-presence matches rather than tag-frequency-scored ones. Add
one line under the section header, not per-signal:
  ↳ *(bootstrapping: this namespace has no learnings yet — signals shown are unscored tag matches)*

**Carry-forward from last session:** *(only if incomplete list is non-empty — P5)*
For each item, show text plus any reason annotation if present (P14):
  • "goal text" [blocked: waiting on Y API]
  • "goal text" [abandoned]
  • "goal text"  ← no status captured
Flag all as goal candidates.

**Tactical backlog:** *(only if `backlog_tactical` is non-empty — docs/namespace-backlog-standard.md)*
`backlog_tactical` is already in `read`'s output — no extra command needed. These are
items the namespace's own reality.md marked ready-to-act-on (`## Backlog` → `### Tactical`),
distinct from carry-forward (unfinished *goals* from last session) — a Tactical backlog
item may never have been a session goal at all. Show each with its `[source: ...]` tag intact:
  • "item text" [source: code-review-2026-07-10]
  • "item text" [source: research]
Flag all as goal candidates, same as carry-forward.

**Drift / gaps detected:**
<1–3 specific gaps between intent and reality — be direct>

**Goal hit-rate (P0.2):**
<last_session_hit_rate or "no prior session">

**Reality completeness (P22):** *(omit if score is null)*
<reality_completeness.score>% of reality bullets carry completion markers (<achieved>/<total>)

**Session complexity (P24):** *(omit if session not open or avg_last_5 is null)*
<session_complexity.current> compass calls (avg: <session_complexity.avg_last_5> over last 5 sessions)
```

**P56 — Top learnings zone-grouping rule:** each entry in `top_learnings` may carry a `zone` field (`golden` | `warning` | `preference` | absent). If **2 or more** of the surfaced learnings carry a zone, group the "Top learnings in play" block by zone instead of one flat list:

```
**Top learnings in play:**

  ✓ Golden (replicate):
    ★ "<learning text>" [w:3]

  ⚠ Warning (avoid):
    · "<learning text>" [w:1]

  ○ Preference:
    · "<learning text>" [w:2]

  (unclassified):
    ★ "<learning text>" [w:3]
```

If **fewer than 2** surfaced learnings carry a zone, render the original flat weight-sorted list (today's format) — grouping a mostly-unclassified corpus into near-empty buckets is worse than the flat list it replaces.

**Conditional advisories:** check these fields from the current `read` output —
`session_complexity.elevated` · `goal_stats.hit_rate < 70` ·
`reality_completeness.regression` · `exploration_ratio.low` ·
`reality_structure_warnings` non-empty · `outcome_rate < 0.3` ·
`contract_coverage < 0.4` · `quality_trend == "declining"` ·
`quality_plateau.plateaued` with a pulled-forward cadence ·
`quality_trend == "declining"` AND `quality_plateau.trend == "improving"` (P75 —
takes priority over the plain declining message when both hold).
If **any** hold, read `scripts/prompts/orient-brief-advisories.md` and append the
matching block(s), in the order listed there. If **none** hold, skip entirely —
don't load that file for a brief with nothing to add.

**Record surfacing (P32/P44):** immediately after presenting the brief above (and the
global cross-project block, if shown), run silently:
```bash
python3 ~/.claude/skills/compass/scripts/compass.py surface-learnings <namespace>
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

### Step 2b.1 — Strategic Assumption Audit (P47)

Check `assumption_audit_due` from read output. If `false`, skip entirely. If `true`, read
`~/.claude/skills/compass/scripts/prompts/assumption-audit.md` and follow it — it covers
candidate selection, the P50/P59 advisory add-ons, the V/C/D/S response mapping, and the
counter reset.

### Step 2b.3 — Periodic code quality review (moved)

`code_review_due` and its complexity-pull-forward signal are still read at Step 1
OBSERVE — nothing fires here. The prompt, spawn-agent flow, and defer/escalate path
now live in **Step 3f**, item 1 (v1 consolidation, 2026-09-01).

---

### Step 2b.3b — Periodic CLAUDE.md hygiene review

Check `claude_review_due` from read output. If `false`, skip entirely. If `true`, check
whether a `CLAUDE.md` exists for this namespace (`repo_path/CLAUDE.md`, or
`~/.claude/skills/<namespace>/CLAUDE.md` if `repo_path` is unset) — if none, reset the
counter (`record-claude-review <namespace>`) and skip silently. Otherwise read
`~/.claude/skills/compass/scripts/prompts/claude-md-hygiene-review.md` and follow it.

---

### Step 2b.4 — Periodic external research (P11) — compass-research-scope (moved)

`research_due` and its complexity-pull-forward signal are still read at Step 1 OBSERVE —
nothing fires here. The prompt, `compass-research-scope` invocation, and defer/escalate
path now live in **Step 3f**, item 2 (v1 consolidation, 2026-09-01).

---

### Step 2b.4b — Periodic SKILL.md optimisation pass (P-SkillOpt)

Check `skill_opt_due` from read output. If true, check `skill_opt_status.friction_gate.sufficient_signal` (P61c) before choosing which prompt to surface.

**`sufficient_signal: true`** (≥3 unresolved `skill_feedback.jsonl` entries) — surface the standard prompt:
```
🔧 SKILL.md optimisation pass due — <sessions_since_skill_opt> sessions since last run.
   Run evidence collection? [Y / n / later]
```

**`sufficient_signal: false`** — thin signal, same failure mode as a too-small validation split (diagnose→propose has little to act on). Surface the friction-aware variant instead:
```
🔧 SKILL.md optimisation pass due — <sessions_since_skill_opt> sessions since last run.
   Only <open_feedback_count> skill-feedback entries logged since the last pass (threshold: <threshold>).
   Proceed anyway, or wait for more signal? [Y — proceed / n / later — wait]
```
Treat **Y** identically to the standard prompt's Y (run evidence collection). **n / later** defer exactly as below — do not add a second deferral path.

#### Y — Run evidence collection + reflect pass

Read `~/.claude/skills/compass/scripts/prompts/skillopt-protocol.md` and follow it. It
covers the full S1–S6 cycle: freezing/scoring the holdout, the three-pass reflect
protocol (analyst_error → analyst_success → merge_final), edit proposal/apply/reject,
and the promote/reject/re-freeze decision at the end.

#### n or later — Defer

Continue silently. Counter is not reset.

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

`dream_due` is still read at Step 1 OBSERVE — nothing fires here. The prompt and
`dream-pass-protocol.md` invocation now live in **Step 3f**, item 3 (v1 consolidation,
2026-09-01).

### Step 2c — Reality validation protocol (P0.1)

Check `reality_stale_bullets` from the `read` output. If empty, skip to Step 2d's check
below. If non-empty, read `~/.claude/skills/compass/scripts/prompts/reality-and-hypothesis-validation.md`
and follow its Step 2c section — it covers the implicit filesystem/git-history auto-verify
pass and the manual confirmation prompt for anything that survives it. **This is a gating
step** — do not proceed to DECIDE until it resolves.

### Step 2d — Hypothesis validation (P1.1)

Check `pending_validations` from the `read` output. If empty, skip to Step 2e. If
non-empty, read the same `reality-and-hypothesis-validation.md` file's Step 2d section —
it covers the Confirmed/Disproven/Untested prompt per expired hypothesis.

### Step 2e — Deferred skill escalation (P1.3)

Check `escalation_candidates` from read output. If any opportunities have defer_count >= 2:

```
⬆ Deferred skill escalation — <N> opportunity(ies) escalated from prior deferrals.

- "<opportunity text>" (deferred 2 times)
  Commit to goal this session? Y/N

- "<opportunity text>" (deferred 2 times)
  Y/N?
...
```

For each escalation candidate, ask: **Y**es (commit to goal) or **N**o (acknowledge but skip this session).

Map responses:
- **Y** (Commit) → add to this session's goals in DECIDE step; the opportunity is treated as a priority gap
- **N** (Skip) → note it; if not completed by CLOSE, another deferral will be recorded at CLOSE time

If no escalation candidates exist, skip silently and continue to DECIDE.

### Step 2f — Session hygiene precondition check (P4.2)

Before proceeding to DECIDE, check if this namespace was last closed less than 4 hours ago
(or less than the configured cooldown, if customised):

```bash
python3 ~/.claude/skills/compass/scripts/compass.py read <namespace>
```

Check `last_close` timestamp. Calculate hours elapsed. If elapsed time is less than the
configured `session_cooldown_hours` (default: 4):

```
⏸ Session hygiene violation — namespace reopened within 4h cooldown.
Last closed: <N>h ago.

Why are you resuming so soon? (brief reason)
```

**This is a blocking check.** User must provide a reason (one sentence minimum) to proceed.

Once they provide a reason, acknowledge the violation:

```bash
python3 ~/.claude/skills/compass/scripts/compass.py acknowledge-cooldown-violation <namespace> '{
  "reason": "<user-provided reason>",
  "raw_impulse": "<raw_impulse from Step 0, or omit the key entirely if it was skipped>"
}'
```

This opens the session immediately. Continue to DECIDE as normal — the `open` call in
Step 4 is idempotent when a session is already open: it will update `planned_actions`
without re-incrementing any counters. No special handling required.

If no violation (last_close is more than cooldown_hours ago, or never closed), skip silently.

### Step 2g — Git vs reality reconciliation

Run **only if** `gitlog` returned commits this session. Silently scan two things:

**A — Stale "missing" bullets:** read reality's `## Backlog` section (or, for a namespace
not yet migrated to the standard — docs/namespace-backlog-standard.md — whichever of
`## What's missing` / `## What's next` it still uses). For each bullet, check whether
any commit subject or changed filename from the gitlog plausibly addresses it. Match
loosely — component names, feature names, file paths.

**B — New files not in reality:** identify files created in commits (new file additions)
that do not appear anywhere in reality's "What exists" section.

If **no matches** from either scan: skip silently — no prompt, no output.

If **matches found**, surface once before DECIDE. Present each match with the specific
proposed change, not a vague signal:

```
⚠ Git vs reality — <N> proposed update(s):

  • PROPOSED: move "reality bullet about X" → "What exists and works"
    Source: commit a3f2c1 "<commit subject>" (<N>d ago)
    Accept? [Y / N / M(anual edit)]

  • PROPOSED: add "path/to/file.py — <inferred description>" to "What exists"
    Source: new file in commit b7d2e3
    Accept? [Y / N / M(anual edit)]
```

Per-item responses:
- **Y** → mark as accepted; hold the exact proposed wording for CLOSE. Each accepted
  item here is a single bullet — a "move" is one `remove-reality-bullet` (Backlog) plus
  one `append-reality-bullet` (What exists and works); a "new file" add is one
  `append-reality-bullet`. Prefer these over reconstructing the whole document at close
  (2026-08-28 audit finding #4), unless several other bullets are also being reworded
  in the same close, in which case a single `update-reality` covering all of them is
  the better trade.
- **N** → skip this update; the bullet stays as-is.
- **M** → ask for the user's preferred wording; note it for CLOSE.

**Rule:** this is a signal, not a gate. If the user rejects all, do not block goal-setting.
The wiki-frontend failure mode (two sessions building already-shipped features) is the
specific thing this prevents.

**Match confidence filter:** only propose an update when the match is specific — a commit
subject or changed filename containing the exact component or feature name from the reality
bullet. Do not propose updates based on loose keyword proximity alone.

### Step 2h — Intent drift check (P1.2)

Check `intent_changed` from the read output. If `false`: skip silently.

If `true`, surface before DECIDE — this is a gating check, the session direction depends on it:

```
⚠ Intent drift detected.

**Previous:** <prev_intent from read output>
**Current:**  <intent from read output>

Deliberate shift? Y (confirm) / N (revert)
```

**Y — confirm:**
```
Why did intent shift? (one sentence for the record)
```
Record immediately:
```bash
python3 ~/.claude/skills/compass/scripts/compass.py set-intent <namespace> \
  '{"text": "<current intent text>", "reason": "<user reason>"}'
```

**N — revert:**
```bash
python3 ~/.claude/skills/compass/scripts/compass.py set-intent <namespace> \
  '{"text": "<prev_intent text>", "reason": "reverted at ORIENT — drift was not intentional"}'
```
Confirm: `✓ Intent reverted. Proceeding with original direction.`

**Rule:** action happens now, at ORIENT — do not re-prompt at close.

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

### Step 3b — Vague goal detection (P_GOAL)

After the user confirms (or edits) the goal list, scan each goal for vagueness signals
**before** locking:

**Vague if any of:**
- Contains: "work on", "look at", "investigate", "sort out", "pick up", "think about",
  "explore", "continue with", "review", "have a look"
- Fewer than 5 words with no concrete output verb (e.g. "auth service", "the Redis thing")
- No observable deliverable implied (cannot be marked done without subjective judgment)

If **one or more goals** are vague, surface once — do not flag each individually:

```
💭 Goal(s) <N, M> look broad — a sharp definition of done will help compass track
   them accurately.

Sharpen with /goal? [Y / n / skip all]
```

- **Y** → for each flagged goal, invoke the `goal` skill in `--session` mode with the
  vague goal text pre-loaded as context. Replace the flagged goal(s) in the list with
  the returned `SESSION_GOALS` criteria. Re-present the updated goal list for a final
  confirm before locking.
- **n / skip** → lock the goals as-is; note in the open payload that these goals have
  no measurable criteria (they will surface as low-confidence at close if incomplete).

**Rule:** only fire once per DECIDE. If the user says skip, do not re-flag at close.

### Step 3f — Consolidated DECIDE-tail batch (v1 consolidation, 2026-09-01)

Run **after** Step 3b (vague goal detection), **before** Step 3c (architecture check).
Replaces the formerly-separate prompts for code review due (old 2b.3), research due
(old 2b.4), dream pass due (old 2b.6), goal ambition nudge (old 3b.5), goal type tagging
(old 3b.6), and verification contract offer (old 3b.8) — plus folds in the Y/n gate half
of hypothesis elicitation (old Step 4.5's first question; the free-text-per-hypothesis
follow-up still runs separately after Step 4, since that needs new user-supplied content
the assistant cannot infer — see `docs/2026-08-31-consolidate-open-close-prompts-plan.md`'s
own batching rule).

Build one numbered list, one line per item, **omitting any line whose trigger is false**
(never renumber around a gap — a line's number is stable across a session):

1. **`[CODE REVIEW]`** — only if `code_review_due` is true. Cite `sessions_since_review`/
   `interval_sessions` (or the complexity-pull-forward basis from `code_review_status`,
   same framing rule the old Step 2b.3 used) and `code_review_defer_count`. Default: **yes**.
2. **`[RESEARCH]`** — only if `research_due` is true. Cite `sessions_since_research`/
   `interval_sessions` (or the pull-forward basis) and `research_defer_count`. Default: **yes**.
3. **`[DREAM PASS]`** — only if `dream_due` is true. Cite `sessions_since_dream`/
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

Render:
```
1. [CODE REVIEW] due (<sessions_since_review>/<interval_sessions> sessions, deferred <N>×) → run? (default: yes)
2. [RESEARCH] due (<sessions_since_research>/<interval_sessions> sessions, deferred <N>×) → scope Q&A? (default: yes)
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

**Parsing overrides:** a bare **Enter** accepts every default. Otherwise parse
space-separated tokens, each `<N><value>` (or `<N>:<value>` for a longer value):
`1n` / `1later` / `4Y` / `5:EEXX` / `6S` / `7Y`, in any combination.

**Route each item's answer exactly as its original step did:**

| Item | On yes / non-default | On no / default-skip |
|---|---|---|
| 1 CODE REVIEW | Spawn the review agent: `mkdir -p ~/.claude/loop/<namespace>/code_reviews`, spawn an Agent with the prompt template at `~/.claude/skills/compass/scripts/prompts/code-review-agent.md` (substitute `<target_path_or_diff>` with `<repo_path>`, pass verbatim). Then: save the report to `~/.claude/loop/<namespace>/code_reviews/<YYYY-MM-DD>.md`; scan it for `[CRITICAL]` lines and add each directly to the session todo list (prefixed `[code review]`, no asking); call `record-review <namespace>`; surface the report path plus up to 5 `[HIGH]`/`[CRITICAL]` findings and offer *"Extract top issues as next-session goals? [Y/n]"* — **Y** appends up to 3 (user-editable) to `reality.md`'s `## Backlog` → `### Tactical` via `append-reality-bullet`, tagged `[source: code-review-<date>]`. | `later` and `n` both defer: `defer-code-review <namespace>`. Check `defer_count`/`escalate`. `escalate: false` → no further comment. `escalate: true` → append *"Run periodic code quality review (overdue)"* to the already-confirmed goal list and say so (see escalation note below — DECIDE has already run by this point in the v1 flow). |
| 2 RESEARCH | Invoke `/compass-research-scope namespace=<namespace>` — handles its own Q&A, agent spawn, `record-research`, and result parsing (the sub-skill has no gate of its own; this Y/non-default answer *is* the gate). | `later`/`n` → `defer-research <namespace>` (called directly here, never via the sub-skill). Check `defer_count`/`escalate`. `escalate: true` → append *"Run periodic external research pass (overdue)"* to the confirmed goal list, same escalation-after-lock note as item 1. |
| 3 DREAM PASS | Read `~/.claude/skills/compass/scripts/prompts/dream-pass-protocol.md` and follow it. | Skip; counter untouched (the old Step 2b.6 had no defer command either — `dream_due` simply re-fires next ORIENT). |
| 4 STRETCH GOAL | Ask: *"One sentence: what's the open design question or hypothesis worth tackling?"* Add as an additional confirmed goal. | Proceed silently. |
| 5 GOAL TYPES | Parse the string into a `goal_types` array (e.g. `"EXE"` → `["exploit", "explore", "exploit"]`). Store alongside goals; passed in the close payload as `"goal_types": [...]`. | Default-fill `goal_types` as all `"exploit"` for the confirmed goal count (see the item-5 note above). |
| 6 CONTRACTS | Read `~/.claude/skills/compass/scripts/prompts/contract-and-architecture-checks.md`'s "Verification contract offer" section and follow its Y/S per-goal criteria capture (`log-goal-contract` per goal). | `goal_contracts` stays empty for this session. |
| 7 HYPOTHESES | Proceed into Step 4.5's existing per-assumption free-text loop (unchanged — this is the part that needs new user-supplied content, so it stays interactive after Step 4's lock). | Skip Step 4.5 entirely; no hypotheses logged. |

**Escalation-after-lock note (items 1/2 only):** Step 3 (DECIDE) has already run and the
goal list is already confirmed by the time Step 3f fires — a deliberate ordering change
from the pre-v1 flow, where code-review/research escalation used to happen *before* goals
were locked, as part of the original numbered DECIDE proposal. An `escalate: true` response
here means: append the item to the already-confirmed list and say so plainly, e.g.:
```
⬆ Code review escalated (deferred <N>×) — adding to this session's already-confirmed goals.
```
Do not re-open the whole goal list for edits at this point — just append and continue.

**Rules:**
- One rendered screen. One round of overrides. No follow-up screen for items 1–3, 6 —
  their Y-path work (agent spawn, sub-skill invocation, contract capture) happens
  immediately after parsing the response, same as it did inline in their original steps.
- Item 4's stretch-goal free text and item 7's hypothesis free text are still separate
  interactive follow-ups **after** this screen resolves — batching a yes/no gate does not
  mean inventing content the user hasn't supplied yet.
- Advisory items (4) never block; cadence items (1–3) never block; 5/6/7 never block.

### Step 3c — Architecture constraint check (P_ARCH)

Run **only if** the namespace has a `repo_path` configured. If no `repo_path`, skip
silently — do not even read the prompts file. If configured, read the same
`contract-and-architecture-checks.md` file's Step 3c section — it covers the
service/module-name detection and the pom.xml dependency-declaration check.

### Step 3d — Pre-implementation research fork detection (P13)

Run **after** architecture check, **before** Step 4 lock. Advisory only — never blocks DECIDE.

**Trigger:** scan each confirmed goal for decision verbs:
*implement, migrate, replace, integrate, adopt, switch, introduce, rewrite, refactor, extract, redesign*

If **no goal** contains a decision verb: skip silently — no output. If at least one goal
does, read `~/.claude/skills/compass/scripts/prompts/research-fork-detection.md` and follow
it — it covers the query/scan, the hits-found and no-hits prompts, and the no-go/deferred
override sub-flow.

---

### Step 3e — Domain novelty check (P21)

Run **after** Step 3d, **before** Step 4 lock. Advisory only — never blocks DECIDE.

**Trigger:** for each confirmed goal, extract domain/component tokens — technology names, service names, architectural terms. Exclude decision verbs already handled by Step 3d.

*Example extraction: "Implement LCLM session index in cmd_read" → tokens: `LCLM`, `session index`, `cmd_read`*

For each token set, query learnings:
```bash
python3 ~/.claude/skills/compass/scripts/compass.py query-learnings <namespace> \
  '{"query": "<extracted tokens>", "limit": 3}'
```

**Classify results:**
- If query returns **no results**, or all results have `date` older than 90 days: the domain is unexplored or stale — flag it.
- If recent results exist (< 90 days): domain is familiar — skip silently.

**If any goals touch unexplored domains**, surface **once** as a single non-blocking block:

```
🔍 Domain novelty — limited recent learnings for:
  · "<goal text, condensed>" (token: <domain token>, last learning: <date or "none">)

  Worth adding an exploratory goal — hypothesis, spike, or research question? [Y / n / skip]
```

- **Y** → prompt: *"One sentence: what would you want to learn or confirm about this area?"* Add as a goal candidate in the DECIDE list or log as a hypothesis if the user prefers not to make it a full goal.
- **n / skip** → dismiss silently; do not re-flag.

**Rules:**
- One prompt maximum per DECIDE — batch all unexplored-domain hits.
- Do not fire if fewer than 3 prior sessions exist (insufficient history to distinguish "new" from "early").
- **Relationship to P13:** P13 checks for *prior decisions* on a chosen approach. P21 checks for *absence of prior experience* in a domain — complementary, not redundant.

---

### Step 4 — Lock and open

Once confirmed, open the session:
```bash
python3 ~/.claude/skills/compass/scripts/compass.py open <namespace> '<json_array_of_goals>' '<raw_impulse from Step 0, or omit the third arg entirely if it was skipped>'
```
The third argument is `raw_impulse` captured at Step 0 (P74 Phase 1) — omit it if the
user skipped that prompt. This call only stashes the value on a genuine first open; if
`acknowledge-cooldown-violation` already opened the session this cycle (Step 2f), that
call already carried `raw_impulse` and this one must not repeat it — `open`'s idempotent
re-open branch ignores a fourth positional value regardless, so passing it again here is
harmless but redundant, not double-counted.

Then populate the todo list with the confirmed goals, if this runtime has a
TodoWrite-equivalent tool (use it). **Either way**, as each confirmed goal actually
completes during the session, also call:
```bash
python3 ~/.claude/skills/compass/scripts/compass.py set-goal-status <namespace> '{"index": <0-based goal index>, "done": true}'
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

- **Y** (from Step 3f item 7) → for each assumption the user provides, log as a hypothesis. If the assumption clearly bets on one or more of this session's confirmed goals, include `goal_origin` (the goal's index/indices) in the same call — `log-learning` already accepts and stores it on create, so this does not require a second write:
  ```bash
  python3 ~/.claude/skills/compass/scripts/compass.py log-learning <namespace> '{
    "text": "<assumption text>",
    "tags": ["<inferred tag>"],
    "learning_type": "hypothesis",
    "confidence": "<high|medium|low — ask if unclear>",
    "test_window": "end of session",
    "goal_origin": [<goal index(es), or omit if cross-goal/exploratory>]
  }'
  ```
  Accept one assumption per prompt; ask "Another? [Y / n]" until done or n.
  These surface in the next close's `pending_validations`.

**Rule:** the Y/n gate (Step 3f item 7) fires once, never re-asked mid-session; the
free-text loop above only runs at all when that gate came back Y. No script changes
needed (log-learning already supports `goal_origin` on both create and weight-increment
paths — see `_upsert_learning`).

Confirm to the user:
```
✓ Session open. Todo list set. Let's go.
```

### Step 4.6 — Goal-conditioned supplemental retrieval (P43b)

After the confirm, run silently:
```bash
python3 ~/.claude/skills/compass/scripts/compass.py query-learnings <namespace> \
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

**Prompt-count tally (2026-09-01, `docs/2026-08-31-consolidate-open-close-prompts-plan.md`
in the `compass` skill):** keep a running count of every distinct interactive screen
actually presented to the user across Steps 0–4.6 — an "interactive screen" is one round
trip that waited for a response, not one script call or one bullet within a screen (Step
3f's batch counts as **one**, however many of its 7 items rendered). Carry this number
forward in working memory as `open_prompt_count` for the rest of the session — it gets
passed into the final `close` payload at `compass-close`'s Step 6, alongside its own
`close_prompt_count` tally, purely as write-only session metrics (no script or trend
reads it yet — see the Strategic backlog for the deferred read-side `avg_last_5`).
