# Beaver's Choice Paper Company – Multi-Agent System: Reflective Report

Framework: smolagents · Database: SQLite (starter helpers) · Agents: 5 (1 orchestrator + 4 workers) · Deliverables: `project_starter.py`, `multi_agent_workflow.png` / `.mmd`, `test_results.csv`, this report

## 1. Executive summary

This report explains the multi-agent system built for the Beaver's Choice Paper Company, how it was evaluated on all 20 requests in `quote_requests_sample.csv`, and how each project requirement is met. The system answers inventory questions, decides when to reorder, prices quotes with bulk discounts informed by historical quotes, and finalizes sales by writing to the database.

On the sample set the system fully fulfilled 10 requests, partially fulfilled 9, and rejected 1, with a clear customer-facing reason for every line it could not supply. Cash moved from $45,059.70 to $46,804.87, and all built-in requirement checks pass. Runs driven by a live LLM produced the same outcomes, and the final live run matched the offline results exactly, including final cash of $46,804.87 (section 11).

The solution is a rework of an earlier submission. That version crashed under the real smolagents package, used no smolagents agents, and silently dropped requested items while reporting success. Section 9 lists what was wrong and how it was fixed.

## 2. System architecture and decision process

### 2.1 Why five agents

The brief requires three business capabilities (inventory, quoting, sales finalization) and recommends an orchestrator. A fourth worker, the finance agent, was added for two reasons. Spending decisions (how much cash may be committed to supplier purchases) are a different responsibility from stock decisions and from booking sales. It also gives a natural owner to the helpers `get_cash_balance` and `generate_financial_report`, which the instructions require an agent to use. Five is the permitted maximum, and each agent's duties do not overlap.

### 2.2 Agent responsibilities

| Agent | Responsibility | Tools and the starter helper each uses |
| --- | --- | --- |
| `orchestrator_agent` | Parses the inquiry (items, quantities, deadline, event), registers it, delegates to workers in order, stops early on rejection, returns the customer reply | `register_order_tool`, `compose_customer_response_tool` (shared order ledger only) |
| `inventory_agent` | Maps customer wording to exact catalog names, compares stock with demand, checks supplier arrival against the deadline, decides reorders | `inventory_snapshot_tool` → `get_all_inventory`; `stock_check_tool` → `get_stock_level`; `supplier_lead_time_tool` → `get_supplier_delivery_date`; `assess_order_inventory_tool` → all three |
| `quote_agent` | Prices accepted lines and applies a bulk discount informed by history | `quote_history_tool`, `build_quote_tool` → `search_quote_history` |
| `finance_agent` | Checks cash and financial health, approves or trims supplier purchases | `cash_balance_tool` → `get_cash_balance`; `financial_report_tool` → `generate_financial_report`; `authorize_purchases_tool` → `get_cash_balance` |
| `sales_agent` | Books supplier purchases and customer sales | `fulfill_order_tool` → `create_transaction` |

### 2.3 Workflow

Customer inquiry → orchestrator → inventory\_agent → quote\_agent → finance\_agent → sales\_agent → orchestrator → customer reply.

1. The orchestrator registers the order and passes only an `order_key` to each worker.
2. The inventory agent resolves each line to a catalog item. It rejects lines that are not in the catalog or that the supplier cannot deliver by the deadline, and plans purchases for any shortage.
3. If no line survives, the flow returns to the orchestrator with a rejection. Otherwise the quote agent prices the accepted lines.
4. The finance agent checks cash. It can reject the order if required purchases are unaffordable.
5. The sales agent writes the `stock_orders` and `sales` transactions.
6. The orchestrator composes the customer reply.

The diagram in `multi_agent_workflow.png` (source in `multi_agent_workflow.mmd`) shows each agent, its tools, the purpose of each tool, the starter helper it calls, and the data flow to the SQLite database.

### 2.4 Key design decisions

