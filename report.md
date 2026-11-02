# Beaver's Choice Paper Company – Multi-Agent Quoting & Inventory System

Reflection report · framework: **smolagents** · database: SQLite (starter helpers) · agents: **5** (1 orchestrator + 4 workers)

## 1. What was reviewed and what changed

The submitted solution was reviewed against project requirements. The review found that the original `project_starter.py` did not meet several requirements, so it was reworked.

| # | Finding in the original submission | Requirement affected | Fix in this version |
|---|---|---|---|
| 1 | Script **crashed on import** with the real `smolagents` package (tool docstrings had no per-argument descriptions). The shipped `test_results.csv` was produced by a silent fallback that bypassed smolagents. | Framework use; "all tests pass" | All tools have valid Google-style docstrings; smolagents is imported unconditionally. |
| 2 | "Agents" were plain Python classes; no smolagents `Agent` object existed, and the orchestrator was not an agent. | Implementation matches diagram, smolagents used | Five real `ToolCallingAgent`s; the orchestrator holds the four workers as `managed_agents`. |
| 3 | The item parser **silently dropped items** (e.g. request 10 lost 300 cardstock, request 11 lost 300 matte, request 8 lost 3,000 recycled paper) yet reported "ORDER FULFILLED". Customers were told they were fully served. | Accuracy, transparency | New parser; every requested line is either supplied or listed with a reason. |
| 4 | Rejection reasons were hard-coded to the sample (`balloons`, `tickets`, `cardboard`); `envelopes` was declared unsupported although it is in the catalog. | Robustness | Generic catalog resolution; unsupported items are detected, not pre-listed. |
| 5 | No reorder decision after a sale; items first stocked via a supplier order never entered the `inventory` table, so reported inventory value was wrong. | "Decide when to reorder"; financial accuracy | Reorder-point logic plus `ensure_inventory_record`. |
| 6 | Restocks were bought at full catalog price and sold at or below it (negative margin). | Profitability | Supplier cost = 60% of catalog price (documented assumption). |
| 7 | Quote agent only *counted* history matches; history never influenced price. | "Consider historical quote data" | Volume tier is blended with the median discount found in matching historical quotes. |
| 8 | Zip had no real report (only the starter README) and no `.mmd` file; the diagram did not show how tool purposes map to helpers in every case. | Documentation, diagram | This report, `multi_agent_workflow.mmd` + PNG, new README. |

## 2. Architecture and decision process

**Why five agents with this split.** The brief names three business capabilities (inventory, quoting, sales) and recommends an orchestrator. A separate **finance_agent** was added because spending decisions (cash, reserve) are a different responsibility from stock decisions and from booking sales; it also owns the otherwise orphaned `get_cash_balance` and `generate_financial_report` helpers. Five is the permitted maximum; each agent has non-overlapping duties.

| Agent | Responsibility | Tools → starter helper |
|---|---|---|
| `orchestrator_agent` | Parse the inquiry (items, quantities, deadline, event), register it, delegate in order, stop early on rejection, return the customer reply | `register_order_tool`, `compose_customer_response_tool` (ledger only) |
| `inventory_agent` | Map wording to exact catalog names, compare stock with demand, check supplier arrival against the deadline, decide reorders | `inventory_snapshot_tool` → `get_all_inventory`; `stock_check_tool` → `get_stock_level`; `supplier_lead_time_tool` → `get_supplier_delivery_date`; `assess_order_inventory_tool` → all three |
| `quote_agent` | Price accepted lines; bulk discount informed by history | `quote_history_tool`, `build_quote_tool` → `search_quote_history` |
| `finance_agent` | Check cash and financial health; approve or trim supplier purchases | `cash_balance_tool`, `authorize_purchases_tool` → `get_cash_balance`; `financial_report_tool` → `generate_financial_report` |
| `sales_agent` | Book supplier purchases and customer sales | `fulfill_order_tool` → `create_transaction` |

**Flow.** Customer → orchestrator → inventory → quote → finance → sales → orchestrator → customer. A line the inventory agent cannot supply (not in catalog, or supplier cannot deliver by the deadline) is rejected with a reason; if no line survives, the flow ends at the orchestrator. The finance agent can reject the order if required purchases are unaffordable.

**Key design decisions.**
- *Business rules live in plain functions; tools are thin wrappers.* Arithmetic and database writes are deterministic and auditable regardless of what an LLM does.
- *Shared order ledger.* Workers exchange an `order_key` rather than re-typing data, which removes transcription errors in LLM-driven runs. Fulfilment is idempotent (an order cannot be booked twice).
- *Partial fulfilment (switchable).* Supplyable lines are shipped and the rest are explained (`ALLOW_PARTIAL_FULFILLMENT`). Setting it to `False` gives all-or-nothing behaviour (10 fulfilled / 10 rejected on the sample).
- *Pricing.* Volume tiers: ≥500 units 5%, ≥1,000 units 7%, ≥5,000 units 10%, capped at 15%. For orders that qualify for a tier, the discount is the average of the tier and the median discount found in matching historical quotes (history mostly shows 10%). Small orders get no discount.
- *Reordering.* If a sale leaves an item below its minimum stock level, it is topped up to 2× that level. These "safety" purchases are dropped first if cash is tight; a cash reserve of $1,000 is never spent.
- *Two ways to drive the agents.* The agents are ordinary smolagents agents. By default they run on `RuleBasedModel`, a deterministic stand-in for an LLM that emits real tool calls, so the evaluation is reproducible and offline. `USE_LLM=1` with `UDACITY_OPENAI_API_KEY` swaps in `OpenAIServerModel` against the Vocareum endpoint for live LLM orchestration.

