# ORDER_FLOW_NOTES.md

## 1. Example order-event sequence

Example order: `ORD-DEMO-01`, total amount `1000` paise, initially `PLACED` and `UNPAID`.

| Step | Event | Expected order/payment/refund state | Actual result |
|---|---|---|---|
| 1 | `payment_confirmed` (`E1`) | `PLACED`, `PAID`, refund due `0`, refunded `0` | `PLACED`, `PAID`, refund due `0`, refunded `0` |
| 2 | `restaurant_accepted` (`E2`) | `ACCEPTED`, `PAID`, refund due `0`, refunded `0` | `ACCEPTED`, `PAID`, refund due `0`, refunded `0` |
| 3 | `dispatch_started` (`E3`) | `OUT_FOR_DELIVERY`, `PAID`, refund due `0`, refunded `0` | `OUT_FOR_DELIVERY`, `PAID`, refund due `0`, refunded `0` |
| 4 | `cancel_requested` (`E4`) | Cancellation should be rejected after dispatch; remain `OUT_FOR_DELIVERY`, `PAID`, refund due `0`, refunded `0` | `OUT_FOR_DELIVERY`, `PAID`, refund due `0`, refunded `0` |

Each event has a distinct event ID. The service records processed IDs and ignores a repeated event ID, preventing the same event from being applied twice.

## 2. Three edge cases

### 1. Cancellation after dispatch

- **Input/event sequence:** Start with an order in `OUT_FOR_DELIVERY`, then apply `cancel_requested` with event ID `E-CANCEL`.
- **Expected behaviour:** The cancellation must not change the order because dispatch has already started.
- **Actual observed result:** The order remains `OUT_FOR_DELIVERY`. Its payment and refund values remain unchanged.
- **Verifying test:** `test_cancel_after_dispatch_is_ignored` in `tests/test_contract.py`.

### 2. Partial refund must not mark payment as fully refunded

- **Input/event sequence:** Start with a `CANCELLED` order whose payment status is `PAID` and refund due is `1000` paise. Apply `refund_processed` with event ID `R1` and amount `400` paise.
- **Expected behaviour:** Record the partial refund, but do not mark the payment as fully refunded.
- **Actual observed result:** `refunded_paise` becomes `400`; `payment_status` remains `PAID`.
- **Verifying test:** `test_partial_refund_is_not_marked_complete` in `tests/test_contract.py`. The parameterized test `test_valid_partial_and_full_refunds` in `tests/test_acceptance_matrix.py` also checks partial and full refund amounts.

### 3. Refund event without an event ID

- **Input/event sequence:** Start with a `CANCELLED` order, payment status `PAID`, and refund due `1000` paise. Apply `refund_processed` for `400` paise without providing an `event_id`.
- **Expected behaviour:** Do not process the refund because a refund event must have an event ID for replay protection.
- **Actual observed result:** `refunded_paise` remains `0`, `payment_status` remains `PAID`, and no event ID is recorded.
- **Verifying test:** `test_refund_without_event_id_is_ignored` in `tests/test_contract.py`.

## 3. Assumptions and limitations

### Cancellation eligibility
- A cancellation request is accepted only when the order status is `PLACED` or `ACCEPTED`.
- A cancellation request after dispatch has started is ignored.
- If a paid order is cancelled, the full order total is recorded as the refund due.
- An unpaid cancellation does not create a refund due.

### Dispatch and delivery transitions
- A restaurant can accept or reject an order only while it is `PLACED`.
- Dispatch starts only from `ACCEPTED`, changing the status to `OUT_FOR_DELIVERY`.
- Delivery is recorded only from `OUT_FOR_DELIVERY`, changing the status to `DELIVERED`.
- Events that do not satisfy these transition conditions do not change the order status.

### Refund due and refunded amounts
- Amounts are stored as integer paise, not floating-point rupees.
- A paid order that is rejected or cancelled has its full `total_paise` recorded as `refund_due_paise`.
- A refund must have a positive integer amount and cannot exceed the outstanding refund due.
- Refunds are accepted only for orders in `REJECTED` or `CANCELLED` status with payment status `PAID`.
- Partial refunds increase `refunded_paise` but leave payment status as `PAID`.
- Payment status changes to `REFUNDED` only when the full refund due has been processed.
- Refund events require an event ID. Repeated event IDs are ignored.

### Remaining limitations
- Non-refund events without an event ID can still be processed, but cannot be reliably deduplicated. The service therefore cannot guarantee replay protection for those events.
- Refund events without an event ID are ignored to avoid processing financial events without replay protection.
- The reconciliation function compares records by `order_id` and reports missing records, amount mismatches, and status mismatches. It uses `total_paise` for internal records and `amount_paise` for provider records, with a fallback to the alternate field.
- The offline demo and the automated test suite were run, but the complete 219-event dataset was not replayed end-to-end.
- Performance improvements have not been benchmarked, and no performance gains are claimed.
