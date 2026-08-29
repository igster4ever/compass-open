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

Read the JSON from `compass.py read`. Also fetch git signals:
```bash
python3 ~/.claude/skills/compass/scripts/compass.py gitlog <namespace>
```

Gather:
- `intent` — the stated north star
- `reality` — last known ground truth
- `top_learnings` — highest-weight learnings (already ranked; used as fallback)
- `planned_actions` from last session (if `last_close` is recent)
- `session_index` — compact index of the last 20 sessions (P28: one line each — id, date, goals, top learning, tags). Scan this to identify sessions relevant to carry-forward, hypothesis validation, or prior decisions. Call `expand-session <id>` to fetch the full snapshot for those sessions only. Skip the rest.
- git commits since last close (from gitlog)

```bash
python3 ~/.claude/skills/compass/scripts/compass.py expand-session <namespace> <session-id>
```

**BM25 goal-relevance query (R9):** if `session_index` contains planned actions for the most recent session, run:
```bash
python3 ~/.claude/skills/compass/scripts/compass.py query-learnings <namespace> \
  '{"query": "<last session planned actions joined>", "limit": 5}'
```
Use the returned `results` as `top_learnings` in the orient brief instead of the
weight-ranked list. Fall back to `top_learnings` from `read` if the query returns
fewer than 2 results (sparse corpus). Skip if no prior planned actions exist.

**Carry-forward check (P5):** also run:
```bash
python3 ~/.claude/skills/compass/scripts/compass.py carry-forward <namespace>
```
Capture the `incomplete` array — these are unfinished actions from the previous session and are natural goal candidates for DECIDE.

**Global orientation (P7):** if the current namespace is not `global`, also run:
```bash
python3 ~/.claude/skills/compass/scripts/compass.py read global
```
From the global `top_learnings`, surface up to 3 entries whose tags overlap with this namespace's top-learning tags. Hold these for inclusion in the orient brief. Skip silently if `global` has no learnings or no tag overlap.

**Artefact orientation (P41):** fetch recent artefacts whose tags overlap with the session context:
```bash
python3 ~/.claude/skills/compass/scripts/compass.py list-artefacts <namespace> \
  '{"tags": [<top 3 learning tags from top_learnings>], "limit": 5}'
```
From the returned list, keep at most 2 whose `tags` share ≥1 tag with the top-learnings. Hold for the orient brief. Skip silently if `artefacts` is empty.

**Cross-namespace feed (P54):** if `config.watches` is non-empty, call:
```bash
python3 ~/.claude/skills/compass/scripts/compass.py watch-signals <namespace>
```
If `response.empty` is false, hold the `signals` array for the orient brief. Skip entirely when `empty: true` — no output, no mention.

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

**Cadence-gate convention (applies to 2b.1, 2b.3, 2b.3b, 2b.4, 2b.4b, 2b.6):** each of
these checks a `*_due` flag from `read`. `false` → skip entirely, no prompt. `true` →
surface once as part of ORIENT, never re-surface at close or mid-session. This is assumed
below; only deviations from it are called out per step.

### Step 2b.1 — Strategic Assumption Audit (P47)

Check `assumption_audit_due` from read output. If `false`, skip entirely. If `true`, read
`~/.claude/skills/compass/scripts/prompts/assumption-audit.md` and follow it — it covers
candidate selection, the P50/P59 advisory add-ons, the V/C/D/S response mapping, and the
counter reset.

### Step 2b.3 — Periodic code quality review

Run this check **only if** the namespace has a `repo_path` configured. Check `code_review_due` from read output. If true, surface once as part of the orient brief.

`code_review_due` fires from two independent gates — a session/day cadence (`sessions_since_review >= interval_sessions` AND days-since ≥ `interval_days`) **or** a complexity pull-forward (`code_review_status.pulled_forward_by_complexity`): a git-diff-derived signal (lines changed, new files/modules, a newly added `docs/` design doc, or clustered complex/chaotic decisions since the last review) crossing threshold, gated at a minimum of `review_complexity_min_sessions` (default 2) sessions so a single big commit moments after opening doesn't demand a review every session. This directly answers the P-review-cadence feedback that a pure session-count trigger under- or over-fires relative to how much actually changed.

If `code_review_status.pulled_forward_by_complexity` is true, say so and cite the basis from `code_review_status.complexity_signal`:

```
🔬 Code review due — pulled forward by complexity (<sessions_since_review>/<interval_sessions> sessions in):
   <lines_changed> lines changed, <new_files> new file(s)<if new_design_docs> incl. <new_design_docs> design doc(s)</if><if complex_decisions>, <complex_decisions> complex/chaotic decision(s)</if> since last review.
Spawn a quality review agent? [Y/n/later]
```

