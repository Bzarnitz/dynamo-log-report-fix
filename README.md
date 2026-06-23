# log-report — fixed Terminal-Bench 3 / Harbor task

This repo contains the **corrected** `log-report` task for the Project Dynamo
"Fix the Broken Terminal-Bench Task" assessment.

The underlying task is simple, everyday software engineering: parse an
Apache-style access log at `/app/access.log` and write a JSON summary to
`/app/report.json` (`total_requests`, `unique_ips`, `top_path`). The work here
was repairing the **task authoring** into correct Harbor format.

## Task layout

```
log-report/
├── task.toml                 # manifest (artifacts is an array; environment_mode = "separate")
├── instruction.md            # exact output path + format + 4 numbered success criteria
├── environment/
│   ├── Dockerfile            # pinned base image; no leaked solution
│   └── access.log            # input log (fixed, 6 request lines)
├── solution/
│   └── solve.sh              # reference oracle solution
└── tests/
    ├── Dockerfile            # verifier image (pinned python + uv)
    ├── test.sh               # runs pytest, writes reward.txt + ctrf.json to /logs/verifier/
    └── test_output.py        # asserts the real values, 1 test per success criterion
```

## Verification (Harbor 0.7.1)

```
harbor run -p log-report -a oracle     # reference solution  -> reward 1.0 (PASS)
harbor run -p log-report --agent nop   # no-op agent         -> reward 0.0 (FAIL)
```

- oracle: `reward.txt = 1`, ctrf `4 passed / 0 failed`
- nop:    `reward.txt = 0`, ctrf `0 passed / 4 failed`
- bugged solve.sh (off-by-one count): `reward.txt = 0`, ctrf `3 passed / 1 failed`
  (the verifier catches a wrong value, not just a missing file)

See [FIXES.md](./FIXES.md) for the full defect-by-defect breakdown.
