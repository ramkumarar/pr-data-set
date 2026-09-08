# Kiro-vs-Claude Java Review Bench — Specification

Bench tool to compare **Kiro CLI** vs **Claude Code** as automated code-review agents on **Java pull requests**, using the AACR-Bench evaluation methodology. Standalone tool, stripped and adapted from `alibaba/aacr-bench` (Apache-2.0, keep attribution).

## 1. Purpose

Answer one question reproducibly: *given the same repo state, the same review prompt, and the same findings-reporting channel, does Kiro CLI or Claude Code produce better review comments?* Better = higher recall of known issues, higher precision, better line localization — measured the AACR-Bench way.

## 2. Scope

| In | Out |
| --- | --- |
| Java PRs only (AACR-Bench Java subset = 30 PRs, local, as smoke corpus) | Other languages |
| Reviewers: `claude`, `kiro` | `ocr`, `codex` (dropped) |
| Offline bundles: `corpus/*.bundle` as the repo source | Live GitLab API wiring (scaffold only, added later) |
| Static review (agent may read/grep, no build) | Annotation pipeline, data curation |

## 3. Architecture (data flow)

```
data/<bench>.jsonl (standard format)
   │  load (schema.load_instances)
   ▼
review stage (pipeline.run_review_stage)
   │  per instance: clone/fetch → checkout head_commit → run reviewer
   │    claude: claude -p /code-review <base>...<head>  [MCP report server + stream-json fallback]
   │    kiro:   kiro-cli chat --no-interactive ...      [MCP report server + stream-json fallback]
   │  both write: results/<bench>/<reviewer>/<run_id>/<safe_id>.json
   ▼
eval stage (pipeline.run_eval_stage → evaluate.py → judge.py)
   │  4-stage matching: path → side → line(k) → semantic (LLM judge)
   ▼
metrics/<bench>/<reviewer>/<run_id>/metrics_<reviewer>_<ts>.json
   (semantic + line Precision/Recall/F1, noise rate, per-sample flags, missing instances)
```

## 4. Components: pick / drop / modify

### Pick as-is (no changes)

| File | Why |
| --- | --- |
| `schema.py` | Standard JSONL format + per-line validation. Core contract, keep verbatim. |
| `config.py` | Path/env-var conventions. Add Kiro + judge-key-optional vars. |
| `repo_utils.py` | git clone/fetch/checkout/clean. `clone_url` is already generic — works with local mirrors/bundles. |
| `mcp_finding_server.py` | Shared findings channel for BOTH reviewers (env-bound instance id, atomic append). This is the parity mechanism. |
| `reviewers/claude.py` | Claude Code reviewer (MCP incremental + stdout/stream fallback). Keep whole. |
| `judge.py` | 4-stage matching + metrics. Modify only the LLM gate (see below). |
| `evaluate.py` | Evaluation orchestration. Reuse `build_target_comments_from_claude` for Kiro findings (same `{file,line,summary,failure_scenario}` shape from the shared MCP server); add `kiro` alias. |

### Copy then modify

| File | Change |
| --- | --- |
| `pipeline.py` | `--reviewer` choices → `{claude, kiro}`; add `kiro` branch; drop `--ocr-command`/`--max-tools`; keep repo-group concurrency model. |
| `converters/aacr_bench.py` | Keep as reference + `--lang java` filter (project_main_language == Java). |
| `.env.example` | Replace with: `ANTHROPIC_BASE_URL`/`ANTHROPIC_AUTH_TOKEN`(placeholder allowed)/`ANTHROPIC_MODEL`; `KIRO_API_KEY`(optional, stub mode)/`KIRO_MODEL`; `JUDGE_BASE_URL`/`JUDGE_MODEL` — **no token required**. |

### New

| File | Purpose |
| --- | --- |
| `reviewers/kiro.py` | Drive `kiro-cli chat --no-interactive --trust-tools=read,grep,find --output-format stream-json --model <KIRO_MODEL> "<review prompt>"`. Writes project-level `.kiro/settings/mcp.json` into the checked-out workspace (MCP server with `env: REVIEW_RESULTS_DIR/REVIEW_INSTANCE_ID`), then parses stream-json as fallback. Mirrors claude.py's dual-channel + `save_result` structure. |
| `converters/gitlab_mr.py` | Scaffold: reads exported GitLab MR JSON (title/desc/diff/notes) → standard format. No API wiring yet. |
| `make_bundles.py` | Run on a machine with internet: mirror-clone each repo, validate base/head commits, emit `corpus/<owner>__<name>.bundle`. |
| `docs/...` | This spec + metrics notes. |

### Drop

`reviewers/ocr.py`, `reviewers/codex.py`, `mcp_codex_finding_server.py`, `hooks/on_stop_failure.py` (Claude-only retry hook — keep only if Claude retry handling is wanted), `evaluation/README.*` (rewrite short one).

## 5. Judge protocol (key adaptation)

`judge.py:25` gates on `JUDGE_USE_MOCK=true OR JUDGE_API_KEY missing` → forces Mock mode. Your gateway needs **no bearer token**, so:

