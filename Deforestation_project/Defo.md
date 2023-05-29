# SQL Queries Used

Project introduction (Create a View):
```
DROP VIEW IF EXISTS forestation;
CREATE VIEW forestation
AS
SELECT f.country_code,
        f.country_name,
        f.year,
        f.forest_area_sqkm,
        r.region,
        r.income_group,
        ( l.total_area_sq_mi * 2.59 ) AS total_area_sqkm,
        Round(( ( f.forest_area_sqkm / ( l.total_area_sq_mi * 2.59 ) ) * 100 )::numeric, 2) AS percent_forest
 FROM   forest_area f
        JOIN land_area l
          ON f.country_code = l.country_code
             AND f.year = l.year
        JOIN regions r
          ON r.country_code = f.country_code;
```

Global Situation

A.
```
SELECT Sum(forest_area_sqkm) AS total_forest_area
FROM forestation
WHERE year = 1990
    AND country_name = 'World';
```

B.
```
SELECT Sum(forest_area_sqkm) AS total_forest_area
FROM forestation
WHERE year = 2016
    AND country_name = 'World';
```
C.
```
SELECT (
         SELECT Sum(forest_area_sqkm) AS total_forest_area
         FROM forestation
         WHERE year = 1990
              AND country_name = 'World'
    ) - (
         SELECT Sum(forest_area_sqkm) AS total_forest_area
         FROM forestation
         WHERE year = 2016
              AND country_name = 'World'
    ) AS change_forest_area;
```
D.
```
WITH total_forest_area_1990 AS (
SELECT Sum(forest_area_sqkm) AS total_forest_area
         FROM   forestation
         WHERE  year = 1990
                AND country_name = 'World'
)
, total_forest_area_2016 AS (
SELECT Sum(forest_area_sqkm) AS total_forest_area
         FROM   forestation
         WHERE  year = 2016
                AND country_name = 'World')


SELECT ROUND(((((ELECT * FROM total_forest_area_1990) - (SELECT * FROM total_forest_area_2016)) / (SELECT * FROM total_forest_area_1990)) * 100)::numeric, 2) AS diff_percent;
```
E.
```
WITH total_forest_area_1990 AS (
SELECT Sum(forest_area_sqkm) AS total_forest_area
         FROM   forestation
         WHERE  year = 1990
                AND country_name = 'World'
)
, total_forest_area_2016 AS (
SELECT Sum(forest_area_sqkm) AS total_forest_area
         FROM   forestation
         WHERE  year = 2016
                AND country_name = 'World')


SELECT country_name,
total_area_sqkm,
ROUND(((abs(((SELECT * FROM total_forest_area_1990) - (SELECT * FROM total_forest_area_2016)) - total_area_sqkm)/total_area_sqkm)*100)::numeric, 2) AS err
FROM forestation
WHERE year = '2016'
ORDER BY err
LIMIT 1;
```

2.Regional Outlook
--all question
```
WITH total_percent_forest AS (
SELECT region,
year,
ROUND(((sum(forest_area_sqkm) / sum(total_area_sqkm))*100)::numeric, 2) AS percent_forest
FROM forestation
WHERE year IN (1990, 2016)
GROUP BY 1, 2)

SELECT region,
sum(case when year = 1990 then percent_forest else 0 end) as percent_forest_1990,
sum(case when year = 2016 then percent_forest else 0 end) as percent_forest_2016,
sum(case when year = 2016 then percent_forest else percent_forest*-1 end) as decrease_forest
FROM total_percent_forest
GROUP BY 1
ORDER BY 4;
```

