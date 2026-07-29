## Week 7 — Issue selection

**Issue link:** https://github.com/ascherj/pathreview/issues/152

**Issue title:** Faithfulness checker can never mark short claims as supported

**Tier:** [x] Tier 1  [ ] Tier 2  [ ] Tier 3

**Problem summary:**
The `FaithfulnessChecker` in `rag/evaluator/faithfulness_checker.py` has two compounding bugs that prevent short, factual claims from ever being scored as "supported." First, the `_extract_claims()` method filters out any sentence shorter than 10 characters, so short claims like "Knows SQL." are silently discarded before they even reach the scoring logic. Second, `_is_supported()` requires at least 2 non-stopword tokens to overlap between the claim and the context, but short claims often share only one meaningful keyword (e.g., "python") with a fully supporting context chunk, so they always fail the threshold check. Together, these bugs cause feedback made up of short, fully supported claims to score 0.0 when it should score close to 1.0. The fix involves lowering the minimum character-length threshold in `_extract_claims()` and reducing the required overlap count in `_is_supported()` to 1 when the claim has fewer than 2 meaningful tokens.

**Is this right for me? — checklist reasoning:**
- The issue is scoped to a single file (`rag/evaluator/faithfulness_checker.py`) with clear reproduction steps and three named failing tests (`test_partial_support_returns_middle_score`, `test_multiple_context_chunks`, `test_multiple_claims_varying_support`).
- No external API keys or services are needed to reproduce — it's pure Python logic with no network calls.
- The root cause is visible in under 100 lines of code, and the fix requires changing two numeric thresholds plus adding a guard condition.
- The issue has existing test coverage that will confirm when the fix is correct.

**Branch name:** fix/152-faithfulness-short-claims

**Setup confirmation:** [ ] App runs locally at localhost:5173

**Cohort ledger:** [ ] Issue added to cohort ledger

---

## Week 8 — Reproduction & solution planning

**Reproduction commit link:** [docs: add PLAN.md and Week 8 reproduction notes for #152](https://github.com/tesfishzana/pathreview/commit/93feec4)

**Reproduction summary:**
Ran `pytest tests/unit/test_faithfulness_checker.py` targeting the three tests named in the issue. All three failed with `assert 0.2 < 0.0` — confirming that every call to `checker.check()` on short-claim feedback returns `0.0`. The log output showed `claims_count=1` for inputs that should have produced 3 claims, and `supported_count=0` even when the single extracted claim had clear keyword overlap with the context.

**PLAN.md link:** [PLAN.md](https://github.com/tesfishzana/pathreview/blob/fix/152-faithfulness-short-claims/PLAN.md)

**Walkthrough video (recommended):** [pending]

**Blockers or open questions:**
None — confirmed locally that `len(s.strip()) > 3` retains all target claims including `"Knows Rust"` (10 chars). The two-part fix (lower length threshold + short-claim overlap guard) is fully scoped to `faithfulness_checker.py` with no external dependencies or API calls needed.

---

## Week 9 — Solution building & PR submission

### Check-in 1 (mid-week)

**Current progress:**
Completed all implementation. The fix touched `_extract_claims()`, `_is_supported()`, and the `check()` context-join line — all within the single file `rag/evaluator/faithfulness_checker.py`. Four bugs were resolved: the `> 10` length filter, `.split()` keeping commas on tokens, the `>= 2` overlap threshold being too high for short claims, and `chunk.get("text", "")` returning `None` when the key exists with a `None` value. All 22 unit tests pass; ruff, black, and mypy all clean.

**Next steps:**
Push the implementation commit, open a draft PR against `ascherj/pathreview`, fill in the PR template, and mark ready for review.

**Blockers:**
None.

---

### Check-in 2 (end of week)

**PR link:** [fix(rag): repair faithfulness checker for short claims — #152](https://github.com/tesfishzana/pathreview/compare/main...tesfishzana:pathreview:fix/152-faithfulness-short-claims)

**Branch:** `fix/152-faithfulness-short-claims`

**What you built:**
Fixed four compounding bugs in `FaithfulnessChecker` that caused every short claim to score 0.0. The solution lowers the min-character filter in `_extract_claims()` from `> 10` to `> 3`, adds a conjunction split so multi-claim sentences produce separate scoreable sub-claims, switches tokenization in `_is_supported()` from `.split()` to `re.findall(r'\w+')` to strip embedded punctuation, and applies a 1-token overlap threshold for claims with ≤ 5 meaningful tokens (vs. 2 for longer claims). A fourth fix guards against `None` values in context chunk text.

**Tests added or updated:**
No new tests were needed — `tests/unit/test_faithfulness_checker.py` already contained 22 tests covering all cases, including the three named failing tests. All 22 pass after the fix.

**Self-review confirmation:** [x] make check passes  [x] make test-unit passes

**Draft PR feedback received from:** none
