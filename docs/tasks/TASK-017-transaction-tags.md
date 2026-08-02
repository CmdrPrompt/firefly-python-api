# TASK-017 Transaction tags

## Status

todo

## Requirements

**Binding:** REQ-012
**BDD mode:** BDD-ACTIVE
**Depends on:** TASK-005 (introduced `TransactionRead` and
`_split_to_transaction_read()`), TASK-016 (introduced
`_get_transactions_by_type()`, through which both fetch methods now build
their records)
**Precedence:** The requirements above are the binding definition of this task.
The story and scenarios below are derived from them. On any discrepancy, the
requirements document wins. Stop and report discrepancies; do not build from
the story.

## Story (context, not binding)

As a consumer application (firefly-bills-analyzer), I want each transaction to
carry its tags, so that a user can mark one grocery purchase as personal, or
one purchase in a personal category as shared, without having to reorganize
their categories to express the exception.

## Description

Add `tags: list[str]` to `TransactionRead`, read from the split's
`attributes.transactions[].tags` in the Firefly III response.

The change is confined to `_split_to_transaction_read()` and the `TypedDict`
definition. Both `get_withdrawal_transactions()` and
`get_deposit_transactions()` build their records through
`_get_transactions_by_type()`, which calls that helper, so both gain the field
without either method changing (UC-012-3).

Absent or `null` tags become `[]`, never `None` (UC-012-2). This deviates from
the `None`-defaulting convention the other optional fields follow, and does so
deliberately: a consumer asking "does this transaction carry tag X" wants a
container to test, and the distinction between "no tags" and "tags not
reported" carries no meaning here.

Tag strings are stored exactly as returned (UC-012-4). Do not case-fold, trim,
or sort them. A consumer matching tags decides its own comparison rules, and a
library that normalizes silently makes the consumer's rules unpredictable.

The field is additive; no existing field or signature changes.

## Branch

**Branch name:** `task/017-transaction-tags`
**Switch/create:** `git checkout -b task/017-transaction-tags`
**Make target:** `make branch-task f=TASK-017`

## Acceptance criteria (Gherkin)

**Feature files:** tests/bdd/features/TASK-017-transaction-tags.feature

- [ ] 1. Scenario: Tags are returned on a withdrawal
      Given a withdrawal split carrying two tags in the API response
      When `get_withdrawal_transactions(start, end)` is called
      Then the returned record's `tags` holds both tag strings, in the order
      the API returned them

- [ ] 2. Scenario: Tags are returned on a deposit
      Given a deposit split carrying one tag in the API response
      When `get_deposit_transactions(start, end)` is called
      Then the returned record's `tags` holds that tag

- [ ] 3. Scenario: An absent tags field becomes an empty list
      Given a split whose API response contains no `tags` key
      When either fetch method is called
      Then the returned record's `tags` is `[]` and not `None`

- [ ] 4. Scenario: A null tags field becomes an empty list
      Given a split whose API response contains `"tags": null`
      When either fetch method is called
      Then the returned record's `tags` is `[]`

- [ ] 5. Scenario: Tag strings are preserved verbatim
      Given a split tagged `" Hushåll "` with surrounding whitespace and mixed case
      When either fetch method is called
      Then the returned tag string is byte-identical to the API's value

- [ ] 6. Scenario: Each split carries its own tags
      Given a multi-split transaction whose two splits carry different tags
      When either fetch method is called
      Then each returned record carries the tags of its own split

- [ ] 7. Scenario: Existing fields are unchanged
      Given the completed implementation
      When the existing `get_withdrawal_transactions()` and
      `get_deposit_transactions()` tests are run
      Then they pass unmodified

- [ ] 8. Scenario: Type checking and quality gates pass
      Given the completed implementation
      When `mypy --strict` is run on `src/`, and `make lint && make test` are run
      Then `mypy --strict` passes, `make lint && make test` pass, and unit
      test coverage does not drop below the task-start baseline

## Out of scope

- Any tag-based server-side filter. Firefly III can filter by tag at the API,
  but the requesting consumer reads a full window and filters in memory; adding
  a query parameter nothing calls would be speculative.
- Writing, creating, or modifying tags. This library's tag support is read-only
  here.
- Normalizing, deduplicating, or sorting tag values.
- Any tag field on the resources other than transactions (bills, accounts,
  budgets).

## Blockers

None.

## Completion

**Date:**
**Summary:**
**Files changed:**
**Branch:**
**Stage:**
**Commit:**
