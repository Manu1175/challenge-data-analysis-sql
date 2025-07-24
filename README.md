# challenge-data-analysis-sql

Which percentage of companies are under which juridical form?

SQL Statement in enterprise joining code tables:
```sql
SELECT 
  e.JuridicalForm, 
  c.Description, 
  COUNT(*) AS juridical_form_count
FROM 
  enterprise AS e
INNER JOIN 
  code AS c 
    ON e.JuridicalForm = c.Code 
WHERE 
  c.Category = 'JuridicalForm'
GROUP BY 
  e.JuridicalForm, c.Description
ORDER BY 
  juridical_form_count DESC;

# Showing Top to down activities in Belgium using Nace2025:
sql
SELECT 
  substr(c."Code", 1, 2) AS nace_division,
  cd."Description" AS division_description,
  COUNT(e."EnterpriseNumber") AS count_enterprises,
  ROUND(
    100.0 * COUNT(e."EnterpriseNumber") / SUM(COUNT(e."EnterpriseNumber")) OVER (), 
    2
  ) AS percentage
FROM 
  enterprise AS e
INNER JOIN 
  activity AS a ON e."EnterpriseNumber" = a."EntityNumber"
INNER JOIN 
  code AS c ON a."NaceCode" = c."Code"
-- Second join to get description from division-level code
LEFT JOIN 
  code AS cd 
    ON substr(c."Code", 1, 2) = cd."Code"
    AND cd."Category" = 'Nace2025'
    AND cd."Language" = 'FR'
WHERE 
  c."Category" = 'Nace2025'
  AND c."Language" = 'FR'
GROUP BY 
  nace_division, division_description
ORDER BY 
  count_enterprises DESC;

## In the 43 Construction category, show top down percentage sub-categories:

sql
WITH division_total AS (
  SELECT 
    COUNT(e."EnterpriseNumber") AS total_enterprises
  FROM enterprise AS e
  INNER JOIN activity AS a ON e."EnterpriseNumber" = a."EntityNumber"
  INNER JOIN code AS c ON a."NaceCode" = c."Code"
  WHERE c."Category" = 'Nace2025'
    AND c."Language" = 'FR'
    AND substr(c."Code", 1, 2) = '43'
),

subcategory_counts AS (
  SELECT 
    c."Code" AS nace_code,
    c."Description",
    COUNT(e."EnterpriseNumber") AS count_enterprises
  FROM enterprise AS e
  INNER JOIN activity AS a ON e."EnterpriseNumber" = a."EntityNumber"
  INNER JOIN code AS c ON a."NaceCode" = c."Code"
  WHERE c."Category" = 'Nace2025'
    AND c."Language" = 'FR'
    AND substr(c."Code", 1, 2) = '43'
  GROUP BY c."Code", c."Description"
)

SELECT 
  s.nace_code,
  s."Description",
  s.count_enterprises,
  ROUND(100.0 * s.count_enterprises / dt.total_enterprises, 2) AS percentage_within_division
FROM 
  subcategory_counts AS s,
  division_total AS dt
ORDER BY 
  percentage_within_division DESC;

# Average, Min and Max Age per Sub Categories in Construction:
sql
SELECT
  a."NaceCode",
  c."Description",
  ROUND(AVG(
    (julianday('2025-07-24') - julianday(
      substr(e."StartDate", 7, 4) || '-' ||   -- YYYY
      substr(e."StartDate", 4, 2) || '-' ||   -- MM
      substr(e."StartDate", 1, 2)             -- DD
    )) / 365.25
  ), 2) AS avg_age_years,
  MIN(
    CAST(
      (julianday('2025-07-24') - julianday(
        substr(e."StartDate", 7, 4) || '-' ||   
        substr(e."StartDate", 4, 2) || '-' ||   
        substr(e."StartDate", 1, 2)             
      )) / 365.25 AS INTEGER
    )
  ) AS min_age_years,
  MAX(
    CAST(
      (julianday('2025-07-24') - julianday(
        substr(e."StartDate", 7, 4) || '-' ||   
        substr(e."StartDate", 4, 2) || '-' ||   
        substr(e."StartDate", 1, 2)             
      )) / 365.25 AS INTEGER
    )
  ) AS max_age_years
FROM 
  enterprise AS e
INNER JOIN 
  activity AS a ON e."EnterpriseNumber" = a."EntityNumber"
INNER JOIN
  code AS c ON a."NaceCode" = c."Code"
WHERE 
  a."NaceCode" LIKE '43%'
  AND c."Category" = 'Nace2025'
  AND c."Language" = 'FR'
  AND e."StartDate" IS NOT NULL
  AND length(e."StartDate") = 10
  AND substr(e."StartDate", 3, 1) = '-'
  AND substr(e."StartDate", 6, 1) = '-'
  AND CAST(substr(e."StartDate", 1, 2) AS INTEGER) BETWEEN 1 AND 31
  AND CAST(substr(e."StartDate", 4, 2) AS INTEGER) BETWEEN 1 AND 12
GROUP BY
  a."NaceCode", c."Description"
ORDER BY
  avg_age_years DESC;
