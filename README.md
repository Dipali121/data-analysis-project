**write a query to print top 5 cities with highest spends and their percentage contribution of total credit card spends**

WITH CitySpends AS (
    SELECT 
        city,
        SUM(amount) AS total_spend
    FROM 
        credit_card_transcations$
    GROUP BY 
        city
),
TotalSpend AS (
    SELECT 
        SUM(amount) AS total_credit_card_spend
    FROM 
        credit_card_transcations$
)
SELECT 
    cs.city,
    cs.total_spend,
    ROUND((cs.total_spend / ts.total_credit_card_spend) * 100, 2) AS percentage_contribution
FROM 
    CitySpends cs
CROSS JOIN 
    TotalSpend ts
ORDER BY 
    cs.total_spend DESC
OFFSET 0 ROWS FETCH NEXT 5 ROWS ONLY;


**write a query to print highest spend month and amount spent in that month for each card type**

WITH MonthlySpends AS (
    SELECT 
        card_type,
        YEAR(transaction_date) AS year,
        MONTH(transaction_date) AS month,
        SUM(amount) AS total_spend
    FROM 
        credit_card_transcations$
    GROUP BY 
        card_type, YEAR(transaction_date), MONTH(transaction_date)
),
MaxMonthlySpends AS (
    SELECT 
        card_type,
        MAX(total_spend) AS highest_month_spend
    FROM 
        MonthlySpends
    GROUP BY 
        card_type
)
SELECT 
    ms.card_type,
    ms.year,
    ms.month,
    ms.total_spend AS highest_month_spend
FROM 
    MonthlySpends ms
JOIN 
    MaxMonthlySpends mms
    ON ms.card_type = mms.card_type
    AND ms.total_spend = mms.highest_month_spend
ORDER BY 
    ms.card_type, ms.year, ms.month;

**write a query to print the transaction details(all columns from the table) for each card type when it reaches a cumulative of 1000000 total spends(We should have 4 rows in the o/p one for each card type)**

WITH RunningTotal AS (
    SELECT 
        *,
        SUM(amount) OVER (PARTITION BY card_type ORDER BY transaction_date) AS cumulative_spend
    FROM 
        credit_card_transcations$
)
SELECT *
FROM RunningTotal
WHERE cumulative_spend >= 1000000
    AND cumulative_spend - amount < 1000000
ORDER BY card_type, transaction_date;

**write a query to find city which had lowest percentage spend for gold card type**

WITH CitySpend AS (
    SELECT 
        city,
        SUM(amount) AS city_total_spend
    FROM 
        credit_card_transcations$
    WHERE 
        card_type = 'Gold'
    GROUP BY 
        city
),
TotalGoldSpend AS (
    SELECT 
        SUM(amount) AS total_gold_spend
    FROM 
        credit_card_transcations$
    WHERE 
        card_type = 'Gold'
)
SELECT 
    cs.city,
    cs.city_total_spend,
    ROUND((cs.city_total_spend / ts.total_gold_spend) * 100, 2) AS percentage_spend
FROM 
    CitySpend cs
CROSS JOIN 
    TotalGoldSpend ts
ORDER BY 
    percentage_spend ASC;

**write a query to print 3 columns:  city, highest_expense_type , lowest_expense_type (example format : Delhi , bills, Fuel)**

WITH CityExpense AS (
    SELECT
        city,
        exp_type,
        SUM(amount) AS total_expense
    FROM
        credit_card_transcations$
    GROUP BY
        city, exp_type
),
RankedExpenses AS (
    SELECT 
        city,
        exp_type,
        total_expense,
        RANK() OVER (PARTITION BY city ORDER BY total_expense DESC) AS rank_high,
        RANK() OVER (PARTITION BY city ORDER BY total_expense ASC) AS rank_low
    FROM 
        CityExpense
)
SELECT 
    city,
    (SELECT exp_type FROM RankedExpenses WHERE city = re.city AND rank_high = 1) AS highest_expense_type,
    (SELECT exp_type FROM RankedExpenses WHERE city = re.city AND rank_low = 1) AS lowest_expense_type
