-- MARKETING A/B HYPOTHESES TEST.

![card](assets/Screenshot_6-7-2026_15385_localhost.jpeg)

### Table of Contents

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

- Calculate user total and split by group.
- Calculate conversion rates by group.
- Ads exposure. 
	- Not all users saw the same number of ads. 
	- Does seeing more ads make you more likely to convert? And if it does, is that causation or correlation?
- Which day of the week and time of day drives the most conversions?
- Test for statistical significance.

---
## Methodology

- Import raw data to SQL server for cleaning and analysis.
- Tailored analysis to answering business questions using key metrics (conversions, conversion rate).
- Connect Python to SQL (query) database to perform statistical test (chi-square, p-value) to understand difference between groups.
- Share recommendations based on insights from data to be used for strategic business decision.

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

```python

from scipy.stats import chi2_contingency

table = [[14423, 550154], [420, 23104]]
chi2, p, dof, expected = chi2_contingency(table)
print(chi2, p)

```

## Tools

| Tool | Purpose|
|---|---|
| SQL | Data cleaning and analysis |
| Python | Test statistical significance: chi-square, p-value |

---

## Insights

- A chi-square value of 54 (p < 0.005) confirms a statistically significant difference between the test groups.

- The 'PSA' variant yielded a 1.79% conversion rate (420 conversions from 23,524 users), whereas the 'Ad' variant achieved 2.55% (14,423 conversions from 564,577 users).

- Within the 'Ad' group, performance scales notably with audience reach: ads seen by 51 to 100 users have an 11.63% conversion rate, which jumps to 17.14% for those seen by over 100 users.

- Regarding timing, Monday and Tuesday drive the highest volume of conversions, with only a slight difference between them.

- Most conversions occur between 1pm and 4pm, the peak conversion rates are observed at 4pm and 8pm. Notably, 8 pm stands out with the highest rate of all, despite having a lower number of conversions compared to the afternoon rush.

---

## Recommendations

- Shift majority of the budget from the 'psa' variant to the 'ad' variant. Given the statistical significance (p<0.005) and the higher overall conversion rate, continuing to spend heavily on 'psa' is inefficient.

- Consolidate ad spend into fewer, high-traffic inventory slots to ensure individual ads are served to at least 51 users, with a clear KPI to push for 100+ users per placement. Avoid spreading the budget across hundreds of low-traffic pages, as this dilutes reach and drastically lowers the conversion rate.

- Split Monday/Tuesday strategy into two distinct campaigns by
   - Scheduling heavy budgets between 1 pm and 4 pm to capture the highest absolute number of conversions.
   - Schedule a separate, targeted budget specifically for the 8 pm slot. Despite lower traffic, this slot delivers the highest conversion rate, this can be used to test premium creatives or retargeting strategies, as the audience active at this hour is clearly highly engaged.

- Observe why 8 pm performs so well. Is it the creative, the user's free time, or the device type? Copy the successful elements from the 8 pm slot and test them during the 1 to 4pm window to see if it can artificially boost the afternoon rates.