## 3. Evaluation results (`test_results.csv`)

All 20 requests in `quote_requests_sample.csv` were processed. Result: **10 fulfilled, 9 partially fulfilled, 1 not fulfilled**. Cash moved from $45,059.70 to $46,804.87; inventory value ended at $4,404.55.

| Request | Date | Outcome | Cash change ($) |
|---|---|---|---|
| 1 | 2025-04-01 | FULFILLED | +100.00 |
| 2 | 2025-04-03 | PARTIAL | +9.50 |
| 3 | 2025-04-04 | PARTIAL | +151.26 |
| 4 | 2025-04-05 | FULFILLED | +55.35 |
| 5 | 2025-04-05 | FULFILLED | +48.53 |
| 6 | 2025-04-06 | FULFILLED | +15.85 |
| 7 | 2025-04-07 | FULFILLED | +443.76 |
| 8 | 2025-04-07 | FULFILLED | +200.52 |
| 9 | 2025-04-07 | FULFILLED | +19.00 |
| 13 | 2025-04-08 | PARTIAL | +12.00 |
| 12 | 2025-04-08 | FULFILLED | +14.50 |
| 10 | 2025-04-08 | FULFILLED | +35.12 |
| 11 | 2025-04-08 | FULFILLED | +50.05 |
| 14 | 2025-04-09 | PARTIAL | +24.38 |
| 15 | 2025-04-12 | NOT FULFILLED | +0.00 |
| 16 | 2025-04-13 | PARTIAL | +105.60 |
| 17 | 2025-04-14 | PARTIAL | +47.50 |
| 18 | 2025-04-14 | PARTIAL | +8.00 |
| 19 | 2025-04-15 | PARTIAL | +47.25 |
| 20 | 2025-04-17 | PARTIAL | +357.00 |

Built-in checks (printed at the end of every run) all pass:
- all 20 requests evaluated; ≥3 requests fulfilled (19); ≥1 rejected with a clear reason (request 15); ≥3 requests changed cash (19)
- every one of the seven required helpers is called by at least one tool (measured at run time, not asserted by hand)
- no customer reply contains cash, margin or error text

Additional integrity checks run during review: booked cash reconciles exactly with the transactions table; stock never goes negative on any date; every stocked item appears in the inventory table; the sum of quoted totals equals booked sales ($4,142.92); two consecutive runs give identical output; a cash-starved scenario is rejected with a customer-safe reason.

**Strengths.**
- *Honest, explainable replies.* Each reply lists items with unit prices, the discount and why it applies, an estimated delivery date, and a reason for every line not supplied (e.g. request 15: supplier cannot deliver 10,000 sheets by April 15 – earliest April 19 – and cardboard is not in the catalog).
- *Impossible constraints are recognised.* Request 15 is rejected in full; requests 2, 3, 14 and 20 (and five others) are shipped partially with the impossible lines explained.
- *Correct books.* Every sale and purchase goes through `create_transaction`; cash and inventory value update after every request.
- *Margin protection.* Discounts are capped, and replenishment is costed at supplier cost.
- *Modularity.* Parser, inventory, pricing, finance and fulfilment are independently testable functions.

**Areas for improvement (observed).**
- Nine requests were only partly fulfilled, mostly because the 7-day supplier lead time for orders over 1,000 units collides with 5–10-day deadlines. Several customers would receive a fraction of what they asked for.
- Size and variant words (A3, A5, "recycled", "biodegradable") are not catalog attributes. A3 glossy/matte paper is matched to the size-agnostic catalog item with a visible note, but "A3 paper" alone is rejected; this is a judgement call.
- The default offline run exercises the agent graph deterministically; the live-LLM path was wired and schema-validated but **not run against the Vocareum API in this review**.

## 4. Suggested further improvements

1. **Split-delivery and substitution offers.** When a line misses the deadline, offer an in-stock partial shipment now and the remainder on the supplier arrival date, or suggest a substitute catalog item (e.g. Standard copy paper for A4). This turns most of the nine partial outcomes into complete sales.
2. **Demand-driven reorder points and supplier lead-time buffers.** Use `quote_requests.csv` (about 400 historical inquiries) to forecast demand per item and set reorder points from lead time × expected demand, so popular items such as cardstock, glossy and A4 paper are already stocked before the large orders arrive.
3. **Customer clarification loop.** Let the orchestrator ask a follow-up question when an item is ambiguous (size, finish, "reams" vs sheets) instead of assuming, and add a human-approval gate for orders above a value threshold.
4. **Price optimisation.** Replace the fixed tier table with a margin-aware model trained on historical quotes (price vs accepted/rejected), so discounts are larger only where margin allows.

## 5. Assumptions

- Supplier cost is 60% of catalog price; the starter has no cost field.
- Sales and the matching supplier purchases are booked on the request date; supplier arrival is checked against the deadline but not modelled as a delayed inventory event.
- "Reams", "packets" and "rolls" are treated one-to-one with catalog units, as the historical quotes do.
- Customer replies deliberately omit stock levels, cash and margin; the job title is not repeated.

## 6. How to run

```
pip install -r requirements.txt
python project_starter.py            # deterministic run, writes test_results.csv
USE_LLM=1 python project_starter.py  # live LLM orchestration (needs UDACITY_OPENAI_API_KEY in .env)
```

## 11. Live-LLM run review and hardening

A live run (`USE_LLM=1`) exposed weaknesses that the offline run could not: the model skipped or reordered workers, invented order keys, produced malformed item JSON, hallucinated a $0.00 cash balance, and wrote its own customer replies that leaked internal terms (cash, order status). Replies and the database sometimes disagreed (e.g. request 1 was reported "not fulfilled" although the sale was booked).

Fixes: (1) the customer reply is always rendered from the order ledger, never from model free text; (2) any workflow stage the agents leave undone is completed by the same business functions the tools call; (3) tools ignore invented order keys and take request date, deadline and event from the trusted request text; (4) item JSON is repaired or replaced by the rule parser when unusable; (5) the live model runs at temperature 0. Verified with a deliberately faulty model: all 20 replies and the final cash ($46,804.87) are identical to the clean run.

### 11.2 Second live run: results

The second `USE_LLM=1` run met every requirement and passed all built-in checks that existed at the time.

| Requirement | Evidence |
|---|---|
| At least 3 orders fulfilled | 10 fully confirmed (requests 1, 4, 5, 6, 7, 8, 9, 10, 11, 12) and 9 partially confirmed |
| At least 1 order rejected with a clear reason | Request 15 rejected in full: 10,000 A4 and 5,000 A3-colored sheets cannot arrive by April 15 (earliest April 19); cardboard not in catalog; no charge made |
| Impossible constraints recognised | Request 14 (earliest April 16 vs April 15), request 19 (April 22 vs April 20), request 20 (tickets not in catalog), each reported per line |
| At least 3 requests change cash | 19 of 20 |
| All seven helpers used | create_transaction 121, get_all_inventory 42, get_stock_level 229, get_supplier_delivery_date 153, get_cash_balance 70, generate_financial_report 38, search_quote_history 165 |
| Transparent, no internal data | No cash, status or order-key text in any reply |

Final cash $47,095.30, inventory value $4,404.55. All 20 outcomes and quote totals match the offline run.

### 11.3 Finding: duplicate booking in the second live run, and its fix

Final cash was $290.43 higher than offline. Requests 8, 11 and 14 differed, by exactly that amount. The live model asked the inventory agent to re-check an order after it was booked; the re-run reset the order status, so the order was priced, authorized and booked a second time. Replies were correct (they come from the ledger) but the database held two sales for those orders.

Reproduced with a model that re-runs the inventory step after completion (sales $8,238.34 vs the correct $4,142.92), then fixed: (1) each step runs only from its own status; (2) a `booked` flag makes an order bookable once; (3) a seventh check compares booked sales with quoted totals. After the fix all test models give identical replies and final cash of $46,804.87, and all seven checks pass. A third live run with the updated script confirmed this (section 11.4).

### 11.4 Third live run: confirmation

The third live run with the fixed script matched the offline results exactly: final cash $46,804.87, final inventory value $4,404.55, and all seven built-in checks passing, including the booking-integrity check.

| Check | Third live run | Offline run |
|---|---|---|
| Outcomes (fulfilled / partial / not fulfilled) | 10 / 9 / 1 | 10 / 9 / 1 |
| Cash change after each of the 20 requests | identical | identical |
| Final cash | $46,804.87 | $46,804.87 |
| Booking-integrity check | PASS | PASS |
| `create_transaction` calls | 100 | 100 |
| Replies containing cash, status or order-key text | none | none |

The number of database writes equals the offline run (100), so no order was booked twice. Read-only helper calls differ (for example 112 supplier-date lookups against 39) because the live model explores extra tool calls; these change no data. The terminal output starts at request 1 and does not show the line naming the model driving the agents, so the live mode is taken from the run description rather than the output itself.

Request 15 remains the required rejection: turned down in full because 10,000 A4 and 5,000 A3-colored sheets cannot arrive by April 15 (earliest April 19) and cardboard is not in the catalog, with the reason stated to the customer and no charge.
