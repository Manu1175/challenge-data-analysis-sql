# Description

## Enterprise Data Analysis with SQL in Construction

This repository includes a series of SQL queries used to analyze Trends in the Construction Industry.

---

### 1. Top Activities in Belgium (NACE2025 Divisions)

**Question:** *Show top-down economic activity divisions based on number of enterprises.*

```sql
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
```

---

### 2. Subcategories in Construction (Division 43)

**Question:** *In the 43 Construction category, show top-down percentage sub-categories.*

```sql
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
```

---

### 3. Company Age Statistics in Construction

**Question:** *Average, min, max, stdev age per sub-category (in years) for NACE codes starting with '43'.*

```sql
SELECT
  sub."NaceCode",
  c."Description",
  ROUND(AVG(sub.age), 2) AS avg_age_years,
  ROUND(SQRT(AVG(sub.age * sub.age) - AVG(sub.age) * AVG(sub.age)), 2) AS stddev_age_years
FROM (
  SELECT DISTINCT
    e."EnterpriseNumber",
    a."NaceCode",
    (julianday('2025-07-24') - julianday(
      substr(e."StartDate", 7, 4) || '-' ||   -- YYYY
      substr(e."StartDate", 4, 2) || '-' ||   -- MM
      substr(e."StartDate", 1, 2)             -- DD
    )) / 365.25 AS age
  FROM 
    enterprise AS e
  INNER JOIN activity AS a ON e."EnterpriseNumber" = a."EntityNumber"
  WHERE 
    a."NaceCode" LIKE '43%'
    AND e."StartDate" IS NOT NULL
    AND length(e."StartDate") = 10
    AND substr(e."StartDate", 3, 1) = '-'
    AND substr(e."StartDate", 6, 1) = '-'
    AND CAST(substr(e."StartDate", 1, 2) AS INTEGER) BETWEEN 1 AND 31
    AND CAST(substr(e."StartDate", 4, 2) AS INTEGER) BETWEEN 1 AND 12
) AS sub
INNER JOIN code AS c ON sub."NaceCode" = c."Code"
WHERE 
  c."Category" = 'Nace2025'
  AND c."Language" = 'FR'
GROUP BY
  sub."NaceCode", c."Description"
ORDER BY
  avg_age_years DESC;
```

---

### 4. Aging Balance Top 5 Sub-Categories
**Question:** *How is the age spread in the top 5 sub-categories.*

```sql
WITH company_ages AS (
  SELECT DISTINCT
    e."EnterpriseNumber",
    d."Denomination",
    e."StartDate",
    a."NaceCode",
    c."Description" AS nace_description,
    CAST(
      (julianday('2025-07-24') - julianday(
        substr(e."StartDate", 7, 4) || '-' ||
        substr(e."StartDate", 4, 2) || '-' ||
        substr(e."StartDate", 1, 2)
      )) / 365.25 AS INTEGER
    ) AS age_years
  FROM 
    enterprise AS e
  INNER JOIN activity AS a ON e."EnterpriseNumber" = a."EntityNumber"
  INNER JOIN code AS c ON a."NaceCode" = c."Code"
  INNER JOIN denomination AS d 
    ON e."EnterpriseNumber" = d."EntityNumber"
    AND d."TypeOfDenomination" = 1
  WHERE 
    c."Category" = 'Nace2025'
    AND c."Language" = 'FR'
    AND d."Language" = 1
    AND a."NaceCode" IN (43333, 43222, 43320, 43332, 43343)
    AND e."StartDate" IS NOT NULL
    AND length(e."StartDate") = 10
    AND substr(e."StartDate", 3, 1) = '-'
    AND substr(e."StartDate", 6, 1) = '-'
    AND CAST(substr(e."StartDate", 1, 2) AS INTEGER) BETWEEN 1 AND 31
    AND CAST(substr(e."StartDate", 4, 2) AS INTEGER) BETWEEN 1 AND 12
)

SELECT
  "NaceCode",
  nace_description,
  COUNT(*) AS total_companies,

  COUNT(CASE WHEN age_years > 20 THEN 1 END) AS older_than_20_years,
  ROUND(100.0 * COUNT(CASE WHEN age_years > 20 THEN 1 END) / COUNT(*), 2) AS pct_older_than_20,

  COUNT(CASE WHEN age_years BETWEEN 10 AND 20 THEN 1 END) AS between_10_and_20_years,
  ROUND(100.0 * COUNT(CASE WHEN age_years BETWEEN 10 AND 20 THEN 1 END) / COUNT(*), 2) AS pct_between_10_and_20,

  COUNT(CASE WHEN age_years < 10 THEN 1 END) AS between_0_and_10_years,
  ROUND(100.0 * COUNT(CASE WHEN age_years < 10 THEN 1 END) / COUNT(*), 2) AS pct_between_0_and_10

FROM company_ages
GROUP BY "NaceCode", nace_description
ORDER BY "NaceCode";
```

