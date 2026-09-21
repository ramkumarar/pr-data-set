# CRH-Bench — Code Review Harness Benchmark (Spec)

Status: ready-for-agent (pending issue tracker setup)
Scope: Java / Spring Boot ecosystem, v1
Owner: (you)

---

## Problem Statement

I run a **Kiro-CLI-based code review harness** that produces review comments, but I have no
reproducible way to know whether those comments are good. I cannot call LLM APIs directly —
my only LLM path is an agent CLI (Kiro), which is slow and not free — so every existing
benchmark harness (AACR-Bench's Python `evaluation/`, Martian's Code Review Bench, which
assume a direct SDK/API call) is a bad fit or requires rework.

I need to measure my harness's review comments against ground truth in two shapes:

1. **A human-curated sample set** ("pure comments"): fixed corpus, curated comments only.
2. **Merged MRs/PRs that already have comments**: real discussions where I must first filter
   the chit-chat out and keep only the comments that carry real review value, then compare the
   harness output against them.

Without this, I cannot compare harness configurations/models, cannot tell if a change made
review quality worse, and cannot show whether the harness catches real issues or just makes noise.

The closest public work (AACR-Bench, Martian Code Review Bench) provides methodology but not a
tool I can run in my environment: AACR is Python and expects an LLM API endpoint; Martian's is
per-tool product benchmarking, not an agent-driven harness. I want the *methodology* in a Go
component that fits my stack.

## Solution

A standalone **Go module + CLI** ("CRH-Bench") that:

- ingests **review comments produced by the harness** in a harness-agnostic standard format
  (v1 adapter: the Kiro harness's Markdown findings contract, with an opportunistic findings-JSON
  reader);
- ingests **ground truth** from two corpus types — the AACR-Bench Java subset (Corpus A) and a
  curated merged-MR set modelled on Martian's golden comments (Corpus B, seeded by the
  `ls1intum/Artemis` set);
- uses **Kiro CLI itself as the judge and the comment-quality classifier** (no direct LLM API),
  with caching so re-runs are cheap and stable;
