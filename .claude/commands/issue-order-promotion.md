Reference details for Issue 1: Order Promotion Debugging

## Context
**Setup:** `qa3-devrelayonm`
- Cluster: `gke_greymatter-qa_us-east1_qa3-cluster`
- Namespace: `qa3-cluster-devrelayonm-greymatter`

## Identifiers
- **RTR ID:** `fcff337d-9811-4d11-8eb6-c2623548693d`
- **Found in:** `scheduler.2026-02-24.2.log.gz`
- **Task Key:** `3b213c11-9d24-4d1a-98c6-de3ae35029a3`
- **Task Type:** `tote_task` → `storable_to_pps`

## Assignment Details
- **First Assignment:** `2026-02-24T09:36:54.336Z` (request `UNj7xcmdSjy2bOoV91FNHg==`)
- **Assigned Ranger:** Bot 117, heading to PPS 7 at `(70, 1)`
- **Serviced Bin:** `7-1` (1 item, 4.536 kg)
- **Order:** `961158`, priority `MEDIUM`, not critical

## Bin Configuration
**PPS 7 / Bin 7-1:**
- `is_virtual_bin_used: false` (physical bin)
- `is_msio: true` (Multi-Slot Input/Output)

## Finding
Bin `7-1` is a **physical MSIO bin**. The task references it as the service bin but `promoted_to_physical_bin: false` for the order — mismatch between task bin expectation and PPS 7's bin configuration.

Per the HTM Assignment Rule: if all orders are on MSIO/virtual bins and none are promoted, the planner must withhold assignment. In this case the order was not promoted yet the task was assigned.

## Status
- [ ] Open
- [x] Root cause identified

## Remediation
- Planner must check `promoted_to_physical_bin` on all orders in `serviced_orders` before assigning a tote to an HTM bot.
- If all orders are on MSIO bins and none have `promoted_to_physical_bin: true`, skip HTM assignment for that task until promotion occurs.
- Related rule: see "HTM Assignment Rule for MSIO / Virtual Bins" in `CLAUDE.md`.
