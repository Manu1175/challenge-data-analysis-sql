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
