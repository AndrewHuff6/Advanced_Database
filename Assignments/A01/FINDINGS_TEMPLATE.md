# FINDINGS — Assignment 01

> Copy this to `starter_code/FINDINGS.md` and fill it in. Keep answers
> short: one to three sentences each, and every answer should point at a
> **number** from `results.md` or a **plan line** from `results.csv`.

## Environment

- SQLite version (`sqlite3 --version`):
- Python version (`python --version`):
- Machine (CPU, RAM, SSD or spinning disk):
- Row counts (copy from the `make_dbs.py` output):

| Size | purchases | customers | products |
| :--- | --------: | --------: | -------: |
| 10k  |      10,000     |     500      |     100     |
| 100k |     100,000      |     5,000      |     500     |
| 1m   |      1,000,000     |     50,000      |     5,000     |

- Indexes in `idx_1m.db` (paste the output of `sqlite3 data/idx_1m.db ".indexes"`):

```
idx_cards_customer        idx_purchases_cust_amount     idx_purchases_date                 idx_purchases_department      idx_purchases_product
idx_customers_zipcode     idx_purchases_customer        idx_purchases_date_cust_amount     idx_purchases_prod_amount     idx_zipcodes_state
```

## Results

Paste your whole `results.md` here. All times are median milliseconds.

| Query | noidx_10k | idx_10k |
| :--- | ---: | ---: |
| Q01 | 0.06 | 0.06 |
| Q02 | 0.10 | 0.10 |
| Q03 | 1.59 | 0.45 |
| Q04 | 1.19 | 1.65 |
| Q05 | 1.35 | 0.08 |
| Q06 | 2.13 | 2.10 |
| Q07 | 3.32 | 1.42 |
| Q08 | 20.84 | 4.71 |
| Q09 | 12.34 | 2.64 |
| Q10 | 6.36 | 1.29 |
| Q11 | 0.49 | 0.40 |
| Q11-fast | 0.12 | 0.14 |
| Q12 | 2.02 | 1.85 |
| Q12-fast | 0.10 | 0.10 |
| Q13 | 7.63 | 5.83 |
| Q13-fast | 3.22 | 2.11 |
| Q14 | 12.57 | 9.07 |
| Q14-fast | 0.72 | 0.75 |
---

## Phase 1 — Simple reads (Q01–Q03)

| Query | Plan summary in `noidx_1m` | Plan summary in `idx_1m` |
| :---- | :------------------------- | :----------------------- |
| Q01   |"SEARCH c USING INTEGER PRIMARY KEY (rowid=?)", "SEARCH z USING INDEX sqlite_autoindex_zipcodes_1 (zipcode=?)" | "SEARCH c USING INTEGER PRIMARY KEY (rowid=?)","SEARCH z USING INDEX sqlite_autoindex_zipcodes_1(zipcode=?)"|
| Q03   |"SCAN purchases", "USE TEMP B-TREE FOR ORDER BY"|"SEARCH purchases USING INDEX idx_purchases_date (purchase_date>? AND purchase_date<?)"|

*A plan summary is the one or two `SCAN` / `SEARCH` lines that matter, e.g. `SCAN purchases + TEMP B-TREE FOR ORDER BY`.*

1. Why is Q01 fast even in `noidx_*`?

2. What does SQLite do differently for Q03 in the two databases?

## Phase 2 — Where indexes matter (Q04–Q10)

| Query | `noidx_1m` ms | `idx_1m` ms | Speedup (noidx ÷ idx) |
| :---- | ------------: | ----------: | --------------------: |
| Q04   |      82.70         |      42.31       |         1.95              |
| Q05   |      81.77         |       2.33      |            35.09           |
| Q06   |      84.06         |      42.91       |          1.96             |
| Q07   |      191.63         |      89.31       |          2.15             |
| Q08   |      4623.31         |      41.19       |            112.24           |
| Q09   |      2302.50         |      268.52       |          8.57             |
| Q10   |      840.99         |      117.97       |          7.13             |

1. Which query had the largest speedup? Quote its two plan summaries and explain the difference.

2. Q09 reads every purchase in both databases. Which word in its `idx_1m` plan explains why it's still faster?

3. Why did Q07 only get about 2× faster?

## Phase 3 — When an index can't save you (Q11–Q14)

| Query                          | slow, `noidx_1m` | slow, `idx_1m` | `/fast`, `idx_1m` |
| :----------------------------- | ---------------: | -------------: | ----------------: |
| Q11 offset → keyset            |       31.54           |        30.88        |         0.09          |
| Q12 random sort → random ids   |       120.02           |       117.27         |        0.13           |
| Q13 revenue by month → summary |       682.51         |       534.70         |        8.74           |
| Q14 cube → summary             |        1318.65          |       2307.55         |        7.14           |

For each, write one or two sentences: **why the slow version is slow** (point at the plan) and **what the rewrite gives up**.

- **Q11:**
- **Q12:**
- **Q13:**
- **Q14:** (also: why is the slow version slower in `idx_1m` than in `noidx_1m`?)

## Phase 4 — Write concurrency (Q15)

| Run  | busy_timeout | workers |    n | succeeded (201) | failed (503) | seconds | writes/sec |
| :--- | -----------: | ------: | ---: | --------------: | -----------: | ------: | ---------: |
| A    |            0 |       1 |  200 |        200         |      0        |    2.05     |     97.56       |
| A    |            0 |       1 |  500 |        93         |       407       |    8.52     |     58.69       |
| B    |         5000 |       1 |  200 |        62         |       138       |    1.45     |     137.93       |
| B    |         5000 |       1 |  500 |        142         |      358        |    8.40     |     59.52       |
| C    |            0 |       4 |  200 |        49         |      151        |    1.54     |      129.87      |
| C    |            0 |       4 |  500 |        195         |       305       |    8.36     |     59.80       |
| D    |         5000 |       4 |  200 |        200         |       0       |    1.60     |      125      |
| D    |         5000 |       4 |  500 |        500         |       0       |     9.06    |      55.19      |

Exact error text from a failed request:

```
INFO:     127.0.0.1:56978 - "POST /purchases?db=writes HTTP/1.1" 503 Service Unavailable
```

Summary-table check (`COUNT(*) FROM purchases` vs. `SUM(num_purchases) FROM monthly_sales`):

```
COUNT(*) = 102990
SUM(num_purchases) = 100000
```

1. What does `busy_timeout` do, and why did B succeed where A failed?

2. Did 4 worker processes make SQLite write faster? Why or why not?

3. Why don't the two summary-check numbers match? What does that cost the `/fast` routes from Phase 3?

## Phase 5 — Conclusions

For each scenario: **SQLite yes or no**, 1–2 sentences, and one number from your results.

1. Internal read-only dashboard, ~10 analysts:
2. Mobile app offline local storage:
3. Thousands of events per second from many servers:
4. Fresh throwaway database per automated test:
