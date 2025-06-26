# Kiva Microloan Analysis: SQL Queries 

This document provides SQL queries and their conceptual results, demonstrating analysis performed solely on the kiva_loans_utf8 table. These are suitable for documentation and GitHub.


1. Describe the process of ingesting raw Kiva loan data into MS SQL Server and outlining its initial schema
   
SELECT
    COLUMN_NAME,
    DATA_TYPE,
    CHARACTER_MAXIMUM_LENGTH,
    IS_NULLABLE
FROM
    INFORMATION_SCHEMA.COLUMNS
WHERE
    TABLE_NAME = 'kiva_loans_utf8'
ORDER BY
    ORDINAL_POSITION;


2. How did you perform initial data exploration in SQL to understand the distribution and basic structure of the Kiva dataset?
   
--Top 100 rows to see data structure
SELECT TOP 100 *
FROM kiva_loans_utf8;

-- Count of records and distinct values for key columns
SELECT
    COUNT(*) AS TotalRecords,
    COUNT(DISTINCT country) AS DistinctCountries,
    COUNT(DISTINCT sector) AS DistinctSectors
FROM kiva_loans_utf8;


3. Explain how you handled and formatted DATE and DATETIME columns directly within SQL Server.
   
SELECT
    posted_time,
    CAST(posted_time AS DATE) AS PostedDateOnly,
    YEAR(posted_time) AS PostedYear,
    MONTH(posted_time) AS PostedMonth,
    FORMAT(posted_time, 'yyyyMM') AS PostedYearMonth
FROM kiva_loans_utf8
WHERE posted_time IS NOT NULL
LIMIT 5; -- Using LIMIT for sample, use TOP in SQL Server


4. What SQL queries did you use to identify and manage missing values in crucial Kiva columns?
   
-- Identify missing values:
SELECT COUNT(*) AS MissingFundedAmount
FROM kiva_loans_utf8
WHERE funded_amount IS NULL;

SELECT COUNT(*) AS MissingBorrowerGenders
FROM kiva_loans_utf8
WHERE borrower_genders IS NULL OR borrower_genders = '';


5. How did you detect and de-duplicate records within the Kiva dataset?
   
WITH CTE_LoanDuplicates AS (
    SELECT
        *,
        ROW_NUMBER() OVER(PARTITION BY id ORDER BY posted_time ASC) as rn -- Assuming 'id' is the unique loan identifier
    FROM
        kiva_loans_utf8
)
SELECT *
FROM CTE_LoanDuplicates
WHERE rn > 1; -- Shows duplicate rows based on 'id'


6. Provide examples of data type conversions you performed in SQL.
   
SELECT
    term_in_months,
    CAST(term_in_months AS INT) AS TermAsInteger
FROM kiva_loans_utf8
WHERE term_in_months IS NOT NULL
LIMIT 5;


7. Describe how you might standardize text fields (e.g., country names) or apply basic normalization using SQL.

-- UPDATE kiva_loans_utf8
-- SET sector = UPPER(sector);

-- Example: Replace common misspellings/variations for a specific country
-- UPDATE kiva_loans_utf8
-- SET country = 'Congo (Dem. Rep.)'
-- WHERE country IN ('DR Congo', 'Democratic Republic of the Congo');

-- Query to see current distinct values before/after standardization
SELECT DISTINCT country FROM kiva_loans_utf8 ORDER BY country;


8. Write a SQL query to calculate the total number of loans in the dataset.
   
SELECT COUNT(id) AS TotalLoans
FROM kiva_loans_utf8;


9. Calculate the total loan_amount and funded_amount across all Kiva loans.
    
SELECT
    SUM(loan_amount) AS TotalLoanAmount,
    SUM(funded_amount) AS TotalFundedAmount
FROM kiva_loans_utf8;



10. Determine the overall percentage of fully funded loans.
    
SELECT
    CAST(SUM(CASE WHEN loan_amount = funded_amount THEN 1 ELSE 0 END) AS DECIMAL(10, 2)) * 100.0 / COUNT(*) AS FullyFundedPercentage
FROM kiva_loans_utf8
WHERE loan_amount IS NOT NULL AND funded_amount IS NOT NULL;



11. Calculate the total unfunded amount (loan_amount - funded_amount).
    
SELECT
    SUM(loan_amount - funded_amount) AS TotalUnfundedAmount
FROM kiva_loans_utf8
WHERE loan_amount > funded_amount; -- Only for partially or unfunded loans (if any exist)


12. Find the average time_to_fund_days for all loans.
    
