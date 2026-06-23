# log-report — defects found & fixes

Task: repair the broken Terminal-Bench 3 / Harbor task `log-report`
(parse `/app/access.log` → write a JSON summary to `/app/report.json`).

Verified with Harbor 0.7.1:
- `harbor run -p log-report -a oracle` → **reward 1.0 (PASS)**, ctrf 4/4 passed
- `harbor run -p log-report --agent nop` → **reward 0.0 (FAIL)**, ctrf 0/4 passed
- Verifier wrote both `/logs/verifier/reward.txt` and `/logs/verifier/ctrf.json`.

## Root cause: four-way path inconsistency
`instruction.md` (no path) ≠ `task.toml artifacts` (`/app/out.json`) ≠ verifier (`/app/report.json`) ≠ `solve.sh` (`/app/report.json`).
Everything is now unified on **`/app/report.json`**.

## Defects and fixes

| # | File | Defect | Fix |
|---|------|--------|-----|
| 1 | `task.toml` | `artifacts = "/app/out.json"` — a **string**, and the wrong path (the task never writes `out.json`) | `artifacts = ["/app/report.json"]` — a TOML **array** naming the real output |
| 2 | `task.toml` | `schema_version = "1.1"` (off-convention) | `schema_version = "1.0"` (matches all canonical TB3 tasks) |
| 3 | `task.toml` | `verification_explanation` was vague ("Check the report file.") | Describes the actual value assertions (6 / 3 / `/index.html`) |
| 4 | `environment/Dockerfile` | `FROM python:latest` — unpinned, not reproducible | `FROM python:3.13-slim-bookworm` (pinned) |
| 5 | `environment/Dockerfile` + `solution_hint.py` | **Reference solution leaked** into the agent image via `COPY solution_hint.py` | Removed the `COPY` and **deleted** `solution_hint.py` |
| 6 | `tests/test_output.py` | Only asserted the file **exists / is non-empty** — a no-op that writes any bytes would "pass" the spirit of the check | Asserts the **real computed values**: `total_requests == 6`, `unique_ips == 3`, `top_path == "/index.html"`, plus valid-JSON-object. Each test's docstring names the success criterion it covers (1:1) |
| 7 | `tests/test.sh` | Wrote reward to `/app/reward.txt`; produced **no** `ctrf.json`; missing `pytest-json-ctrf` | Writes `reward.txt` **and** `ctrf.json` to `/logs/verifier/`; runs pinned `pytest==8.4.1` + `pytest-json-ctrf==0.3.5` via `uvx` |
| 8 | `instruction.md` | No output path, no format, no numbered criteria — not consistent with the verifier | States exact path `/app/report.json`, the JSON schema (`total_requests` int, `unique_ips` int, `top_path` str), 4 numbered success criteria (1:1 with the tests), no leaked hints, ends with the "You have 120 seconds…" line |
| 9 | several files | missing `harbor-canary` header | Added the canary (existing task GUID) to `instruction.md`, both `Dockerfile`s, `solve.sh`, `test.sh` |

## Four-way consistency (final)
`instruction.md` output path == `task.toml` `artifacts` == verifier assertions == `solve.sh` write target == **`/app/report.json`**.

## Expected values (derived from the fixed 6-line `access.log`)
- `total_requests` = 6
- `unique_ips` = 3 (`192.168.0.1`, `192.168.0.2`, `10.0.0.5`)
- `top_path` = `/index.html` (3 hits vs `/about.html` 2, `/api/login` 1)