- `JUDGE_API_KEY` becomes **optional**; when unset, pass a placeholder key (`"dummy"`) to the OpenAI client — real auth is provided by network/transport (proxy) or gateway ignores it. Add an explicit `JUDGE_REQUIRE_KEY=false` switch so Mock stays opt-in.
- `judge.py:100` sends `extra_body={"chat_template_kwargs": {"enable_thinking": False}}` (vLLM-specific). Make it a config `JUDGE_EXTRA_BODY` (default `{}`), so arbitrary OpenAI-compatible proxies don't reject the request.
- `JUDGE_MODEL` default: `sonnet-4.5`; `opus-4.6` for final runs. (Only GPT-5.2 / Sonnet 4.5 / Opus 4.6 available at gateway.)

Reviewer parity defaults: `ANTHROPIC_MODEL=sonnet-4.5`, `KIRO_MODEL=sonnet-4.5`; second config both `opus-4.6`.

## 6. Standard data format (unchanged from source)

JSONL, one `ReviewInstance` per line: `instance_id`, `repo` (`owner/name`), `base_commit`, `head_commit`, optional `clone_url`, `reference_comments[]` (`path`, `start_line`/`end_line` closed interval, `side` left/right, `text`). Line convention and validation identical to `schema.py` — do not fork this.

## 7. Metrics (unchanged)

- **Line**: P/R/F1 where generated line within `k` (default 1) of reference interval.
- **Semantic**: P/R/F1 via LLM judge equivalence on the same path/side/line-filtered candidate.
- Noise rate (`1 − precision`), avg comments per patch.
- Missing result files excluded from denominators (≠ "model found nothing").
- Multi-round eval averaging (`--eval-rounds`).

## 8. Run model

```
python -m pipeline run --stage all --reviewer {claude|kiro} \
    --dataset data/java_bench.jsonl --run-id baseline --concurrency 2
```

- Concurrency grouped by repo (same repo serial, different repos parallel).
- Per-run isolation: `results/<bench>/<reviewer>/<run_id>/`, `metrics/...` mirrored.
- `--preview` clones/checks out without invoking the LLM.

## 9. Repository acquisition (no GitHub egress)

The Java corpus (30 PRs) references GitHub repos; the harness never assumes GitHub is reachable. **Chosen path: offline bundles** (frozen, reproducible corpus). Options considered:

1. **Offline bundles (chosen)**: fetch once on a machine with internet → ship bundles into the enterprise.
2. **Internal GitLab mirror (later, when GitLab data path arrives)**: import/mirror repos into internal GitLab, `clone_url` → GitLab. Same harness.
3. **Corporate proxy/allowlist (fallback)**: if github.com is reachable via enterprise proxy, git `http.proxy` suffices.
4. **Not viable**: per-commit GitHub archive tarballs (snapshots without history) — reviewers need real history for cross-file context.

### Bundle generation (`make_bundles.py`, new helper, run on the internet machine)

```
git clone --mirror https://github.com/<owner>/<name> /tmp/rep.git
git bundle create <owner>__<name>.bundle --all
```

- **One bundle per repo** (shared by all instances of that repo), named `owner__name.bundle` → `corpus/` dir (env `CORPUS_DIR`, default `<tool>/corpus/`).
- The script reads the converted dataset (`data/java_bench.jsonl`) and for **each instance validates** `git cat-file -e <base>^{commit}` and `<head>^{commit}` inside the mirror **before** emitting the bundle — a bundle that cannot serve every instance must not ship.
- Big repos (e.g. dbeaver): `--mirror --filter=blob:none` + `--shallow-since=<PR window start>` to shrink; validation step is what guards correctness.

### Harness integration

- Converter sets `clone_url = <abs path>/corpus/<owner>__<name>.bundle` for every instance (the derived-github-URL fallback is never used).
- `repo_utils.py:47-74` needs **no change**: `git clone <bundle>` works, and `git fetch --all --tags` / `git fetch origin <commit>` treat the bundle as origin.
- Frozen caveat: a bundle is static — fine for a benchmark (reproducibility); regenerate only when the corpus changes.
- Preflight (optional, `--verify-corpus`): for each instance, `git bundle verify` + commit presence, fail fast before a long review run.

## 10. Known risks / decisions to verify at implementation time

1. **Kiro MCP tool naming**: does Kiro expose the findings server's tool as `mcp__findings__report` (Claude naming) or differently? Verify with `kiro-cli` MCP discovery; the findings prompt must name the right tool. If Kiro cannot call MCP tools in headless mode, fall back to stream-json parsing only (parity note: then Claude also runs without MCP for comparison).
2. **Kiro auth**: headless needs `KIRO_API_KEY` (Kiro portal) and may be subject to admin governance (MCP restrictions, model policy) — confirm MCP + chosen model are permitted. Until then, `kiro.py` is testable against `--no-interactive --output-format stream-json` with a stub.
3. **Gateway proxy quirks**: verify the proxy accepts OpenAI-compatible `/chat/completions` without auth header and rejects/ignores unknown `extra_body`.
4. **`ANTHROPIC_AUTH_TOKEN`**: Claude Code requires the env var to be set even if the proxy ignores it — set a placeholder.

## 11. Deliverable shape

New standalone repo (dir: `bench-tool/` at workspace root) containing the picked files above + short README + this spec. License: Apache-2.0 with AACR-Bench attribution header in copied files.

## 12. Roadmap

- [x] Spec + component pick (this doc)
- [ ] Standalone repo with claude + kiro reviewers, Java-filtered converter, optional-key judge
- [ ] Smoke run on AACR-Bench Java subset (30 PRs), mock judge first, then Sonnet 4.5 judge
- [ ] GitLab MR converter (export JSON) when MR corpus is available
