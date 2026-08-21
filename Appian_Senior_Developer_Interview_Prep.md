# Appian Senior Developer – Interview Prep

---

## 1. Record Types vs. Data Store Entities

**What is a Record Type?**
A Record Type is Appian's data-fabric object that exposes business data (from one or more sources — synced DB tables, unsynced DB, web services, or a mix) as a unified, secure, end-user-facing model. It bundles the data definition with relationships, record actions, views, list/summary UI, security (row-level via filters), and — if synced — a fast internal cache that Appian queries instead of hitting the source every time.

**What is a Data Store Entity (DSE)?**
A DSE is the older, lower-level construct: a direct mapping between a CDT (Custom Data Type) and a database table/view through a Data Store connection. It's purely a data-access object — no built-in UI, no relationships beyond what you code, no security layer, no caching. Everything (queries, joins, security filtering) is your responsibility in expression rules/process models.

**When to use each:**
- **Record Type (synced)** – default choice for new development: end-user data with lists/dashboards, row-level security, relationships, Process HQ/Process Reports, AI features, and good query performance (data-fabric cache).
- **Record Type (unsynced)** – when data must stay live at the source (e.g., real-time balances) or the source can't be synced (some legacy/SOAP systems), but you still want the Records UX layer.
- **DSE/CDT** – legacy apps already built on it, very complex custom SQL/stored-procedure needs, write-only integration tables, staging/audit tables that end users never browse, or sources that data fabric doesn't support well.

**Why has Appian shifted toward Record Types?**
- Data Fabric gives a caching/indexing layer that makes `queryRecordType()` much faster than hitting the source DB every time.
- Declarative row/column-level security instead of custom filter logic everywhere.
- Native relationships (no manual joins), Related Actions, Record Views/Dashboards out of the box.
- One object powers UI, integrations, process automation, and Process HQ/AI features — reducing duplicate objects (CDT + DSE + rules).
- Aligns with Appian's low-code direction: less custom expression logic, more configuration.

---

## 2. queryRecordType() vs. queryEntity()

**When to use `queryRecordType()`:** Default for any new development against a Record Type — list views, grids, dashboards, reports, dropdowns, related-record lookups. It respects Record Type security, relationships, and (for synced types) reads from the fast internal cache.

**When would you still use `queryEntity()`:** Only for objects still on CDT/DSE (legacy apps not yet migrated), or narrow cases like Data Store-only staging/audit tables where you deliberately never wrapped them in a Record Type.

**Which performs better, and why:** `queryRecordType()` against a **synced** record type is generally faster because it reads from Appian's internally maintained, indexed copy of the data instead of issuing a live query to the source database every time — it also pushes filters/paging down efficiently and benefits from data-fabric's query optimizer. `queryEntity()`/unsynced record queries always hit the live source, so performance depends entirely on the source DB and network latency.

