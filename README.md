-- MARKETING A/B HYPOTHESES TEST.

### Table of Contents

- [Overview](#overview)
- [Objective](#objective)
- [Questions](#questions)
- [Methodology](#methodology)
- [Tools](#tools)
- [Insights](#insights)
- [Recommendations](#recommendations)


## Objective

The idea of the dataset is to analyze the groups, find if the ads were successful, how much the company can make from the ads, and if the difference between the groups is statistically significant. [Data](https://github.com/alabiibrahim/Marketing_AB_Testing/blob/main/assets/dataset/marketing_AB.csv)

---

## Questions

1. Calculates total and split by group.
2. Calculates conversion rates by group.
3. Ads exposure. 
- Not all users saw the same number of ads. 
- Does seeing more ads make you more likely to convert? And if it does, is that causation or correlation?
4. Which day of the week and time of day drives the most conversions?
5. Test for statistical significance.

---
## Methodology
1. Import raw data to SQL server for cleaning and analysis.
2. Tailored analysis to answering business questions using key metrics.
3. Connect Python to SQL (query) database to perform statistical test (chi-square, p-value) to understand difference between groups.
4. Share recommendations based on insights from data to be used for strategic business decision.

```sql

SELECT COUNT(*) FROM [dbo].[marketing_AB] AS ROW_COUNT; -- dataset row check. 

SELECT [test_group], 
	COUNT(*) AS row_count,
	ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER(), 2) AS split_pct
  FROM [dbo].[marketing_AB]
  GROUP BY [test_group]             -- group split (1) 'ad' 564577 & split_pct of 96%. (2) 'psa' - 23524 & split_pct of 4%.


/* bq2: Calculates conversion rates by test_group.

The core question: do people who see ads convert at a higher rate? 
Calculate the rate for each group and the relative lift between them.       */

SELECT 
    test_group AS test_gr,
    COUNT(*) AS row_ct,
    SUM(CASE WHEN converted = 1 THEN 1 ELSE 0 END) AS convert_users,
    ROUND(
        SUM(CASE WHEN converted = 1 THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 
        2
    ) AS convert_pct
FROM [dbo].[marketing_AB]
GROUP BY test_group;

/*
-- bq3: Ads exposure. 
Does seeing more ads make you more likely to convert? And if it does, is that causation or correlation?    */

WITH Aggregated AS (
    SELECT 
        test_group,
        AVG(total_ads) AS ads_avg_total,
        MAX(total_ads) AS max_ads,
        MIN(total_ads) AS min_ads
    FROM [dbo].[marketing_AB]
    GROUP BY test_group
),
Median AS (
    SELECT DISTINCT
        test_group,
        PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY total_ads) 
            OVER (PARTITION BY test_group) AS med_ads
    FROM [dbo].[marketing_AB]
)
SELECT 
    a.test_group,
    a.ads_avg_total,
    a.max_ads,
    a.min_ads,
    m.med_ads
FROM Aggregated a
INNER JOIN Median m ON a.test_group = m.test_group;
 

WITH ad_seen  AS ( 
    SELECT 
        CASE 
            WHEN [total_ads] BETWEEN 1 AND 20 THEN '1 to 20'
            WHEN [total_ads] BETWEEN 21 AND 50 THEN '21 to 50'
            WHEN [total_ads] BETWEEN 51 AND 100 THEN '51 to 100'
            ELSE '100+' END AS 'Ads_seen',
            converted
    FROM [dbo].[marketing_AB] 
    WHERE [test_group] = 'ad' 
    )   SELECT Ads_seen,
            COUNT(*) AS 'Users',
            ROUND(SUM(CASE WHEN [converted] = 1 THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 2) AS 'conv_rate_pct'
        FROM ad_seen
        GROUP BY Ads_seen;

 bq 4:   

-- Which day of week drives the most conversion?

    SELECT [most_ads_day],
        COUNT(*) AS counts,
        SUM(CASE WHEN [converted] = 1 THEN 1 ELSE 0 END) AS conversions,
        ROUND(SUM(CASE WHEN [converted] = 1 THEN 1 ELSE 0 END), 2) * 100.0 / COUNT(*) AS weekday_conv_rate
    FROM [dbo].[marketing_AB] 
    WHERE [test_group] = 'ad'
    GROUP BY [most_ads_day]
    ORDER BY conversions DESC;

-- Which hour drives the most conversion?

    SELECT [most_ads_hour],
        COUNT(*) AS counts,
        SUM(CASE WHEN [converted] = 1 THEN 1 ELSE 0 END) AS conversions,
        ROUND(SUM(CASE WHEN [converted] = 1 THEN 1 ELSE 0 END), 2) * 100.0 / COUNT(*) AS Ads_hr_conv_rate
    FROM [dbo].[marketing_AB] 
    WHERE [test_group] = 'ad'
    GROUP BY [most_ads_hour]
    ORDER BY conversions DESC;
  
-- calculate average ads seen by test group.
    SELECT 
    test_group,
    COUNT(*) AS total_users,
    SUM(CASE WHEN converted = 1 THEN 1 ELSE 0 END) AS conversions,
    ROUND(
        SUM(CASE WHEN converted = 1 THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 
        2
    ) AS conversion_rate_pct,
    ROUND(AVG(total_ads), 1) AS avg_ads_seen
FROM [dbo].[marketing_AB] 
GROUP BY test_group;

```

**Testing Hypotheses**

![card](assets/Screenshot_6-7-2026_15378_localhost.jpeg)

![card](assets/Screenshot_6-7-2026_153731_localhost.jpeg)

![card](assets/Screenshot_6-7-2026_15385_localhost.jpeg)

## Tools

| Tool | Purpose|
|---|---|
| SQL | Data cleaning and analysis |
| Python | Test statistical significance: chi-square, p-value |

---

## Insights

- Chi-square = 54 and p-value < 0.005 shows difference in statistical significance.
- 'psa' have a conversion rate of 1.79%, 23,524 users and 420 conversion.
- 'ad' have a conversion rate of 2.55%, 564,577 users and 14,423 conversions.
- ad seen by 100+ users have conversion rate of 17.14%.
- ad seen between 51-100 users have a conversion rate of 11.63%.
- Monday & Tuesday drives the most conversion with a slight difference between them.
- Most conversion happens between 1 to 4pm. Also,
- 4pm and 8pm stands out with the highest conversion_rate, especially 8pm having a lower conversion but higher conversion rate.

---

## Recommendations

- Running of Ads should target a minimum of 50 people and above to increase conversion rate.
- Running of ads should be timed, and also target the first two workday of the week between lunch time and closing hour.
- 
