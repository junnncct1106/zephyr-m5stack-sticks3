# Process record — PR #16 gate-logic fix (Issue #7)

Self-review trail for the fix that addresses thc1006's maintainer review on PR #16
(review id 4472219975) and Copilot's `field()` note. This document lives on the
fork's `work/es8311-gate-selfreview` branch only; the PR branch
(`chore/es8311-upstream-readiness`) carries just the clean script fix.

Date: 2026-06-11. Author: junnncct1106 (with Claude Code). Method requested by the
repo owner: research → cross-analysis → implement → self-test → push record to fork
→ quantitative Socratic self-eval → push to PR only if it passes.

## 1. What the maintainer asked for

1. **PR still auto-closes #7.** Body says "Refs #7" / "does not close #7", but
   `closingIssuesReferences` still returns #7, so merging would close the issue.
2. **Gate verdict looser than the ADR it enforces.** The script flipped to
   GATE OPEN when #110205 **OR** #107655 merged; ADR 0004's gate is #107655 merged
   **AND** the ES8311 work back on a live PR. Our own board PR (#110205) merging
   alone would wrongly print "GATE OPEN".
3. **`field()` returns "?" on failure** (Copilot, confirmed by maintainer). The
   merge check counted any non-empty value as merged, so "?" biases toward a false
   GATE OPEN — it stacks with #2.
4. (Copilot) malformed table in `0007-...readiness.md` — maintainer could not
   reproduce; treated as a non-issue, no change made.

## 2. Cross-analysis (ADR text × script × maintainer review × live API)

ADR 0004's authoritative gate (Update 2026-06-05): *"Wait for the base board
(#107655) to land AND the ES8311 work to resume on a live PR."* So the gate is a
conjunction of `#107655 MERGED` and `a live successor PR exists`. #110205 ("our
own board PR") is never an ADR gate condition.

Empirically verified against `zephyrproject-rtos/zephyr` on 2026-06-11:

| Probe | Result | Implication |
| ----- | ------ | ----------- |
| a merged PR (#110975) | `state=MERGED`, `mergedAt=2026-06-10T18:54:01Z` | `state==MERGED` is the authoritative merge signal; `mergedAt` is ISO-8601 |
| `gh` on a bad PR / outage | prints `?` (old `field`) | confirms the false-merge vector |
| #107655 / #107660 / #110205 | all `OPEN`, `mergedAt=null` | gate correctly closed today |
| `gh search prs es8311 --state open` | only #107660 + #110205 | "no live successor" confirmed |

## 3. Implementation (the fix)

`scripts/check_es8311_upstream_gate.sh`, +60/-18:

- `field()` now returns **empty on any gh error** (was `?`), so an outage cannot
  masquerade as a real value.
- New `is_merged()` trusts **`state == "MERGED"`** plus an **ISO-8601 `mergedAt`**
  regex. `""`, `null`, `?` all fail.
- Gate is now **AND**: `base_merged (#107655)` **and** `successor`. #110205 removed
  from the trigger entirely (kept only in an exclude list).
- **Live-successor** detection (decision: exclude-list, automated): any OPEN
  es8311 PR whose number ∉ {#107660, #110205} and whose author ∉ {thc1006,
  junnncct1106} is marked `*` and sets `successor=1`.
- Constants + `in_list()` helper added for clarity; license header, tabs, ADR-
  referencing comments and the usage line preserved.

## 4. Self-test

- `bash -n`: clean. (shellcheck unavailable in env; array-based rewrite is
  written to be shellcheck-clean — no SC2086 word-splitting.)
- Live run: prints STILL HOLDING; #107660 and #110205 correctly **not** marked as
  successors.
- `scripts/test_check_es8311_gate.sh` (mock `gh`, offline) — 4/4 PASS:

| Scenario | Expect | Result |
| -------- | ------ | ------ |
| base #107655 MERGED + live successor #109999 | GATE OPEN | PASS |
| base #107655 MERGED, no successor | STILL HOLDING | PASS |
| only our #110205 MERGED (old false-open) | STILL HOLDING | PASS |
| gh failing entirely (old "?" false-merge) | STILL HOLDING | PASS |

`set -e` early-exit ruled out empirically: scenarios 2–4 and the live run all
printed the full VERDICT line.

## 5. Quantitative Socratic self-evaluation

Adversarial Q&A, scored 0–2. **Pass = total ≥ 90% (≥ 14.4/16) AND no item = 0.**

| # | Question (asked against the change) | Score | Evidence |
| - | ----------------------------------- | :---: | -------- |
| 1 | Does GATE OPEN now require BOTH base merged AND a successor? | 2 | `&&` in verdict; S1/S2 |
| 2 | Is the #110205-alone false-open gone? | 2 | removed from trigger; S3 holds |
| 3 | Is the `field()`="?" false-merge gone? | 2 | empty-on-error + `is_merged`; S4 holds |
| 4 | Can a non-merged state pass `is_merged`? | 2 | needs `state==MERGED` + ISO `mergedAt`; CLOSED→fail |
| 5 | Successor false +/- risk? | 1 | excludes are exact; residual: successor must contain "es8311" in search text (pre-existing limitation, documented) |
| 6 | `set -e` early-exit safety? | 2 | full VERDICT printed in S2–S4 + live |
| 7 | Side-effects / idempotent? | 2 | read-only (gh view/search); re-runnable |
| 8 | Style/altitude match to repo? | 2 | header, tabs, ADR comments, verdict wording kept |

**Total 15/16 = 93.75% → PASS** (no item scored 0). The single 1 (Q5) is a
pre-existing search-text limitation the maintainer's own review relied on, not a
regression introduced here.

## 6. Item left for human action (cannot be done by CLI)

Maintainer point #1 (#7 still a closing reference): the PR **body** is already
correct (`Refs #7`, no `Closes`). The remaining link is a **manual "Development"
sidebar link**, which has no public API/CLI removal path — it must be unlinked in
the GitHub web UI. Raised with the repo owner for a decision; not auto-changed.
