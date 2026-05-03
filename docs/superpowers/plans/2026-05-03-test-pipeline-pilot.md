# Test Pipeline Pilot — E2E Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Bootstrap `rmbrmv/pipeline-pilot-test` with a trivial Python `add(a, b) -> int` function and drive it through all 4 pipeline stages (plan → develop → test → release) to validate end-to-end pipeline-orchestrator machinery.

**Architecture:** The artifact is intentionally minimal — a single Python function with 2 tests — so all complexity lives in the orchestration layer (worktree lifecycle, journal persistence, PR/merge, watcher notifications). Four stages are executed sequentially; each stage ends with a journal event and passes state to the next. All journal events are written via `python3 -c "import json..."` to guarantee valid JSONL.

**Tech Stack:** Python 3.11, pytest ≥ 7, setuptools src-layout, GitHub CLI (`gh`), git worktrees.

---

## File Structure

| Path | Status | Responsibility |
|---|---|---|
| `docs/superpowers/plans/2026-05-03-test-pipeline-pilot.md` | CREATE (Stage 1) | This plan file |
| `pyproject.toml` | CREATE (Stage 2) | Package metadata, src layout, pytest dev dependency |
| `src/pilot/__init__.py` | CREATE (Stage 2) | Public re-export of `add` |
| `src/pilot/math.py` | CREATE (Stage 2) | `def add(a: int, b: int) -> int` — the entire deliverable |
| `tests/test_math.py` | CREATE (Stage 2) | 2 asserts: `add(2,3)==5`, `add(-1,1)==0` |
| `.gitignore` | CREATE (Stage 2) | Exclude `.venv/`, `__pycache__/`, `*.egg-info/` |
| `.pipeline/journal.jsonl` | ORCHESTRATOR-MANAGED | Append-only event log; must be in merge commit |
| `.pipeline/state.json` | ORCHESTRATOR-MANAGED | Pipeline state; updated and committed at release |

**Do NOT touch:** `README.md`, `docs/specs/`, any CI workflow files.  
**Never use `git add -A` or `git add .`** — always stage files explicitly to prevent `.pipeline/` leaks.

---

## Stage 0 — Preflight

Run once before any stage work. Fail fast on auth/remote issues rather than discovering them after local work is done.

### Task 0: Preflight checks

- [ ] **Step 1: Verify worktree exists and branch is correct**

```bash
git -C /srv/pipeline/projects/test-pipeline-pilot branch --show-current
```
Expected: `feature/pipeline-test-pipeline-pilot`

- [ ] **Step 2: Verify remote points at target repo**

```bash
git -C /srv/pipeline/projects/test-pipeline-pilot remote -v
```
Expected: `origin` → `git@github.com:rmbrmv/pipeline-pilot-test.git` (or https equivalent) for both fetch and push.

- [ ] **Step 3: Verify GitHub CLI auth**

```bash
gh auth status
```
Expected: `Logged in to github.com` — no auth errors.

- [ ] **Step 4: Verify access to target repo**

```bash
gh api repos/rmbrmv/pipeline-pilot-test --jq '.full_name'
```
Expected: `rmbrmv/pipeline-pilot-test`. If 404 → repo does not exist or no access; stop and investigate before proceeding.

- [ ] **Step 5: Check for CI workflows on target repo**

```bash
gh api repos/rmbrmv/pipeline-pilot-test/contents/.github/workflows 2>&1 | head -3
```
Record result:
- If `404` → **no CI**, set `HAS_CI=false`. Release will merge immediately after PR is open.
- If JSON array → **CI present**, set `HAS_CI=true`. Release must wait for green checks.

- [ ] **Step 6: Check for branch protection on master**

```bash
gh api repos/rmbrmv/pipeline-pilot-test/branches/master/protection 2>&1 | head -5
```
Record result:
- If `404` → no protection, direct merge allowed.
- If JSON → note `required_pull_request_reviews.required_approving_review_count` field. If > 0 → a human reviewer must approve before merge. The designated reviewer is the repo owner (`rmbrmv`). Maximum wait: 60 minutes. If no approval after 60 min → mark pipeline `blocked`, do NOT remove worktree.

---

## Stage 1 — Plan (current stage)

