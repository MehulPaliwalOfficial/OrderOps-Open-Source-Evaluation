# Approach

## 1. Baseline

- **Commands run before editing:** The initial baseline command history and output are not available in the retained project notes. I have therefore not reconstructed or assumed an initial test result.
- **Initial test result:** Not recorded in the available logs.
- **Demo/output observed:** The initial demo output before implementation was not retained. During later investigation, the reconciliation demo exposed amount mismatches for `ORD-101` and `ORD-102`, which led to a separate investigation of the amount-field mapping.

## 2. Codebase Understanding

- **Important files and functions inspected:**
  - `service.py`: `apply_order_event()` and `reconcile_orders()`.
  - `models.py`: `new_order()` and the default order structure.
  - `fixtures.py`: data used by the offline demonstration.
  - `data/records.json` and `data/README.txt`: synthetic records and event examples.
  - `tests/test_acceptance_matrix.py` and `tests/test_contract.py`: expected behaviour and regression coverage.
  - `main.py`: offline demonstration entry point.
  - `pyproject.toml`, `requirements-dev.txt`, `.gitignore`, and `.github/workflows/tests.yml`: project configuration and CI workflow.

- **Order lifecycle and data flow traced:**

  The order begins in `PLACED` state with an `UNPAID` payment status. Events update the order through payment confirmation, restaurant acceptance or rejection, cancellation, dispatch, and delivery. When a paid order is rejected or cancelled, a refund becomes due. Refund events update the refunded amount and can change the payment status to `REFUNDED` once the full amount is processed.

  Reconciliation compares internal order summaries with provider records using `order_id`, amount, and status.

- **How I confirmed my understanding:**

  I inspected the model defaults, event-processing conditions, fixture generation, synthetic data, demonstration output, and acceptance and contract tests. I also used targeted tests and the offline demonstration to verify the behaviour of individual changes.

## 3. Investigation and Implementation Strategy

### Issue 1: Duplicate event processing

- **Observed symptom:** Replaying an event with an already processed `event_id` could apply its state change more than once.
- **Hypothesis tested:** The event processor needed to check whether the identifier had already been recorded before applying an event.
- **Evidence:** The targeted idempotency test selection passed 20 tests after the duplicate-event guard was implemented.
- **Fix:** Added an early return when a non-`None` event ID is already present in `processed_event_ids`.

### Issue 2: Unsafe refund processing

- **Observed symptom:** Refund processing needed stronger checks around eligibility, amount validity, and the remaining refund due.
- **Hypothesis tested:** Refunds should only be accepted for paid orders in `REJECTED` or `CANCELLED` states, and the amount should be a positive integer that does not exceed the outstanding balance.
- **Evidence:** The targeted refund validation selection passed 67 tests, covering valid partial and full refunds, invalid amounts, eligibility, and refund limits.
- **Fix:** Added eligibility and amount validation. The payment status changes to `REFUNDED` only when the accumulated refund exactly reaches the refund due.

### Issue 3: Incorrect reconciliation classification

- **Observed symptom:** Records with different statuses could be classified as matched when their amounts were equal.
- **Hypothesis tested:** Matching amounts alone were insufficient; status also needed to agree.
- **Evidence:** The reconciliation tests passed after adding a separate status-mismatch category. The full suite at that stage passed 212 tests.
- **Fix:** Added `status_mismatches` to the report and compared statuses after checking amounts.

### Issue 4: Internal and provider amount fields differed

- **Observed symptom:** The offline demo reported `ORD-101` and `ORD-102` as amount mismatches.
- **Hypothesis tested:** Internal fixtures used `total_paise`, while provider records used `amount_paise`; comparing `amount_paise` on both sides would therefore produce an incorrect result.
- **Evidence:** Inspection of `fixtures.py` and the resulting demo output identified the field-name difference. A regression test was added using an internal `total_paise` value and a provider `amount_paise` value.
- **Fix:** Updated reconciliation to read the appropriate amount field from each record, with a fallback for the alternative field name.

### Issue 5: Events without IDs were treated as duplicates

