Investigate an MVTS issue starting from an Order ID.

**Arguments:** `<order_id> [namespace]`

Example: `/debug-order 961158 qa3-cluster-devrelayonm-greymatter`

---

Parse `$ARGUMENTS` to extract:
- `order_id` — first token
- `namespace` — second token if provided; otherwise ask the user before proceeding

Then run all steps below in sequence using `kubectl exec`. Carry extracted values forward into each subsequent step automatically. At the end, print a structured summary of all findings.

## Step 1 — Find the log file

Search `scheduler.log` first, then archived logs descending. Stop at the first file that matches.

```bash
kubectl exec mvts-0 -n <namespace> -c ml-engine-mvts -- sh -c \
  'grep -l "<order_id>" /app/data/logs/scheduler.log 2>/dev/null; zgrep -l "<order_id>" /app/data/logs/*.gz 2>/dev/null | sort -r | head -1'
```

Extract: `log_file`

## Step 2 — Find the task for this order

Message lines are 3–4 MB — use a Python script. Write the script locally, copy it into the pod, run it:

```bash
kubectl cp /tmp/parse_order.py mvts-0:/tmp/parse_order.py -n <namespace> -c ml-engine-mvts
kubectl exec mvts-0 -n <namespace> -c ml-engine-mvts -- python3 /tmp/parse_order.py
```

**Script to write to `/tmp/parse_order.py`:**
```python
import re, json, gzip, sys

logfile = "/app/data/logs/<log_file>"
order_id = "<order_id>"
opener = gzip.open if logfile.endswith(".gz") else open

with opener(logfile, "rt") as f:
    for line in f:
        if order_id not in line:
            continue
        m = re.search(r"Message: (\{.*)", line)
        if not m:
            continue
        data = json.loads(m.group(1))
        for task in data.get("task_list", []):
            for order in task.get("serviced_orders", []):
                if str(order.get("order_id")) == order_id:
                    print("task_key:", task.get("task_key"))
                    print("task_type:", task.get("task_type"))
                    print("task_subtype:", task.get("task_subtype"))
                    print("transport_entity_id:", task.get("transport_entity_id"))
                    print("order:", json.dumps(order, indent=2))
                    sys.exit(0)
```

Extract: `task_key`, `task_type`, `task_subtype`, `transport_entity_id`, order details

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

Use a Python script if the line is too large to parse inline. Extract `serviced_bins` and `serviced_orders` for the specific `task_key`.

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
Order ID:           <order_id>
Log file:           <log_file>
Task key:           <task_key>
Task type:          <task_type> → <task_subtype>
Transport entity:   <transport_entity_id>
First assign:       <timestamp>  request=<request_id>
Ranger:             Bot <ranger_id>
Serviced bin:       <bin_id>  is_msio=<>  is_virtual_bin_used=<>
promoted_to_physical_bin: <>
Verdict:            VALID assignment / INVALID — assignment should have been withheld
```