This stage produces the plan you are reading. Exit criteria: plan file committed on `feature/pipeline-test-pipeline-pilot`.

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
git -C /srv/pipeline/projects/test-pipeline-pilot \
  add docs/superpowers/plans/2026-05-03-test-pipeline-pilot.md
git -C /srv/pipeline/projects/test-pipeline-pilot \
  commit -m "plan: add E2E pilot implementation plan"
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
build-backend = "setuptools.build_meta"

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

**Files:** `.gitignore`, `pyproject.toml`, `src/pilot/__init__.py`, `src/pilot/math.py`, `tests/test_math.py`

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
git -C /srv/pipeline/projects/test-pipeline-pilot status --porcelain \
  | grep -v '^?? \.pipeline'
```
Expected output: only the 5 new files (`.gitignore`, `pyproject.toml`, `src/pilot/__init__.py`, `src/pilot/math.py`, `tests/test_math.py`) — nothing else.

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
git -C /srv/pipeline/projects/test-pipeline-pilot \
  commit -m "feat: add pilot.add with pytest harness"
```
Expected: `1 commit, 5 files changed`.

---

## Stage 3 — Test

### Task 7: Reproducibility check, branch push, and journal commit

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
git -C /srv/pipeline/projects/test-pipeline-pilot \
  push -u origin feature/pipeline-test-pipeline-pilot
```
Expected: branch tracking set on `origin`.

- [ ] **Step 4: Append test_pass event to journal (valid JSON via python3)**

```bash
python3 -c "
import json, datetime, pathlib
j = pathlib.Path('/srv/pipeline/projects/test-pipeline-pilot/.pipeline/journal.jsonl')
event = {'ts': datetime.datetime.utcnow().strftime('%Y-%m-%dT%H:%M:%S'),
         'type': 'test_pass',
         'payload': {'suite': 'pytest', 'result': '1 passed', 'stage': 'test'}}
j.open('a').write(json.dumps(event) + '\n')
print('appended:', json.dumps(event))
"
```

- [ ] **Step 5: Validate journal JSONL integrity**

```bash
python3 -c "
import json
lines = open('/srv/pipeline/projects/test-pipeline-pilot/.pipeline/journal.jsonl').readlines()
[json.loads(l) for l in lines if l.strip()]
print(f'OK — {len(lines)} valid JSON lines')
"
```
Expected: `OK — N valid JSON lines` (no exception).

- [ ] **Step 6: Commit journal after test_pass (so it persists if process interrupted)**

```bash
git -C /srv/pipeline/projects/test-pipeline-pilot \
  add .pipeline/journal.jsonl
git -C /srv/pipeline/projects/test-pipeline-pilot \
  commit -m "test: record test_pass journal event"
git -C /srv/pipeline/projects/test-pipeline-pilot push
```
Expected: commit pushed. Stage 3 state is now durable.

---

## Stage 4 — Release

### Task 8: Finalize journal, update state.json, and commit before PR

- [ ] **Step 1: Append stage_end and stage_start events**

```bash
python3 -c "
import json, datetime, pathlib
j = pathlib.Path('/srv/pipeline/projects/test-pipeline-pilot/.pipeline/journal.jsonl')
ts = datetime.datetime.utcnow().strftime('%Y-%m-%dT%H:%M:%S')
events = [
    {'ts': ts, 'type': 'stage_end', 'payload': {'stage': 'test'}},
    {'ts': ts, 'type': 'stage_start', 'payload': {'stage': 'release', 'expected_k': 30}},
]
f = j.open('a')
for e in events:
    f.write(json.dumps(e) + '\n')
    print('appended:', json.dumps(e))
