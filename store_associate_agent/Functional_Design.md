# Store Associate Agent (SAA)
## Functional Design — Design Challenge Submission

**Use Case**: Buy Online, Pick Up In Store (BOPIS) — Retail Order Picking
**Domain**: Retail Store Operations · SAP S/4HANA Cloud

---

## Table of Contents

1. [Problem Statement](#1-problem-statement)
2. [Why an Agent — Not a Workflow](#2-why-an-agent--not-a-workflow)
3. [Scope](#3-scope)
4. [Target Personas](#4-target-personas)
5. [Business Flows](#5-business-flows)
   - Flow 1 — Standard Order Picking
   - Flow 2 — Item Unavailability Exception (Customer HITL)
   - Flow 3 — Shelf Depletion Alert
   - **Flow 4 — Joule Conversational Assistant**
6. [Exception Handling Design](#6-exception-handling-design)
7. [Key Design Decisions](#7-key-design-decisions)
8. [Assumptions](#8-assumptions)
9. [Open Questions](#9-open-questions)

---

## 1. Problem Statement

Retail stores operating BOPIS fulfillment require store associates to manually pick ordered items before customers arrive to collect them. The picking process has a well-understood happy path — scan, confirm, complete — but becomes operationally complex under three exception conditions:

- **Stock unavailability**: An ordered item is not on the shelf at the customer's chosen store.
- **Supply resolution**: The system must autonomously determine whether a transfer from a nearby store, an in-store substitute, or a forced exclusion is appropriate — and in some cases, must ask the customer to decide.
- **Shelf depletion**: Post-pick stock levels may fall below reorder thresholds; the store manager must be alerted proactively with enough context to act.

The challenge is not the happy path. It is reasoning under uncertainty: evaluating three supply sources in parallel, presenting only the relevant options to the right actor (customer, not associate), and executing the confirmed resolution against a live SAP S/4HANA Cloud order management system — all within the constraints of a mobile-first, low-latency store operations environment.

---

## 2. Why an Agent — Not a Workflow

A rule-based workflow could handle the happy path. An agent is justified here because three conditions hold simultaneously:

| Test | This use case |
|------|---------------|
| Requires reasoning over ambiguous or changing signals? | **Yes** — stock levels, substitute compatibility, transfer feasibility, and customer preferences all vary per situation |
| Outcome depends on context that varies per call? | **Yes** — the resolution path (transfer / substitute / exclude) is determined by runtime data, not a fixed decision tree |
| Autonomous action is required, not just a report? | **Yes** — the agent executes order mutations, creates stock transfer orders, and sends customer notifications |

The key agentic behaviors in this design are:
- **Conversational reasoning via Joule**: Answering the associate's natural language questions mid-pick by dynamically selecting the right tool(s), calling live SAP data or the RAG knowledge base, and synthesising a grounded answer — the same reasoning capability that powers exception resolution, now available as an interactive conversational interface
- **Pre-pick risk reasoning**: Scanning all order lines before picking begins and flagging likely exceptions before they occur
- **Parallel evidence gathering**: Running three independent availability checks concurrently and synthesising the results into a ranked recommendation
- **Substitute scoring**: Applying semantic judgment (category fit, unit equivalence, price proximity) to rank candidates — a task that cannot be reduced to a deterministic rule
- **Exception pattern recognition**: Detecting whether multiple unavailability events across a session indicate a systemic supply gap, not isolated incidents

---

## 3. Scope

### In Scope

| Capability | Description |
|-----------|-------------|
| BOPIS order picking management | Guide the associate through scanning and confirming all order lines |
| **Joule conversational assistant** | Natural language Q&A for associates mid-pick; ReAct agent with full tool access; read-only by default — write actions require explicit associate confirmation before execution |
| Pre-pick exception forecasting | Batch-scan all order lines at order open and flag high-risk items |
| Picking instruction synthesis | Generate per-item handling guidance from product attributes and store SOPs |
| Item unavailability resolution | Three-way parallel check: transfer, substitute, global exclusion |
| Customer substitution preference | Customer sets preference at order time: **auto** (agent selects best substitute), **notify** (interactive approval required), or **strict** (no substitute — original item or nothing); stored on order header as `ZZ1_SubstitutionPreference`; governs **substitute decisions only** — transfer is attempted for all preferences regardless |
| Substitute scoring and ranking | AI-ranked top 2–3 substitutes with rationale; confidence tier (high ≥90% / medium 70–89% / low <70%) determines which authorization path is taken |
| Transfer source selection | Reason across candidate source stores on ETA, stock reliability, and consolidation trade-offs |
| Customer HITL notification | Interactive mobile/SMS notification with option selection; window length is tier-dependent: 2 hours for transfer and medium-confidence cases; 30 minutes for low-confidence and strict-preference cases |
| Exception pattern recognition | Detect systemic supply gaps from multiple exceptions in a session |
| Shelf depletion alerting | Post-pick depletion detection with AI-enriched reorder recommendation to store manager |
| Order status lifecycle management | Track and update order from IN_PICKING to READY_FOR_PICKUP via SAP S/4HC |
| Audit trail | Structured event log of every state-changing action per associate, order, and tenant |

### Out of Scope

| Item | Reason |
|------|--------|
| Customer order placement or modification | The agent operates downstream of order creation; it does not own the commerce layer |
| Store manager replenishment execution | The agent alerts the manager; it does not create purchase orders or replenishment requests |
| Unsupervised substitute selection by the associate | For medium and low-confidence substitutes, the customer is the sole authorising actor; associates may only confirm AI-scored high-confidence substitutes (≥90%) or when the customer's preference is set to auto |
| Cross-customer order consolidation | The agent resolves exceptions per order; cross-order inventory pooling is out of scope |
| Offline-first data sync for mobile | The mobile application handles offline state; the agent assumes connectivity |

---

## 4. Target Personas

| Persona | Flows | Role in this system |
|---------|-------|---------------------|
| **Store Associate** | Flow 1, Flow 2, Flow 4 (Joule) | Initiates all picking actions; scans items; reports unavailability; receives AI-assisted instructions and risk flags; confirms high-confidence AI-scored substitutes with a single tap; asks natural language questions via Joule mid-pick; not authorised to select low-confidence substitutes on behalf of customers |
| **Customer** | Flow 2 | Sets substitution preference at order placement (auto / notify / strict); receives exception notifications only when the confidence tier or their preference requires explicit approval; makes the binding selection (transfer or substitute) in those cases; communicates only via push/SMS — never interacts with the agent directly |
| **Store Manager** | Flow 3 | Receives enriched shelf depletion alerts with AI-generated reorder recommendations; responsible for acting on replenishment |

---

## 5. Business Flows

### Flow 1 — Standard Order Picking

**Trigger**: Store associate opens an assigned BOPIS order on the mobile app.

**What the agent does autonomously:**
- Fetches all order lines from SAP S/4HC including SKUs, quantities, and storage locations
- Runs a pre-pick batch risk scan across all lines and highlights items with predicted availability risk
- For each scanned item: validates the barcode match, confirms stock, and synthesises a picking instruction if the product has handling attributes
- After each confirmed pick: checks post-pick stock against the depletion threshold and sends an enriched alert to the store manager if breached
- Once all lines are resolved: updates the order status to READY_FOR_PICKUP and notifies the customer

**Completion criteria**: All order lines are in a terminal state (PICKED, SUBSTITUTED, AWAITING_TRANSFER, or ITEM_EXCLUDED) and the order header is updated.

---

### Flow 2 — Item Unavailability Exception (Tiered Resolution)

**Trigger**: Store associate marks an order line as unavailable.

**What the agent does autonomously:**
- Reads `ZZ1_SubstitutionPreference` from the order header — this governs **substitute decisions only**, not transfer
- Checks transfer availability for **all** preferences — a transfer delivers the customer's original item from a different source store; substitution preference does not apply
- For **STRICT**: only transfer is checked; if no transfer is found, the item is excluded immediately with no substitute check
- For **AUTO / NOTIFY**: transfer and substitute checks run in parallel; transfer takes priority in routing
- Scores and ranks substitute candidates only when no transfer is found and preference is not STRICT
- Detects systemic supply gaps across the session (AUTO / NOTIFY only)

**Resolution paths — in priority order:**

| Priority | Path | Triggered when | Who acts | Timeout |
|----------|------|---------------|----------|---------|
| 1 | **Transfer** | Transfer available — any preference | Agent executes; customer informed of delay | None |
| 2 | **Immediate exclude** | STRICT, no transfer | Agent only | None |
| 3 | **Associate confirms substitute** | AUTO, no transfer, substitutes found | Associate taps confirm on mobile | None |
| 4 | **Customer chooses substitute** | NOTIFY, no transfer, substitutes found | Customer selects via push/SMS | 30 min; item excluded if no response |
| 5 | **Auto-exclude** | Any preference, no transfer, no substitutes | Agent only | None |

**Completion criteria**: Order line is in AWAITING_TRANSFER, SUBSTITUTED, or ITEM_EXCLUDED; customer notified of outcome; pick flow resumes.

---

### Flow 3 — Shelf Depletion Alert (Fully Autonomous)

**Trigger**: Post-pick stock level at the pickup store falls to or below a configured threshold.

**What the agent does autonomously:**
- Checks the pending BOPIS order queue for the affected SKU to assess demand velocity
- Generates a reorder recommendation with quantity, urgency level, and estimated depletion time
- Sends an enriched alert to the store manager — not just "stock is low" but "reorder N units now, shelf empty in approximately Z minutes given current demand"

**No human decision required.** The agent alerts and recommends; the store manager decides whether to act.

---

### Flow 4 — Joule Conversational Assistant

**Trigger**: Store associate asks a natural language question at any point during the picking session — typed or spoken on the mobile app.

**What makes this agentic:**
Joule uses a ReAct (Reasoning + Acting) pattern. Rather than a fixed sequence of steps, the agent reasons about the question, decides which tool(s) to invoke from the existing tool inventory, calls them against live SAP data or the RAG knowledge base, observes the result, and synthesises a grounded natural language answer. It may chain multiple tool calls when the question requires it.

**Representative interactions:**

| Associate asks | Agent does |
|----------------|-----------|
| "Where is the organic milk in this store?" | Reads storage location from the current order line in SAP S/4HC; returns aisle and bin reference |
| "How many items are left to pick on this order?" | Reads pick progress from current state; returns count with item names |
| "This item has no barcode — could item X be a valid substitute?" | Calls substitute finder and scorer; returns ranked compatibility rationale |
| "Tell me how to handle this frozen item" | Retrieves product handling guidelines from RAG knowledge base |
| "This SKU keeps running out — is it a known supply issue?" | Reviews exception history from current session; surfaces systemic gap if pattern detected |

**Human confirmation gate:**
Joule is read-only by default. If the associate asks Joule to execute a write action (e.g., "mark this line as unavailable" or "notify the customer"), the agent presents the proposed action for explicit confirmation before any order mutation is made. This gate ensures no unintended changes from ambiguous natural language.

**Completion criteria**: Associate receives a grounded, tool-backed answer on-screen. No order state changes without explicit confirmation.

---

## 6. Exception Handling Design

The exception resolution design embeds several deliberate principles:

### Three-Way Parallel Check

All three availability checks run concurrently to minimise latency in the exception flow. The results are consolidated before scoring begins. This avoids sequential waterfall logic where an early "no transfer available" check would delay the substitute candidate retrieval.

### Decision Hierarchy

| Situation | Agent Action | Human Required? |
|-----------|-------------|-----------------|
| Transfer available — any preference | Create STO, update line to AWAITING_TRANSFER; inform customer of delay | No |
| STRICT, no transfer | Exclude immediately — no substitute check | No |
| AUTO, no transfer, substitutes found | Associate confirms AI's top-ranked substitute; customer notified after | Yes — associate confirms |
| NOTIFY, no transfer, substitutes found | Send interactive notification; 30-min window; exclude if no response | Yes — customer selects |
| Any preference, no transfer, no substitutes | Auto-exclude and notify customer | No |

### Customer Notification Design

- The customer receives an interactive notification (push/SMS) — not a plain text message.
- Transfer option is presented as "expected delay in days" — no internal store names, plant IDs, or logistics detail are exposed.
- Substitutes are presented with description and price delta — no internal material numbers.
- The customer's response is delivered to the agent via a BTP Notification Service callback, encoded as a structured `process_customer_response` action.

### Async HITL State

The agent does not hold a live connection while waiting for the customer. The exception state (options, timeout, resolution context) is written to Object Store (S3) keyed by a unique `exception_id`. When the customer responds, the agent retrieves the state by ID and resumes. This design supports the up-to-30-minute response window without blocking a server thread.

---

## 7. Key Design Decisions

### Decision 1: Transfer-universal, preference governs substitutes only

- **What**: The customer's substitution preference (`ZZ1_SubstitutionPreference`) governs substitute decisions only — it does not affect transfer eligibility. A transfer delivers the customer's original item from a different source store; from the customer's perspective, they receive exactly what they ordered. Therefore transfer is checked for **all** preferences before any substitute logic runs. For **STRICT**: only transfer is attempted — if no transfer is available, the item is excluded immediately with no substitute check. For **AUTO** and **NOTIFY**: transfer and substitute checks run in parallel; transfer takes priority in routing.
- **Consideration**: The alternative design — applying the preference gate before all availability checks — meant a STRICT customer could lose an item that was perfectly transferable from a nearby store. That is a bad outcome for both the customer (who loses an item they explicitly wanted) and the retailer (who loses fulfilment revenue on an item that could have been delivered). Separating the transfer decision from the substitution preference removes this false constraint without violating the customer's stated intent: they said no to substitutes, not to their original item arriving from a different location.
- **Limitation**: Transfer availability is bounded by `ZZ1_TRANSFER_LEADTIME` configuration between plant pairs and live stock in adjacent stores. If a STRICT customer's item is unavailable at the pickup store and no transfer source exists, the item is excluded with no fallback — this is the customer's explicit choice and must be communicated clearly in the customer notification.
- **Assumption**: `ZZ1_SubstitutionPreference` is captured at order placement by the commerce layer and stored on the sales order header. If the field is absent, the agent defaults to **notify** (most conservative tier for substitutes; transfer is still attempted regardless).

---

### Decision 2: Async HITL via Object Store

- **What**: Exception state is parked to S3 between notification and customer callback. No in-process state is held.
- **Consideration**: The customer response arrives as a separate HTTP callback up to 30 minutes later. Holding a server-side connection for that duration would exhaust connection pools under concurrent load.
- **Limitation**: If S3 is unavailable at callback time, the response cannot be processed. The associate would need to handle the exception manually.
- **Assumption**: BTP Object Store availability SLA is sufficient for the HITL response window. Recovery from S3 failure during a 30-minute window is an edge case, not the common path.

---

### Decision 3: Application Backend and AI Workflow Service are separate components

- **What**: The Application Backend handles all deterministic picking logic and all SAP mutations. The AI Workflow Service is a dedicated microservice invoked only for tasks that require AI reasoning.
- **Consideration**: This separation keeps AI reasoning isolated from order state mutations. It also enables independent scaling — AI inference is compute-intensive and should not contend with high-throughput picking confirmations.
- **Limitation**: Cross-service calls add latency on the exception path. For pre-pick forecasting, this means the associate waits for an AI call at order open — this must be bounded (e.g., 2-second timeout with graceful degradation).
- **Assumption**: The AI Workflow Service can be deployed on the same SAP BTP subaccount as the Application Backend, making A2A calls an internal network hop.

---

### Decision 4: S/4HC as the sole system of record

- **What**: No new domain data store is introduced. All order, inventory, product, and transfer data is read from and written to SAP S/4HC via standard OData APIs.
- **Consideration**: This eliminates dual-write consistency problems, reduces operational complexity, and leverages existing data governance and access control in the customer's S/4HC instance.
- **Limitation**: The agent is bounded by what S/4HC OData APIs expose. Complex queries (e.g., real-time demand velocity, cross-plant consolidation scoring) may require multiple API calls and in-agent aggregation.
- **Assumption**: The customer's S/4HC instance has the standard APIs enabled (`API_SALES_ORDER_SRV`, `API_MATERIAL_STOCK_SRV`, `API_PRODUCT_SRV`, `API_STOCK_TRANSFER_ORDER_SRV`) and the associate's user has the required authorisations.

---

### Decision 5: Key User Extensibility for custom fields

- **What**: Order line status (`ZZ1_LineStatus`), source plant (`ZZ1_SourcePlant`), and order status (`ZZ1_OrderStatus`) are added to standard S/4HC objects via RAP-based Key User Extensibility, not custom Z-tables.
- **Consideration**: Using Key User Extensibility avoids modification, keeps the extension upgrade-safe, and is visible in the standard OData API without custom service development.
- **Limitation**: Key User Extensibility fields are tied to the S/4HC release schedule. Field type and length constraints must be respected. The extension cannot be done by the agent at runtime — it requires an S/4HC administrator action as a one-time setup.
- **Assumption**: The S/4HC instance is on a release that supports Key User Extensibility for `A_SalesOrder` and `A_SalesOrderItem`.

---

### Decision 6: Joule as a first-class interaction mode, sharing the same tool inventory

- **What**: The Joule conversational assistant does not get a separate data access layer. It runs as a ReAct agent within the AI Workflow Service with access to the same tools used by the structured picking and exception flows. The same tool functions that the deterministic agent calls in a fixed sequence become available to the LLM to select dynamically based on the associate's question.
- **Consideration**: This architecture means adding Joule costs near-zero incremental development once the tool layer is built. The tools are already tested, the SAP OData bindings are established, and the observability instrumentation is in place. A new conversational capability is layered on top without duplication. This is the strategic advantage of designing a unified tool inventory from the start.
- **Limitation**: The ReAct loop is unbounded by default. A verbose or ambiguous question could trigger a chain of tool calls that exceeds the 3-second mobile UX target. A max-iterations cap (4 iterations) and a 2.5-second timeout must be enforced. Additionally, Joule conversation history is held in the session state only — if the associate's session is interrupted, conversational context is lost.
- **Assumption**: SAP Joule is available in the customer's BTP subaccount. If it is not, the same conversational capability can be surfaced as a custom chat widget on the mobile app using the same AI Workflow Service endpoint — the agent design does not change, only the front-end entry point.

---

## 8. Assumptions

| # | Assumption | Impact if false |
|---|-----------|----------------|
| 1 | The customer has an active push or SMS notification channel registered at order time | The HITL notification path fails; the auto-exclusion path must be extended as fallback |
| 2 | Product substitution relationships are maintained in the SAP material master (`API_PRODUCT_SRV/A_ProductSubstitute`) | The substitute candidate search returns empty; the agent can only offer transfer or exclusion |
| 3 | Transfer lead times between plant pairs are configured in the extension table (`ZZ1_TRANSFER_LEADTIME`) | ETA cannot be computed; the agent can confirm transfer availability but not the expected delay |
| 4 | All order lines for a given BOPIS order are assigned to a single pickup store (no split fulfilment) | The consolidation logic assumes one destination plant; multi-store orders would require redesign |
| 5 | The associate's BTP session token can be exchanged for an S/4HC bearer token via OAuth2 User Token Exchange | All S/4HC mutations would use a shared service account, losing per-associate audit identity |
| 6 | The RAG knowledge base (product handling instructions, store SOPs, substitution rules) is populated and maintained by the customer's operational team | Picking instruction synthesis and compatibility guardrails produce low-quality output if the knowledge base is sparse or stale |
| 7 | Associates have a text input or voice input interface on the mobile app for natural language questions | Joule conversational assistant cannot be used in environments where only scan buttons and tap interfaces are available |
| 8 | Customer substitution preference is set at order placement by the commerce layer and stored as `ZZ1_SubstitutionPreference` on the sales order header | If absent, the agent defaults to **notify** (middle tier); the auto path is unavailable and all substitutions require customer notification |

---

## 9. Open Questions

| # | Question | Impact |
|---|---------|--------|
| 1 | Should the substitution confidence thresholds (≥90% high, 70–89% medium, <70% low) be global constants or configurable per tenant or product category? | Per-category tuning would allow a grocery retailer to apply tighter thresholds for allergen-sensitive categories (baby food, nut products) than for general merchandise — but adds configuration complexity |
| 2 | Should the depletion threshold be global (one value for all SKUs) or SKU-class-specific? | Affects the depletion check logic and the extension table design |
| 3 | Is the store manager a single role per store, or can multiple managers receive alerts? | Affects the notification fan-out design |
| 4 | Are there regulatory or customer-consent requirements for sending AI-generated substitute recommendations to customers in the target markets? | May require guardrail extensions and customer opt-in messaging |
| 5 | Should the pre-pick risk forecast be blocking (associate waits for results before seeing the pick list) or non-blocking (pick list shown immediately, flags appear asynchronously)? | Trade-off between latency and value — non-blocking is safer for store SLA; blocking gives the associate more time to pre-arrange options |
| 6 | What is the scope of write actions Joule is permitted to confirm and execute — and who defines the permission boundary? | A narrow scope (Joule can only answer questions, never act) is safest but limits utility; a wider scope (Joule can mark lines unavailable or notify the customer on confirmation) is more powerful but increases the risk of mis-triggered mutations in a noisy store environment |
