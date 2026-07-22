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