"
```

- [ ] **Step 2: Update state.json to reflect release stage**

Read current state.json, then update `current_stage`, `status`, and `updated_at`:
```bash
python3 -c "
import json, datetime, pathlib
p = pathlib.Path('/srv/pipeline/projects/test-pipeline-pilot/.pipeline/state.json')
state = json.loads(p.read_text())
state['current_stage'] = 'release'
state['stage_iteration'] = 0
state['updated_at'] = datetime.datetime.utcnow().strftime('%Y-%m-%dT%H:%M:%S')
p.write_text(json.dumps(state, indent=2))
print('state.json updated: current_stage=release')
"
```
Expected: `state.json updated: current_stage=release`

- [ ] **Step 3: Append pipeline_complete event (must be BEFORE merge to appear in merge commit)**

```bash
python3 -c "
import json, datetime, pathlib
j = pathlib.Path('/srv/pipeline/projects/test-pipeline-pilot/.pipeline/journal.jsonl')
ts = datetime.datetime.utcnow().strftime('%Y-%m-%dT%H:%M:%S')
event = {'ts': ts, 'type': 'pipeline_complete', 'payload': {'status': 'success'}}
j.open('a').write(json.dumps(event) + '\n')
print('appended:', json.dumps(event))
"
```

- [ ] **Step 4: Validate journal JSONL one final time**

```bash
python3 -c "
import json
lines = open('/srv/pipeline/projects/test-pipeline-pilot/.pipeline/journal.jsonl').readlines()
[json.loads(l) for l in lines if l.strip()]
print(f'OK — {len(lines)} valid JSON lines')
"
```

- [ ] **Step 5: Check journal-commit author identity**

```bash
git -C /srv/pipeline/projects/test-pipeline-pilot config user.email
git -C /srv/pipeline/projects/test-pipeline-pilot config user.name
```
Verify this matches the expected orchestrator identity. If wrong, set before committing:
```bash
# Only if needed — do not override if already correct:
# git -C /srv/pipeline/projects/test-pipeline-pilot config user.email "pipeline-bot@example.com"
# git -C /srv/pipeline/projects/test-pipeline-pilot config user.name "Pipeline Bot"
```

- [ ] **Step 6: Commit journal and state before PR (this is the commit that will be in the merge)**

```bash
git -C /srv/pipeline/projects/test-pipeline-pilot add \
  .pipeline/journal.jsonl \
  .pipeline/state.json
git -C /srv/pipeline/projects/test-pipeline-pilot \
  commit -m "chore: finalize pipeline journal for merge"
git -C /srv/pipeline/projects/test-pipeline-pilot push
```
Expected: commit pushed. Merge commit will include `journal.jsonl` with `pipeline_complete` event.

### Task 9: Create PR

- [ ] **Step 1: Create PR (separate from merge — no `--merge` flag here)**

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
- `pipeline_complete` event present in journal before merge

## Pipeline journal
See `.pipeline/journal.jsonl` on this branch for stage-by-stage event log.
EOF
)"
```
Expected: PR URL printed (capture PR number for next steps).

### Task 10: Wait for checks (if CI) and merge PR

- [ ] **Step 1: Decide CI wait path based on Task 0 Step 5 result**

If `HAS_CI=false` (Task 0 Step 5 returned 404):
```bash
echo "No CI workflows found — proceeding to merge immediately."
```

If `HAS_CI=true`:
```bash
gh pr checks --repo rmbrmv/pipeline-pilot-test --watch
```
Wait until all checks are green (exit 0). If any check fails → investigate; do NOT force-merge.

- [ ] **Step 2: Merge with --merge strategy (NOT --squash, NOT --rebase)**

Using `--merge` preserves the journal commit in the merge commit diff, satisfying the spec requirement.
```bash
gh pr merge --repo rmbrmv/pipeline-pilot-test \
  --merge \
  --delete-branch
```
Expected: PR merged, remote branch `feature/pipeline-test-pipeline-pilot` deleted, PR closed.

If this fails due to branch protection requiring reviewer (identified in Task 0 Step 6):
- Request review from repo owner: `gh pr review --repo rmbrmv/pipeline-pilot-test --request rmbrmv`  
- Wait up to 60 minutes for approval.
- If no approval after 60 min → mark pipeline `blocked`, do NOT remove worktree; stop here.

- [ ] **Step 3: Verify journal appears in merge commit diff**

```bash
gh api repos/rmbrmv/pipeline-pilot-test/commits \
  --jq '.[0] | {sha: .sha, message: .commit.message}'
```
Then:
```bash
gh api repos/rmbrmv/pipeline-pilot-test/commits/master \
  --jq '[.files[].filename]' | grep journal
```
Expected: `.pipeline/journal.jsonl` in the file list. If absent → spec failure criterion; log as incident.

### Task 11: Watcher check and worktree cleanup

- [ ] **Step 1: Check watcher notification (best-effort with timeout)**

