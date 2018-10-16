Get the first or last row per group in MySQL
========================================

This is a quick code-through based on [this post](https://www.xaprb.com/blog/2006/12/07/how-to-select-the-firstleastmax-row-per-group-in-sql/).

# The Setup 

This data was created using [this data set](https://github.com/mjfrigaard/mysql-adventures/blob/master/data/fruit_example.csv).

```sql
USE fruit_db;
SHOW TABLES;
/*
+--------------------+
| Tables_in_fruit_db |
+--------------------+
| fruits             |
+--------------------+
1 row in set (0.01 sec)
*/
SELECT * FROM fruits;
/*
+---------+------------+-------+
| ﻿type   | variety    | price |
+---------+------------+-------+
| apple   | gala       |  2.79 |
| apple   | fuji       |  0.24 |
| apple   | limbertwig |  2.87 |
| orange  | valencia   |  3.59 |
| orange  | navel      |  9.36 |
| pear    | bradford   |  6.05 |
| pear    | bartlett   |  2.14 |
| cherry  | bing       |  2.55 |
| cherry  | chelan     |  6.33 |
+---------+------------+-------+
9 rows in set (0.00 sec)
*/
```

The goal is to get these data into a table with '*one maximum row from each group*'.

***WHAT WE WANT:***

```
+--------+----------+-------+
| type   | variety  | price |
+--------+----------+-------+
| apple  | fuji     |  0.24 | 
| orange | valencia |  3.59 | 
| pear   | bartlett |  2.14 | 
| cherry | bing     |  2.55 | 
+--------+----------+-------+
```

## 1 - The self-join

This can be done using the following:

```sql
SELECT type, 
  MIN(price) as minprice 
FROM fruits 
GROUP BY type;
/*
+--------+----------+
| type   | minprice |
+--------+----------+
| apple  |     0.24 |
| orange |     3.59 |
| pear   |     2.14 |
| cherry |     2.55 |
+--------+----------+
4 rows in set (0.00 sec)
*/
```

This gets me to one row per minimum price. 

> *select the rest of the row by joining these results back to the same table. Since the first query is grouped, it needs to be put into a subquery so it can be joined against the non-grouped table*:

```sql
SELECT 
  fruits.type, 
  fruits.variety, 
  fruits.price
FROM (
-- this is a subquery of the grouped query above
   SELECT type, MIN(price) AS minprice
   FROM fruits GROUP BY type
) AS grouped_fruits 
-- join back to itself
 INNER JOIN fruits AS fruits 
 ON fruits.type = grouped_fruits.type 
 AND fruits.price = grouped_fruits.minprice;
/*
+--------+----------+-------+
| type   | variety  | price |
+--------+----------+-------+
| apple  | fuji     |  0.24 |
| orange | valencia |  3.59 |
| pear   | bartlett |  2.14 |
| cherry | bing     |  2.55 |
+--------+----------+-------+
4 rows in set (0.00 sec)
*/
```

## 2 - The correlated subquery

This is less efficient because it is two `WHERE` statements instead of the `JOIN`. 

```sql
SELECT 
	type, 
    variety, 
    price
FROM 
	fruits
    -- this is where we add the MIN()
WHERE 
	price = (
    SELECT MIN(price) 
    FROM fruits AS subq_fruits 
    -- and another WHERE instead of a JOIN
    WHERE subq_fruits.type = fruits.type);
    
/*
+--------+----------+-------+
| type   | variety  | price |
+--------+----------+-------+
| apple  | fuji     |  0.24 |
| orange | valencia |  3.59 |
| pear   | bartlett |  2.14 |
| cherry | bing     |  2.55 |
+--------+----------+-------+
4 rows in set (0.01 sec)
*/
```

## 3 - Select the top N rows from each group 

> *"...Finding the first several from each group is not possible with that method because aggregate functions only return a single value."*

```sql
SELECT 
  type, 
  variety, 
  price
FROM 
  fruits
WHERE (
   SELECT
  COUNT(*) FROM fruits AS fruit_count
  /* select the variety from each type 
     where the variety is no more than
     the second-cheapest of that type */
   WHERE fruit_count.type = fruits.type 
   /* this is the 'second cheapest bit */
   AND fruit_count.price <= fruits.price
) <= 2;

/*
+--------+----------+-------+
| type   | variety  | price |
+--------+----------+-------+
| apple  | gala     |  2.79 |
| apple  | fuji     |  0.24 |
| orange | valencia |  3.59 |
| orange | navel    |  9.36 |
| pear   | bradford |  6.05 |
| pear   | bartlett |  2.14 |
| cherry | bing     |  2.55 |
| cherry | chelan   |  6.33 |
+--------+----------+-------+
8 rows in set (0.01 sec)
*/
```

## 4 - UNION ALL

Here we specify the `type` for each `SELECT` statement, then `ORDER BY` the `price` and only return the top `2` with `LIMIT`.

```sql
(SELECT * FROM fruits WHERE type = 'apple' ORDER BY price LIMIT 2)
UNION ALL
(SELECT * FROM fruits WHERE type = 'orange' ORDER BY price LIMIT 2)
UNION ALL
(SELECT * FROM fruits WHERE type = 'pear' ORDER BY price LIMIT 2)
UNION ALL
(SELECT * FROM fruits WHERE type = 'cherry' ORDER BY price LIMIT 2);
/*
+--------+----------+-------+
| type   | variety  | price |
+--------+----------+-------+
| apple  | fuji     |  0.24 |
| apple  | gala     |  2.79 |
| orange | valencia |  3.59 |
| orange | navel    |  9.36 |
| pear   | bartlett |  2.14 |
| pear   | bradford |  6.05 |
| cherry | bing     |  2.55 |
| cherry | chelan   |  6.33 |
+--------+----------+-------+
8 rows in set (0.00 sec)
*/
```

This would only work with a small number of `types`. 







