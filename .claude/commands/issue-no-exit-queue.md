Reference details for Issue 2: No Exit Queue at Conveyor (On-Demand Bot)

## Context
**Setup:** `stg001-walmartpotstg`
- Cluster: `gke_gm-prod-sams-atl_us-east1_stg001-cluster`
- Namespace: `stg001-cluster-walmartpotstg-greymatter`

## Problem
Totes waiting at PPS 5 conveyor exit point `(46, 62)` with no bot to pick them up. MVTS returning empty `rangerIDList` for on-demand bot request.

**GMC Timestamp:** `2026-02-24 19:43:01.593`

## Grep Patterns
```bash
# Find on-demand bot API calls
zgrep "OnDemandBotRequiredAtPPSInputMessage" /app/data/logs/<file>.log.gz

# Find bots available within 5s of start_time
# Parse Message JSON: ranger_list[].available_at_time <= start_time + 5000
```

## Timeline of Events
```
19:41:25  MSG IN   Problem statement [zBwEHH4uT1en...]
                   • Bot 102: only available bot, ready @ (60,71)
                   • PPS 5: 8 totes going_to_conveyor_exit, assigned_ranger_id=null

19:41:55  MSG IN   Problem statement [9LW72woISuur...]
                   • Same state persists

19:42:26  MSG IN   Problem statement [ItehZmzAT7qk...]  ← last before issue
                   • Bot 102: available_at_time = start_time - 145ms
                   • PPS 5: 8 totes unassigned at conveyor exit
                   • Planner assigns bot 117 → PPS 5 (relay_to_pps, HA20001805)
                   • Bot 102: NOT assigned by planner

19:42:36  API IN   OnDemandBotRequiredAtPPS { ppsID:5, blacklisted:[] }
          API OUT  ✅ rangerIDList: [102]  — bot 102 dispatched to PPS 5

          ⏳ ~25s — bot 102 now busy, no other bots free

19:43:01  API IN   OnDemandBotRequiredAtPPS { ppsID:5, blacklisted:[] }
          API OUT  ❌ rangerIDList: []     ← PROBLEM (GMC timestamp)

19:43:25  MSG IN   Problem statement [qjQelGd8TLq7...]
                   • New planning cycle begins
```

## Root Cause
Only **1 bot (102)** was available fleet-wide for `HTM_HAI_K50_L_E1` version. The first on-demand call at `19:42:36` dispatched it to PPS 5. When GMC called again 25s later, bot 102 was already occupied and no other bots were free → empty list → totes stuck at conveyor exit.

## Status
- [ ] Open
- [x] Root cause identified

## Remediation
- Short-term: ensure minimum bot fleet size per `HTM_HAI_K50_L_E1` version so at least 2 bots are available for on-demand dispatch.
- Long-term: MVTS should include a warning in the API response when `rangerIDList` is empty due to fleet exhaustion (vs. other reasons), so GMC can surface it as a fleet capacity alert rather than a silent failure.
- For investigation, use `/debug-on-demand-bot <pps_id> <timestamp> <namespace>` to reproduce the timeline.