- **Business rules live in plain functions; tools are thin wrappers.** Arithmetic and database writes are deterministic and auditable no matter what a language model does.
- **Shared order ledger.** Workers exchange an `order_key` instead of retyping data, which removes transcription errors in LLM-driven runs. Fulfilment is idempotent: an order cannot be booked twice.
- **Partial fulfilment.** Lines that can be supplied are shipped and the rest are explained. A constant, `ALLOW_PARTIAL_FULFILLMENT`, switches to all-or-nothing behaviour (10 fulfilled and 10 rejected on the sample).
- **Pricing.** Discount tiers are 5% from 500 units, 7% from 1,000 and 10% from 5,000, capped at 15%. For an order that qualifies for a tier, the discount is the average of the tier rate and the median discount found in matching historical quotes (history mostly shows 10%). Orders under 500 units get no discount.
- **Reordering.** If a sale leaves an item below its minimum stock level, it is topped up to twice that level. These safety purchases are dropped first when cash is tight, and a $1,000 cash reserve is never spent.
- **Supplier cost.** The starter has no cost field, so supplier cost is assumed to be 60% of catalog price. This keeps discounted sales profitable.
- **Two ways to drive the agents.** The agents are ordinary smolagents `ToolCallingAgent`s. By default they run on `RuleBasedModel`, a deterministic stand-in for an LLM that emits real tool calls, so the evaluation is reproducible and works offline. Setting `USE_LLM=1` with `UDACITY_OPENAI_API_KEY` swaps in `OpenAIServerModel` against the Vocareum endpoint.

## 3. Implementation notes

- **Parsing.** The orchestrator extracts the request date, deadline, event and line items. Quantities are found first and each product phrase runs up to the next quantity, so comma- or "and"-separated lists are handled correctly. Dates, percentages and sizes such as 24x36 are removed before quantities are detected.
- **Catalog resolution.** Customer wording is mapped to exact catalog names in three passes: specific aliases, exact catalog names, then generic aliases, with a fuzzy match as a last resort. Unmatched products produce a customer-safe reason.
- **Inventory.** Stock comes from `get_stock_level` and a full snapshot from `get_all_inventory`. Supplier arrival comes from `get_supplier_delivery_date` for the shortage quantity, and is compared with the customer's deadline.
- **First-time stocking.** An item bought from the supplier for the first time is registered in the `inventory` table, so `generate_financial_report` values it correctly.
- **Code quality.** snake\_case naming, docstrings on every function and tool, dataclasses for state, and the section layout: configuration, ledger and business logic, tools, models, agents, evaluation.

## 4. Evaluation method

The system processes every request in `quote_requests_sample.csv` in date order against a freshly initialised database, then writes `test_results.csv` (request id, date, cash balance, inventory value, response, outcome, cash change). At the end of each run the script prints seven checks (the six listed here plus a booking-integrity check, section 11):

1. All sample requests were evaluated.
2. At least 3 requests were fulfilled.
3. At least 1 request was rejected with a stated reason.
4. At least 3 requests changed the cash balance.
5. Each of the seven required helpers was called by an agent tool. This is measured at run time by wrapping the helpers, not asserted by hand.
6. No customer reply contains cash, margin or error text.

Additional checks were run during review:

- Booked cash reconciles exactly with the transactions table.
- Stock never goes negative on any date.
- Every stocked item appears in the inventory table.
- The sum of quoted totals equals booked sales ($4,142.92).
- Two consecutive runs give identical output.
- A cash-starved scenario is rejected with a customer-safe reason.
- Fulfilling the same order twice is blocked.

## 5. Evaluation results

Starting position: cash $45,059.70, inventory value $4,940.30. Final position: cash $46,804.87, inventory value $4,404.55. Totals: 10 fulfilled, 9 partial, 1 not fulfilled.