---

### 5. Oldest companies

**Question:** *10 Oldest companies for the top 5 categories in sub-categories construction.*

```sql
WITH company_ages AS (
  SELECT DISTINCT
    e."EnterpriseNumber",
    d."Denomination",
    e."StartDate",
    a."NaceCode",
    c."Description" AS nace_description,
    CAST(
      (julianday('2025-07-24') - julianday(
        substr(e."StartDate", 7, 4) || '-' ||   -- YYYY
        substr(e."StartDate", 4, 2) || '-' ||   -- MM
        substr(e."StartDate", 1, 2)             -- DD
      )) / 365.25 AS INTEGER
    ) AS age_years
  FROM 
    enterprise AS e
  INNER JOIN 
    activity AS a ON e."EnterpriseNumber" = a."EntityNumber"
  INNER JOIN 
    code AS c ON a."NaceCode" = c."Code"
  INNER JOIN 
    denomination AS d 
      ON e."EnterpriseNumber" = d."EntityNumber"
     AND d."TypeOfDenomination" = 1  -- main name
  WHERE 
    c."Category" = 'Nace2025'
    AND c."Language" = 'FR'
    AND d."Language" = 1
    AND a."NaceCode" IN (43333, 43222, 43320, 43332, 43343)
    AND e."StartDate" IS NOT NULL
    AND length(e."StartDate") = 10
    AND substr(e."StartDate", 3, 1) = '-'
    AND substr(e."StartDate", 6, 1) = '-'
    AND CAST(substr(e."StartDate", 1, 2) AS INTEGER) BETWEEN 1 AND 31
    AND CAST(substr(e."StartDate", 4, 2) AS INTEGER) BETWEEN 1 AND 12
),
ranked_companies AS (
  SELECT 
    *,
    ROW_NUMBER() OVER (PARTITION BY "NaceCode" ORDER BY age_years DESC) AS rn
  FROM 
    company_ages
)
SELECT 
  "NaceCode",
  nace_description,
  "Denomination",
  "EnterpriseNumber",
  "StartDate",
  age_years
FROM 
  ranked_companies
WHERE 
  rn <= 10
ORDER BY 
  "NaceCode", age_years DESC;
```

---

### 6. Newest companies

**Question:** *10 newest companies for the top 5 categories in sub-categories construction.*
```sql
WITH company_ages AS (
  SELECT DISTINCT
    e."EnterpriseNumber",
    d."Denomination",
    e."StartDate",
    a."NaceCode",
    c."Description" AS nace_description,
    CAST(
      (julianday('2025-07-24') - julianday(
        substr(e."StartDate", 7, 4) || '-' ||   -- YYYY
        substr(e."StartDate", 4, 2) || '-' ||   -- MM
        substr(e."StartDate", 1, 2)             -- DD
      )) / 365.25 AS REAL
    ) AS age_years
  FROM 
    enterprise AS e
  INNER JOIN activity AS a ON e."EnterpriseNumber" = a."EntityNumber"
  INNER JOIN code AS c ON a."NaceCode" = c."Code"
  INNER JOIN denomination AS d 
      ON e."EnterpriseNumber" = d."EntityNumber"
     AND d."TypeOfDenomination" = 1
  WHERE 
    a."NaceCode" IN (43333, 43222, 43320, 43332, 43343)
    AND c."Category" = 'Nace2025'
    AND c."Language" = 'FR'
    AND d."Language" = 1
    AND e."StartDate" IS NOT NULL
    AND length(e."StartDate") = 10
    AND substr(e."StartDate", 3, 1) = '-'
    AND substr(e."StartDate", 6, 1) = '-'
    AND CAST(substr(e."StartDate", 1, 2) AS INTEGER) BETWEEN 1 AND 31
    AND CAST(substr(e."StartDate", 4, 2) AS INTEGER) BETWEEN 1 AND 12
),
ranked_newest AS (
  SELECT 
    *,
    ROW_NUMBER() OVER (PARTITION BY "NaceCode" ORDER BY age_years ASC) AS rn
  FROM company_ages
)
SELECT 
  "NaceCode",
  nace_description,
  "Denomination",
  "EnterpriseNumber",
  "StartDate",
  ROUND(age_years, 2) AS age_years
FROM 
  ranked_newest
WHERE 
  rn <= 10
ORDER BY 
  "NaceCode", age_years ASC;
```

---

