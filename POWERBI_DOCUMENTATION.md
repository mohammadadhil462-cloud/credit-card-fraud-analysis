# Power BI Documentation: Credit Card Fraud Dashboard

This document describes how the Power BI dashboard was built: the data source, the data model, every DAX measure, the content of each page, and how the results were checked against the Python analysis. The overview and screenshots are in the [Power BI README](../powerbi/README.md).

## 1. Objective

Present the fraud findings from the Python analysis as an interactive dashboard that answers three questions:

1. How big is the fraud problem, and where does the money go?
2. Where, when and among which customers is fraud concentrated?
3. Which simple fraud rules are worth running once the cost of reviewing false alerts is counted?

The dashboard is aimed at fraud, risk and operations teams, and at hiring managers reviewing a data analyst portfolio.

## 2. Data Source and Preparation

| Step | Detail |
|---|---|
| Source dataset | `credit_card_fraud_detection_2026.csv`, 100,000 transactions, January to December 2026 |
| Cleaning | Done in Python (see the [Python documentation](PROJECT_DOCUMENTATION.md)): blank `browser` values labelled `None (not online)`, `transaction_datetime` converted to datetime, `Month`, `Month_Name`, `age_band` and `account_age_band` added |
| Export | `x.to_csv("fraud_cleaned.csv", index=False)` produces a 100,000 row by 47 column file |
| Load | Power BI Desktop, **Get data → Text/CSV**, then **Transform Data** to check the column types |

**Column types checked in Power Query**

| Column | Type |
|---|---|
| `transaction_datetime` | Date/Time |
| `transaction_amount`, `fraud_amount_lost`, `amount_vs_avg_ratio` | Decimal number |
| `is_fraud`, `transaction_hour`, `account_age_days`, `num_chargebacks_ever` | Whole number |

**Load check:** 100,000 transactions and 2,055 fraud transactions, matching the Python results.

## 3. Data Model

| Table | Type | Rows | Purpose |
|---|---|---|---|
| `Fraud` | Imported | 100,000 | Transaction-level fact table |
| `Date` | Calculated table | 365 | Calendar dimension for 2026 |
| `Rule Hits` | Calculated table | about 102,000 | One row per transaction flagged by each rule |
| `Cost per False Alert` | What-if parameter | 20 | Cost per false alert, values 1 to 20, default 5 |

**Relationships**

```
Date[Date]  ------------------>  Fraud[Transaction Date]      (one to many)
Rule Hits[transaction_id]  --->  Fraud[transaction_id]        (many to one, single direction)
```

`Cost per False Alert` is a disconnected table. It is read only through its measure.

### Calendar table

```dax
Date =
ADDCOLUMNS(
    CALENDAR(DATE(2026,1,1), DATE(2026,12,31)),
    "Month Number", MONTH([Date]),
    "Month", FORMAT([Date], "MMM"),
    "Day Name", FORMAT([Date], "dddd")
)
```

`Date[Month]` is sorted by `Date[Month Number]` so months appear in calendar order.

### Rule Hits table

Each rule becomes a block of rows (transaction and rule name), and the blocks are stacked with `UNION`. This lets one set of measures score all seven rules in a single table or chart.

| Rule | Logic |
|---|---|
| High-risk country | `high_risk_country = 1` |
| Overnight (0-4am) | `transaction_hour <= 4` |
| New account (<=90 days) | `account_age_days <= 90` |
| 2+ chargebacks | `num_chargebacks_ever >= 2` |
| International + card not present | `international_transaction = 1` and `card_present = 0` |
| Amount > 3x customer average | `amount_vs_avg_ratio > 3` |
| Combined (4 rules) | High-risk country, overnight, new account or 2+ chargebacks |

```dax
Rule Hits =
UNION(
    SELECTCOLUMNS(FILTER(Fraud, Fraud[high_risk_country] = 1),
        "transaction_id", Fraud[transaction_id], "Rule", "High-risk country"),
    SELECTCOLUMNS(FILTER(Fraud, Fraud[transaction_hour] <= 4),
        "transaction_id", Fraud[transaction_id], "Rule", "Overnight (0-4am)"),
    SELECTCOLUMNS(FILTER(Fraud, Fraud[account_age_days] <= 90),
        "transaction_id", Fraud[transaction_id], "Rule", "New account (<=90 days)"),
    SELECTCOLUMNS(FILTER(Fraud, Fraud[num_chargebacks_ever] >= 2),
        "transaction_id", Fraud[transaction_id], "Rule", "2+ chargebacks"),
    SELECTCOLUMNS(FILTER(Fraud, Fraud[international_transaction] = 1 && Fraud[card_present] = 0),
        "transaction_id", Fraud[transaction_id], "Rule", "International + card not present"),
    SELECTCOLUMNS(FILTER(Fraud, Fraud[amount_vs_avg_ratio] > 3),
        "transaction_id", Fraud[transaction_id], "Rule", "Amount > 3x customer average"),
    SELECTCOLUMNS(FILTER(Fraud, Fraud[high_risk_country] = 1 || Fraud[transaction_hour] <= 4
                         || Fraud[account_age_days] <= 90 || Fraud[num_chargebacks_ever] >= 2),
        "transaction_id", Fraud[transaction_id], "Rule", "Combined (4 rules)")
)
```