| Req | Date | Outcome | Quote | Notes |
| --- | --- | --- | --- | --- |
| 1 | 2025-04-01 | Fulfilled | $100.00 | All items in stock, no discount (under 500 units) |
| 2 | 2025-04-03 | Partial | $129.50 | Balloons not in catalog |
| 3 | 2025-04-04 | Partial | $468.00 | A3 paper not in catalog; 10,000 A4 arrives by 2025-04-11 |
| 4 | 2025-04-05 | Fulfilled | $80.94 | 7.5% discount |
| 5 | 2025-04-05 | Fulfilled | $123.53 | 8.5% discount |
| 6 | 2025-04-06 | Fulfilled | $70.45 | 8.5% discount |
| 7 | 2025-04-07 | Fulfilled | $622.20 | 4 product lines, 8.5% discount |
| 8 | 2025-04-07 | Fulfilled | $648.00 | 6,500 units, 10% discount |
| 9 | 2025-04-07 | Fulfilled | $32.50 | Size note on glossy paper |
| 10 | 2025-04-08 | Fulfilled | $134.12 | Both lines supplied (previously one was silently dropped) |
| 11 | 2025-04-08 | Fulfilled | $142.45 | Both lines supplied; size note |
| 12 | 2025-04-08 | Fulfilled | $48.10 | Three lines |
| 13 | 2025-04-08 | Partial | $30.00 | 500 A4 printing paper cannot arrive by 04-10 (earliest 04-12) |
| 14 | 2025-04-09 | Partial | $69.38 | Large lines cannot arrive by 04-15 (earliest 04-16) |
| 15 | 2025-04-12 | Not fulfilled | – | Deadline impossible for large quantities; cardboard not stocked |
| 16 | 2025-04-13 | Partial | $114.00 | 500 A4 printer paper cannot arrive in time |
| 17 | 2025-04-14 | Partial | $47.50 | Most lines miss the deadline |
| 18 | 2025-04-14 | Partial | $20.00 | Two large lines miss the deadline |
| 19 | 2025-04-15 | Partial | $137.25 | Glossy and matte lines miss the 04-20 deadline |
| 20 | 2025-04-17 | Partial | $1,125.00 | Tickets not in catalog; 7,000 units at 10% discount |

The cash balance changed after 19 of 20 requests (only the full rejection left it unchanged). The complete per-request replies are in `test_results.csv`.

### 5.1 Strengths

- **Explainable replies.** Each reply lists items with unit prices, the discount and why it applies, an estimated delivery date, and a reason for every line not supplied.
- **Impossible constraints are recognised.** Request 15 is rejected outright with specific reasons, and unsupplyable lines in the other orders are called out individually.
- **Honest completion.** No line is dropped silently. The customer always knows what is and is not covered.
- **Correct books.** Every sale and purchase goes through `create_transaction`, and the reported cash and inventory value update after every request.
- **Margin protection.** Discounts are capped and replenishment is costed at supplier cost.
- **Modularity.** Parsing, inventory, pricing, finance and fulfilment are separate testable functions.

### 5.2 Weaknesses and limits

- Nine requests were only partly fulfilled, mostly because the 7-day supplier lead time for orders over 1,000 units collides with 5 to 10 day deadlines. Several customers would receive a fraction of what they asked for.
- Size and variant words (A3, A5, recycled, biodegradable) are not catalog attributes. A3 glossy and matte paper is matched to the size-agnostic catalog item with a visible note, while generic "A3 paper" is rejected. This is a judgement call.
- The default evaluation uses the rule-based model. The live-LLM mode was run three times (section 11); the first two runs exposed weaknesses that were fixed, and the third matched the offline results exactly. A model can still transcribe an item badly, in which case an order may be rejected, but the reply will state an accurate reason.
- Sales and the matching supplier purchases are booked on the request date. Supplier arrival is checked against the deadline but not modelled as a delayed inventory event.

## 6. Requirement traceability

### 6.1 Agent workflow diagram

| Requirement | How it is met |
| --- | --- |
| Architecture with agent responsibilities and orchestration; at most five agents | The diagram shows five agents, each in its own box with its role stated. Orchestration order and the early-exit paths are drawn. |
| Tools shown per agent, with purpose and the helper function used, plus data flow | Every tool box states its purpose and its helper (for example `stock_check_tool` → `get_stock_level`). Arrows show agent-to-tool and tool-to-database interactions. |

### 6.2 Multi-agent implementation

| Requirement | How it is met |
| --- | --- |
| Distinct orchestrator and worker roles matching the diagram | `build_multi_agent_system()` creates `orchestrator_agent` plus `inventory_agent`, `quote_agent`, `finance_agent` and `sales_agent`, with the same names, tools and order as the diagram. |
| Uses an allowed framework | smolagents `ToolCallingAgent`, with the workers passed to the orchestrator as `managed_agents`. |
| All seven helpers used in at least one tool | `create_transaction`, `get_all_inventory`, `get_stock_level`, `get_supplier_delivery_date`, `get_cash_balance`, `generate_financial_report` and `search_quote_history` are each called by tools. A runtime counter proves it (check 5). |
| Required helpers assigned to agents | `get_all_inventory` → inventory\_agent, `get_cash_balance` and `generate_financial_report` → finance\_agent. |