FROM 
    RankedExpenses re
GROUP BY 
    city;

**write a query to find percentage contribution of spends by females for each expense type**

WITH TotalSpend as (
	SELECT
		exp_type,
		SUM(amount) as total_expense
	FROM
		credit_card_transcations$
	GROUP BY
		exp_type
),
FemaleSpend AS (
SELECT
exp_type,
SUM(amount) as female_expense
from
credit_card_transcations$
where
gender = 'F'
group by 
exp_type
)
SELECT
ts.exp_type,
ROUND ((fs.female_expense / ts.total_expense) * 100, 2) AS female_percentage_contribution
from TotalSpend ts
left join
FemaleSpend fs on ts.exp_type = fs.exp_type
order by
female_percentage_contribution desc;

**which card and expense type combination saw highest month over month growth in Jan-2014**

WITH MonthlySpend AS (
    SELECT
        card_type,
        exp_type,
        YEAR(transaction_date) AS year,
        MONTH(transaction_date) AS month,
        SUM(amount) AS total_spend
    FROM
        credit_card_transcations$
    WHERE
        (YEAR(transaction_date) = 2013 AND MONTH(transaction_date) = 12) OR 
        (YEAR(transaction_date) = 2014 AND MONTH(transaction_date) = 1)
    GROUP BY
        card_type, exp_type, YEAR(transaction_date), MONTH(transaction_date)
),
GrowthCalculation AS (
    SELECT
        m1.card_type,
        m1.exp_type,
        m1.total_spend AS jan_2014_spend,
        m2.total_spend AS dec_2013_spend,
        CASE
            WHEN m2.total_spend = 0 THEN NULL  -- Prevent division by zero
            ELSE (m1.total_spend - m2.total_spend) / m2.total_spend * 100
        END AS growth_percentage
    FROM
        MonthlySpend m1
    LEFT JOIN
        MonthlySpend m2 ON m1.card_type = m2.card_type
        AND m1.exp_type = m2.exp_type
        AND m2.year = 2013 AND m2.month = 12
    WHERE
        m1.year = 2014 AND m1.month = 1
)
SELECT TOP 1
    card_type,
    exp_type,
    growth_percentage
FROM
    GrowthCalculation
ORDER BY
    growth_percentage DESC;

**during weekends which city has highest total spend to total no of transcations ratio**

WITH WeekendSpend AS (
    SELECT
        city,
        SUM(amount) AS total_spend,
        COUNT(transaction_id) AS total_transactions
    FROM
        credit_card_transcations$
    WHERE
        DATEPART(weekday, transaction_date) IN (1, 7) -- 1 for Sunday, 7 for Saturday
    GROUP BY
        city
)
SELECT TOP 1
    city,
    total_spend,
    total_transactions,
    CAST(total_spend AS FLOAT) / total_transactions AS spend_to_transaction_ratio
FROM
    WeekendSpend
ORDER BY
    spend_to_transaction_ratio DESC;

**which city took least number of days to reach its 500th transaction after the first transaction in that city**

WITH CityTransactions AS (
    SELECT
        city,
        transaction_id,
        transaction_date,
        ROW_NUMBER() OVER (PARTITION BY city ORDER BY transaction_date) AS transaction_rank
    FROM
        credit_card_transcations$
),
FirstTransaction AS (
    SELECT
        city,
        MIN(transaction_date) AS first_transaction_date
    FROM
        credit_card_transcations$
    GROUP BY
        city
),
FifthHundredTransactionDate AS (
    SELECT
        ct.city,
        DATEDIFF(DAY, ft.first_transaction_date, ct.transaction_date) AS days_to_500th_transaction
    FROM
        CityTransactions ct
    JOIN
        FirstTransaction ft ON ct.city = ft.city
    WHERE
        ct.transaction_rank = 500
)
SELECT TOP 1
    city,
    days_to_500th_transaction
FROM
    FifthHundredTransactionDate
ORDER BY
    days_to_500th_transaction ASC;
