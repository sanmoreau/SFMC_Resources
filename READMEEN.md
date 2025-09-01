# CloudPage in SFMC — Limitations of CreateSalesforceObject() and UpdateSingleSalesforceObject()

**Date:** 2025-09-01  
**Author:Santi

## TL;DR
- These are **synchronous, per-record** calls — no bulk.  
- They don’t consume the **org’s daily API limit**, but they **are** subject to **concurrency**, **timeouts**, and **locks** in Salesforce.  
- They fire **Validation Rules/Flows/Triggers** and require proper **FLS/CRUD** for the integration user.  
- Error handling is **basic**; implement **logging** to a Data Extension.  
- Use them for **few records** (forms, simple creates). Avoid them with **volume** or **traffic spikes**.

---

## Key limitations
1. **1:1 granularity (no bulk)**  
   Each call creates/updates **one** record. Looping many records from a CloudPage adds latency and timeout risk.

2. **Execution and concurrency limits**  
   - They don’t count against the org’s **daily API limit**, but **concurrency** and **response time** limits still apply.  
   - Under concurrent traffic you can see waits, saturation, and failures.

3. **Record locking / UNABLE_TO_LOCK_ROW**  
   Simultaneous updates on the same record (or parents) can lock. Common with data skew or parallel automations.

4. **Validations and automations run**  
   Validation Rules, Flows/Process Builder, and Triggers execute; they can block, cascade changes, or extend transaction time.

5. **Permissions and visibility (FLS/CRUD)**  
   Execution uses the **integration user**; non-visible or non-editable objects/fields ⇒ error. Consider network/IP policies if applicable.

6. **Strict data formats (especially dates)**  
   Use ISO‑8601/UTC for Date/DateTime. Normalize picklists, required fields, and lengths to avoid rejections.

7. **CloudPages runtime timeouts**  
   Loops with many calls in a single request can exhaust the Marketing Cloud runtime.

8. **Limited observability**  
   - `CreateSalesforceObject()` returns the **Id**.  
   - `UpdateSingleSalesforceObject()` returns **1/0**.  
   A DE log is essential (payload and result).

9. **Performance**  
   Every call has network/serialization overhead; throughput drops as you scale up.

10. **Compatibility/scope**  
   - Works with standard and custom objects.  
   - No **upsert by External Id**; you need custom logic for create vs update (or use APIs that support native upsert).

---

## When to use (and when not to)
**Use if…**  
- There are **few records per interaction** (1–5) and you need a **synchronous** response (e.g., preference form, simple case/appointment creation).

**Avoid if…**  
- There’s **volume** and/or **concurrency** (long lists, traffic spikes, multi-object cascades).

---

## Recommended mitigation patterns
1. **Asynchronous staging in a Data Extension**  
   Write first to a DE with idempotency (anti-resubmit token) and process in **Automation Studio** (SSJS/WSProxy) in **batches** with **retries** and **rate limiting**.

2. **Salesforce APIs for volume**  
   For large loads, use **REST/Bulk API 2.0** from a worker/Function with **External IDs (upsert)**, exponential **backoff**, and **row lock** handling.

3. **Optimize Salesforce automations**  
   Review **Validation Rules/Flows/Triggers**: bulk‑safe, non‑recursive, and time‑bounded to reduce locks and timeouts.

4. **Permissions and IPs**  
   Ensure **FLS/CRUD** for the integration user and, if applicable, allowlist Marketing Cloud IPs in Salesforce (**Network Access** or access policies).

5. **Logging and observability**  
   - DE audit (timestamp, object, payload, result, error).  
   - Correlate by request id/session id for troubleshooting.  
   - Basic alerts (Automation + query for recent errors).

---

## Pre‑go‑live checklist
- [ ] Validate the integration user’s FLS/CRUD per object/field.  
- [ ] Test with **representative volume** and **simulated concurrency**.  
- [ ] Normalize **formats** (dates, picklists, lengths).  
- [ ] Review **Validation Rules/Flows/Triggers** for impact and timings.  
- [ ] Implement **DE logging** and **retry** policies.  
- [ ] Define a **fallback** (DE staging if the synchronous call fails).

---

### Note
The AMPscript update function is `UpdateSingleSalesforceObject()` (often informally abbreviated as *UpdateSalesforceObject*).