### 6.3 Evaluation and reflection

| Requirement | How it is met |
| --- | --- |
| Evaluate on the full sample set and submit `test_results.csv` | All 20 requests are processed, and `test_results.csv` has 20 rows. |
| At least three requests change cash | 19 requests did (check 4). |
| At least three requests fulfilled | 19 fulfilled or partially fulfilled, 10 of them in full (check 2). |
| Not all requests fulfilled, with reasons | Request 15 is rejected in full with reasons, and nine more are partial with per-line reasons (check 3). |
| Report explains the diagram, roles and decision process | Section 2 of this report. |
| Report discusses results and strengths | Section 5. |
| At least two distinct improvement suggestions | Section 7 (four suggestions). |

### 6.4 Industry best practices

| Requirement | How it is met |
| --- | --- |
| Transparent customer outputs with rationale | Replies include line prices, the discount and its justification, delivery date and a reason for each unsupplied line. |
| No sensitive internal data or PII | Replies omit stock levels, cash, margins and error text, and do not repeat the customer's job title. Check 6 scans every reply. |
| Readable, commented, modular code | Naming conventions, docstrings and function-level modularity as described in section 3. |

## 7. Suggested improvements

1. **Split delivery and substitution offers.** When a line misses its deadline, offer an in-stock partial shipment now and the remainder on the supplier arrival date, or suggest a substitute catalog item (for example Standard copy paper for A4). This would convert most of the nine partial outcomes into complete sales.
2. **Demand-driven reorder points.** Use the roughly 400 historical inquiries in `quote_requests.csv` to forecast demand per item and set reorder points from supplier lead time times expected demand, so popular items such as cardstock, glossy and A4 paper are stocked before large orders arrive.
3. **Customer clarification loop and approval gate.** Let the orchestrator ask a follow-up question when an item is ambiguous (size, finish, reams versus sheets) instead of assuming, and add human approval for orders above a value threshold.
4. **Margin-aware price optimisation.** Replace the fixed tier table with a model trained on historical quotes, so larger discounts are offered only where margin allows.

## 8. Assumptions

- Supplier cost is 60% of catalog price.
- Sales and their supplier purchases are booked on the request date.
- Reams, packets and rolls are treated one-to-one with catalog units, as the historical quotes do.
- Product qualifiers that the catalog cannot express (recycled, biodegradable) are not enforced.

## 9. Review of the earlier submission

| Finding | Fix |
| --- | --- |
| The script crashed on import with real smolagents because tool docstrings lacked argument descriptions; the shipped results came from a silent fallback. | Valid docstrings on every tool, and smolagents imported unconditionally. |
| "Agents" were plain Python classes, and the orchestrator was not an agent. | Five real smolagents agents with managed-agent delegation. |
| The parser silently dropped items (requests 8, 10 and 11) yet reported success. | New parser and per-line accept or reject with reasons. |
| Rejection reasons were hard-coded to sample words, and envelopes were wrongly unsupported. | Generic catalog resolution. |
| No reorder logic, and newly stocked items were missing from the inventory table. | Reorder-point logic and automatic inventory record creation. |
| Restocks cost as much as or more than the sale price. | Supplier cost modelled at 60% of catalog price. |
| Quote history was counted but never used. | History-informed discount. |
| No real report and no Mermaid source. | This report, plus `.mmd` and PNG diagram. |

## 10. How to run

```
pip install -r requirements.txt
python project_starter.py            # deterministic run, writes test_results.csv
USE_LLM=1 python project_starter.py  # live LLM orchestration (needs UDACITY_OPENAI_API_KEY in .env)
```

## 11. Live-LLM run review and hardening

A live run (`USE_LLM=1`) exposed weaknesses that the offline run could not: the model skipped or reordered workers, invented order keys, produced malformed item JSON, hallucinated a $0.00 cash balance, and wrote its own customer replies that leaked internal terms (cash, order status). Replies and the database sometimes disagreed (for example, request 1 was reported as not fulfilled although the sale was booked).

Fixes applied:

1. The customer reply is always rendered from the order ledger, never from model free text.
2. Any workflow stage the agents leave undone is completed by the same business functions the tools call.
3. Tools ignore invented order keys and take the request date, deadline and event from the trusted request text.
4. Item JSON is repaired, or replaced by the rule parser when unusable.
5. The live model runs at temperature 0.

