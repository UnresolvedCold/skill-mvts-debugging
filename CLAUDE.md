# MVTS Debugging Guide

## Setup

- **VM IP:** `192.168.9.237`
- **User:** `shubham.kumar`
- **SSH:** Key-based auth via existing `~/.ssh` config

## Cluster & Namespace

- **Cluster:** `gke_greymatter-qa_us-east1_qa3-cluster`
- **Namespace:** `qa3-cluster-devrelayonm-greymatter`
- **Pod:** `mvts-0` (container: `ml-engine-mvts`)

Switch to cluster and namespace:
```bash
kubectl config use-context gke_greymatter-qa_us-east1_qa3-cluster
kubectl config set-context --current --namespace=qa3-cluster-devrelayonm-greymatter
```

## Log Location

Logs are inside the pod at `/app/data/logs/`:
- **Active log:** `scheduler.log`
- **Archived logs:** `scheduler.YYYY-MM-DD.N.log.gz` (multiple rotations per day)

Exec into pod:
```bash
kubectl exec mvts-0 -n qa3-cluster-devrelayonm-greymatter -c ml-engine-mvts -- sh -c '<command>'
```

## Terminology

| External Term | MVTS Log Term |
|---|---|
| ETR ID | RTR ID (`rtr_ids`) |

## Key Behavioral Notes

### Assignment Output
- `serviced_bins: []` and `serviced_orders: []` in Output assignments is **expected** — the output does not carry bin/order commitments back.
- Bot assignment is reflected only in the **Output** line, NOT in subsequent Message inputs (Messages always show `assigned_ranger_id: null` for unassigned/queued tasks).

### HTM Assignment Rule for MSIO / Virtual Bins
> A tote **can** be assigned to an HTM only if at least one order in `serviced_orders` has `promoted_to_physical_bin: true`, OR at least one order's bin is not MSIO and not virtual.
> If ALL orders are on MSIO/virtual bins and none are promoted → planner must withhold assignment.

### Task Key Transitions
- The same tote can be served by multiple sequential tasks with different `task_key`s (e.g., first task completes, new task created for added orders).
- When diagnosing a tote issue, always check if a **prior task** existed for the same `transport_entity_id` — the prior task's assignment may explain why the tote was already in motion.
- After a task is assigned (Output), subsequent Messages may not include it in `task_list` (removed from planning queue once assigned).

---

## Skills

Skills live in `.claude/commands/` as individual `.md` files. **Always create a new `.md` file for each new skill/investigation, commit, and push to origin.**

### Utilities

| Skill | Arguments | Description |
|---|---|---|
| `/setup-context` | `<env-name>` | Switch kubectl context + namespace by env name |
| `/find-logs` | `[namespace]` | List all log files in the pod, newest first |

### Debug Workflows

| Skill | Arguments | Description |
|---|---|---|
| `/debug-rtr` | `<rtr_id> [namespace]` | Full automated investigation from an RTR ID |
| `/debug-order` | `<order_id> [namespace]` | Full automated investigation from an Order ID |
| `/debug-on-demand-bot` | `<pps_id> <gmc_timestamp> [namespace]` | Investigate empty rangerIDList at a PPS conveyor exit |

### Issue Records

| Skill | Description |
|---|---|
| `/issue-order-promotion` | Issue 1: Order promotion mismatch — MSIO bin assigned without promotion (2026-02-24) |
| `/issue-no-exit-queue` | Issue 2: Empty rangerIDList at PPS 5 conveyor exit — fleet exhaustion (2026-02-24) |
