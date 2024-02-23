# Postgres

This activity creates a file postgres sql schema creation. It then creates a docker container from that file and allows you to run some commands. Some concepts this demonstrates are Docker, Joins, Relational Database Design, psql, grouping and aggregating data. The SQL Querries are designed to answer business questions, maybe your interested in purchasing cattle, or investigating why cattle do better in certain regions. 

Lets create a database
```sh
rm -rf postgres 2> /dev/null
mkdir postgres 2> /dev/null
cat << EOF > postgres/setup.sql
DROP TABLE IF EXISTS FeedingLog CASCADE;
DROP TABLE IF EXISTS LocationCattle CASCADE;
DROP TABLE IF EXISTS CattleSales CASCADE;
DROP TABLE IF EXISTS Location CASCADE;
DROP TABLE IF EXISTS Catle CASCADE;
DROP TABLE IF EXISTS Ranchers CASCADE;

CREATE TABLE Ranchers (
  rancher_id SERIAL PRIMARY KEY,
  rancher_name VARCHAR(255) NOT NULL,
  rancher_address VARCHAR(255),
  rancher_phone VARCHAR(20)
);

CREATE TABLE Catle (
  catle_id SERIAL PRIMARY KEY,
  catle_name VARCHAR(255) NOT NULL,
  age INTEGER,
  breed VARCHAR(255),
  weight DECIMAL(10,2),
  rancher_id INTEGER REFERENCES Ranchers(rancher_id)
);

CREATE TABLE Location (
  location_id SERIAL PRIMARY KEY,
  location_name VARCHAR(255) NOT NULL
);

CREATE TABLE LocationCattle (
  location_id INTEGER REFERENCES Location(location_id),
  catle_id INTEGER REFERENCES Catle(catle_id),
  PRIMARY KEY (location_id, catle_id)
);

CREATE TABLE FeedingLog (
  log_id SERIAL PRIMARY KEY,
  catle_id INTEGER REFERENCES Catle(catle_id),
  feeding_date DATE NOT NULL,
  feeding_time TIME NOT NULL,
  feed_type VARCHAR(100) NOT NULL,
  feed_amount DECIMAL(10,2),
  CONSTRAINT CHK_FeedAmount CHECK (feed_amount >= 0)
);

CREATE TABLE CattleSales (
  sale_id SERIAL PRIMARY KEY,
  seller_rancher_id INTEGER REFERENCES Ranchers(rancher_id),
  buyer_rancher_id INTEGER REFERENCES Ranchers(rancher_id),
  cattle_id INTEGER REFERENCES Catle(catle_id),
  sale_date DATE,
  sale_price DECIMAL(10,2)
);
-- Seed data for Ranchers table
INSERT INTO Ranchers (rancher_name, rancher_address, rancher_phone)
VALUES
  ('John Doe', '123 Main St', '555-1234'),
  ('Jane Smith', '456 Elm St', '555-5678'),
  ('David Johnson', '789 Oak St', '555-9012'),
  ('Sarah Williams', '321 Maple St', '555-3456'),
  ('Michael Brown', '654 Pine St', '555-7890'),
  ('Jennifer Davis', '987 Cedar St', '555-2345'),
  ('Daniel Wilson', '147 Birch St', '555-6789'),
  ('Emily Taylor', '258 Walnut St', '555-0123'),
  ('Christopher Clark', '369 Ash St', '555-4567'),
  ('Jessica Martinez', '951 Poplar St', '555-8901');

-- Seed data for Catle table
INSERT INTO Catle (catle_name, age, breed, weight, rancher_id)
SELECT
  'Cattle ' || i,
  FLOOR(RANDOM() * 10) + 1,
  CASE FLOOR(RANDOM() * 3)
    WHEN 0 THEN 'Angus'
    WHEN 1 THEN 'Hereford'
    WHEN 2 THEN 'Holstein'
  END,
  (FLOOR(RANDOM() * 500) + 500)::numeric(10, 2),
  (FLOOR(RANDOM() * 10) + 1)
FROM generate_series(1, 60) AS i
ORDER BY i;

-- Seed data for Location table
INSERT INTO Location (location_name)
SELECT
  'Location ' || i
FROM generate_series(1, 4) AS i
ORDER BY i;

-- Seed data for LocationCattle table
WITH cattle_locations AS (
  SELECT
    catle_id,
    (ROW_NUMBER() OVER ())::int - 1 AS row_num,
    ((ROW_NUMBER() OVER ())::int % 4) + 1 AS location_id
  FROM Catle
)
INSERT INTO LocationCattle (location_id, catle_id)
SELECT
  cl.location_id,
  cl.catle_id
FROM cattle_locations cl;

-- Seed data for FeedingLog table
INSERT INTO FeedingLog (catle_id, feeding_date, feeding_time, feed_type, feed_amount)
SELECT
  (FLOOR(RANDOM() * 60) + 1),
  CURRENT_DATE - (FLOOR(RANDOM() * 30) + 1) * INTERVAL '1 day',
  CURRENT_TIME - (FLOOR(RANDOM() * 24) + 1) * INTERVAL '1 hour',
  'Feed ' || (FLOOR(RANDOM() * 30) + 1),
  (FLOOR(RANDOM() * 10)::numeric(10, 2) + 1)
FROM generate_series(1, 30) AS i;

-- Seed data for CattleSales table
INSERT INTO CattleSales (seller_rancher_id, buyer_rancher_id, cattle_id, sale_date, sale_price)
SELECT
  (FLOOR(RANDOM() * 10) + 1),
  (FLOOR(RANDOM() * 10) + 1),
  (FLOOR(RANDOM() * 60) + 1),
  CURRENT_DATE - (FLOOR(RANDOM() * 30) + 1) * INTERVAL '1 day',
  (FLOOR(RANDOM() * 1000)::numeric(10, 2) + 1)
FROM generate_series(1, 30) AS i;


EOF
```

