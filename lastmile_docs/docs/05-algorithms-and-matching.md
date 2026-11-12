# Algorithms and Matching Design

## Configurable pilot defaults
| Parameter | Initial value |
|---|---:|
| Pickup geofence | 150 m (site-test) |
| Matching window | 60 s |
| Maximum shared wait | 120 s from original booking creation |
| Offer timeout | 20 s |
| Group min/max | 2 / 4 passengers |
| Maximum detour | 25% per passenger |
| GPS updates | 3–5 s |
| PostgreSQL GPS snapshots | 15–30 s + key events |
All are configurable; max total trip duration is pending.

## Zone-level batching
One batch per pickup zone (initially PDEU). Acquire one zone-level lease to prevent overlapping batch processors. Batch membership is cut off at a timestamp; late bookings go to next batch. Persist batch and membership records. Claim each booking so it cannot belong to two active offers. New batch may be triggered early after rejection, but original booking deadline is not reset.

## Eligibility
A booking is eligible only if requested type is SHARED, pending/eligible, not cancelled/expired, within original max-wait deadline, not already claimed, belongs to same pickup zone, and not paused pending reconfirmation after repeated timeout. PRIVATE never enters shared matching.

## Greedy candidate grouping
```text
eligible = loadEligibleBookings(batch)
while eligible not empty:
    seed = chooseOldestOrPriority(eligible)
    candidates = shortlistCompatibleNearby(seed, eligible)
    best = none
    for candidate combination containing seed, size 2..min(4, capacity):
        matrix = OSRM_TABLE(commonPickup + destinations)
        order = enumerateAndScoreDropoffPermutations(matrix)
        metrics = computeDetourAndDuration(order)
        if all constraints pass:
            retain best feasible group
    if best exists:
        atomically claim bookings
        persist group + offer + response rows
        remove those bookings from eligible
    else:
        keep eligible until deadline or fallback policy
```
Candidate shortlist size is pending; bound it to avoid combinatorial growth.

Trade-off: greedy is simple/explainable and appropriate for one low-volume zone, but not globally optimal.

## Route optimization
For up to four destinations:
1. Call OSRM Table API once for pairwise travel-time/distance matrix.
2. Enumerate all drop-off orders locally.
3. Four passengers means `4! = 24` orders.
4. Calculate passenger shared time for each order.
5. Reject infeasible orders.
6. Select lowest-scoring feasible order.
7. Call OSRM Route API once for final route geometry.

Do not call OSRM once per permutation.

Proposed objective:
`score = total_vehicle_travel_time + λ * sum(passenger_detour_penalties)`
Exact λ and normalization are pending. Simpler MVP baseline: minimum total route time, detour as tie-breaker; compare in tests.

## Detour
`detour_i_percent = ((shared_time_i - direct_time_i) / direct_time_i) * 100`
Direct time is pickup→passenger destination estimate; shared time is pickup through chosen stop order until passenger i is dropped. Each passenger must be within initial 25% max detour. Also enforce max trip duration (threshold pending). Do not silently substitute straight-line distance when OSRM fails.

## Fare
MVP formula:
`F_i = F_shared_total * solo_fare_i / sum(solo_fare_j)`
This allocates group total proportionally to individual solo estimates. Base fare/tariff, group total derivation, minimum fare, rounding/remainder rule and currency policy remain pending. Advanced future method discussed: marginal cost `m_i = C(S) - C(S without i)`; not MVP due to complexity.

## Driver matching
Filter to approved, non-suspended, online/available drivers with active vehicle capacity sufficient and no active assignment. Rank by estimated ETA to pickup. Send offer to best; on reject/timeout try next until dispatch deadline. Use conditional atomic update AVAILABLE→RESERVED and unique active assignment index. Confirm ride only after driver accepts. Advanced bipartite matching is future work.

## Offer policy
- All passengers accept same offer version.
- Reject group != cancel request.
- Timeout differs from rejection.
- First timeout: temporarily eligible; second consecutive timeout: pause/reconfirm; expire if no reconfirmation.
- Rejection thresholds proposed: first tries different candidates; second asks whether to continue; third stops auto matching and offers private/cancel. Confirm before coding.
- Never repeat exact failed group immediately.
- Any membership/route/fare change requires new offer and fresh acceptance.

## Cancellation
If passenger cancels after assignment but before trip start: record cancellation, remove passenger, recompute remaining route/detour/fare; preserve only if min group, capacity, route, detour and duration still pass. Otherwise rematch/cancel remaining according to preference. Late-cancel direction: strike/cooldown, not money. Proposed: first warning; second in 30 days = 15-minute shared cooldown; third = 60-minute cooldown; expire strikes after 30 days; exempt driver/platform failures; admin can reverse. Confirm thresholds before enforcement.
