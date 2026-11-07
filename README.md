# SQL Portfolio Roadmap

A plan for eight SQL analytics projects on free, real-world datasets: four in business analytics and four in finance. Each project lists its business question, data source, planned schema, analysis steps with starter queries (PostgreSQL), and the findings it will report.

Completed projects live in their own repositories and are linked in the tracker below.

## Progress tracker

| # | Project | Domain | Key SQL skills | Status | Repository |
| --- | --- | --- | --- | --- | --- |
| 5 | Nifty 50 stock analysis | Finance | Rolling windows, drawdown, self-joins | 🟡 In progress | [nifty-fifty-analytics](https://github.com/areebalam49-lgtm/nifty-fifty-analytics) |
| 1 | E-commerce customer analysis (Olist) | Business | Multi-table joins, RFM, cohorts | ⚪ Planned | – |
| 2 | Food-delivery market study (Zomato) | Business | Text cleaning, normalisation, ranking | ⚪ Planned | – |
| 3 | Sales performance (Superstore) | Business | Star schema, YoY with LAG, pivots | ⚪ Planned | – |
| 4 | HR attrition analysis (IBM) | Business | Conditional aggregation, lift | ⚪ Planned | – |
| 6 | Loan default risk (Lending Club) | Finance | Banding, risk scoring | ⚪ Planned | – |
| 7 | Mutual fund performance | Finance | LATERAL date joins, CAGR | ⚪ Planned | – |
| 8 | Bank marketing campaign (UCI) | Finance | Funnels, cumulative shares | ⚪ Planned | – |

Status key: ⚪ Planned · 🟡 In progress · 🟢 Complete

## Contents

- [Tools and workflow](#tools-and-workflow)
- [Project 1: E-commerce customer analysis (Olist)](#project-1-e-commerce-customer-analysis-olist)
- [Project 2: Food-delivery market study (Zomato)](#project-2-food-delivery-market-study-zomato)
- [Project 3: Sales performance (Superstore)](#project-3-sales-performance-superstore)
- [Project 4: HR attrition analysis (IBM)](#project-4-hr-attrition-analysis-ibm)
- [Project 5: Nifty 50 stock analysis](#project-5-nifty-50-stock-analysis)
- [Project 6: Loan default risk (Lending Club)](#project-6-loan-default-risk-lending-club)
- [Project 7: Mutual fund performance](#project-7-mutual-fund-performance)
- [Project 8: Bank marketing campaign (UCI)](#project-8-bank-marketing-campaign-uci)

## Tools and workflow

| Tool | Purpose |
| --- | --- |
| PostgreSQL 16+ | Database (all queries below are PostgreSQL) |
| DBeaver / pgAdmin | Query editor and CSV import |
| Power BI Desktop | Dashboards |
| GitHub | Version control and write-ups |

Every project follows the same workflow:

1. **Frame the questions:** 5–8 business questions before touching the data.
2. **Inspect:** row counts, columns, date formats, missing values.
3. **Load raw:** a `raw` schema with every column as `TEXT`, loaded as-is.
4. **Clean:** typed tables in a `clean` schema with primary keys.
5. **Validate:** row counts, duplicate keys, value ranges, join fan-out.
6. **Analyse:** one `.sql` file per question, with the question and answer in a comment header.
7. **Report:** README with findings, charts and recommendations.

Standard repository layout:

```
project-name/
  data/       (links to source data; large CSVs are not committed)
  sql/
    01_create_raw.sql
    02_load.sql
    03_clean.sql
    04_validate.sql
    05_analysis_*.sql
  images/     (charts, ER diagram, dashboard screenshots)
  README.md
```

Standard loading pattern:

```sql
CREATE SCHEMA raw;
CREATE SCHEMA clean;

CREATE TABLE raw.orders (order_id TEXT, customer_id TEXT, order_date TEXT, amount TEXT);

\copy raw.orders FROM 'orders.csv' WITH (FORMAT csv, HEADER true);

CREATE TABLE clean.orders (
  order_id    TEXT PRIMARY KEY,
  customer_id TEXT NOT NULL,
  order_date  DATE,
  amount      NUMERIC(12,2)
);
INSERT INTO clean.orders
SELECT order_id, customer_id, order_date::DATE, NULLIF(amount,'')::NUMERIC
FROM raw.orders;

SELECT COUNT(*), COUNT(DISTINCT order_id), MIN(order_date), MAX(order_date) FROM clean.orders;
```

---

## Project 1: E-commerce customer analysis (Olist)

**Business question:** Who are the most valuable customers, how many come back, and where do late deliveries hurt satisfaction?

**Data:** [Brazilian E-Commerce Public Dataset by Olist](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce) on Kaggle. About 100K orders (2016–2018) across 9 linked tables.

| Table | Key columns | Joins to |
| --- | --- | --- |
| orders | order_id, customer_id, order_status, purchase / delivered / estimated dates | customers, items, payments, reviews |
| customers | customer_id, customer_unique_id, city, state | orders |
| order_items | order_id, product_id, seller_id, price, freight_value | orders, products, sellers |
| order_payments | order_id, payment_type, installments, payment_value | orders |
| order_reviews | order_id, review_score | orders |
| products | product_id, category name | category translation |
| sellers | seller_id, city, state | order_items |
| geolocation | zip prefix, lat, lng | optional |
| category_name_translation | Portuguese → English | products |

**Data note:** `customer_id` changes with every order; `customer_unique_id` identifies the real customer and is used for all repeat-purchase analysis.

**Analysis plan:**

1. Load all 9 tables with primary keys; composite key `(order_id, order_item_id)` on order_items.
2. Build a base view of delivered orders:

```sql
CREATE VIEW clean.v_orders AS
SELECT o.order_id, c.customer_unique_id AS customer,
       c.customer_state AS state,
       o.order_purchase_timestamp::date AS order_date,
       o.order_delivered_customer_date::date AS delivered_date,
       o.order_estimated_delivery_date::date AS estimated_date,
       SUM(p.payment_value) AS order_value
FROM clean.orders o
JOIN clean.customers c ON c.customer_id = o.customer_id
JOIN clean.order_payments p ON p.order_id = o.order_id
WHERE o.order_status = 'delivered'
GROUP BY 1,2,3,4,5,6;
```

3. RFM scoring with `NTILE(5)`:

```sql
WITH rfm AS (
  SELECT customer,
         (SELECT MAX(order_date) FROM clean.v_orders) - MAX(order_date) AS recency,
         COUNT(*) AS frequency,
         SUM(order_value) AS monetary
  FROM clean.v_orders GROUP BY customer
)
SELECT *,
  NTILE(5) OVER (ORDER BY recency DESC) AS r,
  NTILE(5) OVER (ORDER BY frequency)    AS f,
  NTILE(5) OVER (ORDER BY monetary)     AS m
FROM rfm;
```

4. Named segments (Champions, At Risk, Lost…) with `CASE`; customers and revenue per segment.
5. Revenue concentration: top-decile share of revenue with `NTILE(10)`.
6. Monthly cohort retention:

```sql
WITH firsts AS (
  SELECT customer, DATE_TRUNC('month', MIN(order_date)) AS cohort
  FROM clean.v_orders GROUP BY customer
), activity AS (
  SELECT f.cohort,
         (EXTRACT(YEAR FROM AGE(DATE_TRUNC('month', v.order_date), f.cohort))*12
          + EXTRACT(MONTH FROM AGE(DATE_TRUNC('month', v.order_date), f.cohort))) AS month_no,
         v.customer
  FROM clean.v_orders v JOIN firsts f USING (customer)
)
SELECT cohort, month_no, COUNT(DISTINCT customer) AS active
FROM activity GROUP BY 1,2 ORDER BY 1,2;
```

7. Late-delivery rate and average delay by state; review scores for late vs on-time orders.

**Findings to report:** top-10% revenue share · repeat-purchase rate · worst 3 states for late delivery · RFM segment sizes with a recommended action for each.

---

## Project 2: Food-delivery market study (Zomato)

**Business question:** Where should a new restaurant open, and what cuisine is under-served?

**Data:** Zomato Bangalore Restaurants dataset on Kaggle (about 51K rows): name, online_order, book_table, rate, votes, location, rest_type, cuisines, approx cost for two, listing type.

**Analysis plan:**

1. Load raw and rename awkward columns (`approx_cost(for two people)` → `cost_for_two`).
2. Clean and de-duplicate:

```sql
CREATE TABLE clean.restaurants AS
SELECT DISTINCT ON (name, location)
  ROW_NUMBER() OVER () AS rest_id,
  name, location, rest_type,
  (online_order = 'Yes') AS online_order,
  (book_table  = 'Yes') AS book_table,
  CASE WHEN rate IN ('NEW','-','') OR rate IS NULL THEN NULL
       ELSE SPLIT_PART(rate,'/',1)::NUMERIC END AS rating,
  votes::INT AS votes,
  REPLACE(cost_for_two, ',', '')::INT AS cost_for_two,
  cuisines
FROM raw.zomato
ORDER BY name, location, votes::INT DESC;
```

3. Normalise cuisines into a bridge table:

```sql
CREATE TABLE clean.rest_cuisine AS
SELECT rest_id, TRIM(c) AS cuisine
FROM clean.restaurants,
     UNNEST(STRING_TO_ARRAY(cuisines, ',')) AS c;
```

4. Locality overview: count, average rating, median cost (`PERCENTILE_CONT(0.5)`), % online ordering; `HAVING COUNT(*) >= 50`.
5. Price bands with `CASE` and ratings per band.
6. Top restaurants per locality with `DENSE_RANK()`.
7. White-space analysis (demand per competitor):

```sql
SELECT r.location, rc.cuisine,
       COUNT(*)                AS supply,
       SUM(r.votes)            AS demand_votes,
       ROUND(AVG(r.rating),2)  AS avg_rating,
       ROUND(SUM(r.votes)::NUMERIC / COUNT(*), 0) AS votes_per_restaurant
FROM clean.restaurants r
JOIN clean.rest_cuisine rc USING (rest_id)
GROUP BY 1,2
HAVING COUNT(*) BETWEEN 3 AND 15
ORDER BY votes_per_restaurant DESC
LIMIT 20;
```

8. Rating and votes with vs without online ordering and table booking.

**Findings to report:** most and least saturated localities · 2–3 recommended cuisine × locality entries · best-rated price band · online-ordering rating gap.

---

## Project 3: Sales performance (Superstore)

**Business question:** Where does the business make and lose money, and at what discount do orders become unprofitable?

**Data:** Sample Superstore dataset on Kaggle (about 10K order lines, 4 years).

**Data note:** dates are in M/D/YYYY format; parse with `TO_DATE(col, 'MM/DD/YYYY')`.

**Analysis plan:**

1. Split the flat file into a star schema:

```sql
CREATE TABLE clean.dim_product AS
SELECT DISTINCT product_id, category, sub_category, product_name FROM raw.superstore;

CREATE TABLE clean.fact_sales AS
SELECT row_id::INT, order_id,
       TO_DATE(order_date,'MM/DD/YYYY') AS order_date,
       TO_DATE(ship_date,'MM/DD/YYYY')  AS ship_date,
       customer_id, product_id, region, state,
       sales::NUMERIC, quantity::INT, discount::NUMERIC, profit::NUMERIC
FROM raw.superstore;
```

2. Profit and margin by region × category.
3. Discount bands vs margin:

```sql
SELECT CASE WHEN discount = 0      THEN '0%'
            WHEN discount <= 0.10  THEN '1-10%'
            WHEN discount <= 0.20  THEN '11-20%'
            WHEN discount <= 0.30  THEN '21-30%'
            ELSE 'over 30%' END AS discount_band,
       COUNT(*) AS lines,
       ROUND(SUM(profit)/SUM(sales)*100, 1) AS margin_pct
FROM clean.fact_sales
GROUP BY 1 ORDER BY 1;
```

4. Year-on-year growth:

```sql
WITH yearly AS (
  SELECT EXTRACT(YEAR FROM order_date) AS yr, SUM(sales) AS sales
  FROM clean.fact_sales GROUP BY 1
)
SELECT yr, sales,
       ROUND((sales - LAG(sales) OVER (ORDER BY yr)) / LAG(sales) OVER (ORDER BY yr) * 100, 1) AS yoy_pct
FROM yearly;
```

5. Year-to-date running totals; top and bottom 10 products by profit; days-to-ship by mode; category × year pivot.
6. Power BI dashboard: KPI cards, region map, discount-band chart, monthly trend.

**Findings to report:** loss-making discount threshold · loss-making sub-categories · YoY growth by year · profit lost on bottom 10 products.

---

## Project 4: HR attrition analysis (IBM)

**Business question:** Which employee groups leave most often, and what does attrition cost?

**Data:** IBM HR Analytics Employee Attrition & Performance dataset on Kaggle (1,470 rows; a fictional dataset created by IBM data scientists).

**Analysis plan:**

1. Load with a `left_flag` (1 = left) and drop constant columns.
2. Baseline attrition rate.
3. Attrition by department, role, overtime, travel:

```sql
SELECT department,
       COUNT(*) AS headcount,
       SUM(left_flag) AS leavers,
       ROUND(AVG(left_flag)*100, 1) AS attrition_pct
FROM clean.employees
GROUP BY department
ORDER BY attrition_pct DESC;
```

4. Income quartiles (`NTILE(4)`) and tenure bands (`CASE`).
5. Two-factor cross-tab:

```sql
WITH e AS (
  SELECT *, NTILE(4) OVER (ORDER BY monthly_income) AS income_q
  FROM clean.employees
)
SELECT income_q,
  ROUND(AVG(CASE WHEN overtime='Yes' THEN left_flag END)*100,1) AS ot_yes_pct,
  ROUND(AVG(CASE WHEN overtime='No'  THEN left_flag END)*100,1) AS ot_no_pct
FROM e GROUP BY income_q ORDER BY income_q;
```

6. Lift vs baseline for each group; a combined high-risk profile.
7. Cost estimate under a stated replacement-cost assumption.

**Findings to report:** baseline rate and highest-risk groups · overtime effect · high-risk profile size and rate · estimated annual attrition cost.

---

## Project 5: Nifty 50 stock analysis

**Business question:** Which Nifty 50 stocks delivered the best return for the risk taken, and how did they behave in drawdowns?

**Data:** NIFTY-50 Stock Market Data (2000–2021) on Kaggle (one CSV per stock plus metadata with industry). Index benchmark from [niftyindices.com](https://www.niftyindices.com).

**Data note:** prices are not adjusted for splits and bonus issues; days with jumps over 25% are reviewed and excluded from return calculations.

**Analysis plan:**

1. Schema:

```sql
CREATE TABLE clean.stocks (
  symbol TEXT PRIMARY KEY, company TEXT, industry TEXT
);
CREATE TABLE clean.prices (
  symbol TEXT REFERENCES clean.stocks, trade_date DATE,
  open NUMERIC, high NUMERIC, low NUMERIC, close NUMERIC,
  volume BIGINT,
  PRIMARY KEY (symbol, trade_date)
);
```

2. Load only `series = 'EQ'` rows.
3. Daily returns and split detection:

```sql
CREATE TABLE clean.returns AS
SELECT symbol, trade_date, close,
       close / LAG(close) OVER (PARTITION BY symbol ORDER BY trade_date) - 1 AS daily_ret
FROM clean.prices;

SELECT * FROM clean.returns WHERE ABS(daily_ret) > 0.25 ORDER BY symbol, trade_date;
```

4. 50/200-day moving averages and crossover signals:

```sql
WITH ma AS (
  SELECT symbol, trade_date, close,
    AVG(close) OVER (PARTITION BY symbol ORDER BY trade_date
                     ROWS BETWEEN 49 PRECEDING AND CURRENT ROW)  AS ma50,
    AVG(close) OVER (PARTITION BY symbol ORDER BY trade_date
                     ROWS BETWEEN 199 PRECEDING AND CURRENT ROW) AS ma200
  FROM clean.prices
)
SELECT symbol, trade_date,
  CASE WHEN ma50 > ma200 AND LAG(ma50) OVER w <= LAG(ma200) OVER w THEN 'Golden cross'
       WHEN ma50 < ma200 AND LAG(ma50) OVER w >= LAG(ma200) OVER w THEN 'Death cross' END AS signal
FROM ma
WINDOW w AS (PARTITION BY symbol ORDER BY trade_date);
```

5. Annualised return, volatility and return per unit of risk:

```sql
SELECT symbol,
  ROUND(AVG(daily_ret)*252*100, 2)            AS ann_return_pct,
  ROUND(STDDEV(daily_ret)*SQRT(252)*100, 2)   AS ann_vol_pct,
  ROUND(AVG(daily_ret)*252 / (STDDEV(daily_ret)*SQRT(252)), 2) AS return_per_risk
FROM clean.returns
WHERE ABS(daily_ret) < 0.25
GROUP BY symbol ORDER BY return_per_risk DESC;
```

6. Best and worst stock per year (`FIRST_VALUE` / `LAST_VALUE`, `RANK()`).
7. Maximum drawdown:

```sql
WITH peaks AS (
  SELECT symbol, trade_date, close,
    MAX(close) OVER (PARTITION BY symbol ORDER BY trade_date
                     ROWS UNBOUNDED PRECEDING) AS running_peak
  FROM clean.prices
)
SELECT symbol, ROUND(MIN(close/running_peak - 1)*100, 1) AS max_drawdown_pct
FROM peaks GROUP BY symbol ORDER BY max_drawdown_pct;
```

8. Sector averages; COVID-crash window (20 Feb – 23 Mar 2020) via self-join.

**Findings to report:** top and bottom 5 by risk-adjusted return · deepest drawdown and recovery time · golden-cross hit rate · most resilient sectors in March 2020.

---

## Project 6: Loan default risk (Lending Club)

**Business question:** Which borrowers default most, and can a simple points-based score separate good from bad loans?

**Data:** Lending Club accepted loans 2007–2018 on Kaggle (2M+ rows; analysis uses a subset of issue years).

**Data note:** only finished loans (Fully Paid, Charged Off, Default) have a known outcome; current loans are excluded.

**Analysis plan:**

1. Clean fields:

```sql
CREATE TABLE clean.loans AS
SELECT id,
  loan_amnt::NUMERIC AS amount,
  TRIM(term) AS term,
  REPLACE(int_rate,'%','')::NUMERIC AS int_rate,
  grade, purpose, home_ownership,
  NULLIF(annual_inc,'')::NUMERIC AS annual_inc,
  NULLIF(dti,'')::NUMERIC AS dti,
  emp_length,
  TO_DATE(issue_d, 'Mon-YYYY') AS issue_date,
  CASE WHEN loan_status IN ('Charged Off','Default') THEN 1 ELSE 0 END AS defaulted
FROM raw.loans
WHERE loan_status IN ('Fully Paid','Charged Off','Default');
```

2. Baseline default rate overall and by issue year; default by grade.
3. Banded analysis:

```sql
SELECT CASE WHEN annual_inc < 40000  THEN '1. under 40K'
            WHEN annual_inc < 70000  THEN '2. 40-70K'
            WHEN annual_inc < 100000 THEN '3. 70-100K'
            ELSE '4. 100K+' END AS income_band,
       COUNT(*) AS loans,
       ROUND(AVG(defaulted)*100, 1) AS default_pct
FROM clean.loans GROUP BY 1 ORDER BY 1;
```

4. Default by purpose and term; ranked two-factor segments (min 500 loans).
5. Rule-based risk score:

```sql
WITH scored AS (
  SELECT *,
    (CASE WHEN grade IN ('E','F','G') THEN 3 WHEN grade IN ('C','D') THEN 2 ELSE 0 END)
  + (CASE WHEN term = '60 months' THEN 2 ELSE 0 END)
  + (CASE WHEN dti > 25 THEN 2 WHEN dti > 15 THEN 1 ELSE 0 END)
  + (CASE WHEN annual_inc < 40000 THEN 2 ELSE 0 END)
  + (CASE WHEN purpose = 'small_business' THEN 1 ELSE 0 END) AS risk_score
  FROM clean.loans
)
SELECT risk_score, COUNT(*) AS loans, ROUND(AVG(defaulted)*100,1) AS default_pct
FROM scored GROUP BY risk_score ORDER BY risk_score;
```

6. Simplified expected loss by grade (default rate × amount lent, recoveries ignored).

**Findings to report:** grade A vs G default rate · 3 riskiest segments · default rate at lowest vs highest score · whether interest rates compensate for risk.

---

## Project 7: Mutual fund performance

**Business question:** Do Indian active equity funds beat the Nifty 50, and which do so consistently?

**Data:** NAV history from [AMFI](https://www.amfiindia.com) (or Kaggle mutual fund datasets); Nifty 50 TRI from [niftyindices.com](https://www.niftyindices.com). Scope: 15–30 large-cap and flexi-cap Direct Growth plans.

**Analysis plan:**

1. Schema:

```sql
CREATE TABLE clean.schemes (
  scheme_code INT PRIMARY KEY, scheme_name TEXT, amc TEXT, category TEXT
);
CREATE TABLE clean.nav (
  scheme_code INT REFERENCES clean.schemes, nav_date DATE, nav NUMERIC,
  PRIMARY KEY (scheme_code, nav_date)
);
CREATE TABLE clean.benchmark (index_date DATE PRIMARY KEY, tri NUMERIC);
```

2. Nearest-date NAV lookup (handles market holidays):

```sql
SELECT s.scheme_code, s.scheme_name,
       now_nav.nav AS nav_end, past.nav AS nav_start
FROM clean.schemes s
CROSS JOIN LATERAL (
  SELECT nav FROM clean.nav n
  WHERE n.scheme_code = s.scheme_code AND n.nav_date <= DATE '2026-10-31'
  ORDER BY nav_date DESC LIMIT 1) now_nav
CROSS JOIN LATERAL (
  SELECT nav FROM clean.nav n
  WHERE n.scheme_code = s.scheme_code AND n.nav_date <= DATE '2023-10-31'
  ORDER BY nav_date DESC LIMIT 1) past;
```

3. CAGR for 1, 3 and 5 years: `POWER(nav_end / nav_start, 1.0 / years) - 1`.
4. Excess return over Nifty 50 TRI; rank and quartile within category.
5. Rolling 1-year returns at each month-end; % of periods beating the benchmark.
6. Volatility, return per unit of risk, and drawdown.

**Findings to report:** share of funds beating the TRI over 3 and 5 years · most consistent fund · rank persistence of top performers.

---

## Project 8: Bank marketing campaign (UCI)

**Business question:** Who should a bank call for a term-deposit campaign, and how many calls are worth making?

**Data:** [UCI Bank Marketing](https://archive.ics.uci.edu/dataset/222/bank+marketing) dataset, `bank-additional-full.csv` (41,188 rows, semicolon-separated).

**Data notes:** `pdays = 999` means never contacted before; `duration` is only known after a call ends, so it is not used for targeting.

**Analysis plan:**

1. Load and type:

```sql
\copy raw.bank FROM 'bank-additional-full.csv' WITH (FORMAT csv, HEADER true, DELIMITER ';');

CREATE TABLE clean.contacts AS
SELECT ROW_NUMBER() OVER () AS contact_id,
  age::INT, job, marital, education, contact, month, day_of_week,
  campaign::INT AS calls,
  NULLIF(pdays,'999')::INT AS days_since_prev,
  previous::INT, poutcome,
  (y = 'yes')::INT AS converted
FROM raw.bank;
```

2. Baseline conversion; campaign funnel.
3. Conversion by job, age band, education, month and contact type.
4. Diminishing returns on call attempts:

```sql
SELECT LEAST(calls, 10) AS call_attempts,
       COUNT(*) AS customers,
       ROUND(AVG(converted)*100, 1) AS conversion_pct
FROM clean.contacts
GROUP BY 1 ORDER BY 1;
```

5. Previous-campaign outcome effect.
6. Targeting curve (share of conversions captured vs share of customers called):

```sql
WITH seg AS (
  SELECT job, contact, COUNT(*) n, SUM(converted) yes
  FROM clean.contacts GROUP BY 1,2
), ranked AS (
  SELECT *, SUM(n)   OVER (ORDER BY yes::NUMERIC/n DESC) * 1.0 / SUM(n)   OVER () AS cum_share_called,
            SUM(yes) OVER (ORDER BY yes::NUMERIC/n DESC) * 1.0 / SUM(yes) OVER () AS cum_share_converted
  FROM seg
)
SELECT * FROM ranked ORDER BY cum_share_called;
```

**Findings to report:** best and worst segments with lift · call-attempt cut-off · previous-success effect · conversions captured by calling the top segments.

---

*Datasets belong to their respective publishers and are used for educational analysis only. Large data files are not stored in this repository.*
