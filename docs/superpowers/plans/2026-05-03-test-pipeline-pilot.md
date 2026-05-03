# Test Pipeline Pilot — E2E Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Bootstrap `rmbrmv/pipeline-pilot-test` with a trivial Python `add(a, b) -> int` function and drive it through all 4 pipeline stages (plan → develop → test → release) to validate end-to-end pipeline-orchestrator machinery.

**Architecture:** The artifact is intentionally minimal — a single Python function with 2 tests — so all complexity lives in the orchestration layer (worktree lifecycle, journal persistence, PR/merge, watcher notifications). Four stages are executed sequentially; each stage ends with a journal event and passes state to the next.

**Tech Stack:** Python 3.11, pytest ≥ 7, setuptools src-layout, GitHub CLI (`gh`), git worktrees.

---

## File Structure

| Path | Status | Responsibility |
|---|---|---|
| `pyproject.toml` | CREATE | Package metadata, src layout, pytest dev dependency |
| `src/pilot/__init__.py` | CREATE | Public re-export of `add` |
| `src/pilot/math.py` | CREATE | `def add(a: int, b: int) -> int` — the entire deliverable |
| `tests/test_math.py` | CREATE | 2 asserts: `add(2,3)==5`, `add(-1,1)==0` |
| `.gitignore` | CREATE | Exclude `.venv/`, `__pycache__/`, `*.egg-info/` |
| `.pipeline/journal.jsonl` | ORCHESTRATOR-MANAGED | Append-only event log; must be in merge commit |
| `.pipeline/state.json` | ORCHESTRATOR-MANAGED | Pipeline state; committed at release |

**Do NOT touch:** `README.md`, `docs/`, `.pipeline/state.json` (except at release), any CI workflow files.

---

## Stage 1 — Plan (current stage)

This stage produces the plan you are reading. Exit criteria: plan file committed on feature branch.

### Task 1: Commit Plan

**Files:**
- Create: `docs/superpowers/plans/2026-05-03-test-pipeline-pilot.md`

- [ ] **Step 1: Verify plan file exists**

```bash
ls -la /srv/pipeline/projects/test-pipeline-pilot/docs/superpowers/plans/
```
Expected: `2026-05-03-test-pipeline-pilot.md` present.

- [ ] **Step 2: Stage and commit plan only**

```bash
git add docs/superpowers/plans/2026-05-03-test-pipeline-pilot.md
git commit -m "plan: add E2E pilot implementation plan"
```
Expected: 1 file changed, commit on `feature/pipeline-test-pipeline-pilot`.

---

## Stage 2 — Develop

Working directory: `/srv/pipeline/projects/test-pipeline-pilot/`
Branch: `feature/pipeline-test-pipeline-pilot`

### Task 2: Create .gitignore

**Files:**
- Create: `.gitignore`

- [ ] **Step 1: Write .gitignore**

```
.venv/
__pycache__/
*.egg-info/
dist/
.pytest_cache/
```

- [ ] **Step 2: Verify file written**

```bash
cat /srv/pipeline/projects/test-pipeline-pilot/.gitignore
```
Expected: 5 lines, no blank lines at start.

### Task 3: Create pyproject.toml

**Files:**
- Create: `pyproject.toml`

- [ ] **Step 1: Write pyproject.toml**

```toml
[build-system]
requires = ["setuptools>=68", "wheel"]
build-backend = "setuptools.backends.legacy:build"

[project]
name = "pilot"
version = "0.1.0"
requires-python = ">=3.11"
description = "Pipeline E2E pilot — trivial add function"

[project.optional-dependencies]
dev = ["pytest>=7"]

[tool.setuptools.packages.find]
where = ["src"]
```

- [ ] **Step 2: Validate TOML parses**

```bash
python3.11 -c "import tomllib; tomllib.load(open('/srv/pipeline/projects/test-pipeline-pilot/pyproject.toml','rb')); print('OK')"
```
Expected: `OK`

### Task 4: Create src/pilot package

**Files:**
- Create: `src/pilot/__init__.py`
- Create: `src/pilot/math.py`

- [ ] **Step 1: Create src/pilot directory**

```bash
mkdir -p /srv/pipeline/projects/test-pipeline-pilot/src/pilot
```

- [ ] **Step 2: Write src/pilot/math.py**

```python
def add(a: int, b: int) -> int:
    """Return the sum of two integers."""
    return a + b
```

- [ ] **Step 3: Write src/pilot/__init__.py**

```python
from pilot.math import add

__all__ = ["add"]
```

### Task 5: Create tests/test_math.py

**Files:**
- Create: `tests/test_math.py`

- [ ] **Step 1: Create tests directory**

```bash
mkdir -p /srv/pipeline/projects/test-pipeline-pilot/tests
```

- [ ] **Step 2: Write tests/test_math.py**

```python
from pilot import add


def test_add():
    assert add(2, 3) == 5
    assert add(-1, 1) == 0
```