### What-if parameter

Created with **Modeling → New parameter → Numeric range**: name `Cost per False Alert`, whole number, minimum 1, maximum 20, increment 1, default 5. It creates the measure `[Cost per False Alert Value]`.

## 4. Measures

### Core measures (Fraud table)

| Measure | Purpose | DAX |
|---|---|---|
| Total Transactions | Number of transactions | `COUNTROWS(Fraud)` |
| Fraud Transactions | Number of fraud cases | `SUM(Fraud[is_fraud])` |
| Fraud Rate % | Share of transactions that are fraud | `DIVIDE([Fraud Transactions], [Total Transactions])` |
| Total Amount | Total transaction value | `SUM(Fraud[transaction_amount])` |
| Fraud Loss | Money lost to fraud | `SUM(Fraud[fraud_amount_lost])` |
| Avg Fraud Amount | Average value of a fraud transaction | `CALCULATE(AVERAGE(Fraud[transaction_amount]), Fraud[is_fraud] = 1)` |
| Avg Legit Amount | Average value of a legitimate transaction | `CALCULATE(AVERAGE(Fraud[transaction_amount]), Fraud[is_fraud] = 0)` |

### Rule measures (Rule Hits table)

| Measure | Purpose | DAX |
|---|---|---|
| Rule Flagged | Transactions flagged by the rule | `COUNTROWS('Rule Hits')` |
| Rule Frauds Caught | Flagged transactions that are fraud | `SUMX('Rule Hits', RELATED(Fraud[is_fraud]))` |
| Rule False Alerts | Flagged transactions that are legitimate | `[Rule Flagged] - [Rule Frauds Caught]` |
| Rule Recall % | Share of all fraud the rule catches | `DIVIDE([Rule Frauds Caught], [Fraud Transactions])` |
| Rule Precision % | Share of alerts that are real fraud | `DIVIDE([Rule Frauds Caught], [Rule Flagged])` |
| Rule Money Saved | Fraud loss on the flagged fraud | `SUMX('Rule Hits', RELATED(Fraud[fraud_amount_lost]))` |
| Rule Alert Cost | Cost of reviewing false alerts | `[Rule False Alerts] * [Cost per False Alert Value]` |
| Rule Net Benefit | Money saved minus review cost | `[Rule Money Saved] - [Rule Alert Cost]` |

**Why a bridge table?** A transaction can be flagged by several rules, so a single yes/no column cannot hold the result. A long table with one row per transaction and rule avoids repeating long conditional formulas and lets all seven rules appear in one visual.

## 5. Page-by-Page Specification

The canvas is 1920 x 1080. Every page has a navy header band, a light grey background, and white cards with rounded corners.

### Page 1: Executive Overview

| Visual | Fields | Purpose |
|---|---|---|
| Slicer | `merchant_category` | Filters the whole page |
| KPI cards (5) | Total Transactions, Total Amount, Fraud Transactions, Fraud Rate %, Fraud Loss | Headline numbers. Volume cards are navy and fraud cards are red |
| Clustered bar | `fraud_type`, Fraud Loss (Legitimate excluded) | Which fraud type costs the most |
| Clustered column | `transaction_channel`, Fraud Rate % | Fraud rate by channel, Online highlighted |
| Line chart | `Month`, Fraud Rate % | Monthly trend with an average reference line |
| Text boxes | Three insight cards | 53.8% of losses from Card-Not-Present, online 4x riskier than in-store, 2.2% of value lost |

### Page 2: Fraud Patterns

| Visual | Fields | Highlighted |
|---|---|---|
| Bar chart: merchant country | `merchant_country`, Fraud Rate % | Romania, Russia, Nigeria, China |
| Bar chart: merchant category | `merchant_category`, Fraud Rate % | Online Retail, Travel & Airlines, Entertainment & Streaming |
| Column chart: account age | `account_age_band`, Fraud Rate % | Under 3 months |
| Column chart: hour of day | `transaction_hour`, Fraud Rate % | Hours 1 to 4 |
| Column chart: prior chargebacks | `num_chargebacks_ever`, Fraud Rate % | 2, 3 and 4 chargebacks |
| Text boxes | Three insight cards | 11.05% in high-risk countries, 11.3% at 3am, 4.85% on new accounts |

Highlighted bars are red and the rest are grey, so the eye goes to the high-risk groups.

### Page 3: Rule Performance