**Limitations of `queryRecordType()`:**
- Sync scale limits (historically ~4M rows per record type source table, being raised over time) and sync lag (data isn't instantaneous — near-real-time, not transactional-read-your-write in some configurations).
- Field type limits (e.g., text field length caps in the internal sync store).
- Complex multi-source (federated) queries across unsynced/synced sources can result in "unbounded" queries if filters/relationships can't be pushed down to the source — this can hurt performance.
- Some very complex SQL (window functions, exotic joins/stored procs) is easier to express via a custom DSE query or view than through Record Type configuration.

---

## 3. Interface Taking 12 Seconds to Load — Root Cause & Optimization

**Investigate:**
- Use **Appian Health Check / Performance tab** in the interface designer (or "Monitor Performance" / SAIL Designer's view performance) to see per-component/per-rule timings.
- Check **Admin Console > Performance Monitoring** and Log Viewer (Application Server logs) for slow rule executions, DB query times.
- Isolate: strip the interface down section by section (comment out grids/related actions) to find which component is the bottleneck.

**Possible causes:**
- Unbounded/inefficient queries (no pagination, fetching all columns/all rows, missing filters).
- N+1 queries — looping and calling a rule/integration per row instead of batching.
- Unsynced record types or DSE queries with no indexes.
- Too many nested rule calls or redundant re-computation without caching.
- Heavy use of `a!refreshVariable` on load, or unnecessary process/related-action visibility calls.
- Large images/rich text/complex nested grids rendering client-side.
- External integration calls made synchronously inside the interface load.

**Optimizations:**
- Add pagination and only fetch needed fields (`fields` parameter in `a!queryRecordType`).
- Add DB indexes on filter/sort columns; ensure sync is enabled where possible.
- Batch related data instead of looping calls (use relationships / single query with joins instead of per-row rule calls).
- Cache static/reference data in constants or use `a!localVariables` with `submit` semantics instead of recomputation.
- Move non-critical data behind lazy-loading (tabs, "load more", async process for heavy back-end work).
- Use `loadingIndicator` / async smart services for slow integrations instead of blocking the interface.

**Verify improvement:** Re-run the interface performance profiler, compare before/after timings; use browser dev tools network tab for round-trip time; do load testing (multiple concurrent users) to confirm the fix holds under real load, not just in isolation.

---

## 4. Designing for 1 Million Records — Fast, Scalable, Maintainable

- Use a **synced Record Type** on a properly indexed database table (indexes on filter/sort/foreign-key columns).
- Design data model with correct **relationships** to avoid app-side joins; use pagination everywhere (grids, dropdowns).
- Apply **row-level security via record filters** natively rather than post-filtering large result sets in expressions.
- Avoid loading full datasets into interfaces — always query with filters and limits (`a!queryRecordType` with paging).
- Archive/partition historical data (e.g., move closed/old records to an archive table not surfaced in the live Record Type) to keep the "hot" table smaller.
- Use asynchronous process models / queues for any bulk processing rather than synchronous, single-transaction operations.
- Monitor with Process HQ / performance dashboards and set up alerting on slow queries.
- Keep business logic in reusable expression rules (DRY), document naming standards, and modularize by domain (separate apps/objects per bounded context) for maintainability at scale.

---

## 5. Interface With 100 Customers × 4 Related Data Sets Taking 18 Seconds

**What's causing it:** Classic N+1 pattern — for each of the 100 customers, the interface is likely making separate calls for account, loan, and transaction data (400+ round trips), instead of one batched query per data type.

**How to identify the bottleneck:** Use the interface performance profiler / view performance timings to see which section/rule dominates; check if calls are made inside a `a!forEach` per customer rather than once for the whole set.

**Changes to make:**
- Replace per-row rule calls with **single batched queries**: query all accounts/loans/transactions once, filtered by the 100 customer IDs (`in` filter), then map results back client-side (e.g., group by customer ID using `group()`/`index()`).
- Use **Record Type relationships** so related data comes back in one `queryRecordType()` call with related fields instead of four separate lookups.
- Paginate the customer list itself (don't render 100 rows with full detail at once — use a summary grid + on-demand detail/drill-down).
- Only fetch the fields actually displayed (`fields` parameter) instead of full objects.
- Consider a **denormalized "dashboard" Record Type/view** that pre-joins the four datasets if this screen is used heavily.

**Best practices:** batch over loop, fetch only needed fields, paginate, use synced record types with proper indexes, lazy-load secondary details, cache reference/lookup data.

---

## 6. Common Appian Performance Mistakes (and How to Avoid Them)

| Mistake | Fix |
|---|---|
| Looping queries/integration calls per row (N+1) | Batch into a single query filtered by a list of IDs |
| Querying full objects/all columns | Use the `fields` parameter to fetch only what's shown |
| No pagination on grids/dropdowns | Always paginate; use lazy loading for large lists |
| Overusing `refreshVariable`/unnecessary re-computation | Cache values in local variables; only refresh on real triggers |
| Using unsynced Record Types/DSEs for high-traffic screens | Enable sync where possible; add DB indexes |
| Doing heavy logic in the interface layer | Push complex computation to process models/async or precompute in the record source |
| Synchronous long-running integrations blocking UI | Use async smart services / background processes with polling or notifications |
| Poor reuse — copy-pasted expression logic | Centralize in shared expression rules and constants |
| No monitoring in production | Set up Health Check, Performance Monitoring, and log-based alerting proactively |

---

## 7. AI-Driven Document Intake Solution Design

**Business need:** On upload, extract data with AI, create a case, route to the right team, notify, and allow manual review on low confidence.

**Architecture:**
- **Record Types:** `Document`, `Case`, `Team`, `ExtractionResult` (with confidence score), related via foreign keys.
- **Interfaces:** upload form, case worklist, manual-review screen showing extracted fields with confidence highlighting.
- **Integrations:** Appian AI Document Extraction (or a plug-in/connected system to an external AI/OCR service) as an Integration object.

**Process flow:**
1. Upload triggers a process model (event: document creation / Start Process record action).
2. Process calls the **AI extraction integration** (async-capable, since extraction can take time).
3. Branch on confidence score:
   - **High confidence** → auto-create Case record, auto-assign team (via routing rule based on document type/business unit), send notification (email/task).
   - **Low confidence** → create Case in "Pending Review" status, generate a human task for manual review; on completion, update the record and proceed.
4. Notifications sent via Appian's native notification/email smart services at each key milestone.

**Process models / Record Types / Integrations / AI placement:**
- Record Types hold state and drive UI + security.
- Process models orchestrate the workflow, branching, and human tasks.
- Integration objects encapsulate the AI service call (retryable, logged).
- AI is invoked as an integration step, its output written to the Record Type.

**Failures/retries:** Wrap the AI integration call in a sub-process with a **retry loop with exponential backoff** (e.g., 3 attempts), a timeout, and a fallback path to manual review if all retries fail — never let a stuck integration block the whole process indefinitely. Log every attempt for audit.

**Scalability:** Make extraction **asynchronous** (queue-based / event-driven — the upload just enqueues; a separate process pool consumes), so upload spikes don't overload the AI service; use process throughput/parallel execution for independent documents.

---

## 8. Redesigning a Nightly Batch of 50,000 Records (4–5 Hours, Occasional Failures)

- **Chunk the work**: split 50,000 records into batches (e.g., 500–1,000 per batch) processed by parallel sub-processes instead of one giant sequential loop.
- Use **asynchronous/parallel process execution** (multiple process instances or Appian's parallel flow) to use available execution engines concurrently.
- Move heavy per-record logic out of the interface layer entirely — it should already be pure process/integration work.
- Add **checkpointing**: track batch-level status (Not Started / In Progress / Success / Failed) in a tracking Record Type so a restart only reprocesses what's needed.
- Add **structured error handling** (try/catch equivalents — error-handling process nodes) so one bad record doesn't kill the whole batch; log and continue, then report failures at the end.
- Ensure idempotency (safe to re-run) so retries don't create duplicates.
- Move to event/queue-driven kickoff rather than one big nightly script if source data arrives incrementally.

---

## 9. From 5 Hours → 2 Hours, But Batch #37 Failure Restarts Everything — Retry Redesign

- Replace "all-or-nothing" with a **batch tracking table/Record Type**: one row per batch with status, timestamp, retry count, and error detail.
- On failure, only **that batch** is retried (not the whole run) — the orchestrator process checks status and re-queues only "Failed"/"Not Started" batches.
- Add a **retry policy**: max attempts (e.g., 3) with backoff, then mark as "Needs Manual Intervention" and alert a support queue rather than blocking the whole nightly run.
- Make batches **idempotent** (upsert semantics, unique keys) so re-processing a partially-completed batch doesn't duplicate data.
- Consider a **queue-based design** (message queue or Appian process-based work queue) where each batch is a discrete unit of work pulled by a worker process — natural retry/DLQ (dead-letter queue) semantics.
- Report a summary at the end: batches succeeded / retried / failed, so ops has visibility without re-running the whole job to find out.

---

## 10. Retry Design for a Flaky Third-Party API (500s, Slow Responses, Duplicate Risk)

- **Timeouts**: set a sane connect/read timeout on the integration (don't let a 30-second call block indefinitely).
- **Exponential backoff with jitter**: retry on 5xx/timeout (e.g., 1s, 2s, 4s...), capped at a small max attempt count (e.g., 3–5), so you don't hammer a struggling service.
- **Idempotency key**: generate a unique request/transaction ID per logical operation and pass it to the API (if supported) or check "did this already succeed?" before retrying, so a retry after a false-timeout doesn't create a duplicate transaction.
- **Circuit breaker pattern**: if failures spike, stop calling for a cool-down period and fail fast / queue for later instead of continuously retrying.
- **Asynchronous processing**: don't retry inline in a user-facing interface — hand off to a background process so retries don't block the user.
- **Dead-letter/manual-review path**: after max retries, log the failure with full context and route to a support/ops queue rather than silently dropping it.
- **Idempotent downstream writes**: use upsert-by-key logic on your side as a second safety net against duplicates even if the API side can't guarantee it.

---

## 11. Concurrent Edits — Same Customer Record by Two Users

**What could go wrong:** "Lost update" — User A's changes get silently overwritten by User B's save (or vice versa) because each read a stale copy and wrote back the whole record.

**Optimistic vs. pessimistic locking:** Generally prefer **optimistic locking** in Appian (it fits its stateless, form-based UI model far better than pessimistic/row locks, which would require holding a DB lock across a user's think-time — bad for scalability and UX). Pessimistic locking is only reasonable for short, tightly-scoped critical sections (e.g., a brief "claim this case" action), not for general editing.

**How to implement optimistic locking:**
- Add a `version`/`lastModifiedOn` field to the record.
- On save, compare the version the user loaded against the current version in the DB (via a query before write, or a DB-level version check in the write). If they don't match, someone else updated it since — reject/flag the write.

**Conflict handling:** Show the user a clear message ("This record was updated by [user] at [time] since you opened it") with options to **review the latest data and merge/re-apply their change**, or discard their edit and reload the current version. For simple fields, you can auto-merge non-overlapping field changes; for overlapping fields, force manual reconciliation.

**What to show the user:** A conflict-resolution screen highlighting what changed, by whom, and letting them choose to overwrite, merge, or cancel — never a silent overwrite.

---

## 12. Refactoring a Large, Sprawling App (10,000+ Users, Years of Growth)

Approach it as a structured technical-debt program:
- **Modularity**: break the monolith into multiple Appian applications/objects grouped by business domain (bounded contexts), each with clear ownership and its own object naming prefix.
- **Reusability**: audit for duplicated expression rules/interfaces across teams; consolidate into shared, versioned rule libraries and reusable components (e.g., common form patterns, shared constants).
- **Naming standards**: enforce a documented naming convention (prefixes by type/domain) retroactively where feasible, strictly for all new objects.
- **Layered architecture**: enforce clear layers — Data (Record Types) → Business Logic (expression rules) → Process (orchestration) → Presentation (interfaces) — so logic isn't duplicated between UI and process layers.
- **Security**: consolidate group-based security models; move ad-hoc/interface-level security checks into centralized, declarative Record Type security where possible.
- **Integrations**: standardize on Integration objects and Connected Systems instead of scattered custom web-service calls; centralize error handling/retry patterns.
- **Performance**: run the Health Check tool app-wide, fix flagged issues (N+1 queries, missing indexes, unsynced record types under load).
- **Technical debt**: use Appian's Application Health Check / Object Dependency analysis to find dead/orphaned objects, unused rules, and deprecated smart services; prioritize fixes by usage/risk, not just age.
- Do this incrementally (strangler-fig pattern) — migrate module by module, keep both old and new paths working during transition, and use feature flags/constants to cut over safely.

---

## 13. Integration Layer Across 8 External Systems (REST, SOAP, SFTP) — Resilient Onboarding

- Build a **single onboarding process model** as the orchestrator, but treat each of the 8 integrations as an **independent, decoupled step** — never let one system's outage block the whole process.
- Wrap each integration (REST/SOAP alike, using Appian Connected Systems/Integration objects) with **its own timeout, retry, and circuit-breaker logic**.
- For systems that are **temporarily unavailable**, use an **asynchronous, queue-based retry**: mark that step "Pending," let the rest of onboarding proceed where possible, and have a background process periodically retry the failed step until it succeeds or exceeds a retry limit (then escalate).
- For the **SFTP legacy system**, decouple further: write files to an outbound folder/queue and use a scheduled process to pick up and send, with file-based idempotency (checksums/unique file names) — this system's design constraints (batch/file-based) shouldn't force everything else to be synchronous.
- Maintain a central **onboarding status Record Type** tracking per-system completion status, so partial progress is visible and resumable rather than needing a full restart.
- Standardize a common **canonical data model** so all 8 systems map to/from one internal representation, isolating the core process from each system's quirks (adapter/anti-corruption-layer pattern).
- Add monitoring/alerting per integration so failures are visible operationally, not just discovered by users.

---

## 14. Production-Readiness Checklist Before Go-Live

- **Performance**: Health Check passed, load/performance tested under expected concurrent users, no unbounded/N+1 queries.
- **Security**: role-based access verified for every persona, record-level security tested, no objects left with default/broad permissions, sensitive data encrypted/masked as needed.
- **Error handling**: all integrations have timeouts/retries/fallback paths; process models have fault handling (no silent failures).
- **Data**: migration/seed data validated, backup and rollback plan defined, database indexes in place.
- **Environment/config**: environment-specific values externalized to constants (no hardcoded URLs/credentials), promoted correctly through dev → test → prod via package/deployment pipeline.
- **Monitoring & alerting**: logging, Admin Console monitoring, and alerts configured for critical processes.
- **Documentation**: design docs, support runbook, and known-issues list handed to the support team.
- **Disaster recovery**: rollback plan for the deployment itself if issues are found post-go-live.
- **User readiness**: training/UAT sign-off completed, support/on-call plan in place for go-live week.

---

## 15. Critical Process Suddenly 5× Slower, No Recent Deployment (2 AM Call)

**First:** Check **infrastructure/environment health** — CPU, memory, DB connection pool utilization, execution engine queue depth in the Admin Console. A slowdown with no code change is very often infra/resource contention, not application logic.

**Second:** Check for **external factors** — database performance (locking, growing table size, missing/degraded index, a batch job or another app hogging the same DB), a downstream integration/API that's suddenly slow (its SLA changed, not yours), or a data volume spike (more records than usual hitting the same query).

**Third:** Check **recent non-code changes** — infra patches, DB maintenance, a scheduled job that started running at the same time, a certificate/network change, or an increase in concurrent users (seasonal spike). Also check if statistics/table indexes need rebuilding (query plans degrade as data grows even with no code change).

Only after ruling these out would I look at data growth against unindexed queries (a query that was fine at 100K rows may now be slow at 5M rows) — this is a very common "no deployment, but got slower" root cause in Appian apps.

---

## 16. Redesigning 100,000 Synchronous Customer Updates/Day Into a Scalable Architecture

- Move from a **single synchronous process model per update** to an **asynchronous, queue-based, batched architecture**:
  - Incoming updates land in a staging table/queue (Record Type) immediately (fast ack to the external system) instead of being processed inline.
  - A pool of **worker processes** consumes the queue in parallel batches (e.g., 100–500 at a time), decoupling ingestion rate from processing rate.
- Use **bulk/batched writes** (Write Records with multiple records) instead of one write per record.
- Add **backpressure handling**: if the queue backs up, scale out worker concurrency rather than letting it grow unbounded; add monitoring/alerting on queue depth.
- **Prioritize/partition** if some updates are more time-sensitive than others (e.g., separate queues by urgency or customer segment).
- Ensure idempotency (unique update IDs) so replays/retries after failures don't double-apply.
- Track per-record processing status for auditability and easy reprocessing of failures without reprocessing the whole day's volume.

---

## 17. Technical-Health Assessment of a 3-Year-Old Production App

- Run **Appian's Application Health Check** across the app for structural issues (unused objects, missing descriptions, security gaps, performance red flags).
- Analyze **object dependency graphs** to find highly-coupled or "god" objects (interfaces/rules touched by every feature) that are likely regression hotspots.
- Pull **defect/regression history** — which modules generate the most production bugs — to prioritize refactoring where it reduces risk fastest, not just where code looks messy.
- Review **performance monitoring data** (Admin Console) for consistently slow interfaces/processes.
- Interview each team/owner about pain points, tribal knowledge, and undocumented behavior.
- Score modules on a simple matrix: **business criticality × technical debt/risk**, and refactor high-criticality/high-risk areas first (not just the oldest code).
- Establish guardrails going forward (naming standards, shared rule libraries, code review checklist, CI-based static analysis) so debt doesn't reaccumulate immediately after the cleanup.

---

## 18. Security Model for Employees / Managers / Auditors / Administrators

- Use **Appian groups** to represent each role (e.g., `Customer_Employees`, `Customer_Managers`, `Customer_Auditors`, `Customer_Admins`), with users assigned to the appropriate group(s).
- Apply **Record Type row-level security via filters tied to group membership and data attributes**:
  - **Employees**: filter `Customer.assignedTo = loggedInUser()`.
  - **Managers**: filter `Customer.team = user's team` (looked up via a Team/Org Record Type relationship), so they see their team's customers without needing per-record assignment.
  - **Auditors**: full read visibility (no row filter), but **no write/record-action permissions** — enforce via Record Type action security and interface-level read-only rendering (grey out/disable fields, hide save buttons) — never rely on hiding UI alone; back it with actual permission checks.
  - **Administrators**: full read/write, plus access to admin-only functions (user management, config).
- Layer security at multiple levels for defense-in-depth: **Record Type security (row/column level)** + **Process Model security (who can initiate/complete tasks)** + **Interface-level conditional visibility/read-only** + **Record Action security** (who can even see the "Edit" action).
- Periodically **audit group membership and security rules** (especially for auditors/admins) since these are highest-risk for privilege creep over time.

---

*Prepared as concise interview-answer notes — expand any section with a live example/demo if the interviewer wants more depth.*
