# Challenges Faced

## 1. Challenge or confusing behaviour

The most confusing issue was reconciliation reporting amount mismatches for records that appeared to have matching amounts. I also investigated how events without an `event_id` were handled, particularly when processing refunds.

## 2. Evidence

The initial demo output showed:

```text
matched: []
missing_from_provider: ['ORD-103']
missing_from_internal: ['ORD-104']
amount_mismatches: ['ORD-101', 'ORD-102']
status_mismatches: []
```

On inspecting the fixtures, I found that internal records used `total_paise`, whereas provider records used `amount_paise`.

Another issue was that missing event IDs were represented by `None` in the processed event list, causing unrelated events without IDs to be treated as duplicates.

## 3. What I tried

- Ran the demo using `python -m main` and examined its reconciliation output.
- Inspected `fixtures.py`, `models.py` and `service.py` to understand how internal and provider records were constructed.
- Compared the amount fields used by both sources.
- Ran targeted pytest selections while fixing refund validation and event idempotency.
- Added regression tests for different amount field names and events without IDs.

## 4. Resolution

I updated `reconcile_orders()` in `service.py` to compare `total_paise` from internal records with `amount_paise` from provider records, while retaining fallback compatibility.

I also changed event ID handling so missing IDs are not stored or treated as duplicate identifiers. Since refunds involve financial state, I additionally required refund events to have an event ID.

These changes addressed the underlying causes rather than suppressing test failures.

## 5. Proof and regression prevention

The regression test `test_reconcile_internal_total_against_provider_amount()` verifies the amount-field mapping.

The tests `test_events_without_event_id_are_processed_independently()` and `test_refund_without_event_id_is_ignored()` cover missing-ID behaviour.

The final full test suite completed with **215 passed**. The demo also produced the expected reconciliation categories, with ORD-101 matched and ORD-102 reported as a status mismatch.

## 6. Unresolved challenge or next step

Events without IDs cannot be reliably deduplicated because there is no unique identifier to track their replay. I currently allow independent non-financial events without IDs but reject refund events without them. A future improvement would be to investigate event validation at ingestion and require unique identifiers for all financial events.