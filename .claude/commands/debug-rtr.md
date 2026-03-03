Debug an MVTS issue starting from an RTR ID (also called ETR ID externally).

## Usage
Provide an RTR ID and optionally the log file if already known.

## Steps

### 1. Find which log file contains the RTR ID
Search `scheduler.log` first, then archived logs descending:
```bash
kubectl exec mvts-0 -n $NAMESPACE -c ml-engine-mvts -- sh -c \
  'grep -l "<rtr_id>" /app/data/logs/scheduler.log 2>/dev/null; zgrep -l "<rtr_id>" /app/data/logs/*.gz'
```
Stop at the first file that matches.

### 2. Find the task associated with the RTR ID
```bash
kubectl exec mvts-0 -n $NAMESPACE -c ml-engine-mvts -- sh -c \
  'zgrep "Message:.*<rtr_id>" /app/data/logs/<file>.log.gz | head -1'
```
Extract: `task_key`, `task_type`, `task_subtype`, `status`, `serviced_bins`, `serviced_orders`

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

> **Note:** Message lines can be 3–4 MB. Use Python scripts (copied via `kubectl cp`) for reliable JSON parsing rather than raw `zgrep -o`.

### 5. Check if a bin is virtual for a PPS
From the Message JSON, look up `pps_list[id=<pps_id>].bin_details[bin_id=<bin_id>]`:
- `is_virtual_bin_used: true/false` — whether the bin slot is virtual
- `is_msio: true/false` — whether it's a Multi-Slot Input/Output bin
