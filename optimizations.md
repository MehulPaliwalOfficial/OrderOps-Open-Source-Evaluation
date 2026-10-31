# Optimizations Considered and Implemented

## 1. Baseline and objective

The main opportunities for improvement were in correctness, maintainability, and avoiding unnecessary repeated event processing.

During investigation, several issues were identified:

- Duplicate events could be processed more than once.
- Events without an `event_id` were incorrectly treated as duplicates because `None` could be stored in the processed event list.
- Refund processing needed stronger validation to prevent invalid or excessive refunds.
- Reconciliation used different amount field names for internal and provider records, resulting in incorrect mismatch reports.

These findings came from inspecting the implementation, running the supplied tests, and comparing the internal and provider fixtures.

The objective was to make event processing safer, reconciliation more reliable, and the code easier to understand and maintain.

No formal performance baseline was collected.

## 2. Options considered

### Option 1: Improve event idempotency

**Expected benefit:** Prevent the same identified event from being applied repeatedly.

**Risk:** Treating all events without IDs as duplicates could incorrectly discard independent events.

**Decision:** Implemented. Duplicate checking and recording now apply only when an event ID is present.

### Option 2: Add stronger refund validation

**Expected benefit:** Prevent invalid refunds, refunds exceeding the outstanding amount, and refunds on ineligible orders.

**Risk:** Additional conditions make the refund-processing logic slightly longer.

**Decision:** Implemented. Refunds now require an eligible order, a valid positive integer amount, an outstanding refund balance, and an event ID.

### Option 3: Standardize reconciliation amount fields

**Expected benefit:** Compare internal and provider amounts correctly despite differences in their field names.

**Risk:** Supporting multiple field names introduces fallback logic that must remain understandable.

**Decision:** Implemented. Internal records use `total_paise` with an `amount_paise` fallback, while provider records use `amount_paise` with a `total_paise` fallback.

### Option 4: Introduce a more complex event-processing architecture

**Expected benefit:** A dedicated event-processing pipeline could make future event types easier to organize and extend.

**Risk:** It would add abstraction and complexity to a relatively small service.

**Decision:** Not implemented. The existing function-based structure was sufficient for the current requirements.

### Option 5: Introduce caching or additional data structures

**Expected benefit:** Could reduce repeated lookups or processing if the dataset became significantly larger.

**Risk:** Additional memory usage and synchronization complexity without evidence that the current implementation needs them.

**Decision:** Not implemented. No profiling evidence justified introducing caching or more complex structures.

## 3. Changes implemented

### A. Duplicate event protection

- **Change:** Added an event ID check before processing and recorded IDs only when they are present.
- **Reason:** Prevent repeated processing of identified events without treating unrelated events with missing IDs as duplicates.
- **Tests used:** `test_payment_event_replay_idempotent` and `test_events_without_event_id_are_processed_independently`.
- **Measurement:** No performance benchmark was conducted.

### B. Refund validation

- **Change:** Added eligibility, amount-type, positive-amount, outstanding-balance, and event-ID checks.
- **Reason:** Protect refund processing against invalid or repeated financial operations.
- **Tests used:** `test_valid_partial_and_full_refunds`, `test_invalid_refund_amounts_are_ignored`, `test_refund_eligibility_matrix`, `test_refunds_cannot_exceed_due`, and `test_refund_without_event_id_is_ignored`.
- **Measurement:** No performance benchmark was conducted.

### C. Reconciliation field handling

- **Change:** Added explicit fallback handling for the different internal and provider amount field names.
- **Reason:** Avoid reporting false amount mismatches when the underlying values are equal.
- **Tests used:** `test_reconcile_internal_total_against_provider_amount` and the existing reconciliation tests.
- **Measurement:** No timing or memory comparison was conducted.

The complete test suite passed with **215 tests passing** in the latest recorded run. This verifies correctness against the available tests, not a measured performance improvement.

## 4. Trade-offs

The implemented changes prioritize correctness and predictable behaviour over performance claims.

- **Idempotency:** Events with IDs can be checked for duplicates. Events without IDs cannot be reliably deduplicated, so they are not added to the processed ID list.
- **Refund safety:** Additional validation makes the logic more explicit but introduces more conditions to maintain.
- **Reconciliation:** Supporting alternative field names improves compatibility with the supplied records, although it requires clear field-selection logic.
- **Simplicity:** More complex architectures, caching, and premature optimization were avoided because the current dataset and requirements did not demonstrate a need for them.

No speedup, memory reduction, or time-complexity improvement is claimed.

## 5. Further improvements

With more time, I would consider the following improvements:

1. **Input validation:** Validate incoming event structures and required fields before processing. This would make the service more predictable when handling malformed or incomplete data.

2. **Larger dataset testing:** Test reconciliation and event processing with substantially larger synthetic datasets to identify potential bottlenecks.

3. **Performance profiling:** Measure execution time and memory usage before considering caching, indexing, or alternative data structures.

4. **Improved event traceability:** Introduce structured processing results or logs so that ignored, invalid, duplicate, and successfully processed events can be investigated more easily.

5. **Additional edge-case tests:** Cover empty event IDs, missing amount fields, duplicate order IDs, and other malformed input scenarios.

Any future optimization would be implemented only when supported by measurable evidence, with regression tests protecting existing behaviour.