| Visual | Fields | Purpose |
|---|---|---|
| Slicer (dropdown) | `Cost per False Alert` | Sets the cost of each false alert |
| Text box | Formula explanation | Explains how net benefit is calculated |
| Table | Rule, Flagged, Frauds Caught, Recall %, Precision %, False Alerts, Money Saved, Net Benefit | Scorecard sorted by net benefit |
| Bar chart | Rule, Rule Net Benefit | Ranks the rules by profit |
| Scatter chart | X = Recall %, Y = Precision %, Size = Flagged | Shows the trade-off between catching fraud and creating noise |
| Text boxes | Three insight cards | Best rule, widest-catching rule, most precise rule |

## 6. Design Choices

| Element | Choice |
|---|---|
| Colours | Navy `#1F2A44` for structure, red `#C0392B` for fraud, grey `#8C96AE` for non-highlighted data, background `#F4F5F7` |
| Layout | Header band, then KPIs or filters, then charts, then written insights |
| Highlighting | Only the groups that matter are red, so the key finding stands out without annotations |
| Insight text | Each page ends with three short takeaways, so a reader gets the message without studying the charts |
| Rule ranking | Rules are ranked by net benefit rather than by fraud caught, because the business cares about money, not detection counts |

## 7. Validation: Power BI vs Python

Cost per false alert = 5.

**Totals**

| Metric | Python | Power BI |
|---|---|---|
| Transactions | 100,000 | 100,000 |
| Fraud transactions | 2,055 | 2,055 |
| Fraud rate | 2.05% | 2.06% (exactly 2.055%, rounded differently) |
| Total amount | 12,687,197.67 | 12.69M |
| Fraud loss | 279,842.20 | 279.84K |

**Rule scorecard**

| Rule | Flagged | Frauds caught | Recall | Precision | False alerts | Money saved | Net benefit |
|---|---|---|---|---|---|---|---|
| Amount > 3x customer average | 14,645 | 447 | 21.8% | 3.1% | 14,198 | 186,515 | 115,525 |
| International + card not present | 35,749 | 1,634 | 79.5% | 4.6% | 34,115 | 227,760 | 57,185 |
| High-risk country | 4,932 | 545 | 26.5% | 11.1% | 4,387 | 76,226 | 54,291 |
| Combined (4 rules) | 24,730 | 1,228 | 59.8% | 5.0% | 23,502 | 163,181 | 45,671 |
| New account (<=90 days) | 10,608 | 515 | 25.1% | 4.9% | 10,093 | 67,658 | 17,193 |
| 2+ chargebacks | 9,319 | 414 | 20.1% | 4.4% | 8,905 | 53,391 | 8,866 |
| Overnight (0-4am) | 2,456 | 150 | 7.3% | 6.1% | 2,306 | 17,545 | 6,015 |

All values match the Python notebook.

**Sensitivity check:** at a cost of 10, "International + card not present" shows a net benefit of -113,390, and at a cost of 1, "Amount > 3x customer average" shows 172,317. Both match the Python sensitivity table.

## 8. Insights and Recommendations

1. Run an **"amount > 3x customer average"** rule first. It has the highest net benefit and stays profitable at every review cost tested.
2. Add a **high-risk country** rule for precise, low-noise alerts (11.1% precision).
3. Use **step-up authentication** (for example an OTP) on online and card-not-present transactions instead of blocking them. A blanket block creates 34,115 false alerts.
4. Apply **tighter limits to accounts in their first 90 days** and to customers with 2 or more chargebacks.
5. Monitor **1am to 4am** transactions more closely, especially above-average amounts.

## 9. Limitations and Known Issues

- Net benefit assumes every flagged fraud is successfully blocked, so it is an upper bound on savings.
- The cost per false alert is an assumption, which is why the dashboard lets the reader change it.
- Rules are scored on the full dataset with no train/test split, so the results describe this data and are not a forecast.
- Only one combination of rules (the 4-rule OR) is tested. Layered rules are not.
- The insight text boxes are fixed text based on the default cost of 5. They do not change when slicers change.
- The monthly trend chart uses the month number (1 to 12) on the axis.
- No row-level security or scheduled refresh is configured. This is a portfolio report built on a static file.

## 10. Maintenance

- **Refreshing data:** re-run the Python notebook to regenerate `fraud_cleaned.csv`, then in Power BI use **Transform data → Data source settings** to point to the new file and click **Refresh**.
- **Adding a rule:** add one more `SELECTCOLUMNS(FILTER(...))` block to the `Rule Hits` table. All measures, the table, the bar chart and the scatter chart pick it up automatically.
- **Changing the cost range:** edit the `Cost per False Alert` parameter in **Modeling**.

## 11. Future Improvements

- Test layered rules such as "amount > 3x average AND high-risk country".
- Add a predictive model in Python and compare its net benefit with the rule-based approach.
- Add drill-through from a rule to the individual flagged transactions.
- Make the insight text dynamic with DAX text measures.
- Add distance and velocity features (`distance_from_home_km`, `transactions_last_1h`), which are not analysed yet.
