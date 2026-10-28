# Banking Customer & Operations Analytics

An interactive Power BI report that helps bank management understand customer behaviour, account performance, transaction activity, the loan portfolio and customer support quality.

Built as Sprint 2 of the INSTANT Data Analysis Training. The brief was to think like a banking analyst: answer real business questions, not just draw charts.

**Tools:** Power BI Desktop, Power Query (M), DAX, Excel/CSV sources

---

## Business questions

| Area | Question |
|---|---|
| Customers | Who is active, who is drifting away, and who is at risk of leaving? |
| Accounts | Where does the money sit, and how much of it belongs to inactive customers? |
| Transactions | Is activity growing? Which transactions look unusual and need a second look? |
| Loans | How big and how well priced is the portfolio, and what matures next? |
| Support | Are customer problems actually being solved, and who keeps calling back? |

## Report pages

| Page | What it shows |
|---|---|
| Home | Navigation page with buttons to every page |
| Executive Overview | Headline KPIs: customers, activity rate, balances, loan portfolio, support resolution, at-risk customers |
| Customer Analysis | Growth, status mix, balance by status, tenure, states, product mix, top at-risk customers |
| Transaction Analysis | Value trend, money in vs out, amount bands, account activity levels, withdrawals above balance |
| Loan Analysis | Portfolio by type, rate vs exposure, maturity wall, rate mix, new lending by year |
| Customer Support | Call volume and resolution, resolution by issue and quarter, repeat callers, resolution vs target |

Every page has slicers on the left and a short insight line in the footer.

Screenshots go in `assets/`:

```
![Executive Overview](assets/executive-overview.png)
![Customer Analysis](assets/customer-analysis.png)
![Transaction Analysis](assets/transaction-analysis.png)
![Loan Analysis](assets/loan-analysis.png)
![Customer Support](assets/customer-support.png)
```

---

## Data

Training dataset provided by the program. Six tables:

| Table | Rows | Notes |
|---|---|---|
| Customers | 5,050 | Join date, state |
| Accounts | 5,100 | Type, balance, created date |
| Transactions | 20,000 | 12 May 2022 to 11 May 2025 |
| Loans | 2,600 | Type, amount, rate, start and end date |
| Cards | 4,000 | Type, issued and expiration date |
| Support calls | 3,100 | 11 May 2024 to 11 May 2025, resolved Yes/No |

**As-of date:** all "days since" and "active / matured" logic uses **11 May 2025**, the last date in the data.

---

## Data preparation (Power Query)

- Imported every table with parameters for the folder path and file name.
- Converted text timestamps to real dates with one reusable function.
- Trimmed text, standardised values and set correct data types.
- Removed duplicate keys (none were found).
- Dropped personal data that the analysis does not need: email, phone and address. Card numbers are reduced to the last four digits.
- Added helper columns: balance bands, rate bands, loan status, loan term, card status, resolved flag, high-value flag, money direction.
- Built a date table from 2015 to 2035 (loans end as late as 2035) and marked it as the date table.

The full scripts are in [`powerquery/Sprint2_PowerQuery_Scripts.m`](powerquery/Sprint2_PowerQuery_Scripts.m).

### Data quality findings

| Finding | Count | How it was handled |
|---|---|---|
| Nulls, duplicates, orphan keys | 0 | Verified on every table |
| Customers with no account | 1,851 (37%) | Kept. Shown as "No account" |
| Customers with no record in any other table | 50 (IDs 5001-5050) | Kept. Shown as "No products" |
| Accounts created before the customer joined | 2,488 (49%) | Kept and documented. The correct date is unknown |
| Transactions dated before the account was opened | 3,172 (16%) | Flagged in a column |
| Withdrawals larger than the account balance | 236 | Flagged and listed on the Transaction page |
| Accounts dated after the data date | 7 | Flagged (`IsFutureDated`) |
| Loans starting after the data date | 12 | Status "Not started", excluded from the active portfolio |
| State recorded as "Army" | 531 | Renamed "Military (APO/AE)" |

---

## Data model

Star/snowflake schema with customers and accounts as dimensions and transactions, loans, cards and calls as facts.

```
                        dim_date
        ┌────────┬────────┼────────┬────────┐
fact_transactions   fact_loans   fact_cards   fact_support_calls
        │               │            │              │
   dim_accounts         └────────────┴──────────────┤
        │                                           │
        └──────────────── dim_customers ────────────┘
```

| From (many) | To (one) | Active |
|---|---|---|
| fact_transactions[AccountID] | dim_accounts[AccountID] | Yes |
| dim_accounts[CustomerID] | dim_customers[CustomerID] | Yes |
| fact_loans[CustomerID] | dim_customers[CustomerID] | Yes |
| fact_cards[CustomerID] | dim_customers[CustomerID] | Yes |
| fact_support_calls[CustomerID] | dim_customers[CustomerID] | Yes |
| fact_transactions[TransactionDate] | dim_date[Date] | Yes |
| fact_support_calls[CallDate] | dim_date[Date] | Yes |
| fact_loans[LoanStartDate] | dim_date[Date] | Yes |
| fact_loans[LoanEndDate] | dim_date[Date] | No (maturity) |
| fact_cards[IssuedDate] | dim_date[Date] | Yes |
| fact_cards[ExpirationDate] | dim_date[Date] | No (expiry) |

All relationships are single direction. The date table is deliberately **not** linked to customer join dates or account created dates: that would create a second path to the fact tables. Join year and join month are columns on the customer table instead.

---

## Key definitions

