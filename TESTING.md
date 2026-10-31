# TESTING.md — Test Evidence

## Test command and environment

- **Python version:** 3.13.13
- **Test framework:** pytest
- **Command run:** `python -m pytest -q`
- **Actual test summary:** `215 passed in 3.66s`
- **Additional verification:** `python -m main` completed and produced the expected reconciliation output. `git diff --check` reported no whitespace errors.

## Edge-case evidence

| # | Case / input | Expected behaviour | Actual result | Test name or evidence |
|---|---|---|---|---|
| 1 | The same payment event is received more than once with the same event ID. | A previously processed event must not be applied again. | Passed; duplicate event processing was prevented. | `payment_event_replay_idempotent` — 20 targeted cases passed. |
| 2 | A refund event has an invalid amount, including zero, negative, non-integer, or an amount greater than the outstanding refund due. | Invalid refunds must be ignored; the refunded amount must not exceed the outstanding due. | Passed; invalid refund amounts were rejected. | `invalid_refund_amounts_are_ignored`, `refunds_cannot_exceed_due` |
| 3 | A valid refund is processed in partial and full amounts. | A partial refund must retain the partial-refund state; the order becomes fully refunded only when the full due amount has been processed. | Passed; partial and full refund behaviour was verified. | `valid_partial_and_full_refunds` |
| 4 | A refund is attempted for an order that is not both paid and rejected or cancelled. | The refund must not be processed unless the order meets the eligibility conditions. | Passed; ineligible refunds were rejected. | `refund_eligibility_matrix` |
| 5 | Internal and provider records use different amount field names (`total_paise` and `amount_paise`). | Reconciliation must compare the corresponding amounts rather than incorrectly reporting a mismatch. | Passed; matching amounts were reconciled correctly. | `test_reconcile_internal_total_against_provider_amount` |
| 6 | An event arrives without an event ID. | It must not be treated as a duplicate merely because its ID is missing; missing-ID events must not be recorded as processed IDs. | Passed; independent missing-ID events were processed independently. | `test_events_without_event_id_are_processed_independently` |
| 7 | A refund event arrives without an event ID. | The refund must be ignored because it cannot be safely deduplicated. | Passed; the refund was ignored. | `test_refund_without_event_id_is_ignored` |
| 8 | Internal and provider records have matching amounts but different statuses. | Reconciliation must report a status mismatch separately from an amount mismatch. | Passed; the status mismatch was reported. | `python -m main` output and the reconciliation test suite. |

## Regression tests added

The following regression tests were added during the implementation:

- **Duplicate event processing:** Added coverage for replaying a payment event with the same event ID.
- **Refund validation:** Added coverage for valid partial and full refunds, invalid refund amounts, refund eligibility, and preventing refunds beyond the outstanding due.
- **Reconciliation amount mapping:** Added `test_reconcile_internal_total_against_provider_amount` to cover the difference between internal `total_paise` and provider `amount_paise` fields.
- **Missing event IDs:** Added `test_events_without_event_id_are_processed_independently` to ensure missing IDs do not cause unrelated events to be treated as duplicates.
- **Refund event IDs:** Added `test_refund_without_event_id_is_ignored` to ensure refund events without IDs are not processed.

The complete test suite was run after the implementation changes and reported **215 passed**.

## Remaining failures or limitations

- No failing tests remained in the final recorded full test-suite run.
- Events without IDs cannot be reliably deduplicated. The implementation therefore avoids recording missing IDs and rejects refund events without an ID.
- The full 219-event example dataset was not independently replayed end-to-end as a separate test. Verification is based on the pytest suite and the offline `python -m main` demonstration.
- No formal performance benchmark was conducted, so no execution-time or memory improvements are claimed.
- The reconciliation implementation assumes the expected internal and provider record structures; broader malformed-input and schema-validation coverage could be added in future work.