### Task 6: Local verification and commit

**Files:** all 5 new files

- [ ] **Step 1: Create venv and install**

```bash
python3.11 -m venv /tmp/pilot-venv
/tmp/pilot-venv/bin/pip install -e /srv/pipeline/projects/test-pipeline-pilot[dev] --quiet
```
Expected: `Successfully installed pilot-0.1.0 ...`

- [ ] **Step 2: Run pytest**

```bash
/tmp/pilot-venv/bin/pytest /srv/pipeline/projects/test-pipeline-pilot/tests/ -v
```
Expected output:
```
tests/test_math.py::test_add PASSED
1 passed in <1s
```

- [ ] **Step 3: Verify import**

```bash
/tmp/pilot-venv/bin/python -c "from pilot import add; assert add(2,3)==5; print('import OK')"
```
Expected: `import OK`

- [ ] **Step 4: Verify clean worktree (no .pipeline/ leak)**

```bash
git -C /srv/pipeline/projects/test-pipeline-pilot status --porcelain | grep -v '^\?\? \.pipeline'
```
Expected output: only the 5 new files listed (`.gitignore`, `pyproject.toml`, `src/pilot/__init__.py`, `src/pilot/math.py`, `tests/test_math.py`).

- [ ] **Step 5: Stage exactly the 5 new files**

```bash
git -C /srv/pipeline/projects/test-pipeline-pilot add \
  .gitignore \
  pyproject.toml \
  src/pilot/__init__.py \
  src/pilot/math.py \
  tests/test_math.py
```

- [ ] **Step 6: Commit**

```bash
git -C /srv/pipeline/projects/test-pipeline-pilot commit -m "feat: add pilot.add with pytest harness"
```
Expected: `1 commit, 5 files changed`.

---

## Stage 3 — Test

### Task 7: Reproducibility check and branch push

- [ ] **Step 1: Fresh venv pytest run**

```bash
python3.11 -m venv /tmp/pilot-venv2
/tmp/pilot-venv2/bin/pip install -e /srv/pipeline/projects/test-pipeline-pilot[dev] --quiet
/tmp/pilot-venv2/bin/pytest /srv/pipeline/projects/test-pipeline-pilot/tests/ -v
```
Expected: `1 passed`, exit 0.

- [ ] **Step 2: Check test discovery**

```bash
/tmp/pilot-venv2/bin/pytest /srv/pipeline/projects/test-pipeline-pilot --collect-only -q
```
Expected: `tests/test_math.py::test_add`

- [ ] **Step 3: Push branch to origin**

```bash
git -C /srv/pipeline/projects/test-pipeline-pilot push -u origin feature/pipeline-test-pipeline-pilot
```
Expected: branch tracking set on `origin`.

- [ ] **Step 4: Append test_pass event to journal**

```bash
echo '{"ts":"'$(date -u +%Y-%m-%dT%H:%M:%S)'","type":"test_pass","payload":{"suite":"pytest","result":"1 passed","stage":"test"}}' \
  >> /srv/pipeline/projects/test-pipeline-pilot/.pipeline/journal.jsonl
```

---

## Stage 4 — Release

### Task 8: Finalize journal and commit

- [ ] **Step 1: Append stage_end events to journal**

```bash
TS=$(date -u +%Y-%m-%dT%H:%M:%S)
echo '{"ts":"'"$TS"'","type":"stage_end","payload":{"stage":"test"}}' \
  >> /srv/pipeline/projects/test-pipeline-pilot/.pipeline/journal.jsonl
echo '{"ts":"'"$TS"'","type":"stage_start","payload":{"stage":"release"}}' \
  >> /srv/pipeline/projects/test-pipeline-pilot/.pipeline/journal.jsonl
```

- [ ] **Step 2: Commit journal and state before PR**

```bash
git -C /srv/pipeline/projects/test-pipeline-pilot add \
  .pipeline/journal.jsonl \
  .pipeline/state.json
git -C /srv/pipeline/projects/test-pipeline-pilot commit \
  -m "chore: finalize pipeline journal for merge"
git -C /srv/pipeline/projects/test-pipeline-pilot push
```
Expected: journal commit on feature branch, pushed. This ensures journal appears in merge commit diff.

### Task 9: Check branch protection and create PR

- [ ] **Step 1: Probe for branch protection**

```bash
gh api repos/rmbrmv/pipeline-pilot-test/branches/master/protection 2>&1 | head -5
```
If 404 → no protection, proceed. If 200 → note required reviewers/checks.

- [ ] **Step 2: Check for existing CI workflows**

```bash
gh api repos/rmbrmv/pipeline-pilot-test/contents/.github/workflows 2>&1 | head -5
```
If 404 → no workflows, release can merge immediately. If 200 → wait for checks.

- [ ] **Step 3: Create PR**