| Term | Definition |
|---|---|
| Active customer | Account holder with a transaction in the last 90 days |
| Cooling | Last transaction 91 to 180 days ago |
| Inactive | Last transaction more than 180 days ago, or no transaction at all |
| At-risk customer | Inactive and with at least one unresolved support call |
| High-value transaction | Amount of 9,500 or more (about the top 5%) |
| Money out | Withdrawals and payments. Transfers are shown separately because the data does not say which way they go |
| Loan portfolio | Loans that started on or before the as-of date and end after it |

## Customer-level columns

Built on the customer table: number of accounts, total balance, total loan amount, number of cards, number of calls, unresolved calls, last transaction date, days since last transaction, customer status, risk flag, tenure band and product mix.

## Measures

**Customers:** Total Customers, Customers with Account, Account Penetration %, Active Customers 90d, Active Rate %, Inactive Customers, At-Risk Customers, Customers without Card, Cumulative Customers

**Accounts:** Total Accounts, Total Balance, Avg Balance per Account, Avg Balance per Customer, Inactive Balance, At-Risk Balance

**Transactions:** Total Transactions, Transaction Value, Money In, Money Out, Net Flow, High-Value Txns, Withdrawals above Balance, Txns before Account Open, Value L12M, Value Prior 12M, Value Growth L12M %

**Loans:** Active Loans, Loan Portfolio, Avg Interest Rate, Avg Loan Size, Maturing Next 12M, High-Rate Share %, Loan Originations

**Support:** Total Calls, Unresolved Calls, Support Resolution %, Customers Who Called, Repeat Callers, Customers with Unresolved

---

## Key insights

1. **Only 63% of customers hold an account**, and 541 customers hold no product at all.
2. **Activity is low.** 38% of account holders transacted in the last 90 days. 1,312 account holders are inactive and hold about **$89.5M**, around 35% of all balances.
3. **348 customers are at risk** (inactive with an open complaint), holding about $23.9M.
4. **Growth has stalled.** New customers stay around 500 a year, and transaction value is up only 0.8% over the last 12 months.
5. **Money out is about twice money in** ($49.8M against $25.0M, transfers excluded).
6. **Unusual activity:** 1,008 transactions of $9,500 or more, 236 withdrawals above the account balance, and 16% of transactions dated before their account existed.
7. **The loan portfolio is $632.5M across 2,555 active loans.** New lending fell from $146.5M in 2023 to $63.5M in 2024, while $114M to $132M matures every year from 2026 to 2029. About **$126.8M matures in the next 12 months**.
8. **Pricing does not follow loan type.** Average rates sit between 7.3% and 7.7% for the four large loan types, and 15% of the portfolio is priced at 11% or more. Business loans are only $4.4M but carry the highest average rate (9.4%).
9. **Borrowers hold little else with the bank.** 36% of borrowers have no account, and most have loans larger than their deposits.
10. **Support resolves only 49.3% of calls.** Card issues are the weakest (46.5%). 646 customers called more than once and 1,339 still have an unresolved call.

## Recommendations

1. **Win back inactive customers.** Start with the 348 at-risk customers, then the high-balance inactive ones, with a personal call and a clear offer.
2. **Fix first-contact resolution.** Set a resolution target, start recording resolution time, and review the card-issue process first. Close open complaints for customers who also hold loans.
3. **Plan the maturity wall.** Contact customers whose loans end in the next 12 months with refinancing offers, and restart new lending so the portfolio does not shrink.
4. **Review loan pricing.** Check why rates barely differ by loan type and why 15% of the book sits above 11%. Look at whether the small business segment deserves more attention.
5. **Cross-sell.** Offer cards to the 1,426 account holders without one, and offer accounts to borrowers who have none.
6. **Strengthen controls.** Review withdrawals above balance and transactions of $9,500 and above, and fix the source of transaction and account date mismatches.
7. **Improve data quality.** Validate dates at entry and store state values properly.

---

## Assumptions and limitations

- The as-of date is 11 May 2025. First and last months of transactions and calls are partial and are excluded from monthly charts.
- Support calls only carry a resolved Yes/No flag, so resolution time and SLA cannot be measured. The 70% target on the gauge is an assumption.
- Transfer direction is not in the data.
- Card data is modelled and used for product mix and cross-sell, but the report has no dedicated cards page. Card issuance and expiry trends are a natural next step.
- The conflicting dates between customers, accounts and transactions were kept and flagged rather than corrected.

## Challenges

- Choosing a date model: linking the date table to join and created dates created ambiguous paths, so only the fact dates are linked and the others are used through columns.
- Sorting text labels in order (tenure, product mix, bands) caused circular dependencies until the sort columns were built from the original numbers.
- Partial first and last months distorted trend charts and needed filtering.
- Inconsistent dates between tables (accounts before customers, transactions before accounts).
- No resolution time in the support data.

---

## Repository structure

```
.
├── sprint2_bi.pbix
├── powerquery/
│   └── Sprint2_PowerQuery_Scripts.m
├── data/                  # source files (optional)
├── docs/                  # insights and recommendations document, PDF export
├── assets/                # screenshots
└── README.md
```

## How to open

1. Open `sprint2_bi.pbix` in Power BI Desktop.
2. Go to **Home → Transform data → Edit parameters** and set `FolderPath` (with a trailing `\`) and `BankingFile` to where your source files are.
3. Click **Refresh**.

## Author

**[Sherif Azb]**
[LinkedIn](https://www.linkedin.com/in/sherifazb/) · [GitHub](https://github.com/SherifAzb)