The watcher is verified by checking for any pipeline event logged to the PC watcher channel.
- **Automated check** (if orchestrator exposes watcher API): poll watcher endpoint for events from this run's `session_id`.
- **Manual check** (if no API): confirm in watcher tab on PC. If no event visible:
  - Wait up to 30 minutes.
  - After 30 min with no event → mark watcher check as `skipped_watcher_unverifiable`; log to journal; continue with cleanup (do NOT block worktree removal on this).

```bash
python3 -c "
import json, datetime, pathlib
j = pathlib.Path('/srv/pipeline/projects/test-pipeline-pilot/.pipeline/journal.jsonl')
ts = datetime.datetime.utcnow().strftime('%Y-%m-%dT%H:%M:%S')
event = {'ts': ts, 'type': 'watcher_check', 'payload': {'result': 'skipped_watcher_unverifiable'}}
# Replace 'skipped_watcher_unverifiable' with 'confirmed' if watcher received notification
j.open('a').write(json.dumps(event) + '\n')
print('watcher check event appended')
"
```

- [ ] **Step 2: Remove worktree — run from OUTSIDE the worktree**

```bash
cd /tmp && git -C /srv/pipeline/source-repo worktree remove \
  /srv/pipeline/projects/test-pipeline-pilot --force
```
Expected: no error. The `--force` is required because `.pipeline/.archive.lock` may still be present.

- [ ] **Step 3: Prune stale worktree refs**

```bash
git -C /srv/pipeline/source-repo worktree prune
git -C /srv/pipeline/source-repo worktree list
```
Expected: `/srv/pipeline/projects/test-pipeline-pilot` NOT in the list.

- [ ] **Step 4: Verify worktree directory is gone**

```bash
ls /srv/pipeline/projects/test-pipeline-pilot 2>&1
```
Expected: `No such file or directory`.

---

## Risks & Mitigations

| Risk | Likelihood | Mitigation |
|---|---|---|
| Worktree remove fails — lock held or cwd inside worktree | Medium | `cd /tmp` before remove; use `--force`; release stage closes `.archive.lock` first |
| Journal missing from merge commit — PR squash/rebase-merged | Medium | Hardcode `--merge` in Task 10 Step 2; assert journal in diff via Step 3 before marking done |
| Codex review loop > 3 iterations | Low | Hard-stop at iteration 3; dump transcript to journal; mark `blocked` |
| `src` layout breaks pytest discovery | Low | Editable install (`pip install -e .[dev]`) registers `pilot` package — no `sys.path` hacks needed |
| `.pipeline/state.json` accidentally staged during develop | Medium | All `git add` steps are file-explicit; never use `git add -A` |
| Watcher silent > 30 min | Low | Downgraded from blocking to best-effort; watcher check continues to cleanup regardless |
| Branch protection requires reviewer | Low | Identified in Task 0 Step 6; request review from `rmbrmv`; 60 min timeout before `blocked` |
| `pipeline_complete` in wrong commit | Low (fixed) | Appended BEFORE merge commit in Task 8 Step 3 — present in merge commit diff |
| Invalid JSONL in journal | Low (fixed) | All journal writes use `python3 json.dumps()`; validated via `json.loads()` after each write |
| Wrong build backend causes `pip install` failure | Low (fixed) | Fixed to `setuptools.build_meta` (standard) in Task 3 Step 1 |

---

## Open Questions

1. **CI presence** — resolved at runtime in Task 0 Step 5. If CI exists, document what it runs and whether it gates merge.

2. **Watcher verification mechanism** — downgraded from blocking to best-effort (Task 11 Step 1). If a watcher API exists, swap in the automated poll; otherwise manual/timeout path applies.

3. **Journal-finalize commit author** — verified in Task 8 Step 5. Must match expected orchestrator identity.

4. **Postmortem write-back** (out of scope for this pipeline run) — `docs/experiments/m15-e2e-pilot-results.md` goes into the `pipeline-orchestrator` repo after this run completes. Flag as follow-up task.

5. **`.gitignore` in commit** — included in Task 2. Spec mandates minimal scope but doesn't prohibit `.gitignore`. Low risk. If strictly unacceptable, remove Task 2 and adjust Task 6 Step 4-5.
