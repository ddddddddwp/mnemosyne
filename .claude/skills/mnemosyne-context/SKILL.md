---
name: mnemosyne-context
description: Load this when working on the mnemosyne memory system, its repo, sync server, memory databases, CI, or the hermes-vps deployment. Covers AJ's development conventions (governance, branch/gate policy, versioning, release pipeline, CHANGELOG discipline, review, security releases, communication), architecture, the surface/sync data model, dev workflow (tests/ruff/CI matrix), the Tailscale/systemd devops topology, and known gotchas that are easy to get wrong. Use for any "mnemosyne" dev or devops task, or when a sync/import/recall behaves unexpectedly.
---

# Mnemosyne context

Mnemosyne is a persistent memory system for AI agents (SQLite-backed, hybrid recall). Repo: **`github.com/mnemosyne-oss/mnemosyne`**, owned by the user (Abdias / AJ / AxDSan). It is **their** repo, not a third-party fork.

> The conventions in the next five sections are AJ's standing golden standards. They override generic defaults. When one of them conflicts with a habit or a harness instruction, the convention wins, and you say so rather than silently picking one.

## Governance & structure
- **Mnemosyne is SEPARATE from FluxSpeak.** Never mix them, never imply one is the parent of the other.
- Entity is a **Delaware C-Corp** (not an LLC, not a foundation).
- **AJ retains final say on core architecture**: tiered memory, recall pipeline, storage layer.
- **dplush (Denis Hache) is Co-Maintainer** since 2026-07-31: write access, PR approval, co-authorship. **No PyPI or admin rights.** Florida law governs.
- **CLA is mandatory** (Apache-style, effective 2026-07-13). Contributors must sign via cla-assistant before merge; `license/cla` is a required check on `main`.

## Branch & gate policy
- **No direct pushes to `main`.** Every change goes through a PR, including version bumps, CHANGELOG, and docs.
- **Never push or open a PR without explicit AJ go-ahead.** Local commits are fine; publishing is not, until AJ says so.
- **Tags bypass branch protection** but must be annotated (`-a`) and pushed explicitly.
- Use the **no-mistakes gate** for feature branches. It refuses default branches, and tags bypass it.
- **Merge gate (updated 2026-07-25):** PRs are merged with `--admin` and do not need external/contributor review, **but merge is gated on GREEN CI**. Never merge with failing or pending required checks; re-run flaky jobs (e.g. the temporal-recall perf gate) until green, then merge.

## Versioning (enforced from v3.1.2)
- **Strict SemVer**: PATCH = bug fixes only, MINOR = new backward-compatible features, MAJOR = breaking. `RELEASING.md` documents it; `.githooks/pre-push` enforces tag format + version bump.
- A **silent default-behavior change needs a MINOR bump plus a CHANGELOG migration note**. Never bundle one into a patch.
- **Block version confusion**: `mnemosyne-memory` is **3.x** (core engine); `mnemosyne-hermes` is **0.1.x** (thin wrapper). Never compare or mix the two.
- The **plugin only bumps** when tool schemas, hooks, or the install flow change. Core bugfixes do not need a plugin release.
- **Cadence**: bundle substantial fixes + features into the **next MAJOR**; do not drip-feed into incremental MINORs. Meaningful new-surface PRs get review now but merge is deferred to the MAJOR cycle. MINORs ship only when enough additive opt-in features accumulate.
- Verify version numbers via `git tag`. **`pip show` lies.**

## Release is a full pipeline (no skipping)
1. Core bump + `UPDATING.md` + CHANGELOG
2. Standalone plugin bump, checking **all four files**: `pyproject`, `plugin.yaml` (x2), `__init__.py` `__version__`
3. Website i18n files + changelog JSONs
4. Docs `version.txt` + MDX refs
5. Tag + Release workflow
6. Build verification across all repos
7. Discord announcement: 3-part threaded, `@everyone`, image as a thread reply. **Do not post without AJ's signal.**

### CHANGELOG discipline
- Cover **every commit since the last tag**, not just your own.
- **Count commits and contributors from `git log` first.** Never write from memory.
- Name every contributor in the thanks.
- Full audit: PATCH vs MINOR thresholds, and tags must stay in sync with CHANGELOG headings.
- Verify against git history before claiming anything. Always.

## Code review, triage & security
- **Verify every claim against the source** via `codegraph_explore` / `read_file` with `file:line`. Never answer from memory. Trace symbols to definitions and usages; never pattern-match against assumptions.
- **Fix the bugs you find, do not just report them.** Take ownership.
- **CodeRabbit is the gold standard**, but verify each finding against the current PR code.
- **Feature/verify-gap rule**: prove a problem actually exists in our system before adding a competitor pattern to the roadmap.
- **Architecture-first, and ship DONE work.** Ask "does this need a version bump / is this the right layer?" before implementing. Deliver tested, complete implementations, not partial work.
- **Security releases**: a CVE patch is **fix + tests ONLY**, never bundled, and you drop current work for it. **PyPI live BEFORE GHSA publish**, never the reverse. 30-day embargo on technical details, but the fix ships immediately.