SELECT
    AVG(CAST(DATEDIFF(day, posted_time, funded_time) AS DECIMAL(10, 2))) AS AverageTimeToFundDays
FROM kiva_loans_utf8
WHERE posted_time IS NOT NULL AND funded_time IS NOT NULL;


13. Identify the top 10 countrys by the highest number of Kiva loans.

SELECT TOP 10
    country,
    COUNT(id) AS NumberOfLoans
FROM kiva_loans_utf8
WHERE country IS NOT NULL AND country <> ''
GROUP BY country
ORDER BY NumberOfLoans DESC;



14. Determine which countrys receive the largest total loan_amount.

SELECT TOP 10
    country,
    SUM(loan_amount) AS TotalLoanAmountByCountry
FROM kiva_loans_utf8
WHERE country IS NOT NULL AND country <> '' AND loan_amount IS NOT NULL
GROUP BY country
ORDER BY TotalLoanAmountByCountry DESC;


15. How would you use SQL to compare funded_amount versus loan_amount by country?

SELECT
    country,
    SUM(loan_amount) AS TotalLoanAmount,
    SUM(funded_amount) AS TotalFundedAmount,
    (SUM(funded_amount) * 100.0 / SUM(loan_amount)) AS FundingPercentage
FROM kiva_loans_utf8
WHERE country IS NOT NULL AND country <> '' AND loan_amount IS NOT NULL AND loan_amount > 0
GROUP BY country
ORDER BY FundingPercentage DESC;


16. Identify the top 10 sectors by total loan_amount and lender_count.

SELECT TOP 10
    sector,
    SUM(loan_amount) AS TotalLoanAmountBySector,
    SUM(lender_count) AS TotalLenderCountBySector
FROM kiva_loans_utf8
WHERE sector IS NOT NULL AND sector <> ''
GROUP BY sector
ORDER BY TotalLoanAmountBySector DESC, TotalLenderCountBySector DESC;


17. Calculate the average loan_amount across different activity types.

SELECT
    activity,
    AVG(loan_amount) AS AverageLoanAmountByActivity
FROM kiva_loans_utf8
WHERE activity IS NOT NULL AND activity <> '' AND loan_amount IS NOT NULL
GROUP BY activity
ORDER BY AverageLoanAmountByActivity DESC;


18. Determine which sectors have the highest and lowest funding success rates.

SELECT
    sector,
    CAST(SUM(CASE WHEN loan_amount = funded_amount THEN 1 ELSE 0 END) AS DECIMAL(10, 2)) * 100.0 / COUNT(*) AS SectorFundingSuccessPercentage
FROM kiva_loans_utf8
WHERE sector IS NOT NULL AND sector <> '' AND loan_amount IS NOT NULL AND funded_amount IS NOT NULL
GROUP BY sector
HAVING COUNT(*) > 50 -- Only include sectors with a reasonable number of loans
ORDER BY SectorFundingSuccessPercentage DESC;


19. Calculate the average term_in_months per sector.

SELECT
    sector,
    AVG(term_in_months) AS AverageTermInMonthsBySector
FROM kiva_loans_utf8
WHERE sector IS NOT NULL AND sector <> '' AND term_in_months IS NOT NULL
GROUP BY sector
ORDER BY AverageTermInMonthsBySector DESC;


20. Find the distribution of borrower_genders in the Kiva dataset.

SELECT
    borrower_genders,
    COUNT(*) AS NumberOfLoans,
    CAST(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER() AS DECIMAL(5,2)) AS Percentage
FROM kiva_loans_utf8
WHERE borrower_genders IS NOT NULL AND borrower_genders <> ''
GROUP BY borrower_genders
ORDER BY NumberOfLoans DESC;


21. How do loan_amounts differ for female borrowers compared to other gender groups, analyzed in SQL?

SELECT
    borrower_genders,
    AVG(loan_amount) AS AverageLoanAmount
FROM kiva_loans_utf8
WHERE borrower_genders IS NOT NULL AND borrower_genders <> '' AND loan_amount IS NOT NULL
GROUP BY borrower_genders
ORDER BY AverageLoanAmount DESC;


22. Identify the partner_ids that facilitate the most loans or largest loan_amount.

SELECT TOP 10
    partner_id,
    COUNT(id) AS NumberOfLoans,
    SUM(loan_amount) AS TotalLoanAmount
FROM kiva_loans_utf8
WHERE partner_id IS NOT NULL
GROUP BY partner_id
ORDER BY NumberOfLoans DESC, TotalLoanAmount DESC;


