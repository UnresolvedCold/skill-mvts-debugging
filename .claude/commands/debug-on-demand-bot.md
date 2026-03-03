Investigate why MVTS returned an empty `rangerIDList` for an on-demand bot request at a PPS conveyor exit.

**Arguments:** `<pps_id> <gmc_timestamp> [namespace]`

Example: `/debug-on-demand-bot 5 "2026-02-24 19:43:01.593" stg001-cluster-walmartpotstg-greymatter`

---

Parse `$ARGUMENTS` to extract:
- `pps_id` — first token
- `gmc_timestamp` — second token (quoted datetime string)
- `namespace` — third token if provided; otherwise ask the user before proceeding

Then run all steps below in sequence. Carry extracted values forward automatically. Print a structured summary at the end.

## Step 1 — Find the log file

Search `scheduler.log` first, then archived logs descending. Stop at the first file that matches the timestamp date.

```bash
kubectl exec mvts-0 -n <namespace> -c ml-engine-mvts -- sh -c \
  'zgrep -l "OnDemandBotRequiredAtPPSInputMessage" /app/data/logs/*.gz 2>/dev/null | sort -r | head -3'
```

Extract: `log_file`

## Step 2 — Find all on-demand API calls for this PPS near the timestamp

```bash
kubectl exec mvts-0 -n <namespace> -c ml-engine-mvts -- sh -c \
  'zgrep "OnDemandBotRequiredAtPPSInputMessage" /app/data/logs/<log_file> 2>/dev/null | grep "ppsID.*<pps_id>"'
```

Find the failing call (at or just after `gmc_timestamp`) and the preceding successful call if any.
Extract: `request_id` of the failing call, timestamps of both calls.

## Step 3 — Find the Message input just before the failing call

Look for the last planning cycle Message before the failing API call timestamp:

```bash
kubectl exec mvts-0 -n <namespace> -c ml-engine-mvts -- sh -c \
  'zgrep "Message:" /app/data/logs/<log_file> 2>/dev/null | grep -B1 "<failing_timestamp>" | tail -2'
```

Use a Python script for JSON parsing if needed. Extract:
- `ranger_list[]` — all bots and their `available_at_time`, `position`, `bot_type_version`
- `pps_list[id=<pps_id>]` — totes at the PPS, their states, `assigned_ranger_id`

## Step 4 — Check bot availability at the time of the failing call

From the Message JSON, parse `ranger_list` to find bots whose `available_at_time <= failing_call_timestamp + 5000ms`. These would have been eligible.

Write and copy a Python script:
```python
import re, json, gzip

logfile = "/app/data/logs/<log_file>"
pps_id = <pps_id>
target_time = <start_time_ms>  # from the failing Message

opener = gzip.open if logfile.endswith(".gz") else open
with opener(logfile, "rt") as f:
    for line in f:
        if "Message:" not in line:
            continue
        m = re.search(r"Message: (\{.*)", line)
        if not m:
            continue
        data = json.loads(m.group(1))
        start_time = data.get("start_time", 0)
        if abs(start_time - target_time) > 60000:  # within 1 min
            continue
        rangers = data.get("ranger_list", [])
        available = [r for r in rangers if r.get("available_at_time", 9e18) <= start_time + 5000]
        print(f"start_time={start_time}, available bots: {[r['ranger_id'] for r in available]}")
        pps = next((p for p in data.get("pps_list", []) if p.get("id") == pps_id), None)
        if pps:
            totes = pps.get("tote_list", [])
            print(f"PPS {pps_id} totes at conveyor exit: {[t for t in totes if t.get('status') == 'going_to_conveyor_exit']}")
```

## Step 5 — Check what happened to the previously dispatched bot

If a successful on-demand call preceded the failure, find what task that bot was assigned to:

```bash
kubectl exec mvts-0 -n <namespace> -c ml-engine-mvts -- sh -c \
  'zgrep "Output:.*ranger_id.*<bot_id>" /app/data/logs/<log_file> 2>/dev/null | tail -5'
```

## Final Summary

Print a structured report:
```
PPS ID:              <pps_id>
GMC timestamp:       <gmc_timestamp>
Log file:            <log_file>

Successful call:     <timestamp>  → rangerIDList: [<bot_id>]
Failing call:        <timestamp>  → rangerIDList: []

Bots available at successful call:  <list>
Bots available at failing call:     <list>

Bot <bot_id> status at failing call: <busy/unavailable — task_key / location>

Totes stuck at PPS <pps_id>: <count>

Root cause: <one-line explanation>
```
