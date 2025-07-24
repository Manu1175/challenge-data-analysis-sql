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
