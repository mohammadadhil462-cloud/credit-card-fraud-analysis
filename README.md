# Credit Card Fraud Analysis (Python + Power BI)

An end-to-end analysis of **100,000 credit card transactions** to answer a practical business question: *where is fraud concentrated, and which simple, explainable rules would stop the most fraud without drowning the review team in false alerts?*

The project has two parts that use the same dataset and give the same results:

| Part | Tools | What it does | Where |
|---|---|---|---|
| **1. Exploratory analysis** | Python (pandas, NumPy, Matplotlib, Seaborn), Jupyter | Cleans the data, finds fraud patterns, scores detection rules by net benefit | [`Fraud_detection.ipynb`](Fraud_detection.ipynb) |
| **2. Interactive dashboard** | Power BI, DAX | Presents the findings in a 3-page dashboard with a cost-per-false-alert selector | [`powerbi/`](powerbi/) |

---

## Key Results

| Metric | Value |
|---|---|
| Transactions analysed | 100,000 |
| Fraudulent transactions | 2,055 (**2.05%**) |
| Total transaction value | 12,687,197.67 |
| Total amount lost to fraud | 279,842.20 |
| Average fraud amount vs legitimate | 169.93 vs 125.97 |

### What the data says

- **Card-Not-Present fraud is the biggest loss driver.** It accounts for **53.8%** of all money lost (150,504), followed by Account-Takeover at 22.3%.
- **Online is the riskiest channel.** Fraud rate is **3.18%** online vs **0.80%** in-store.
- **A handful of countries are extreme outliers.** Romania (12.9%), Russia (11.4%), Nigeria (10.9%) and China (9.9%) have fraud rates 5-6x the average. Transactions flagged `high_risk_country` have an **11.05%** fraud rate vs 1.59% for the rest.
- **Fraud peaks overnight.** Between 1am and 4am the fraud rate is **7.5% to 11.3%**, versus roughly 2% overall. 3am is the worst hour (11.3%).
- **New accounts are high risk.** Accounts under 3 months old have a **4.85%** fraud rate, which falls steadily to 1.46% for accounts older than 3 years.
- **Chargeback history matters.** Customers with 2+ prior chargebacks show a fraud rate of **4.3% to 4.8%**, more than double those with none (1.76%).
- **Chip and PIN help.** Chip transactions: 0.75% fraud vs 2.65% without. PIN: 0.77% vs 2.34%.
- **Customer age, weekends and email domain show almost no difference** in fraud rate (all close to 2%), so they are not useful for rules.

### Which fraud rules are worth running?

Each rule was scored on **recall** (share of fraud caught), **precision** (share of alerts that are real fraud), and **net benefit** = money saved from caught fraud minus the cost of reviewing false alerts (assumed **5 per false alert**).

| Rule | Recall | Precision | Net benefit (cost = 5) |
|---|---|---|---|
| **Amount > 3x customer average** | 21.8% | 3.1% | **115,525** |
| International + card not present | 79.5% | 4.6% | 57,185 |
| High-risk country | 26.5% | **11.1%** | 54,291 |
| Combined (4 rules) | 59.8% | 5.0% | 45,671 |
| New account (<= 90 days) | 25.1% | 4.9% | 17,193 |
| 2+ chargebacks | 20.1% | 4.4% | 8,866 |
| Overnight (0-4am) | 7.3% | 6.1% | 6,015 |

**Takeaways**

1. **"Amount > 3x customer average" is the most profitable rule** and stays profitable at every review cost tested (1, 5 and 10).
2. **"High-risk country" is the most precise rule** and also stays profitable at every cost tested.
3. **"International + card not present" catches the most fraud (79.5%) but is costly** because it flags 35,749 transactions. It turns *negative* (-113,390) if each false alert costs 10.
4. **Stacking more rules is not automatically better.** The 4-rule combination catches 59.8% of fraud but earns less than the single best rule because it creates 23,502 false alerts.

---

## Charts

| | |
|---|---|
| ![Loss by fraud type](images/01_loss_by_fraud_type.png) | ![Fraud rate by channel](images/02_fraud_rate_by_channel.png) |
| ![Fraud rate by hour](images/04_fraud_rate_by_hour.png) | ![Fraud rate by account age](images/06_fraud_rate_by_account_age.png) |
| ![Fraud rate by merchant category](images/03_fraud_rate_by_category.png) | ![Monthly fraud rate](images/05_monthly_fraud_rate.png) |

![Net benefit by rule](images/07_net_benefit_by_rule.png)

---

## Power BI Dashboard

A three-page interactive dashboard turns the Python findings into a reporting tool. Full details are in the [Power BI README](powerbi/README.md) and the [Power BI documentation](docs/POWERBI_DOCUMENTATION.md).

**Executive Overview:** KPIs, loss by fraud type, fraud rate by channel and month
![Executive Overview](powerbi/screenshots/01_executive_overview.png)