- **Observed symptom:** The duplicate-event check could treat multiple unrelated events without IDs as duplicates because each had `event_id = None`.
- **Hypothesis tested:** Missing IDs should not be inserted into the processed-ID list or used for duplicate detection.
- **Evidence:** Added a regression test applying a payment confirmation and restaurant acceptance without event IDs. The contract tests passed 14 tests, and the full suite passed 214 tests.
- **Fix:** Applied duplicate detection and ID recording only when `event_id` is not `None`.

### Issue 6: Refund events without IDs

- **Observed symptom:** A refund event without an identifier could not be reliably protected against replay.
- **Hypothesis tested:** Financial events should require a deduplication key, even though independent non-financial events can be processed without IDs.
- **Evidence:** The synthetic dataset audit identified seven events without IDs, including a `refund_processed` example for `ORD-1133`. A regression test verified that a refund without an ID does not change the refunded amount or payment status.
- **Fix:** Restricted the refund-processing branch to events with a non-`None` event ID.

## 4. Implementation Decisions

- **Main changes and why:**
  - Used event IDs to prevent duplicate processing.
  - Validated refund eligibility, integer amounts, and outstanding refund limits to protect financial state.
  - Required IDs specifically for refund events because they need replay protection.
  - Kept independent non-financial events without IDs processable.
  - Compared both amount and status during reconciliation.
  - Supported the different amount-field names used by internal and provider records.

- **Alternatives considered and why rejected:**
  - Treating every event without an ID as a duplicate was rejected because it prevents unrelated events from being processed.
  - Allowing refunds without IDs was rejected because repeated financial events cannot be reliably deduplicated without an identifier.
  - Comparing only amounts was rejected because equal amounts do not establish that two records have the same status.
  - Using one fixed amount-field name was rejected because the inspected internal and provider records use different field names.

- **Assumptions:**
  - The order identifier is the key used to associate internal and provider records.
  - Monetary values are represented as integer paise.
  - A refund is eligible only for a paid order that has been rejected or cancelled.
  - Events with IDs use those IDs as their deduplication keys.
  - Events without IDs cannot be reliably deduplicated by event ID.

## 5. Verification

- **Tests added or changed:**
  - Added a regression test for reconciliation between internal `total_paise` and provider `amount_paise`.
  - Added a regression test for independently processing events without IDs.
  - Added a regression test ensuring that a refund without an event ID is ignored.
  - Existing acceptance and contract tests were used to verify idempotency, refund validation, eligibility, and reconciliation behaviour.

- **Commands run and actual results:**

  | Command | Recorded result |
  |---|---|
  | `python -m pytest tests/test_acceptance_matrix.py -k "payment_event_replay_idempotent" -q` | 20 passed |
  | Targeted refund validation tests | 67 passed |
  | `python -m pytest tests/test_contract.py -q` after missing-ID event fix | 14 passed |
  | `python -m pytest tests/test_contract.py -q` after refund-ID fix | 15 passed |
  | `python -m pytest -q` after all implementation changes | 215 passed |
  | `python -m main` after integration | Demo output verified |
  | `git diff --check` | No errors reported |

- **Final demo output:**

  ```text
  matched: ['ORD-101']
  missing_from_provider: ['ORD-103']
  missing_from_internal: ['ORD-104']
  amount_mismatches: []
  status_mismatches: ['ORD-102']
  ```

- **Edge cases verified:**
  - Repeated events with the same event ID.
  - Partial and full refunds.
  - Invalid refund amounts.
  - Refunds exceeding the outstanding amount.
  - Refund eligibility based on order and payment status.
  - Independent events without IDs.
  - Refund events without IDs.
  - Reconciliation with different amount-field names.
  - Reconciliation status mismatches and missing records.

- **Remaining failures or known limitations:**
  - The initial baseline test result was not retained, so no before-and-after test-count comparison is claimed.
  - Events without IDs cannot be reliably deduplicated by event ID. Refund events are therefore rejected when their ID is missing.
  - The final verification establishes that the test suite passes and the offline demo produces the recorded output; it does not establish that every event in the synthetic dataset was replayed end to end.