Use this docker file

```sh
cat << EOF > postgres/dockerfile
FROM public.ecr.aws/docker/library/postgres:13.11-bullseye
COPY setup.sql /docker-entrypoint-initdb.d/
EOF
```

Now build the image with the command below
```
docker build -tmypostgres:custom postgres/.
```

Run the image with this
```
docker run -it --name postgreSQLContainer -e POSTGRES_PASSWORD=catsdogs -d mypostgres:custom
```

Cleanup
```
docker rm -f postgreSQLContainer
docker rmi mypostgres:custom
```

## PSQL

Now to run some sql. Use this command to enter the psql shell.

```sh
docker exec -it postgreSQLContainer psql -U postgres
# this will enter a postgres shell
```

The command below will list the commands you can use. For example, **\l** will list databases and **\dt** database tables.
```
\?
```

Select from the tables. Below is an example of one select. 
```
SELECT * FROM ranchers limit 1;
```
```
 rancher_id | rancher_name | rancher_address | rancher_phone 
------------+--------------+-----------------+---------------
          1 | John Doe     | 123 Main St     | 555-1234
```

### Business Questions
The most important part of data analytics is being able to answer questions with business importance. Below are a list of questions and the corresponding SQL that answer them.

1. How many cattle does each rancher own? Limit your response to 5 and order by rancher_name.
```
SELECT r.rancher_name, COUNT(c.catle_id) AS cattle_count
FROM Ranchers r
LEFT JOIN Catle c ON r.rancher_id = c.rancher_id
GROUP BY r.rancher_id, r.rancher_name
ORDER BY rancher_name ASC limit 5;
```

```
   rancher_name    | cattle_count 
-------------------+--------------
 Christopher Clark |            5
 Daniel Wilson     |            3
 David Johnson     |            5
 Emily Taylor      |            8
 Jane Smith        |            7
(5 rows)
```

2. How much cattle is in each location?