### 7. History company creation construction
**Question:** *Last 10 years created companies in the construction sub-sectors and it's evolution*
```sql
SELECT 
  nace.Code AS NaceCode,
  nace.Description AS NaceDescription,
  SUBSTR(e."StartDate", 7, 4) AS StartYear,
  COUNT(DISTINCT e."EnterpriseNumber") AS EnterpriseCount,
  ROUND(
    100.0 * COUNT(DISTINCT e."EnterpriseNumber") 
    / SUM(COUNT(DISTINCT e."EnterpriseNumber")) OVER (PARTITION BY SUBSTR(e."StartDate", 7, 4)),
    2
  ) AS PercentagePerYear
FROM 
  enterprise AS e
INNER JOIN activity AS a 
  ON e."EnterpriseNumber" = a."EntityNumber"
INNER JOIN code AS nace 
  ON a."NaceCode" = nace.Code 
  AND nace.Category = 'Nace2025'
  AND nace.Language = 'FR'
WHERE 
  a."NaceCode" LIKE '43%' AND
  e."StartDate" IS NOT NULL AND
  LENGTH(e."StartDate") = 10 AND
  SUBSTR(e."StartDate", 3, 1) = '-' AND
  SUBSTR(e."StartDate", 6, 1) = '-' AND
  CAST(SUBSTR(e."StartDate", 7, 4) AS INTEGER) BETWEEN 2015 AND 2025
GROUP BY 
  nace.Code, nace.Description, StartYear
ORDER BY 
  StartYear, EnterpriseCount DESC;
```

---

### 8. History Juridical Form in the construction industry
**Question:** *Last 10 years created Juridical Forms in the construction sub-sectors and it's evolution*
```sql
 SELECT 
  CAST(SUBSTR(e."StartDate", 7, 4) AS INTEGER) AS StartYear,
  jf.Code AS JuridicalFormCode,
  jf.Description AS JuridicalFormDescription,
  COUNT(DISTINCT e."EnterpriseNumber") AS EnterpriseCount,
  ROUND(
    100.0 * COUNT(DISTINCT e."EnterpriseNumber")
    / SUM(COUNT(DISTINCT e."EnterpriseNumber")) OVER (PARTITION BY SUBSTR(e."StartDate", 7, 4)),
    2
  ) AS PercentagePerYear
FROM 
  enterprise AS e
INNER JOIN code AS jf 
  ON CAST(CAST(e."JuridicalForm" AS INTEGER) AS TEXT) = jf.Code
  AND jf.Category = 'JuridicalForm'
  AND jf.Language = 'FR'
WHERE 
  e."StartDate" IS NOT NULL AND
  LENGTH(e."StartDate") = 10 AND
  SUBSTR(e."StartDate", 3, 1) = '-' AND
  SUBSTR(e."StartDate", 6, 1) = '-' AND
  CAST(SUBSTR(e."StartDate", 7, 4) AS INTEGER) BETWEEN 2015 AND 2025
GROUP BY 
  StartYear, jf.Code, jf.Description
ORDER BY 
  StartYear ASC, EnterpriseCount DESC;
  ```

---
### 9. Spread Societe a Commandite
**Question:** *How is Societe a Commandite spread accross sub-categories?*
```sql
SELECT 
  nace.Code AS NaceSubCode,
  nace.Description AS SubCategory,
  COUNT(DISTINCT e."EnterpriseNumber") AS TotalCompanies,
  ROUND(
    100.0 * COUNT(DISTINCT e."EnterpriseNumber") 
    / SUM(COUNT(DISTINCT e."EnterpriseNumber")) OVER (),
    2
  ) AS Percentage
FROM 
  enterprise AS e
INNER JOIN 
  activity AS a ON e."EnterpriseNumber" = a."EntityNumber"
INNER JOIN 
  code AS nace 
    ON substr(CAST(a."NaceCode" AS TEXT), 1, 4) = REPLACE(nace.Code, '.', '') 
   AND nace.Category = 'Nace2025' 
   AND nace.Language = 'FR'
WHERE 
  e."JuridicalForm" IN (12, 13)
  AND a."NaceCode" BETWEEN 43000 AND 43999
GROUP BY 
  nace.Code, nace.Description
ORDER BY 
  TotalCompanies DESC;
  ```

---

### 10. Geographic spread of construction companies
**Question:** *Where are the main companies concentrated?*
```sql
  SELECT 
  addr."Zipcode",
  addr."MunicipalityFR",
  COUNT(DISTINCT e."EnterpriseNumber") AS company_count
FROM 
  enterprise AS e
INNER JOIN 
  activity AS a ON e."EnterpriseNumber" = a."EntityNumber"
INNER JOIN 
  address AS addr ON e."EnterpriseNumber" = addr."EntityNumber"
WHERE 
  a."NaceCode" LIKE '43%'
  AND addr.countryFR IS NULL
GROUP BY 
  addr."Zipcode", addr."MunicipalityFR"
ORDER BY 
  company_count DESC;
  ```

---

# Installation

# Usage
