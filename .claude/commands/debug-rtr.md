Investigate an MVTS issue starting from an RTR ID (also called ETR ID externally).

**Arguments:** `<rtr_id> [namespace]`

Example: `/debug-rtr fcff337d-9811-4d11-8eb6-c2623548693d qa3-cluster-devrelayonm-greymatter`

---

Parse `$ARGUMENTS` to extract:
- `rtr_id` — first token
- `namespace` — second token if provided; otherwise ask the user before proceeding

Then run all steps below in sequence using `kubectl exec`. Carry extracted values forward into each subsequent step automatically. At the end, print a structured summary of all findings.

## Step 1 — Find the log file

Search `scheduler.log` first, then archived logs. Stop at the first file that matches.

```bash
kubectl exec mvts-0 -n <namespace> -c ml-engine-mvts -- sh -c \
  'grep -l "<rtr_id>" /app/data/logs/scheduler.log 2>/dev/null; zgrep -l "<rtr_id>" /app/data/logs/*.gz 2>/dev/null | sort -r | head -1'
```

Extract: `log_file`

## Step 2 — Find the task

```bash
kubectl exec mvts-0 -n <namespace> -c ml-engine-mvts -- sh -c \
  'zgrep "Message:.*<rtr_id>" /app/data/logs/<log_file> 2>/dev/null || grep "Message:.*<rtr_id>" /app/data/logs/<log_file> 2>/dev/null | head -1'
```

Extract from the JSON: `task_key`, `task_type`, `task_subtype`, `status`, `serviced_bins`, `serviced_orders`

## Step 3 — Find first bot assignment

```bash
kubectl exec mvts-0 -n <namespace> -c ml-engine-mvts -- sh -c \
  'zgrep "Output:.*<task_key>" /app/data/logs/<log_file> 2>/dev/null || grep "Output:.*<task_key>" /app/data/logs/<log_file> 2>/dev/null | head -1'
```

Extract: `request_id`, `timestamp`, `ranger_id`

## Step 4 — Get the input Message for that request

```bash
kubectl exec mvts-0 -n <namespace> -c ml-engine-mvts -- sh -c \
  'zgrep "Message:.*<request_id>" /app/data/logs/<log_file> 2>/dev/null || grep "Message:.*<request_id>" /app/data/logs/<log_file> 2>/dev/null | head -1'
```

> Message lines are 3–4 MB. If the raw grep output is truncated or unreliable, write and copy a Python parser instead:
> ```bash
> kubectl cp /tmp/parse_msg.py mvts-0:/tmp/parse_msg.py -n <namespace> -c ml-engine-mvts
> kubectl exec mvts-0 -n <namespace> -c ml-engine-mvts -- python3 /tmp/parse_msg.py
> ```

Extract `serviced_bins` and `serviced_orders` for the specific `task_key`.

## Step 5 — Check bin virtual/MSIO status

From the Message JSON, look up `pps_list[id=<pps_id>].bin_details[bin_id=<bin_id>]`:
- `is_virtual_bin_used` — whether the bin slot is virtual
- `is_msio` — whether it's a Multi-Slot Input/Output bin
- `promoted_to_physical_bin` on each order in `serviced_orders`

Apply the HTM Assignment Rule:
> Assignment is valid only if at least one order has `promoted_to_physical_bin: true`, OR at least one order's bin is not MSIO and not virtual. If all orders are MSIO/virtual and none promoted → assignment should have been withheld.

## Final Summary

Print a structured report:
```
RTR ID:        <rtr_id>
Log file:      <log_file>
Task key:      <task_key>
Task type:     <task_type> → <task_subtype>
First assign:  <timestamp>  request=<request_id>
Ranger:        Bot <ranger_id>
Serviced bin:  <bin_id>  is_msio=<>  is_virtual_bin_used=<>
Orders:        <order_id>  promoted_to_physical_bin=<>
Verdict:       VALID assignment / INVALID — assignment should have been withheld
```
