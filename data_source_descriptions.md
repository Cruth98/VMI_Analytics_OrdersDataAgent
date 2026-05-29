# Data Source Descriptions — Oracle Orders Agent

## vw_Orders_Base
Primary Oracle orders table. One deduplicated row per order line, open 
and shipped. Safe for aggregation and counting. Fulfill lines are 
deduplicated via ROW_NUMBER() — only the most recent fulfill line per 
order line is retained, preventing inflated quantities on rescheduled 
or amended orders. Primarily Open State; select Bailment accounts 
ordering via Oracle may appear.

## vw_Orders
Adds City, State, and pricing strategy fields not available in 
vw_Orders_Base. Not deduplicated — uses a raw LEFT JOIN on fulfill 
lines and will produce duplicate rows when multiple fulfill line 
records exist per order line. Display only. Do not aggregate 
quantities or count orders from this view.

## vw_OpenOrdersWithNPHold
Open orders on active Non-Pricing (NP) holds only. NP holds indicate 
logistics or fulfillment issues requiring planner intervention — the 
highest-priority exception type for the VMI team. Use for NP hold 
triage and exception queues. Do not aggregate quantities.

## vw_OpenOrdersWithHold
Open orders with any active hold, regardless of hold type. Use for 
broad hold triage when the hold type is unknown or not specified. 
Same schema as vw_OpenOrdersWithNPHold. Do not aggregate quantities.
