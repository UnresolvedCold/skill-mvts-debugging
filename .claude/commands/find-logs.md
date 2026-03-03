List all available log files in the MVTS pod, sorted newest first.

**Arguments:** `[namespace]`

Example: `/find-logs qa3-cluster-devrelayonm-greymatter`

---

Parse `$ARGUMENTS` to extract `namespace`. If not provided, ask the user before proceeding.

Run the following commands and present the results in a readable table.

## Step 1 — List all log files with sizes and dates

```bash
kubectl exec mvts-0 -n <namespace> -c ml-engine-mvts -- sh -c \
  'ls -lht /app/data/logs/ | grep -E "scheduler"'
```

## Step 2 — Show date range coverage

```bash
kubectl exec mvts-0 -n <namespace> -c ml-engine-mvts -- sh -c \
  'ls /app/data/logs/scheduler.*.log.gz 2>/dev/null | sort -r'
```

## Output Format

Present results as a table:

```
File                                Size    Modified
────────────────────────────────────────────────────
scheduler.log                       <size>  <date>   ← active
scheduler.2026-02-24.3.log.gz       <size>  <date>
scheduler.2026-02-24.2.log.gz       <size>  <date>
scheduler.2026-02-24.1.log.gz       <size>  <date>
...
```

Also print:
- **Oldest log date** — how far back logs go
- **Total size** — combined size of all log files
- **Tip:** If the identifier you're searching for predates the oldest log, the data may no longer be available.