```
SELECT l.location_name, COUNT(lc.catle_id) AS cattle_count
FROM Location l
LEFT JOIN LocationCattle lc ON l.location_id = lc.location_id
GROUP BY l.location_id, l.location_name;
```

```
 location_name | cattle_count 
---------------+--------------
 Location 3    |           15
 Location 4    |           15
 Location 2    |           15
 Location 1    |           15
(4 rows)
```

3. How much have the cows eaten per day this week?

```
SELECT feeding_date, SUM(feed_amount) AS total_feed_amount, COUNT(feed_amount) AS num_of_feeds
FROM FeedingLog
WHERE feeding_date >= CURRENT_DATE - INTERVAL '1 week'
GROUP BY feeding_date;
```
```
 feeding_date | total_feed_amount 
--------------+-------------------
 2023-07-06   |             34.00
 2023-07-08   |             41.00
 2023-07-09   |             11.00
 2023-07-07   |             15.00
 2023-07-10   |             30.00
 2023-07-11   |             35.00
 2023-07-05   |             17.00
(7 rows)
```

4. What is the average weight of each rancher's cattle, sorted by the highest averge weight?
```
 SELECT r.rancher_name, AVG(c.weight) AS avg_weight
FROM Ranchers r
LEFT JOIN Catle c ON r.rancher_id = c.rancher_id
GROUP BY r.rancher_id, r.rancher_name ORDER BY avg_weight DESC;
```
```
   rancher_name    |      avg_weight      
-------------------+----------------------
 Michael Brown     | 903.5714285714285714
 Emily Taylor      | 760.8750000000000000
 Sarah Williams    | 755.0000000000000000
 John Doe          | 754.2857142857142857
 Jane Smith        | 733.1428571428571429
 David Johnson     | 730.0000000000000000
 Christopher Clark | 714.2000000000000000
 Daniel Wilson     | 705.0000000000000000
 Jennifer Davis    | 693.8750000000000000
```


5. Are there any ranchers with no cattle?
```
SELECT r.rancher_name
FROM Ranchers r
LEFT JOIN Catle c ON r.rancher_id = c.rancher_id
WHERE c.catle_id IS NULL; # this would make an outer join
```

```
 rancher_name 
--------------
(0 rows)
```


6. What is the average feed abount for each cattle breed?

```
postgres=# SELECT c.breed, AVG(fl.feed_amount) AS avg_food_amount
FROM Catle c
JOIN FeedingLog fl ON c.catle_id = fl.catle_id
GROUP BY c.breed;
```

```
  breed   |  avg_food_amount   
----------+--------------------
 Holstein | 4.9615384615384615
 Hereford | 5.0408163265306122
 Angus    | 5.7142857142857143
(3 rows)
```

7. How much food is consumed in each location?
```
SELECT l.location_name, SUM(fl.feed_amount) AS total_food_amount
FROM Location l
JOIN LocationCattle lc ON l.location_id = lc.location_id
JOIN FeedingLog fl ON lc.catle_id = fl.catle_id
GROUP BY l.location_id, l.location_name;
```
```
 location_name | total_food_amount 
---------------+-------------------
 Location 3    |            177.00
 Location 4    |            175.00
 Location 2    |            212.00
 Location 1    |            221.00
(4 rows)
```

8. How many of each type of cow are in each location?

```
SELECT
  lc.location_id,
  c.breed AS cattle_type,
  COUNT(DISTINCT lc.catle_id) AS cattle_count
FROM LocationCattle lc
JOIN Catle c ON lc.catle_id = c.catle_id
GROUP BY lc.location_id, c.breed;
```
```
 location_id | cattle_type | cattle_count 
-------------+-------------+--------------
           1 | Angus       |            1
           1 | Hereford    |            7
           1 | Holstein    |            7
           2 | Angus       |            6
           2 | Hereford    |            3
           2 | Holstein    |            6
           3 | Angus       |            8
           3 | Hereford    |            3
           3 | Holstein    |            4
           4 | Angus       |            5
           4 | Hereford    |            4
           4 | Holstein    |            6
(12 rows)
```

