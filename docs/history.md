# compass-open — moved history and stub text

Verbatim text moved out of SKILL.md on 2026-10-02 (cadence-gated content move). This is
rationale and pointer text only; nothing here is a step the open loop runs.

## Header note: v1 consolidation (formerly lines 13–21)

**v1 consolidation (2026-09-01, `docs/2026-08-31-consolidate-open-close-prompts-plan.md`
in the `compass` skill):** the separate code-review-due / research-due / dream-pass-due
prompts (formerly Steps 2b.3, 2b.4, 2b.6) and the separate ambition-nudge / goal-type-tagging
/ verification-contract prompts (formerly Steps 3b.5, 3b.6, 3b.8) are now one batched screen
— **Step 3f** — fired once, right after the DECIDE goal list is confirmed. The `*_due` flags
themselves are still read at Step 1 OBSERVE; only the *prompt* moved. Step 4.5's Y/n gate
is folded into Step 3f's item 7; only its free-text-per-hypothesis follow-up loop still runs
where Step 4.5 used to be. v2 (per-item confidence-gated auto-apply using P59 override
history) is scoped in the same plan doc but not built — see the Strategic backlog.

## Step 2b.3 stub (moved)

`code_review_due` and its complexity-pull-forward signal are still read at Step 1
OBSERVE — nothing fires here. The prompt, spawn-agent flow, and defer/escalate path
now live in **Step 3f**, item 1 (v1 consolidation, 2026-09-01).

## Step 2b.4 stub (moved)

`research_due` and its complexity-pull-forward signal are still read at Step 1 OBSERVE —
nothing fires here. The prompt, `compass-research-scope` invocation, and defer/escalate
path now live in **Step 3f**, item 2 (v1 consolidation, 2026-09-01).

## Step 2b.6 stub (moved)

`dream_due` is still read at Step 1 OBSERVE — nothing fires here. The prompt and
`dream-pass-protocol.md` invocation now live in **Step 3f**, item 3 (v1 consolidation,
2026-09-01).

## Former Step 2e note (removed 2026-09-09)

*(Former Step 2e — "Deferred skill escalation" (P1.3) — removed 2026-09-09: the only
`defer-opportunity` call site (`compass-close/SKILL.md`, old "Deferred escalations"
paragraph) only re-deferred opportunities already in `escalation_candidates`, and
nothing ever wrote the first entry — `skill-opportunity-detection.md`'s own "D (defer)"
path writes straight to `reality.md`'s Strategic backlog instead, bypassing this
mechanism entirely. `escalation_candidates` could never be non-empty, so this step was
dead code. `deferred_opportunities`/`cmd_defer_opportunity`/`cmd_resolve_opportunity`/
`escalation_candidates` removed from compass's script; Strategic backlog bullets already
provide the same persistent cross-session surfacing this was meant for.)*

## Step 2g stub (moved)

The scan itself (stale "missing" bullets vs `gitlog`; new files not in reality) and its
prompt now run as item 1 of **Step 3b**'s consolidated pre-lock advisory batch (v2
consolidation, 2026-09-16) — grouped with vague-goal/research-fork/domain-novelty since
none of the four has a hard sequential dependency on another. `gitlog` is already fetched
at Step 1 OBSERVE, so nothing needs fetching differently here; only the prompt's timing
moved, from before DECIDE to right after it. See Step 3b for the full scan/prompt/routing
logic — nothing fires at this point in ORIENT anymore.

## Step 3d stub (moved)

Trigger check and full logic now live as item 3 of **Step 3b**'s consolidated pre-lock
advisory batch (v2 consolidation, 2026-09-16) — nothing fires at this point anymore.
`research-fork-detection.md` is unchanged; only its invocation site moved.

## Step 3e stub (moved)

Trigger check and full logic now live as item 4 of **Step 3b**'s consolidated pre-lock
advisory batch (v2 consolidation, 2026-09-16) — nothing fires at this point anymore.
**Relationship to P13** (unchanged): P13/item-3 checks for *prior decisions* on a chosen
approach; P21/item-4 checks for *absence of prior experience* in a domain — complementary,
not redundant, which is why both can appear as separate lines in the same batch screen.

