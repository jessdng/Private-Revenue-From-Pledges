# 📈 Private Revenue From Pledges Report End-to-End Project

## Objective

In this project, I designed and implemented an end-to-end data pipeline that consists of several stages:
1. Extracted data from a CSV file and loaded it into an Oracle database using SQL*Loader for further processing.
3. Transformed and modeled the data using fact and dimensional data modeling concepts using SQL.
4. Using ELT concept, I orchestrated the data pipeline and automated the extraction, transformation, and loading of the data using Oracle DBMS Scheduler into the final tables in the data warehouse.
5. Developed a dashboard on PowerBI.

The sections below will explain additional details on the technologies and files utilized.

## Table of Content

- [Dataset Used](#dataset-used)
- [Technologies](technologies)
- [Date Modeling](#data-modeling)
- [Step 1: Cleaning and Transformation](#step-1-cleaning-and-transformation)
- [Step 2: ELT Data Pipeline Automation](#step-2-elt-data-pipeline-automation) 
- [Step 3: Analytics](#step-3-analytics)
- [Step 4: Dashboard](#step-4-dashboard)

## Dataset Used

This project is for portfolio demonstration only and is based on a real-world project I previously worked on. The data has been modified to showcase the ELT and data modeling process without revealing any sensitive or real-world information. All connection data is intentionally left generic for demonstration purpsoes. The dataset includes data on scheduled payments to pledge donations made towards a fictional educational institution. The dataset used for the following demonstrated can be seen [here](https://github.com/jessdng/Private-Revenue-From-Pledges/blob/main/Project%20Files/Private%20Revenue%20From%20Pledges.csv). 

## Technologies

The following technologies are used to build this project:
- Language: SQL
- Extraction and transformation: Oracle data warehouse, SQL*Loader
- Storage: Oracle data warehouse
- Dashboard: PowerBI

## Data Modeling

The datasets are designed using the principles of fact and dim data modeling concepts. 

![Data Model](https://raw.githubusercontent.com/jessdng/Private-Revenue-From-Pledges/refs/heads/main/Screenshots/PPRF%20-%20Data%20Model.png)

## Step 1: Cleaning and Transformation

In this step, I first created a staging table. This allows for me to clean and transform the data before inserting it into final production tables. 

````sql
CREATE TABLE pledge_staging (
  pledge_id NUMBER,
  donor_id NUMBER,
  donor_type VARCHAR2(20),
  donor_faculty VARCHAR2(20),
  date_pledged DATE DEFAULT SYSDATE,
  total_pledge_amount NUMBER,
  pledge_allocation_id NUMBER,
  allocation_faculty VARCHAR2(50),
  allocation_purpose VARCHAR2(50),
  scheduled_payment_date NUMBER,
  scheduled_payment_amount NUMBER,
  scheduled_payment_status VARCHAR2(1),
  date_modified TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
````
Because we are working inside the Oracle database, I opted to use SQL*Loader instead of Python (cx_Oracle). If I was working with multiple CSV files, or wanted to pre-process the data before loading (ETL vs. ELT), I would opt for Python instead. 

Creating the control file for `sqlldr`:
````sql
LOAD DATA 
INFILE 'scheduled_pledge_payments.csv' 
INTO TABLE pledge_staging 
FIELDS TERMINATED BY ','
(pledge_id, donor_id, donor_type, donor_faculty, date_pledged DATE "YYYY-MM-DD", total_pledge_amount, pledge_allocation_code, allocation_faculty, allocation_purpose, scheduled_payment_date, scheduled_payment_amount, scheduled_payment_status, date_modified)
````
Running the `sqlldr` command:
````sql
sqlldr userid=username/password control=scheduled_pledge_payments.ctl log=scheduled_pledge_payments.log
````

Example of the script I would run if I were to use Python:

```Python

import cx_Oracle

# Connect to Oracle
conn = cx_Oracle.connect("username/password@localhost:1521/orcl") 
cursor = conn.cursor()

# Load CSV into Pandas DataFrame
df = pd.read_csv("pledge_payments_schedule.csv")

# Insert each row into the staging table
for _, row in df.iterrows():
    cursor.execute("""
        INSERT INTO pledge_staging (pledge_id, donor_id, donor_type, donor_faculty, date_pledged, total_pledge_amount, pledge_allocation_code, allocation_faculty, allocation_purpose, scheduled_payment_date, scheduled_payment_amount, scheduled_payment_status)
        VALUES (:1, :2, :3, :4, TO_DATE(:5, 'YYYY-MM-DD'), :6, :7, :8, :9, :10, :11, :12)
    """, (row["pledge_id"], row["donor_id"], row["donor_type"], row["donor_faculty"], row["date_pledged"], row["total_pledge_amount"], row["pledge_allocation_code"], row["allocation_faculty"], row["allocation_purpose"], row["scheduled_payment_date"], row["scheduled_payment_amount"], row["scheduled_payment_status"], row["date_modified"]))

conn.commit()
print("Data loaded into staging table.")

```

The following cleaning and transformation tasks are then performed using SQL:
- Identifying duplicates and keeping only the most recent record added, `(pledge_id, scheduled_payment_date)` should be a unique identifier for every row.
- Removing NULL values from `donor_faculty`. Some donors are not alumni, and will therefore not have a value in `donor_faculty`. NULL values are replaced with an 'Unknown' string value. 
- Converted `scheduled_payment_date` into date format.

```sql
CREATE OR REPLACE PROCEDURE clean_pledge_staging AS BEGIN

-- Identify and delete duplicates, keeping only the most recent record
DELETE FROM pledge_staging
WHERE ROWID IN (
    SELECT ROWID
    FROM (
        SELECT ROWID, ROW_NUMBER() OVER (PARTITION BY pledge_id, scheduled_payment_date ORDER BY date_modified DESC) AS rn
        FROM pledge_staging
    )
    WHERE rn > 1);

-- Handle missing values by replacing NULLs
UPDATE pledge_staging
SET donor_faculty = 'Unknown'
WHERE donor_faculty IS NULL;

-- Convert scheduled_payment_date column to standard format
UPDATE pledge_staging
SET scheduled_payment_date = TO_DATE(scheduled_payment_date, 'YYYYMMDD');

COMMIT;
END clean_pledge_staging;
/

```

After completing the above steps, I created the following fact and dimension tables below:

````sql
-- Allocation Table 
CREATE TABLE allocation ( 
  allocation_id VARCHAR2(10) PRIMARY KEY,
  allocation_purpose VARCHAR2(50),
  allocation_faculty VARCHAR2(50),
  date_added TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
````
| allocation_id | allocation_purpose | allocation_faculty | date_added | 
| ----------- | ----------- | ----------- | ----------- | 
| - | - | - | - |

````sql
-- Donor Table
CREATE TABLE donor ( 
  donor_id NUMBER PRIMARY KEY,
  donor_type VARCHAR(50),
  donor_faculty VARCHAR(50)
  date_added TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
````
| donor_id | donor_type | donor_faculty | date_added |
| ----------- | ----------- | ----------- | ----------- |
| - | - | - | - | 

````sql
-- Pledge Data Table 
CREATE TABLE pledge_data ( 
  pledge_id NUMBER PRIMARY KEY,
  donor_id NUMBER REFERENCES Donor(donor_id),
  date_pledged DATE DEFAULT SYSDATE,
  total_pledge_amount NUMBER(9,2),
  pledge_allocation_id VARCHAR2(20) REFERENCES Allocation(allocation_code),
  date_added TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
````
| pledge_id | donor_id | date_pledged | total_pledge_amount | pledge_allocation_id | date_added |
| ----------- | ----------- | ----------- | ----------- | ----------- | ----------- |
| - | - | - | - | - | - |

````sql
-- Scheduled Payments Table 
CREATE TABLE scheduled_payments ( 
  pledge_id NUMBER REFERENCES Pledges(pledge_num),
  scheduled_payment_date DATE DEFAULT SYSDATE,
  payment_status VARCHAR2(1),
  payment_amount NUMBER(9,2),
  pledge_allocation_id VARCHAR2(20),
  date_modified TIMESTAMP DEFAULT CURRENT_TIMESTAMP
  PRIMARY KEY (pledge_num, scheduled_payment_date)
);
````
| pledge_id | scheduled_payment_date | payment_status | payment_amount | pledge_allocation_id | date_modified |
| ----------- | ----------- | ----------- | ----------- | ----------- | ----------- | 
| - | - | - | - | - | - |

The data can now be moved from the staging table into the dimension and fact tables. 

## Step 2: ELT Data Pipeline Automation

The **PL/SQL procedure** created earlier is automated using **Oracle DBMS Scheduler** to run every Sunday at 3 AM.

````sql
BEGIN
  DBMS_SCHEDULER.create_job (
    job_name        => 'DATA_CLEANUP_JOB',
    job_type        => 'PLSQL_BLOCK',
    job_action      => 'BEGIN clean_pledge_staging(); END;',
    start_date      => SYSTIMESTAMP,
    repeat_interval => 'FREQ=WEEKLY; BYDAY=SUN; BYHOUR=3;',
    enabled         => TRUE
  );
END;
/
````

## Step 3: Analytics

Here's the additional analyses I performed:
1. What is the total amount for each pledge that is paid to date?
 ````sql
SELECT
  pledge_id,
  SUM(payment_amount) amount_paid_to_date
FROM scheduled_payments
WHERE payment_status = 'P'
GROUP BY pledge_id
````  

2. What is the total remaining balance for each pledge?
 ````sql
SELECT
  ps.pledge_id,
  (pd.pledge_amount - SUM(ps.payment_amount)) remaining_balance
FROM scheduled_payments ps
JOIN pledge_data pd ON ps.pledge_num = pd.pledge_num
WHERE ps.payment_status = 'P'
GROUP BY ps.pledge_num
````

3. How long does it take for each donor to pay off their pledges?
 ````sql
WITH payoff_date AS (
  SELECT pledge_id, MAX(scheduled_payment_date) payoff_date
  FROM scheduled_payments
  GROUP BY pledge_id
)

SELECT DISTINCT 
    pod.pledge_id,
    pd.total_pledge_amount, 
    pd.date_pledged, 
    pod.payoff_date, 
    (pod.payoff_date - pd.date_pledged) days_to_payoff,
    ROUND(MONTHS_BETWEEN(pod.payoff_date - pd.date_pledged),2) months_to_payoff
FROM payoff_date pod
JOIN pledge_data pd ON pod.pledge_id = pd.pledge_id 
ORDER BY pod.pledge_id ASC
;
````

4. How long on average does it take for donors to pay off their pledges based on the pledge amount?
 ````sql
WITH payoff_date AS (
  SELECT pledge_id, MAX(scheduled_payment_date) payoff_date
  FROM scheduled_payments
  GROUP BY pledge_id
)

SELECT  
  CASE WHEN pd.total_pledge_amount < 1000 THEN '< 1,000' 
       WHEN pd.total_pledge_amount >= 1000 AND pd.total_pledge_amount < 10000 THEN '1,000 to 10,000' 
       WHEN pd.total_pledge_amount >= 10000 AND pd.total_pledge_amount < 50000 THEN '10,000 to 50,000' 
       WHEN pd.total_pledge_amount >= 50000 AND pd.total_pledge_amount < 100000 THEN '50,000 to 100,000'
       WHEN pd.total_pledge_amount >= 100000 AND pd.total_pledge_amount < 250000 THEN '100,000 to 250,000'
       WHEN pd.total_pledge_amount >= 250000 AND pd.total_pledge_amount < 500000 THEN '250,000 to 500,000'
       WHEN pd.total_pledge_amount >= 500000 AND pd.total_pledge_amount < 1000000 THEN '500,000 to 1,000,000'
       WHEN pd.total_pledge_amount >= 1000000 AND pd.total_pledge_amount < 5000000 THEN '1,000,000 to 5,000,000'
       WHEN pd.total_pledge_amount >= 5000000 AND pd.total_pledge_amount < 10000000 THEN '5,000,000 to 10,000,000'
       WHEN pd.total_pledge_amount >= 10000000 AND pd.total_pledge_amount < 20000000 THEN '10,000,000 to 20,000,000'
  END amount_grouping
  ROUND(AVG(pod.payoff_date - pd.date_pledged),2) avg_days_to_payoff,
  ROUND(AVG(MONTHS_BETWEEN(pod.payoff_date - pd.date_pledged)),2) avg_months_to_payoff
FROM payoff_date pod
JOIN pledge_data pd ON pod.pledge_id = pd.pledge_id
GROUP BY
  CASE WHEN pd.total_pledge_amount < 1000 THEN '< 1,000' 
       WHEN pd.total_pledge_amount >= 1000 AND pd.total_pledge_amount < 10000 THEN '1,000 to 10,000' 
       WHEN pd.total_pledge_amount >= 10000 AND pd.total_pledge_amount < 50000 THEN '10,000 to 50,000' 
       WHEN pd.total_pledge_amount >= 50000 AND pd.total_pledge_amount < 100000 THEN '50,000 to 100,000'
       WHEN pd.total_pledge_amount >= 100000 AND pd.total_pledge_amount < 250000 THEN '100,000 to 250,000'
       WHEN pd.total_pledge_amount >= 250000 AND pd.total_pledge_amount < 500000 THEN '250,000 to 500,000'
       WHEN pd.total_pledge_amount >= 500000 AND pd.total_pledge_amount < 1000000 THEN '500,000 to 1,000,000'
       WHEN pd.total_pledge_amount >= 1000000 AND pd.total_pledge_amount < 5000000 THEN '1,000,000 to 5,000,000'
       WHEN pd.total_pledge_amount >= 5000000 AND pd.total_pledge_amount < 10000000 THEN '5,000,000 to 10,000,000'
       WHEN pd.total_pledge_amount >= 10000000 AND pd.total_pledge_amount < 20000000 THEN '10,000,000 to 20,000,000'
  END 
ORDER BY amount_grouping ASC
;
````  

## Step 4: Dashboard

After completing the analysis, I loaded the relevant tables into PowerBI and created a dashboard.

![Dashboard Pg 1](https://raw.githubusercontent.com/jessdng/Private-Revenue-From-Pledges/refs/heads/main/Screenshots/PPRF%20-%20Overview.png)

![Dashboard Pg 2](https://raw.githubusercontent.com/jessdng/Private-Revenue-From-Pledges/refs/heads/main/Screenshots/PPRF%20-%20FY%20Breakdown.png)

![Dashboard Pg 3](https://raw.githubusercontent.com/jessdng/Private-Revenue-From-Pledges/refs/heads/main/Screenshots/PPRF%20-%20Tables.png)

![Dashboard Pg 4](https://raw.githubusercontent.com/jessdng/Private-Revenue-From-Pledges/refs/heads/main/Screenshots/PPRF%20-%20Filter%20Pane.png)

***