Verified with a deliberately faulty model: all 20 replies and the final cash ($46,804.87) are identical to the clean run.

### 11.2 Second live run: results

The second `USE_LLM=1` run (1 orchestrator + 4 workers driven by a live model) met every requirement and passed all of the built-in checks that existed at the time.

| Requirement | Evidence from the second live run |
| --- | --- |
| At least 3 orders fulfilled | 10 fully confirmed (requests 1, 4, 5, 6, 7, 8, 9, 10, 11, 12) and 9 partially confirmed |
| At least 1 order rejected with a clear reason | Request 15 rejected in full: 10,000 A4 and 5,000 A3-colored sheets cannot arrive by April 15 (earliest April 19), and cardboard is not in the catalog. No charge made. |
| Impossible constraints recognised | Request 14: earliest supplier delivery April 16 versus an April 15 deadline. Request 19: earliest April 22 versus April 20. Request 20: tickets not in the catalog. Each is reported per line. |
| At least 3 requests change cash | 19 of 20 changed cash (only the full rejection did not) |
| All seven helpers used | create\_transaction 121, get\_all\_inventory 42, get\_stock\_level 229, get\_supplier\_delivery\_date 153, get\_cash\_balance 70, generate\_financial\_report 38, search\_quote\_history 165 calls |
| Transparent, no internal data | Replies list items, unit prices, discount and reason, delivery date, and every unsupplied line with its reason. No cash, status or order-key text appears. |

The run ended with cash of $47,095.30 and inventory value of $4,404.55. All 20 outcomes and quote totals match the offline run. The first live run, by contrast, had 15 of 20 orders reported as not fulfilled, hallucinated a $0.00 cash balance, and leaked internal terms; the safeguards described at the start of this section resolved this.

### 11.3 Finding: duplicate booking in the second live run, and its fix

Final cash in the live run was $290.43 higher than in the offline run. Requests 8, 11 and 14 were the only ones whose cash change differed, and by exactly that amount. The cause: the live model asked the inventory agent to re-check an order after it was already booked. Re-running that step reset the order status, so the pipeline priced, authorized and booked the order a second time. Customer replies were correct because they come from the order ledger, but the database held two sales for those three orders.

The mechanism was reproduced with a model that re-runs the inventory step after completion (sales of $8,238.34 against the correct $4,142.92), then fixed:

1. Each workflow step only runs from its own status (inventory from `registered`, quote from `inventory_checked`, authorization from `quoted`). A repeat call returns a no-change message.
2. Each order carries a `booked` flag set before any database write, so it is booked at most once.
3. A seventh built-in check compares booked sales with the sum of quoted totals for fulfilled orders, so any future double booking fails visibly.

After the fix, the re-running model, the faulty model used to test the first fixes and the clean run all give identical replies and the same final cash of $46,804.87, and all seven checks pass. A third live run confirmed this: `USE_LLM=1 python project_starter.py` with the updated script gave the same result (section 11.4).

### 11.4 Third live run: confirmation

The third live run with the fixed script matched the offline results exactly: final cash $46,804.87 and final inventory value $4,404.55, with all seven built-in checks passing, including the booking-integrity check.

| Check | Third live run | Offline run |
| --- | --- | --- |
| Outcomes (fulfilled / partial / not fulfilled) | 10 / 9 / 1 | 10 / 9 / 1 |
| Cash change after each of the 20 requests | identical | identical |
| Final cash | $46,804.87 | $46,804.87 |
| Booking-integrity check (booked sales equal quoted totals) | PASS | PASS |
| `create_transaction` calls | 100 | 100 |
| Customer replies containing cash, status or order-key text | none | none |

Two details support the result. The number of database writes is the same as offline (100), so no order was booked twice. Read-only helper calls differ from offline (for example 112 supplier-date lookups against 39), because the live model explores extra tool calls; these change no data. The terminal output starts at request 1 and does not show the line naming the model driving the agents, so the live-model mode is taken from the run description rather than from the output itself.

Request 15 remains the required rejection: it is turned down in full because 10,000 A4 and 5,000 A3-colored sheets cannot arrive by April 15 (earliest April 19) and cardboard is not in the catalog, and the reply says so without charging the customer.
