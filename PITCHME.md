% SQL and Databases for Research
% Eric Seymour, PhD
% June 14, 2017

# Overview

* Overview of databases for research
* Overview of SQL syntax
* Creating databases
* Using databases and SQL with property data
* Spatial databases and spatial queries 

---

## Database Overview
### (aka why not Excel)

> - DBs best for storing and working with large datasets
> - DBs keep related data files in a single place
> - DBs enable let you link different files using common fields
> - DBs enforce data integrity, e.g., only numbers in number column
> - SQL helps with research reproducibility (reusable code)

---

# Database Design

- DBs contain tables
- Tables consist of rows and columns
- Column have explicitly defined datatypes, e.g.,
	+ Text
	+ Integer
	+ Real (floating point data)

---

## Database Software

| Name           | Type         |
|:---------------|:-------------|
| MS Access      | File based   |
| SQLite         | File based   |
| MS SQL Server  | Server based |
| MySQL (Oracle) | Server based |
| PostgreSQL     | Server based |

---

# SQLite

* Free open source software
* Used can interact w/ DB through command line
* We will use [SQLite Manager plugin for Firefox](https://addons.mozilla.org/en-US/firefox/addon/sqlite-manager/)

---

# SQL Syntax

```sql
SELECT field1, field2 	/* specify fields */
FROM myTable			/* specify fields */
WHERE condition = 'met'	/* specify fields */
ORDER BY field2;		/* specify fields */

```
<!-- 
@[1](Select fields you want returned)
@[2](Select the table or tables you are querying)
@[3](Optional clause to filter results) -->

# Making the slides

```bash
pandoc -t revealjs \\
--highlight-style=zenburn \\
-s -o myslides.html PITCHME.md
```


```python
import sqlite3 as sql

con = sql.connect(db)
```

---

## SQL Functions

* SQL can returns summary information
* MIN, MAX, COUNT, AVG, SUM
* You need to wrap the field in the function

```sql
SELECT COUNT(*), AVG(price), MIN(price), MAX(price)
FROM cars;
```

---

### Changing the unit of analysis

* MCM data includes tract number
* We can use these data to find tracts w/ the highest or lowest percentage of residential structures in good condition
* Need to GROUP BY tract

```bash
head --lines=5 data.csv
```

---

### More Code

```sql
SELECT A.trt AS tract, B.good, A.structures, 
( B.good * 1.0 / A.structures )  * 100 as prct
FROM 
    (SELECT COUNT(*) AS structures, GEOID10_Tract AS trt
    FROM MCM 
    WHERE Condition!='' 
    GROUP BY GEOID10_Tract) AS A 
LEFT JOIN
    (SELECT COUNT(*) AS good, GEOID10_Tract AS trt 
    FROM MCM 
    WHERE Condition='good' 
    GROUP BY GEOID10_Tract) AS B 
ON A.trt = B.trt
ORDER BY ( B.good * 1.0 / A.structures )  * 100
LIMIT 10;
```