9. What location had the most sales

```
SELECT l.location_name, COUNT(cs.sale_id) AS sale_count
FROM Location l
JOIN LocationCattle lc ON l.location_id = lc.location_id
JOIN CattleSales cs ON lc.catle_id = cs.cattle_id
GROUP BY l.location_id, l.location_name
ORDER BY sale_count DESC;
```
 location_name | sale_count 
---------------+------------
 Location 3    |         11
 Location 1    |          8
 Location 4    |          6
 Location 2    |          5


10. Have many sales of each cattle type has there been this past month?

```
SELECT c.breed, COUNT(cs.sale_id) AS sale_count
FROM Catle c
LEFT JOIN CattleSales cs ON c.catle_id = cs.cattle_id
WHERE cs.sale_date >= CURRENT_DATE - INTERVAL '1 month'
GROUP BY c.breed
ORDER BY sale_count DESC;
```
```
  breed   | sale_count 
----------+------------
 Hereford |         12
 Holstein |         11
 Angus    |          7
(3 rows)
```

11. How many sales have each rancher made, having at least 4 sales?

```
SELECT r.rancher_name, COUNT(cs.sale_id) AS sale_count
FROM Ranchers r
LEFT JOIN CattleSales cs ON r.rancher_id = cs.seller_rancher_id
GROUP BY r.rancher_id, r.rancher_name
HAVING COUNT(cs.sale_id) >= 4 -- can not use alias name here
ORDER BY sale_count DESC
;
```

```
  rancher_name  | sale_count 
----------------+------------
 Daniel Wilson  |          6
 Sarah Williams |          4
 Jennifer Davis |          4
 Emily Taylor   |          4
(4 rows)
```


12. Top selling cattle breed for each rancher


```
SELECT
  r.rancher_name,
  c.breed AS top_selling_cattle_type
FROM (
  SELECT
    seller_rancher_id,
    cattle_id,
    ROW_NUMBER() OVER (PARTITION BY seller_rancher_id ORDER BY COUNT(*) DESC) AS rn
  FROM CattleSales
  GROUP BY seller_rancher_id, cattle_id
) AS sub
JOIN Ranchers r ON sub.seller_rancher_id = r.rancher_id
JOIN Catle c ON sub.cattle_id = c.catle_id
WHERE rn = 1;
```
```
   rancher_name   | top_selling_cattle_type 
------------------+-------------------------
 John Doe         | Hereford
 Jane Smith       | Hereford
 David Johnson    | Hereford
 Sarah Williams   | Angus
 Jennifer Davis   | Angus
 Daniel Wilson    | Holstein
 Emily Taylor     | Hereford
 Jessica Martinez | Hereford
```

13. What is the average weight per location per breed of cow.

```
SELECT l.location_name, c.breed, AVG(c.weight) AS average_weight
FROM Location l
JOIN LocationCattle lc ON l.location_id = lc.location_id
JOIN Catle c ON lc.catle_id = c.catle_id
GROUP BY l.location_id, l.location_name, c.breed
ORDER BY average_weight DESC;
```

```
 location_name |  breed   |    average_weight    
---------------+----------+----------------------
 Location 2    | Angus    | 892.0000000000000000
 Location 3    | Angus    | 850.7500000000000000
 Location 1    | Angus    | 804.0000000000000000
 Location 4    | Holstein | 772.8000000000000000
 Location 4    | Hereford | 758.8000000000000000
 Location 1    | Holstein | 724.8000000000000000
 Location 3    | Hereford | 709.0000000000000000
 Location 1    | Hereford | 707.2000000000000000
 Location 2    | Holstein | 695.0833333333333333
 Location 4    | Angus    | 673.0000000000000000
 Location 2    | Hereford | 662.0000000000000000
 Location 3    | Holstein | 596.5000000000000000
(12 rows)
```