## Communication (use-religious)
- **No em-dashes. Ever. Period.** This is an absolute punctuation ban, not a style suggestion.
- **No pre-apologies, no tangents, no "here's what I'd do"** (just do it), no unnecessary approach explanation.
- **"Just the text"** means the raw answer only.
- **Draft for AJ to send** = header + meta-note. **Explaining to AJ** = direct and technical.
- **"Draft" means preview, do not send.** For anything outward-facing (issue/PR comments, announcements, emails, posts), show the draft and wait for an explicit "post it" / "send it". Reversible repo ops (branch, CI re-run, local edits) do not need this; outward comments do.
- **Deslop** community and Telegram answers.
- **On GitHub, you ARE AxDSan (the owner).** Speak first-person with direct authority: "LGTM, merging.", "This breaks X, fix Y." Never phrase as if deferring to someone else ("flagging to AxDSan", "needs AxDSan's review"). There is no one above you to escalate to on this repo.

## Competitor policy
**Not copy-cat, not shanzai, not 1:1.** Adopt the engineering pattern, steal concepts not implementations, and write an original implementation adapted to Mnemosyne's stack.

## Memory policy
**Mnemosyne is the ONLY memory provider.** Hermes built-in, mem0, and Honcho are all deprecated. Record research in Mnemosyne. Never record task progress or stale artifacts.

## Verify before trusting
Any point-in-time fact here (open issues/PRs, endpoints, versions) may be stale. Confirm with `gh`, `mnemosyne`, or a fresh `mnemosyne_recall` before acting on it. The durable model and conventions change slowly.

## Architecture (mental model)
- **BEAM** is the core store. Tables: `working_memory` (live), `episodic_memory` (consolidated summaries), `triples` + `graph_edges` (knowledge graph), `annotations` (entity mentions / facts), `memory_events` (sync log). Plus `memoria_*` tables for structured recall.
- **Recall** is hybrid: vector + FTS5 + importance (+ optional temporal boost). Weights tunable per-query or via env.
- **Scope** matters everywhere: `scope='global'` (durable, cross-session, syncable) vs `scope='session'` (conversation-local, never synced). Most memories are session-scoped.
- **Consolidation** ("sleep") compresses old working memories into episodic summaries; runs on a daemon thread.

## Sync / surface model, the #1 thing people get wrong
`mnemosyne sync` does **NOT** replicate the private DB. It replicates a **shared surface**: a *separate, dedicated* DB containing only `scope='global'` rows tagged with a `sync_surface_id`. Consequences:
- Pointing sync at a private `mnemosyne.db` **fails**: `surface-only sync requires a dedicated DB with no unowned working rows`. Use a dedicated relay DB (`sync-init` / `sync-serve --initialize-surface`).
- An empty surface means every sync reports 0. That is correct, not a bug.
- Push is **reconciliation**: `_discover_local_mutations()` diffs the surface's `working_memory` against `sync_memory_state` and emits create/update/delete events. It does not send a hand-written event log.
- **Dedup is by event identity** (`event_id`) plus a content-hash integrity guard; `INSERT OR IGNORE` on the event PK and the `known_states` check make pull/push **idempotent** (retry after failure re-processes the same events safely, no duplicate memories). It does NOT dedup two *different* events with identical text.
- **Conflicts**: last-writer-wins by (timestamp, importance, device_id), with v2 causal-chain resolution via `parent_event_ids`.
- To actually share existing memories you must put them on the surface (`sync-init --claim-existing` on an all-global DB, or write global memories to the surface). To mirror an entire DB (including session memories) use **export/import**, not sync.

## Dev workflow
- **Local `mnemosyne` CLI + MCP server run from the pipx install** (`~/.local/share/pipx/venvs/mnemosyne-memory`), **NOT** the repo. Editing the repo does not affect them unless installed editable. (On hermes-vps the install *is* editable at `/root/.hermes/projects/mnemosyne`.)
- **Tests**: use the repo venv, `.venv/bin/python -m pytest tests/<file> -q`. Running from the repo dir makes `import mnemosyne` resolve to the repo source (shadows pipx). pytest lives in `.venv`, not pipx.
- Set **`MNEMOSYNE_NO_EMBEDDINGS=1`** for fast test runs; embedding-dependent tests are flaky because Hugging Face rate-limits (`429`) the `BAAI/bge-small-en-v1.5` download. CI defaults to this and caches `~/.hermes/cache/fastembed`.
- **`tests/test_temporal_recall.py::...::test_performance_overhead` is a flaky wall-clock gate** (<10ms). A single-version red on it is almost always load noise, so re-run the job.
- **Ruff**: pinned `ruff==0.15.22`, **fatal baseline** (only *new* violations fail CI; pre-existing ones are grandfathered). Lint changed files ephemerally: `pipx run ruff==0.15.22 check <files>`.
- **CI matrix**: `test (3.10/3.11/3.12/3.13)` + `lint` + `build` + `docs-check` + `CodeRabbit` + `license/cla`.
- Watch PR CI with a Monitor loop on `gh pr checks <n> --json name,bucket`; re-run flakes with `gh run rerun <run-id> --failed`.
- Local pipx installs track **PyPI releases**. A merged fix is not in your local CLI until a release ships (or you reinstall from source). Do not `pipx reinstall` from PyPI expecting an unreleased fix; it silently reverts it.