- runs a **path → side → line(±k) → semantic** match (AACR-Bench's four-stage method) and reports
  **semantic Precision/Recall/F1, line-localization P/R/F1, noise rate, and severity/category
  breakdowns**, with an F-beta option;
- isolates every run (`results/<corpus>/<reviewer>/<run_id>/`) with a **run manifest** recording
  harness, model, prompt versions, corpus hash, and judge model, plus a `compare` command to diff runs.

Everything is file-based (JSON/JSONL), standard-library Go, offline after corpus freeze.

## User Stories

1. As a harness developer, I want to feed the harness's review comments into a benchmark in one
   command, so that I don't hand-assemble inputs each time.
2. As a harness developer, I want a single standard comment format, so that switching harnesses
   doesn't change the benchmark.
3. As a harness developer, I want an adapter for the Kiro harness's Markdown findings, so that I
   can benchmark it without changing the harness.
4. As a harness developer, I want the benchmark to read a findings JSON if the harness emits one,
   so that I get exact fields rather than parsing Markdown.
5. As a harness developer, I want an optional pass-through to run the harness before judging, so
   that one command can go review → judge → metrics.
6. As a benchmark user, I want a fixed, frozen Java corpus, so that numbers are reproducible and
   comparable across harness versions.
7. As a benchmark user, I want to load AACR-Bench's Java positive samples as ground truth, so that
   I start from a validated corpus instead of curating one from scratch.
8. As a benchmark user, I want to load a human-curated merged-MR comment set (Martian
   `golden_comments` shape, seeded by the Artemis set), so that I can measure against real merged
   PR discussions.
9. As a benchmark user, I want comments without file/line anchors to still count for semantic
   matching, so that summary-level ground truth isn't silently dropped.
10. As a benchmark user, I want a per-severity and per-category recall breakdown, so that I can see
    which kinds of issues the harness misses.
11. As a benchmark operator, I want a comment-quality classifier, so that raw MR discussions can be
    reduced to comments that carry real review value.
12. As a benchmark operator, I want the classifier to output a binary good/bad decision plus a
    reason, so that filtering is auditable.
13. As a benchmark operator, I want the classifier applied to harness output too, so that precision
    is measured on valuable comments rather than raw comment count.
14. As a benchmark operator, I want corpus-A quality filtering off by default, so that the curated
    corpus isn't second-guessed unless I ask.
15. As a benchmark operator, I want the semantic judge driven by Kiro CLI, so that I need no direct
    LLM API access.
16. As a benchmark operator, I want batched per-MR judging, so that the slow agent isn't called once
    per comment pair.
17. As a benchmark operator, I want judged/classified verdicts cached by content hash + model +
    prompt version, so that re-runs are fast and stable.
18. As a benchmark operator, I want an optional multi-round majority vote, so that publishable or
    gating numbers aren't at the mercy of judge drift.
19. As a benchmark analyst, I want semantic P/R/F1 as the headline metric, so that I have one
    comparable number.
20. As a benchmark analyst, I want line-localization P/R/F1 where anchors exist, so that I can tell
    whether the harness points at the right place.
21. As a benchmark analyst, I want a noise rate and a count of unscoreable-for-position comments, so
    that I understand the shape of the failures.
22. As a benchmark analyst, I want a configurable F-beta (with F2 available), so that I can weight
    recall over precision when missed bugs matter more.
23. As a benchmark analyst, I want per-comment TP/FP/FN detail exported, so that I can debug specific
    misses without a separate tool.
24. As a benchmark analyst, I want a `compare` command, so that I can diff two harness runs on the
    same corpus side by side.
25. As a benchmark analyst, I want a run manifest, so that any number can be traced to its inputs.
26. As a reviewer of benchmark results, I want a CLI summary at the end of each run, so that I get an
    answer without opening files.
27. As a reviewer of benchmark results, I want a JSON metrics file per run, so that downstream tooling
    and a future dashboard can consume it.
28. As a maintainer, I want the benchmark to be offline and deterministic after corpus freeze, so that
    CI or a laptop rerun gives the same result.
29. As a maintainer, I want the corpus acquisition (GitLab/GitHub fetch, git bundles) to live outside
    the Go core, so that the core stays testable and replaceable.
30. As a maintainer, I want the Kiro invocation behind one interface, so that tests can stub it and
    future agents (Codex, Claude) can be swapped in.

## Implementation Decisions

### Language, artifact, and home
- Build a **standalone Go module + CLI** in `crh-benchmark` (own `go.mod`), harness-agnostic and
  independent of OpenCodeReview's internals. It borrows *methodology and schema* from AACR-Bench
  (Apache-2.0) and *gold formats* from Martian Code Review Bench (MIT); attribution headers required.
- **Standard library only** (plus `os/exec` for `git` and `kiro-cli`). No third-party deps, no DB.
- Module path is an **open item** (needs `github.com/<org>/crh-benchmark` confirmation).

### Standard comment schema (canonical, from OCR's enum)
All sources map lossily into this shape. `severity` may be `null` (AACR has none); never defaulted.

```json
{
  "path": "src/main/java/.../Foo.java",
  "start_line": 112,
  "end_line": 118,
  "side": "right",
  "severity": "high",
  "category": "security",
  "text": "…the review comment…",
  "origin": "harness | golden | aacr_ai | aacr_human",
  "source": "kiro-harness@<run> | ocr@<version> | artemis | aacr"
}
```
- `severity ∈ {critical, high, medium, low, null}` (OCR enum)
- `category ∈ {bug, security, performance, maintainability, test, style, documentation, other}`
  (OCR enum). Mapping tables per source: AACR `{Code Defect, Maintainability and Readability,
  Performance, Security Vulnerability}` → canonical; Martian/Artemis `{bug, security, concurrency,
  data, api, perf, test_gap, doc_defect, style, speculative}` + `{Low, Medium, High, Critical}` →
  canonical.
- `side ∈ {left, right, null}`, `start_line`/`end_line` positive ints or null.

### Corpus model
- A **corpus** is a frozen JSONL file of instances: `{instance_id, repo, base_commit, head_commit,
  clone_url, gold_comments[]}` where `gold_comments[]` uses the canonical schema. Reuses AACR's
  instance idea (`instance_id = owner__name@<head7>`).
- **Corpus A (v1):** AACR-Bench Java subset — 30 PRs / 213 comments across 5 repos
  (elasticsearch 14, keycloak 7, dbeaver 4, kestra 3, spring-ai-alibaba 2). Requires a
  **language filter** (`--lang java`) that upstream's converter lacks. Scope is **all Java**, not
  strictly Spring Boot; Spring is only a preference for Corpus B.
- **Corpus B (v1):** Martian-style golden comments seeded by **`ls1intum/Artemis`** (17 PRs /
  40 comments; `comment`/`severity`/`category`, **no anchors**), with internal GitLab MRs added in
  v2. Merged PRs/MRs only; ground truth = **human reviewer comments only** (bots and MR author
  tagged and excluded by default).
- **Negative samples:** AACR's `negative_samples.json` semantics are undocumented and carry no
  label — **ignored in v1**; may be relabelled later via the classifier and used as a noise probe.

### Harness input
- **Harness-agnostic**: `--comments <file|dir>` accepts standard-schema comments.
- **v1 adapter (Kiro harness):** parse the **Markdown findings contract**
  (`### [CRITICAL|WARNING|INFO] <title>`, `- Location: path:line`, `- Problem:`, `- Fix:`), one
  aspect file per reviewer; use `summary.json` for aspect-level aggregates. If the harness emits a
  **findings JSON**, read it in preference to Markdown (open item: exact path/command not yet
  supplied). OCR JSON (`comments[]`) is a secondary reference adapter.
- Optional `--run-cmd` pass-through to produce comments before judging (ingest is the default).

### Comment-quality classifier (Kiro-driven)
- **Definition — "good"**: identifies a concrete technical issue or improvement with enough
  substance to act on, regardless of whether it was actioned. **"bad"**: empty praise, "+1"/thanks,
  process chatter, restating the diff, vague/generic advice, pure formatting nits with no substance.
- Output: **binary `good|bad` + `reason`**; rubric frozen and versioned.
- Applied to: **Corpus B raw comments (on)**, **harness output tagging (on)**, **Corpus A
  filtering (off by default)**.
- Pure style/formatting comments are classified as their own `style` category (canonical enum), so
  they can be included/excluded rather than only "bad".

### Semantic judge (Kiro-driven)
- **Kiro CLI headless** is the only LLM path:
  `kiro-cli chat --agent-engine v2 --no-interactive --output-format stream-json "<prompt>"`;
  parse the final assistant message. (MCP callback mode deferred.) Exact agent name / model / flags
  are **open items**; `KIRO_MODEL` recorded in the manifest.
- **Matching pipeline:** deterministic pre-filter **path → side → line(±k)** → semantic via Kiro.
  Each generated comment can win at most one line match and one semantic match (AACR's dedup rule).
- **Granularity:** batched per instance/MR (all refs × all candidates in one call, strict JSON
  out), not pairwise.
- **Cache:** JSONL keyed by `sha256(task | ref_text | cand_text | judge_model | prompt_version)`;
  verdicts reused across runs.
- **Determinism:** for publishable/gating numbers, run **3 rounds and majority-vote** (AACR's
  `--eval-rounds` pattern).

### Metrics
- **Headline:** semantic Precision / Recall / F1 + noise rate.
- **Secondary:** line-localization P/R/F1 where anchors exist.
- **Weighting:** configurable **F-beta**, with **F2** (recall-weighted) available; default F1.
- **Breakdowns:** per-severity and per-category recall tables (not headline). No category profiles
  (Martian Strict/Core/All) in v1.
- **Anchor-less comments:** eligible for semantic match, excluded from line metrics, surfaced as an
  "unscoreable for position" count.
- **Missing instances:** excluded from denominators; reported separately (≠ "found nothing").
- Per-comment **TP/FP/FN detail** exported.

### Run model, manifest, comparison
- Layout: `results/<corpus>/<reviewer>/<run_id>/` for comments + matches, `metrics/<corpus>/<reviewer>/<run_id>/`
  for metrics — mirrored, isolated per run (AACR pattern).
- **Manifest** per run: harness + version, model, review/classifier/judge prompt versions, corpus
  version + hash, judge model, seed, cache key basis, timestamp.
- `compare` command diffs N runs on the same corpus (side-by-side table + JSON).

### Corpus acquisition (outside the Go core)
- **Offline `git bundle` per repo** (`git bundle create <owner>__<name>.bundle --all`), validated
  per instance before shipping. Go core reads `clone_url` and shells to `git clone/fetch/checkout`.
- MR/PR **fetching is external** (glab, GitHub API, Artemis export) and emits the frozen corpus
  JSONL. Repos are frozen; bundles regenerate only when the corpus changes.

### Reporting
- Staged: **JSON metrics per run (always)** + **CLI summary table (always)**; **HTML dashboard
  later**, once numbers stabilize.

### Seams (for confirmation)
- **One primary seam:** the **Kiro invocation boundary** (`AgentRunner` interface) — tests inject a
  fake runner (stub executable) so no real Kiro call is made and the whole pipeline runs offline.
- **Secondary (pure-function) seams:** canonical-schema mapping, the Markdown findings parser, the
  four-stage matcher, and metric aggregation. These are pure and need no substitution.
- This is the smallest seam set that keeps every external boundary (LLM, git, filesystem) injectable
  without splitting the core into artificial layers.

## Testing Decisions

- **Good test definition:** exercise external behavior (given inputs → observable outputs), never
  implementation details; deterministic; no network and no real LLM calls.
- **One primary seam:** all agent calls go through a single `AgentRunner` interface. Tests substitute
  a **fake Kiro executable** (fixture script emitting canned stream-json) placed on `PATH`/injected,
  so the entire pipeline (parse → classify → match → metrics → report) runs offline.
- **Pure-function seams** (tested directly, no fake needed): canonical-schema mapping/mapping tables,
  the Kiro Markdown findings parser, the four-stage matcher (`path → side → line(k) → semantic`
  including the one-line/one-semantic dedup rule), and metric aggregation against known match sets.
- **Prior art:** AACR-Bench `evaluation/` judge + schema tests and Martian `offline/tests/`
  step-wise tests are the methodological prior art; mirror their fixture-driven style. OCR's Go test
  style (`*_test.go` per package, table-driven) is the in-language prior art.
- **Fixtures:** small frozen corpora (a few instances), gold comments, harness comment files,
  cached judge verdicts, and a fake Kiro script. Add one end-to-end test using the fake runner.
- **Negative/edge cases:** anchor-less comments, side mismatch, line tolerance boundary `k`, missing
  result files, unparseable judge output, cache hits, severity-absent mapping.

## Out of Scope

- Non-Java languages and non-Spring-relevant ecosystems (v2).
- Using `negative_samples.json` as ground-truth negatives (undocumented semantics).
- Category scoring profiles (Strict/Core/All) and Item Response Theory / ELO.
- HTML dashboard (deferred), online/fresh-corpus mode, and CI gate mode (structure allows later).
- Live GitLab/GitHub fetching inside the Go core; the MCP findings-server channel for Kiro.
- Training or fine-tuning any model; publishing a public leaderboard.
- Corpus B external/internal GitLab ingestion beyond the Artemis seed (v2).

## Further Notes

- **Open items requiring confirmation:**
  1. Go module path (`github.com/<org>/crh-benchmark`).
  2. Kiro invocation specifics: agent name, `--trust-all-tools`, and which model `KIRO_MODEL` maps
     to.
  3. The harness's **findings-JSON** path/command (if it truly emits per-comment JSON); until
     supplied, Markdown is the source of truth.
  4. Confirmation of the proposed seam (single `AgentRunner`) before implementation.
- **Issue tracker:** not configured in this environment; the spec cannot be published yet. Run
  `/setup-matt-pocock-skills` to provide the tracker + triage vocabulary, then publish with the
  `ready-for-agent` label.
- **Attribution/licensing:** AACR-Bench is Apache-2.0; Martian Code Review Bench is MIT. Any reused
  methodology, schema, or fixtures carry attribution headers.
- **Why Kiro is central:** it is the user's only LLM path, so both the semantic judge and the
  value classifier shell out to it; caching and batched calls exist specifically because the agent
  is the slow, costly bottleneck.
- **Reference artifacts consulted:** `/home/dev/work/aacr-bench` (`evaluation/`, `dataset/`),
  `/home/dev/work/code-review-benchmark` (`offline/`, `golden_comments/artemis.json`),
  `/home/dev/work/code-review-analysis` (`.kiro/review/`, `results-ls1intum-Artemis-12384/`),
  `/home/dev/work/open-code-review` (OCR JSON schema).
</content>
</invoke>