3.Country-level detail
Success Stories
SS1.
```
WITH sf_1990 AS
(SELECT country_name, region, forest_area_sqkm
FROM forestation
WHERE year = 1990
AND forest_area_sqkm IS NOT NULL
AND region != 'World'),

sf_2016 AS
(SELECT country_name, region, forest_area_sqkm
FROM forestation
WHERE year = 2016
AND forest_area_sqkm IS NOT NULL
AND region != 'World')

SELECT sf_1990.country_name, sf_1990.region, (sf_2016.forest_area_sqkm - sf_1990.forest_area_sqkm) AS diff_forest_area
FROM sf_2016
JOIN sf_1990
USING (country_name)
WHERE sf_2016.forest_area_sqkm > sf_1990.forest_area_sqkm
ORDER BY 3 desc
LIMIT 5;
```
SS2.
```
SELECT sub1.country_name, sub2.region, ROUND(CAST( (((sub1.forest_area_sqkm - sub2.forest_area_sqkm) * 100) / (sub2.forest_area_sqkm)) AS NUMERIC), 2) AS diff_percent
FROM
  (SELECT country_name, region, forest_area_sqkm
   FROM forestation
   WHERE year = 2016) sub1
   JOIN
   (SELECT country_name, region, forest_area_sqkm
   FROM forestation
   WHERE year = 1990) sub2
ON sub1.country_name = sub2.country_name 
WHERE (sub1.forest_area_sqkm, sub2.forest_area_sqkm) IS NOT NULL    
ORDER BY 3 DESC
LIMIT 5;
```

Largest Concerns
A.
```
WITH sf_1990 AS
(SELECT country_name, region, forest_area_sqkm
FROM forestation
WHERE year = 1990
AND forest_area_sqkm IS NOT NULL
AND region != 'World'),

sf_2016 AS
(SELECT country_name, region, forest_area_sqkm
FROM forestation
WHERE year = 2016
AND forest_area_sqkm IS NOT NULL
AND region != 'World')

SELECT sf_1990.country_name, sf_1990.region, (sf_1990.forest_area_sqkm - sf_2016.forest_area_sqkm) AS diff_forest_area
FROM sf_1990
JOIN sf_2016
USING (country_name)
WHERE sf_1990.forest_area_sqkm > sf_2016.forest_area_sqkm
ORDER BY 3 DESC
LIMIT 5;
```

B.
```
WITH pf_1990 AS
(SELECT country_name, region, forest_area_sqkm
FROM forestation
WHERE year = 1990
AND forest_area_sqkm IS NOT NULL
AND region != 'World'),

pf_2016 AS
(SELECT country_name, region, forest_area_sqkm
FROM forestation
WHERE year = 2016
AND forest_area_sqkm IS NOT NULL
AND region != 'World')

SELECT pf_2016.country_name, pf_2016.region, ROUND(CAST( (((pf_1990.forest_area_sqkm - pf_2016.forest_area_sqkm) * 100) / (pf_1990.forest_area_sqkm)) AS NUMERIC), 2) AS percent_diff
from pf_1990
JOIN pf_2016
USING (country_name)
WHERE pf_1990.forest_area_sqkm > pf_2016.forest_area_sqkm
ORDER BY 3 DESC
LIMIT 5;
```

Quartiles
C.
```
WITH range_quart AS
(SELECT country_name, region,
(CASE
WHEN percent_forest BETWEEN 75 AND 100 THEN 'Fourth'
WHEN percent_forest <= 75 AND percent_forest > 50 THEN 'Third'
WHEN percent_forest <= 50 AND percent_forest > 25 THEN 'Second'
ELSE 'First' END) AS range_quartiles
FROM forestation
WHERE year = 2016
AND region != 'World'
AND percent_forest IS NOT NULL)

SELECT COUNT(country_name) AS most_countries, range_quartiles
FROM range_quart
GROUP BY 2
order by 1 DESC;
```

D.
```
WITH range_quart AS
(SELECT country_name, region, percent_forest,
(CASE
WHEN percent_forest BETWEEN 75 AND 100 THEN 'Fourth'
WHEN percent_forest <= 75 AND percent_forest > 50 THEN 'Third'
WHEN percent_forest <= 50 AND percent_forest > 25 THEN 'Second'
ELSE 'First' END) AS range_quartiles
FROM forestation
WHERE year = 2016
AND region NOT LIKE 'World'
AND percent_forest IS NOT NULL)

SELECT country_name,region, percent_forest
FROM range_quart
WHERE range_quartiles = 'Fourth'
order by 3 DESC;
```

E.
```
SELECT COUNT(country_code) AS countries_higher
FROM forestation
WHERE year = 2016
AND percent_forest > (SELECT percent_forest
                  FROM forestation WHERE year = 2016 AND country_code = 'USA');
```