**Fraud Patterns:** country, merchant category, account age, hour of day and chargebacks
![Fraud Patterns](powerbi/screenshots/02_fraud_patterns.png)

**Rule Performance:** rule scorecard, net benefit and recall vs precision, with an adjustable cost per false alert
![Rule Performance](powerbi/screenshots/03_rule_performance.png)

## Business Questions Answered

1. How big is the fraud problem?
2. Which fraud type costs the most?
3. Where does fraud concentrate (channel, merchant category, country, card security features)?
4. When does fraud happen (hour, weekday vs weekend, month)?
5. Which customers and accounts are most at risk (age, account age, chargebacks, state, email domain)?
6. Which simple rules would catch the most fraud, and are they worth the cost of false alerts?

## Data Cleaning Summary

- 100,000 rows and 43 columns, with **no duplicate transactions**.
- The `browser` column had **41,709 blanks**. They were structural (in-store and ATM transactions have no browser), so they were labelled `None (not online)` instead of deleting rows.
- `transaction_datetime` was stored as text and was converted to a datetime. `Month` and `Month_Name` columns were added.
- Derived columns `age_band` and `account_age_band` were created for segmentation.

## Methodology

- **Segmentation:** fraud rate (`is_fraud` mean) grouped by channel, category, country, hour, month, age band, account age band, chargeback count, state and email domain.
- **Rule evaluation:** each rule is a boolean filter on the data. Recall, precision and false alert counts are calculated against the `is_fraud` label.
- **Cost model:** `Net benefit = fraud_amount_lost on flagged fraud - (false alerts x cost per alert)`.
- **Sensitivity analysis:** the cost per false alert is varied (1, 5, 10) to show which rules are robust.

## Limitations and Assumptions

- The net benefit calculation assumes **every flagged fraud transaction is successfully blocked**, so it is an upper bound on savings.
- The cost per false alert (5) is an **assumption**, not a measured value. This is why the sensitivity analysis is included.
- Rules were evaluated on the **full dataset with no train/test split**, so results describe this data and are not a guaranteed forecast.
- This is an exploratory, rule-based analysis. No machine learning model is built.

## Project Structure

```
credit-card-fraud-analysis/
├── Fraud_detection.ipynb            # Python analysis notebook
├── README.md                        # Project overview (this file)
├── requirements.txt                 # Python dependencies
├── docs/
│   ├── PROJECT_DOCUMENTATION.md     # Python: data dictionary, methodology, detailed findings
│   └── POWERBI_DOCUMENTATION.md     # Power BI: data model, DAX measures, page specs, validation
├── images/                          # Charts exported from the notebook
└── powerbi/
    ├── README.md                    # Power BI project overview
    ├── Fraud_Analysis_Dashboard.pbix  # Power BI report (add your final .pbix here)
    └── screenshots/                 # Dashboard page screenshots
```

## How to Run

1. Clone the repository
   ```bash
   git clone https://github.com/<your-username>/credit-card-fraud-analysis.git
   cd credit-card-fraud-analysis
   ```
2. Install dependencies
   ```bash
   pip install -r requirements.txt
   ```
3. Download the dataset `credit_card_fraud_detection_2026.csv` ( **add the dataset link here** ) and place it in the project folder.
4. In the first notebook cell, update the file path to point to the CSV, for example:
   ```python
   x = pd.read_csv("credit_card_fraud_detection_2026.csv")
   ```
5. Launch Jupyter and run all cells
   ```bash
   jupyter notebook Fraud_detection.ipynb
   ```
6. To explore the dashboard, open [`powerbi/Fraud_Analysis_Dashboard.pbix`](powerbi/) in [Power BI Desktop](https://powerbi.microsoft.com/desktop/). The data is embedded in the file.

## Recommendations

1. Prioritise an **"amount > 3x customer average"** rule as the first line of defence, as it has the highest net benefit.
2. Add a **high-risk country** rule for precise, low-noise alerts.
3. Apply **extra authentication** (for example 3-D Secure or OTP) to online and card-not-present transactions rather than blocking them, since a blanket block creates too many false alerts.
4. Apply **stricter limits and monitoring to new accounts** (first 90 days) and to customers with 2+ chargebacks.
5. Increase monitoring on **overnight (1am-4am)** transactions, especially above-average amounts.
6. **Next step:** test the best rules *together* and build a classification model (for example logistic regression or gradient boosting) with a train/test split to compare against the rule-based approach.

## Tools Used

Python 3.13 | pandas | NumPy | Matplotlib | Seaborn | Jupyter Notebook | Power BI | DAX

## Author

**Mohammad Adhil**
MBA in Business Analytics | Data Analyst
Hyderabad, India

- GitHub: [your-username](https://github.com/your-username)
- LinkedIn: [add your LinkedIn link](https://www.linkedin.com/in/your-profile)
- Email: add your email
