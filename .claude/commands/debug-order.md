Debug an MVTS issue starting from an Order ID.

## Usage
Provide an order ID and optionally the log file if already known.

## Steps

### 1. Find which log file contains the order ID
Search `scheduler.log` first, then archived logs in descending order:
```bash
kubectl exec mvts-0 -n $NAMESPACE -c ml-engine-mvts -- sh -c \
  'grep -l "<order_id>" /app/data/logs/scheduler.log 2>/dev/null'
# then descending archives:
kubectl exec mvts-0 -n $NAMESPACE -c ml-engine-mvts -- sh -c \
  'zgrep -l "<order_id>" /app/data/logs/scheduler.YYYY-MM-DD.N.log.gz'
```
Stop at the first file that matches.

### 2. Find the task associated with the order ID
Message lines are 3–4 MB — use a Python script copied into the pod:
```bash
kubectl cp parse_order.py mvts-0:/tmp/parse_order.py -n $NAMESPACE -c ml-engine-mvts
kubectl exec mvts-0 -n $NAMESPACE -c ml-engine-mvts -- python3 /tmp/parse_order.py
```

**`parse_order.py` template:**
```python
import re, json, gzip

logfile = "/app/data/logs/<file>.log.gz"
order_id = "<order_id>"

with gzip.open(logfile, "rt") as f:
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
                    print("transport_entity_id:", task.get("transport_entity_id"))
                    print("order:", order)
```
Extract: `task_key`, `task_type`, `transport_entity_id`

### 3. Find when the task was first assigned to an HTM bot
```bash
kubectl exec mvts-0 -n $NAMESPACE -c ml-engine-mvts -- sh -c \
  'zgrep "Output:.*<task_key>" /app/data/logs/<file>.log.gz | head -1'
```
Extracts: `request_id`, `timestamp` of first assignment, assigned `ranger_id`

### 4. Get the input Message for that request_id
```bash
kubectl exec mvts-0 -n $NAMESPACE -c ml-engine-mvts -- sh -c \
  'zgrep "Message:.*<request_id>" /app/data/logs/<file>.log.gz | head -1'
```
Parse JSON to extract `serviced_bins` and `serviced_orders` for the specific `task_key`.

### 5. Check if a bin is virtual for a PPS
From the Message JSON, look up `pps_list[id=<pps_id>].bin_details[bin_id=<bin_id>]`:
- `is_virtual_bin_used: true/false` — whether the bin slot is virtual
- `is_msio: true/false` — whether it's a Multi-Slot Input/Output bin