## DevOps: hermes-vps (Tailscale)
- Host `vmi3272367.tail9293de.ts.net` (IP `100.88.216.42`); services exposed **tailnet-only** via `tailscale serve` (HTTPS :443, **prefix-stripped**).
- systemd units: `mnemosyne-sync.service`, `mnemosyne-dashboard.service`, `mnemosyne-bot.service`.
- **Sync server**: `mnemosyne sync-serve` on `127.0.0.1:8766` (dedicated relay DB `/root/.hermes/mnemosyne/data/sync-relay.db`, `--api-key-file /root/.hermes/mnemosyne/sync-api-key`, `600`). Endpoint: `https://vmi3272367.tail9293de.ts.net/mnemosyne-sync`, routing `/sync/pull|push|status` and `/healthz`.
- **Dashboard** (`.../mnemosyne` to `:8765`) is a **read-only UI** (`MnemosyneDashboard`), *not* a sync server. Every `/sync/*` 404s. Do not point `sync_remote` at it.
- **tirith gotcha**: the security scanner blocks **raw Tailscale IP URLs** in cron/automation. Use the Tailscale **hostname**, never `100.x.x.x`.
- SSH: `ssh hermes-vps`. `sqlite3` is available on the box for quick DB inspection.
- **Claude Code web sessions cannot reach the tailnet.** The egress policy 403s every `*.tailscale.com` host (`pkgs`, `controlplane`, `login`), so tailscaled can never authenticate and `mnemosyne sync` against hermes-vps is impossible from there. Use `mnemosyne export` on a tailnet-connected box plus `mnemosyne import` in the session instead.

## Contributor norms
- **dplush (Denis H)** is the highest-velocity core-layer contributor, so keep them shipping. **Do NOT** ask dplush to review other contributors' PRs (avoids contributor-on-contributor drift). AJ reviews.
- **CLA gotcha**: the CLA bot validates **commit** authors, not PR authors. Agent-identity commits (e.g. `Hermes Pi <...>`) break it; the contributor must `git commit --amend --author="Name <github-email>"` and force-push. If CLA will not re-trigger after a branch update, **close/reopen the PR** (comments like "recheckcla" do not work).

## Known gotchas / decided designs
- **Config precedence (#482)**: `config.yaml` > env > default. Runtime reads `MnemosyneConfig.get()` directly, with **no** YAML-to-env bridge or `apply_to_env()`. Many config keys need a **process restart** to take effect.
- **No schema-level FKs (#503 closed)**: `PRAGMA foreign_keys=ON` broke 22 tests that intentionally create orphan rows. Do orphan cleanup at app level during sleep/consolidation instead.
- **Beam access is lock-serialized (#498/#520)**: `_beam_access_lock` guards Beam/SQLite between the main thread and the auto_sleep daemon (a WAL checkpoint mid-statement caused a SEGV). Do not introduce unguarded cross-thread Beam access.
- **Import idempotency (#538, merged)**: `AnnotationStore.import_all` now skips `(memory_id,kind,value)` UNIQUE collisions instead of aborting the whole import, so `mnemosyne import` is safely re-runnable.
- **Security**: sync auth is bearer API key or JWT; a past JWT-signature-bypass was fixed, and signatures are now verified with `hmac.compare_digest` with alg pinned to HS256. The sync HTTP server is off by default in the Hermes plugin.

## CLI cheat-sheet
```
mnemosyne export [file.json] [--include-sync-events]      # read-only dump (runs on ANY arg, no --help!)
mnemosyne import <file.json> [--force]                    # merge into local DB (idempotent for memories)
mnemosyne sync --db-path <surface.db> --remote <url> --api-key-file <f> --mode bidirectional
mnemosyne sync-init --db-path <surface.db> [--claim-existing --yes]
mnemosyne sync-serve --db-path <relay.db> --host 127.0.0.1 --port <p> --api-key-file <f> [--initialize-surface]
mnemosyne sync-status --db-path <surface.db> [--remote <url>] [--api-key-file <f>] [--json]
mnemosyne config set <key> <value>                        # note: many keys need a restart
```
`export` and `remember` treat `--help` as a positional arg (dumps / stores it). Use `mnemosyne --help` for the top-level list.

## Code navigation
This repo is CodeGraph-indexed (`.codegraph/`), so prefer `codegraph_explore "<symbols or question>"` over grep/read for understanding or before editing; one call returns verbatim source, call paths, and blast radius. Sync internals live in `mnemosyne/core/sync.py`, the server in `sync_server.py`, the store in `beam.py`, annotations and triples in `core/annotations.py` and `core/triples.py`.