```bash
gh pr create \
  --repo rmbrmv/pipeline-pilot-test \
  --base master \
  --head feature/pipeline-test-pipeline-pilot \
  --title "feat: pilot.add E2E pipeline pilot" \
  --body "$(cat <<'EOF'
## Summary
- Adds `pilot.add(a, b) -> int` with 2 pytest assertions
- E2E validation of pipeline-orchestrator stages: plan → develop → test → release

## Verification
- `pytest tests/` green (1 passed)
- `.pipeline/journal.jsonl` included in this branch (visible in merge commit diff)

## Pipeline journal
See `.pipeline/journal.jsonl` on this branch for stage-by-stage event log.
EOF
)"
```
Expected: PR URL printed (capture it).

### Task 10: Merge PR and clean up

- [ ] **Step 1: Wait for checks (if CI exists)**

```bash
gh pr checks --repo rmbrmv/pipeline-pilot-test --watch
```
Skip if Step 9.2 confirmed no workflows.

- [ ] **Step 2: Merge with --merge (NOT squash — preserves journal in diff)**

```bash
gh pr merge --repo rmbrmv/pipeline-pilot-test \
  --merge \
  --delete-branch
```
Expected: PR merged, remote branch `feature/pipeline-test-pipeline-pilot` deleted.

- [ ] **Step 3: Verify journal in merge commit**

```bash
gh api repos/rmbrmv/pipeline-pilot-test/commits \
  --jq '.[0] | {sha: .sha, message: .commit.message}'
```
Then verify journal present:
```bash
gh api repos/rmbrmv/pipeline-pilot-test/commits/master \
  --jq '.files[].filename' | grep journal
```
Expected: `.pipeline/journal.jsonl` appears in file list.

- [ ] **Step 4: Append final pipeline_complete event to journal locally**

```bash
echo '{"ts":"'$(date -u +%Y-%m-%dT%H:%M:%S)'","type":"pipeline_complete","payload":{"status":"success"}}' \
  >> /srv/pipeline/projects/test-pipeline-pilot/.pipeline/journal.jsonl
```

### Task 11: Worktree cleanup

- [ ] **Step 1: Confirm watcher received ≥1 notification**

Check PC watcher tab for any event from this pipeline run. If silent after 30 min → spec failure, mark `blocked`.

- [ ] **Step 2: Remove worktree (run from outside worktree)**

```bash
git -C /srv/pipeline/source-repo worktree remove \
  /srv/pipeline/projects/test-pipeline-pilot --force
```
Expected: no error.

- [ ] **Step 3: Prune stale worktree refs**

```bash
git -C /srv/pipeline/source-repo worktree prune
git -C /srv/pipeline/source-repo worktree list
```
Expected: `/srv/pipeline/projects/test-pipeline-pilot` NOT listed.

- [ ] **Step 4: Verify worktree directory gone**

```bash
ls /srv/pipeline/projects/test-pipeline-pilot 2>&1
```
Expected: `No such file or directory`.

---

## Risks & Mitigations

| Risk | Likelihood | Mitigation |
|---|---|---|
| Worktree remove fails — lock held by orchestrator or cwd inside worktree | Medium | `cd /tmp` before remove; use `--force`; release stage explicitly closes `.archive.lock` first |
| Journal missing from merge commit — PR squash-merged | Medium | Hardcode `--merge` flag; assert journal in diff via Step 10.3 before marking done |
| Codex review loop > 3 iterations on 3-line function | Low | Hard-stop at iteration 3; dump transcript to journal; mark `blocked` |
| `src` layout breaks pytest discovery | Low | Editable install (`pip install -e .[dev]`) registers `pilot` package — no `sys.path` hacks needed |
| `.pipeline/state.json` accidentally staged during develop | Medium | All `git add` steps are file-explicit; never use `git add -A` |
| Watcher silent > 30 min | Low | Check watcher heartbeat in Task 11 Step 1 before removing worktree |
| Branch protection requires reviewer on `rmbrmv/pipeline-pilot-test` | Low | Probe in Task 9 Step 1; escalate if required review blocks merge |

---

## Open Questions

1. **CI presence** — does `rmbrmv/pipeline-pilot-test` already have a `.github/workflows/` file on master? Task 9 Step 2 probes this. If yes: what does it run and does it gate merge?

2. **Watcher verification mechanism** — is "watcher на ПК выдал хотя бы одно сообщение" verified automatically (orchestrator polls API) or manually (user confirms)? If manual, Task 11 Step 1 must pause for confirmation.

3. **Journal-finalize commit author** — should it use `pipeline-bot` identity or user git config? Check orchestrator config before Task 8 Step 2.

4. **Postmortem write-back** (out of scope for this pipeline run) — `docs/experiments/m15-e2e-pilot-results.md` goes into the `pipeline-orchestrator` repo, not here. Flag as follow-up task.

5. **`.gitignore` in commit** — included in Task 2 as a low-risk quality-of-life addition. If spec mandates strictly no extra files, drop Task 2 and adjust Task 6 Step 5.
