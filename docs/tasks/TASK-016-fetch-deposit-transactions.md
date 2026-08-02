# TASK-016 Fetch deposit transactions

## Status

todo

## Requirements

**Binding:** REQ-011
**BDD mode:** BDD-ABSENT
**Depends on:** TASK-005 (introduced `get_withdrawal_transactions()`, its
pagination loop and `_split_to_transaction_read()`), TASK-010 (added
`source_name`/`source_id` to `TransactionRead`), TASK-011 (introduced the
`on_page` callback this method mirrors)
**Precedence:** The requirements above are the binding definition of this task.
The story and scenarios below are derived from them. On any discrepancy, the
requirements document wins. Stop and report discrepancies; do not build from
the story.

## Story (context, not binding)

As a consumer application (firefly-bills-analyzer), I want deposit transactions
in a date range with the same pagination and split flattening the withdrawal
side already gets, so that I can detect a recurring salary payment on an asset
account instead of asking the user to type their net income into a
configuration file and keep it up to date by hand.

## Description

Add `FireflyClient.get_deposit_transactions(start, end, on_page=None)`,
returning `list[TransactionRead]`.

The method is `get_withdrawal_transactions()` with `type=deposit` instead of
`type=withdrawal`. Everything else — the page loop, `total_pages` termination,
per-split flattening via `_split_to_transaction_read()`, the `on_page`
callback contract, and `FireflyConnectionError` propagation from `_get` — is
identical. Per REQ-011's constraints, extract the shared body into a private
helper (e.g. `_get_transactions_by_type(txn_type, start, end, on_page)`) and
implement both public methods on top of it, rather than copying the loop.

No new type is introduced. `TransactionRead` already carries both
`source_name` and `destination_name`, and Firefly III fills them per
transaction direction: on a deposit, `source_name` is the revenue account
(the payer) and `destination_name` is the asset account that received the
money. That is the reverse of the withdrawal case, and it is the API's own
convention, not something this library normalizes. Document it in the method
docstring so consumers do not have to rediscover it.

`source_id` is populated on the same terms as today. `destination_id` is not
part of `TransactionRead` and is not added here (see Out of scope).

Firefly III types a movement between two of the user's own asset accounts as
`transfer`. The `type=deposit` filter therefore excludes internal transfers at
the API level (UC-011-5), which is what lets a consumer treat every returned
record as money entering from outside. Assert this in a test against a mocked
response so the claim is enforced rather than assumed.

## Branch

**Branch name:** `task/016-fetch-deposit-transactions`
**Switch/create:** `git checkout -b task/016-fetch-deposit-transactions`
**Make target:** `make branch-task f=TASK-016`

## Acceptance criteria (Gherkin)

Scenarios are inline (BDD-ABSENT) and lift-ready for `.feature` extraction if
BDD tooling is adopted later.

- [ ] Scenario: Deposits are requested with the deposit type filter
      Given a valid date range
      When `get_deposit_transactions(start, end)` is called
      Then the request is `GET /api/v1/transactions` with query parameters
      `type=deposit`, `start`, `end`, and `page=1`

- [ ] Scenario: All pages are followed
      Given an API response reporting `total_pages` of 3
      When `get_deposit_transactions(start, end)` is called
      Then pages 1, 2, and 3 are requested and the returned list contains the
      splits from all three

- [ ] Scenario: Multi-split deposits are flattened
      Given a deposit transaction object with two splits under
      `attributes.transactions`
      When `get_deposit_transactions(start, end)` is called
      Then the returned list contains one `TransactionRead` per split

- [ ] Scenario: Account roles follow the API for a deposit
      Given a deposit split whose `source_name` is a revenue account and whose
      `destination_name` is an asset account
      When `get_deposit_transactions(start, end)` is called
      Then the returned `TransactionRead` carries that revenue account in
      `source_name` and that asset account in `destination_name`

- [ ] Scenario: Absent fields default to None
      Given a deposit split with no `category_name`, no `source_name`, and no
      `source_id` in the API response
      When `get_deposit_transactions(start, end)` is called
      Then those fields are `None` on the returned `TransactionRead`

- [ ] Scenario: Progress callback is invoked per page
      Given an API response reporting `total_pages` of 2
      When `get_deposit_transactions(start, end, on_page=callback)` is called
      Then `callback` is invoked as `(1, 2)` and `(2, 2)`, in that order

- [ ] Scenario: A callback exception stops fetching
      Given a callback that raises on the first page
      When `get_deposit_transactions(start, end, on_page=callback)` is called
      Then the exception propagates to the caller and no further page is
      requested

- [ ] Scenario: Transfers are not returned
      Given an account with both deposits and transfers in the date range
      When `get_deposit_transactions(start, end)` is called
      Then the request carries `type=deposit` and no transfer record appears
      in the result

- [ ] Scenario: Connection failure is reported
      Given the API responds with a non-2xx status or the network call fails
      When `get_deposit_transactions(start, end)` is called
      Then `FireflyConnectionError` is raised

- [ ] Scenario: Withdrawal fetching is unchanged
      Given the refactor extracting the shared page loop
      When the existing `get_withdrawal_transactions()` tests are run
      Then they pass unmodified

- [ ] Scenario: Type checking and quality gates pass
      Given the completed implementation
      When `mypy --strict` is run on `src/`, and `make lint && make test` are run
      Then `mypy --strict` passes, `make lint && make test` pass, and unit
      test coverage does not drop below the task-start baseline

## Out of scope

- Adding `destination_id` to `TransactionRead`. Consumers match asset accounts
  by name today (`source_name` on the withdrawal side), and nothing in the
  requesting use case needs the ID. Add it under its own requirement if a
  consumer ever does.
- Any classification of what a deposit *means* — salary, refund, reimbursement,
  gift. This library returns rows; recognizing a recurring income source is the
  consumer's concern (firefly-bills-analyzer's own requirements).
- Any filter on the deposit side equivalent to a consumer's account or payee
  filtering. The date range and the type filter are the only server-side
  narrowing here.
- Caching. The consumer owns its cache layer.

## Blockers

None.

## Completion

**Date:**
**Summary:**
**Files changed:**
**Branch:**
**Stage:**
**Commit:**
