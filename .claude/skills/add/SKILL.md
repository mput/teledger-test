---
name: skill-add
description: Add missing transactions from bank account screenshots into main.ledger
argument-hint: "<bank-name> (bank, bank-usd or cash)"
---

Add the transactions that are missing from `main.ledger` by reading the user's screenshots of a
bank transaction list, then close with a balance assertion and validate.

## Supported Banks

| Bank | Asset account | Currency |
|------|---------------|----------|
| bank | `Assets:Cards:Bank` | EUR |
| bank-usd | `Assets:Cards:Bank-USD` | USD |
| cash | `Assets:Cash:Main` | EUR |

## Account Name Reference

All accounts and commodities MUST already be declared in `accounts.ledger` (the ledger runs in
`--pedantic` mode). Read `accounts.ledger` before mapping anything; it is the authoritative list.
Use full account names in transactions, never the aliases.

## Steps

### 0. Pull Latest Changes

```bash
git pull
```
If there are conflicts, stop and ask the user for help.

### 1. Determine the Bank

Parse `$ARGUMENTS` for the bank name. If it is missing or unknown, list the supported banks and
ask the user which one to use.

### 2. Get Existing Transactions

Search `main.ledger` for the bank's asset account. Show the user the **last 3 transactions** and
the **current balance** from the most recent balance assertion (`= AMOUNT`), so they can confirm
where to pick up from. Read the last 100 lines of `main.ledger` to know the date range covered.

### 3. Get Bank Transactions

The user provides screenshots of the bank's transaction list. Extract from each entry the date,
the description or merchant, the amount and its currency, and the running balance when shown.

### 4. Map to Expense Accounts

1. Look for the same or a similar merchant earlier in `main.ledger` and reuse its account.
2. Otherwise guess from the merchant name with common sense (a supermarket is
   `Expenses:Food:Groceries`, a fuel station `Expenses:Transportation:Car:Gas`).
3. If unsure, ASK the user, one transaction at a time, with the date, the amount and your best
   guess. Do not guess silently.

Generic merchants such as Amazon say nothing about what was bought: always ask, and write the
answer into the description as `<Merchant> - <what was bought>`.

### 5. Compare with the Ledger

Match by date, amount and an approximate description; skip what is already recorded. Present the
list of NEW transactions to the user.

### 6. Format Transactions

```ledger
2024-03-01 * Description here
    Expenses:Category:Sub  12.34 EUR
    Assets:Cards:Bank  -12.34 EUR
```

Rules: cleared (`*`), four-space indentation, full account names, every posting with an explicit
amount and currency, amounts formatted like the existing entries.

### 7. Present and Confirm

Show every proposed transaction, numbered, and ask for confirmation before touching the ledger.
The user may correct accounts, descriptions or amounts, or skip lines.

### 8. Balance Assertion

As the very last transaction, dated today, assert the balance of every asset account that had
activity, using the balance shown in the screenshot (ask if it is not visible):

```ledger
2024-03-01 * Assert
    Expenses:Common:Missing                    0 EUR
    Assets:Cards:Bank                 = 1,234.56 EUR
```

If validation reports an unbalanced remainder, first look for duplicates or missing entries,
then put the remaining small discrepancy into `Expenses:Common:Missing`.

### 9. Append to main.ledger

After confirmation, append the new transactions followed by the assertion(s) to the very end of
`main.ledger`.

### 10. Validate

```bash
ledger --pedantic -f main.ledger bal > /dev/null
```
Fix until it passes. A missing account belongs in `accounts.ledger`; ask the user before adding one.

### 11. Commit and Push

```bash
git add main.ledger accounts.ledger
git commit -m "Add <bank> transactions <date range>"
git push
```
