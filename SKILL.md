# compass-open

OODA-open sub-skill for `/compass`, invoked by the router when `open_session = false`. Owns
Step 1 (OBSERVE) through Step 4 (lock). **Input:** `namespace`.

Rules that hold throughout: the script is the only write path for loop files; batch every check
into the screens below (never a prompt per check); 3–4 tightly scoped goals beat 6–8 broad ones.
Maintenance (code review, research, dream, SkillOpt, CLAUDE.md hygiene, skill opportunities)
never runs here: it is `/compass maintain`.

---

### Step 1 — OBSERVE

One read-only call:
```bash
~/.claude/skills/compass/scripts/compass.sh brief <namespace> --with-flags
```
Keep this command verbatim (the compass-pulse mod matches it to show ORIENT). It returns
`brief_markdown`, a reality summary, recent sessions S-1 to S-5, an **Advisory inputs** JSON
line, `code_context.md` when present, and the **Gate flags** JSON line (`orient-flags`). Run
`~/.claude/skills/compass/scripts/compass.sh expand-session <namespace> <session-id>` only for a
recent session that carry-forward depends on.

### Step 2 — ORIENT

Open with a 2–3 sentence situation frame as a blockquote before the `## 🧭 Compass` header: where
we are against intent, what the top learnings signal, any risk. Write it from the data, no hedging;
omit it on a first session or with fewer than 3 learnings.

Then show `brief_markdown` (flag every carry-forward and tactical backlog item as a goal
candidate), followed by:
```
**Drift / gaps:** <1–3 specific gaps between intent and reality>
📁 Code context (last session): <code_context.md as-is, only when present>
🧰 Maintenance due: <the Gate flags `due` list> — `/compass maintain` when convenient.
```
Omit the maintenance line when `due` is empty. It is one line, never a prompt.

**Conditional advisories:** if any of `session_complexity.elevated` ·
`goal_stats.total_goals > 0 AND goal_stats.hit_rate < 70` · `reality_completeness.regression` ·
`exploration_ratio.low` · `reality_structure_warnings` non-empty holds in the Advisory inputs, load
`~/.claude/skills/compass/scripts/compass.sh prompt orient-brief-advisories --section "ORIENT brief — conditional advisory blocks"`
and append the matching blocks. None → load nothing.

Right after the brief, run silently `~/.claude/skills/compass/scripts/compass.sh surface-learnings <namespace>`.

### Step 2c — Reality validation

If `stale_bullet_count` is 0, skip. Otherwise load
`~/.claude/skills/compass/scripts/compass.sh prompt reality-and-hypothesis-validation --section "Step 2c — Reality validation protocol (P0.1)"`
and follow it before DECIDE.

### Step 2d — Hypothesis validation

If `pending_validations` is empty, skip. Otherwise load the Step 2d section of the same prompt
(`--section "Step 2d — Hypothesis validation (P1.1)"`).

### Step 2f — Session cooldown

If `last_close` is under `session_cooldown_hours` (default 4) ago, load
`~/.claude/skills/compass/scripts/compass.sh prompt orient-gates --section "Step 2f — Session hygiene"`
and follow it (a blocking reason prompt; `acknowledge-cooldown-violation` opens the session
itself). Otherwise skip.

### Step 2h — Intent drift

If `intent_changed` is true, load
`~/.claude/skills/compass/scripts/compass.sh prompt orient-gates --section "Step 2h"`
and follow it. Otherwise skip.

### Step 3 — DECIDE

Propose up to `suggested_goal_count.count` goals (default 4), grounded in the gaps, carry-forward
and tactical backlog. Each goal names an observable deliverable; rewrite any "look at /
investigate / work on" goal into one. Order by execution sequence (dependencies first, structural
before additive, riskier first) and add `*Order: <reason>*`. Tag each goal `E` (exploit) or `X`
(explore). One screen:

```
1. <goal> [E]
2. <goal> [X]
*Order: <reason>*
Enter to lock, or edit / reorder / retag (e.g. "2E", "drop 3", "add: <goal>").
```

### Step 4 — Lock and open

```bash
~/.claude/skills/compass/scripts/compass.sh open <namespace> '{"goals": [...], "goal_types": ["E", "X", ...]}'
```
Skip `open` if `acknowledge-cooldown-violation` already opened the session. Put the goals on the
host's todo list and, as each lands, run
`~/.claude/skills/compass/scripts/compass.sh set-goal-status <namespace> '{"index": <0-based>, "done": true}'`.
If `seeded_count` > 0, say once: `🌱 Seeded <seeded_count> learning(s) from <seed_source>.`

Tally the screens that waited for a response, in one call:
```bash
~/.claude/skills/compass/scripts/compass.sh tally-prompt <namespace> open <screen>[,<screen>...]
```
Keys: `gate`, `orient`, `hypothesis` (2d), `cooldown` (2f), `drift` (2h), `goals` (3).
`auto-verify-reality` tallies `stale` itself.

Confirm: `✓ Session open. Let's go.` This sub-skill is then done.