23. Calculate the average lender_count for each loan_amount_bucket.

SELECT
    CASE
        WHEN loan_amount < 500 THEN 'Under $500'
        WHEN loan_amount >= 500 AND loan_amount < 1000 THEN '$500 - $999'
        WHEN loan_amount >= 1000 AND loan_amount < 5000 THEN '$1000 - $4999'
        ELSE '$5000+'
    END AS LoanAmountBucket,
    AVG(lender_count) AS AverageLenderCount
FROM kiva_loans_utf8
WHERE loan_amount IS NOT NULL AND lender_count IS NOT NULL
GROUP BY
    CASE
        WHEN loan_amount < 500 THEN 'Under $500'
        WHEN loan_amount >= 500 AND loan_amount < 1000 THEN '$500 - $999'
        WHEN loan_amount >= 1000 AND loan_amount < 5000 THEN '$1000 - $4999'
        ELSE '$5000+'
    END
ORDER BY MIN(loan_amount); -- Order by min amount in bucket for logical sorting


24. What SQL query would you use to find the overall proportion of each repayment_interval type?

SELECT
    repayment_interval,
    COUNT(*) AS NumberOfLoans,
    CAST(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER() AS DECIMAL(5,2)) AS Percentage
FROM kiva_loans_utf8
WHERE repayment_interval IS NOT NULL AND repayment_interval <> ''
GROUP BY repayment_interval
ORDER BY NumberOfLoans DESC;


25. Describe how you might use a Common Table Expression (CTE) or subquery in SQL to analyze loans from the top 5 lending countries.

WITH Top5Countries AS (
    SELECT TOP 5
        country
    FROM kiva_loans_utf8
    WHERE country IS NOT NULL AND country <> ''
    GROUP BY country
    ORDER BY COUNT(id) DESC
)
SELECT
    kl.country,
    AVG(kl.loan_amount) AS AverageLoanAmountInTop5,
    COUNT(kl.id) AS NumberOfLoansInTop5
FROM
    kiva_loans_utf8 kl
INNER JOIN
    Top5Countries t5 ON kl.country = t5.country
GROUP BY kl.country
ORDER BY NumberOfLoansInTop5 DESC;


26. How did you use SQL to analyze temporal trends in loan volume (e.g., posted_time or funded_time) month-over-month or year-over-year?

SELECT
    FORMAT(posted_time, 'yyyy-MM') AS PostedMonthYear,
    COUNT(id) AS NumberOfLoans,
    SUM(loan_amount) AS TotalLoanAmount
FROM kiva_loans_utf8
WHERE posted_time IS NOT NULL
GROUP BY FORMAT(posted_time, 'yyyy-MM')
ORDER BY PostedMonthYear;



27. Calculate the average difference between funded_time and disbursed_time in days for Kiva loans.

SELECT
    AVG(CAST(DATEDIFF(day, funded_time, disbursed_time) AS DECIMAL(10,2))) AS AverageFundingToDisbursalDays
FROM kiva_loans_utf8
WHERE funded_time IS NOT NULL AND disbursed_time IS NOT NULL
AND DATEDIFF(day, funded_time, disbursed_time) >= 0; -- Ensure positive difference


28. What SQL queries did you employ to derive insights from the tags column, if raw tags required parsing?

SELECT
    id,
    loan_amount,
    country,
    tags
FROM kiva_loans_utf8
WHERE tags LIKE '%#Eco-friendly%'; -- Using LIKE for partial match

-- This query counts loans based on the presence of a few common tags.
SELECT
    CASE
        WHEN tags LIKE '%#Woman%' THEN 'Woman Entrepreneur'
        WHEN tags LIKE '%#Parent%' THEN 'Parent'
        WHEN tags LIKE '%#Sustainable%' THEN 'Sustainable'
        WHEN tags LIKE '%#First Loan%' THEN 'First Loan'
        ELSE 'Other/Multiple Tags'
    END AS TagCategory,
    COUNT(id) AS NumberOfLoans
FROM kiva_loans_utf8
WHERE tags IS NOT NULL AND tags <> ''
GROUP BY
    CASE
        WHEN tags LIKE '%#Woman%' THEN 'Woman Entrepreneur'
        WHEN tags LIKE '%#Parent%' THEN 'Parent'
        WHEN tags LIKE '%#Sustainable%' THEN 'Sustainable'
        WHEN tags LIKE '%#First Loan%' THEN 'First Loan'
        ELSE 'Other/Multiple Tags'
    END
ORDER BY NumberOfLoans DESC;


