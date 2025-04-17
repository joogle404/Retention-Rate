# Calculating Retention Rate at Meta  

This article explores how to calculate user retention across months using SQL. I found the original problem statement on [Stratascratch](https://platform.stratascratch.com/coding/2053-retention-rate?code_type=1)
I broke down a seemingly simple problem—monthly retention rate per account—into clear steps using CTEs, date filtering, and conditional logic. It was a great exercise in cohort analysis, handling edge cases, and writing clean, modular SQL that mirrors real-world product analytics work.

## Problem Overview

You are given a dataset tracking user activity on Meta platforms. Each row contains:

- `activity_date`: the date of the user activity  
- `account_id`: the organization or group the activity is tied to  
- `user_id`: the ID of the user performing the activity  

Each row represents a user’s activity on a specific date for a particular account.


## Objective

Calculate the **monthly retention rate** for each `account_id` for:

- **December 2020**
- **January 2021**

A user is considered **retained** in a month if:
- They were active in that month **AND**
- They had activity in **any future month**.

### For example:
- A user active in **December 2020** and again in **January 2021** (or later) is considered **retained** for December.
- A user active in **January 2021** and again in **February 2021** (or later) is considered **retained** for January.


## Final Output

For each `account_id`, calculate:  
Retention Rate Ratio = (Retention Rate in Jan 2021) / (Retention Rate in Dec 2020)  

If there are **no users retained** in December 2020 for an account, set the ratio to **0**.  
The final output should include:
- `account_id`
- `retention_rate_ratio`

## Solution Walkthrough

This solution calculates monthly user retention rates for **December 2020** and **January 2021**, grouped by `account_id`, and then computes the **ratio** of January's retention to December's.


### Understanding the Data

The dataset comes from a table called `sf_events`, which includes:

- `account_id` – the organization the user belongs to  
- `user_id` – the user performing the activity  
- `record_date` – the date the activity happened  

We’ll use this information to measure whether users returned in future months after being active in December or January.

### The Problem

We need to:

1. Calculate retention rate for each `account_id` in **Dec 2020** and **Jan 2021**  
   → A user is **retained** if they were active in the month **and** returned in a future month.

2. Compute the **retention ratio** = (Retention Rate in Jan 2021) / (Retention Rate in Dec 2020)

3. If no users were retained in December, the ratio should default to `0`.


### Step-by-Step Breakdown

#### 1. Find the Latest Activity Date for Each User

```sql
latest_user_activity AS (
SELECT user_id, account_id, MAX(record_date) AS last_active_date
FROM sf_events
GROUP BY user_id, account_id
)
```
We use this to determine if a user was active in future months.  

#### 2. Get Users Active in Each Month  

```sql
dec_users AS (
  SELECT DISTINCT account_id, user_id
  FROM sf_events
  WHERE DATE_TRUNC('month', record_date) = '2020-12-01'
),

jan_users AS (
  SELECT DISTINCT account_id, user_id
  FROM sf_events
  WHERE DATE_TRUNC('month', record_date) = '2021-01-01'
)
```
This filters down to users active in Dec 2020 and Jan 2021.  

#### 3. Claculate Retention Rate For Each Month  

```sql
retention_dec AS (
  SELECT 
    d.account_id,
    SUM(CASE WHEN l.last_active_date > '2020-12-31' THEN 1 ELSE 0 END) * 1.0 / COUNT(*) AS dec_retention_rate
  FROM dec_users d
  JOIN latest_user_activity l ON d.user_id = l.user_id AND d.account_id = l.account_id
  GROUP BY d.account_id
),

retention_jan AS (
  SELECT 
    j.account_id,
    SUM(CASE WHEN l.last_active_date > '2021-01-31' THEN 1 ELSE 0 END) * 1.0 / COUNT(*) AS jan_retention_rate
  FROM jan_users j
  JOIN latest_user_activity l ON j.user_id = l.user_id AND j.account_id = l.account_id
  GROUP BY j.account_id
)
```
Each block calculates the share of users who had future activity, representing the retention rate.  

#### 4. Calculate Retention Ratio  

```sql
SELECT 
  COALESCE(r_dec.account_id, r_jan.account_id) AS account_id,
  COALESCE(r_jan.jan_retention_rate, 0) / COALESCE(r_dec.dec_retention_rate, 1) AS retention_ratio
FROM retention_dec r_dec
FULL OUTER JOIN retention_jan r_jan ON r_dec.account_id = r_jan.account_id;
```
- Combines results using a **FULL OUTER JOIN**

- Handles missing months using `COALESCE`:
  - If no **December** data → assume `dec_retention_rate = 1` (to avoid divide-by-zero)
  - If no **January** data → assume `jan_retention_rate = 0

### Sample Output

| account_id | retention_ratio |
|------------|-----------------|
| A1        | 1.00          |
| A2        | 1.00            |
| A3        | 0.00            |

### Final Result

The final output includes each `account_id` and the calculated **retention ratio** between **January** and **December**.


### Conclusion

This query provides a clear view into **user retention trends** across accounts by calculating the ratio of January to December retention rates.

It uses several key SQL techniques:
- **CTEs (Common Table Expressions)** to modularize logic and keep the query clean
- **`MAX()`** to identify each user's most recent activity
- **`DATE_TRUNC()`** to filter activity by month
- **`CASE WHEN`** logic to check for future activity (i.e., retention)
- **`COALESCE()`** to handle missing data and avoid divide-by-zero errors
- A **`FULL OUTER JOIN`** to ensure no account is left out—even if it's missing from one of the months

Together, these techniques help build a robust and scalable query that:
- Handles missing data.
- Enables cohort-based retention analysis.
- Identifies accounts with improving or declining retention over time.