Otherwise, use the plain cadence framing:

```
🔬 Code review due — <sessions_since_review> sessions since last review (<last_review_at or "never">).
Spawn a quality review agent? [Y/n/later]
```

#### Y — Spawn the review agent

Run immediately:

```bash
mkdir -p ~/.claude/loop/<namespace>/code_reviews
```

Spawn an Agent with the prompt template at `~/.claude/skills/compass/scripts/prompts/code-review-agent.md`, substituting `<repo_path>` from namespace state. The template specifies the exact output structure (Top Issues by severity/theme, then High-leverage harvestable skills/utilities) — the parsing in step 2 below depends on it, so pass it verbatim.

After the agent returns its output:

1. **Save the report:**
   ```bash
   # Write the full output to:
   ~/.claude/loop/<namespace>/code_reviews/<YYYY-MM-DD>.md
   ```

2. **Parse for CRITICAL issues:** Scan the saved report for lines containing `[CRITICAL]`. Collect each as a short string (title only).

3. **If CRITICAL issues exist:** Add them directly to the current session todo list using TodoWrite, prefixed with `[code review]`. Do NOT ask — just add them. Then inform the user:
   ```
   ⚠ <N> critical issue(s) added to your todo list for this session.
   ```

4. **Record the review:**
   ```bash
   python3 ~/.claude/skills/compass/scripts/compass.py record-review <namespace>
   ```

5. **Surface the report path and offer next-session goals:**
   ```
   ✓ Review saved to ~/.claude/loop/<namespace>/code_reviews/<date>.md

   Top findings:
   <list the [HIGH] and [CRITICAL] issues, max 5, one line each>

   Extract top issues as next-session goals? [Y/n]
   ```

   - **Y** → add the top HIGH/CRITICAL issues (max 3, user can edit) to the namespace's `reality.md` under `## Backlog` → `### Tactical` (docs/namespace-backlog-standard.md) — a HIGH/CRITICAL finding is concrete and actionable, not something needing further scoping, so it belongs in Tactical, not Strategic. Tag each with `[source: code-review-<date>]`. Write via `update-reality`, never a direct edit. These surface both at the next ORIENT's Tactical backlog block and as `backlog_tactical` goal candidates at DECIDE.
   - **n** → skip; the file is saved and can be referenced manually

#### n or later — Defer

Record the deferral:
```bash
python3 ~/.claude/skills/compass/scripts/compass.py defer-code-review <namespace>
```

Check the returned `defer_count` and `escalate` fields.

- **`escalate: false`** (first deferral): continue to DECIDE with no further comment.
- **`escalate: true`** (2+ deferrals): add the review as a proposed goal in DECIDE:
  ```
  ⬆ Code review has been deferred <N> times — adding it to this session's goal proposals.
  ```
  It appears in the DECIDE goal list as: *"Run periodic code quality review (overdue)"*. The user can still remove it, but it is no longer silently skipped.

---

### Step 2b.3b — Periodic CLAUDE.md hygiene review

Check `claude_review_due` from read output. If `false`, skip entirely. If `true`, check
whether a `CLAUDE.md` exists for this namespace (`repo_path/CLAUDE.md`, or
`~/.claude/skills/<namespace>/CLAUDE.md` if `repo_path` is unset) — if none, reset the
counter (`record-claude-review <namespace>`) and skip silently. Otherwise read
`~/.claude/skills/compass/scripts/prompts/claude-md-hygiene-review.md` and follow it.

---

### Step 2b.4 — Periodic external research (P11) — compass-research-scope

Check `research_due` from read output. If true, present the prompt below **after** the orient brief, as a standalone interactive step — **do not embed it in the brief block**; present it separately and wait for the user's response before continuing.

**P68 (MemoHarness carryforward) — complexity pull-forward:** if `research_status.pulled_forward_by_complexity` is true, use the pulled-forward framing instead of the plain cadence framing — mirrors the `code_review_due` complexity-pull-forward messaging exactly:

```
🔍 External research due — pulled forward by complexity (<sessions_since_research>/<research_interval_sessions> sessions in):
   <N> complex/chaotic decision(s) clustering on tag "<tag>" since the last research pass.
Scope Q&A before spawning a research agent? [Y / n / later]
```

Otherwise, use the plain cadence framing:

```
🔍 External research due — <sessions_since_research> sessions since last run.
Scope Q&A before spawning a research agent? [Y / n / later]
```

Once the user responds, invoke the `compass-research-scope` sub-skill (Y) or record the deferral (n/later):

```
/compass-research-scope namespace=<namespace> sessions_since_research=<N>
```

The sub-skill handles the full Q&A, agent spawn, result parsing, `record-research` call, and defer path. It returns one of:
- `{"result": "recorded", "signals_count": N}` — research completed; continue to next step.
- `{"result": "deferred", "escalate": false}` — deferred; continue silently.
- `{"result": "deferred", "escalate": true}` — add external research as a DECIDE goal proposal.

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
no confirmation needed. Continue immediately to Step 2b.6.

### Step 2b.6 — Inter-loop dream pass (P12.1)

Check `dream_due` from read output. If `false`, skip entirely. If `true`, read
`~/.claude/skills/compass/scripts/prompts/dream-pass-protocol.md` and follow it. It covers
the dream-status/decision-guidance-candidates prompt, merge/decay candidate review, and
the defer/escalate path.

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

### Step 3b.5 — Goal ambition nudge (P-GC3)

Run **after** vague goal detection, **before** architecture check. Advisory only — never blocks DECIDE.

**Check 1 — All execution tasks:**
Scan the confirmed goal list. If every goal is a concrete implementation task (build X, fix Y, extract Z, wire W) with no exploratory, design, or architectural goals, surface once:

```
💡 All goals this session are execution tasks.
   Consider adding one stretch goal — a design question, architectural decision, or
   hypothesis to test — to counter pure delivery momentum.
   Add one? [Y / n / skip]
```

- **Y** → ask: *"One sentence: what's the open design question or hypothesis worth tackling?"* Add as an additional goal before locking.
- **n / skip** → proceed silently. Do not re-prompt.

**Check 2 — Three consecutive 100% sessions:**
From `goal_completion_trend` (already in the read output), check `trend`. If the last 3 values are all `100.0`:

```
📈 Three sessions in a row at 100% completion.
   Perfect scores can mean great execution — or goals set safely within comfort zone.
   Worth adding a harder goal this session? [Y / n / skip]
```

- **Y** → same prompt as Check 1 — ask for a stretch goal and add it.
- **n / skip** → proceed silently.

**Rules:**
- Only fire each check once per DECIDE. Skip silently if fewer than 3 prior sessions in trend.
- If both checks trigger, batch into one prompt — do not ask twice.
- Advisory only: user can always skip without consequence.

### Step 3b.6 — Goal type tagging (P23)

After the ambition nudge, ask once before locking:

```
🔀 Tag each goal — exploit (E) or explore (X)?
   E = implementation / delivery
   X = research, hypothesis, architectural decision
   (e.g. "EEXEE" for 5 goals — or skip)
```

- **skip / enter** → proceed without types; exploration ratio will not include this session.
- **response** → parse into `goal_types` array (e.g. `"EXE"` → `["exploit", "explore", "exploit"]`). Store alongside goals; passed in close payload as `"goal_types": [...]`.

**Rule:** one prompt only. No blocking — types are optional metadata, not a gate.

### Step 3b.8 — Verification contract offer (P55)

Read `~/.claude/skills/compass/scripts/prompts/contract-and-architecture-checks.md` and
follow its Step 3b.8 section — it covers the Y/S/n prompt and per-goal criteria capture.
Always offered once per DECIDE regardless of `repo_path`.

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

Then populate the todo list with the confirmed goals (use TodoWrite).

**P67 cold-start seeding:** if the response's `seeded_count` is greater than 0 (only
possible on this namespace's genuinely first-ever open), surface once, no action
required:
```
🌱 Seeded <seeded_count> learning(s) from `<seed_source>` — cold-start context, not yet
   validated in this namespace.
```
`seed_source` is `null`/`seeded_count` is `0` on every other open — skip silently.

### Step 4.5 — Pre-session hypothesis elicitation (P15)

Before confirming "Let's go", offer once:

```
🧪 Pre-session assumptions — what are these goals betting on?
   Log any to validate at close? [Y / n]
```

- **n / skip / enter** → skip silently. Continue to confirm.
- **Y** → for each assumption the user provides, log as a hypothesis. If the assumption clearly bets on one or more of this session's confirmed goals, include `goal_origin` (the goal's index/indices) in the same call — `log-learning` already accepts and stores it on create, so this does not require a second write:
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

**Rule:** one prompt only — never ask mid-session. No script changes needed (log-learning already supports `goal_origin` on both create and weight-increment paths — see `_upsert_learning`).

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
