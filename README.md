# sunday_db

Create and activate playground schema:
```SQL
CREATE SCHEMA IF NOT EXISTS sunday_db;

SET search_path TO sunday_db;
```

## Native Example (baseline)

Create an auxallary function to generate random values for text fields:
```SQL
CREATE OR REPLACE FUNCTION generate_pseudo_text(target_length INTEGER) 
RETURNS TEXT AS $$
	SELECT array_to_string(ARRAY(
		SELECT 
			CASE 
				WHEN random()<0.1 THEN chr(32) 
				ELSE chr((65 + round(random() * 25)) :: integer) 
	         END 
		  FROM generate_series(1,target_length)), '')
$$ LANGUAGE sql;
```

Cleanup script (might be helpful if you made a mess with example data and want to start from scratch):
```SQL
DROP TABLE IF EXISTS native_stories_1M;
DROP TABLE IF EXISTS native_stories_500K;
DROP TABLE IF EXISTS native_stories_100K;
DROP TABLE IF EXISTS native_stories_50K;
DROP TABLE IF EXISTS native_stories_25K;
DROP TABLE IF EXISTS native_stories_10K;
DROP TABLE IF EXISTS native_stories_5K;
DROP TABLE IF EXISTS native_stories_1K;
DROP TABLE IF EXISTS native_stories_500;
DROP TABLE IF EXISTS native_stories_100;
```

Example data model representing something like Scrum "Story". There is no indexes (neither primary key) here intentially to show the native baseline.
```SQL
CREATE TABLE native_stories_1M (
	"num" INTEGER NOT NULL,
	"title" TEXT NOT NULL,
	"epic" TEXT NOT NULL,
	"description" TEXT NOT NULL,
	"created_date" DATE NOT NULL,
	"due_date" DATE NOT NULL,
	"status" TEXT NOT NULL,
	"priority" INTEGER NOT NULL,
	"complexity" INTEGER NOT NULL,
	"is_active" BOOLEAN NOT NULL
);
```
Filling native table with generated example data.
```SQL
-- 1_000_000 rows will be inserted.
-- it should take about 2 minutes.
-- most of the time spent in `generate_pseudo_text(...)`
INSERT INTO native_stories_1M ( 
	"num",
	"title",
	"epic",
	"description",
	"created_date",
	"due_date",
	"status",
	"priority",
	"complexity",
	"is_active"
) 
SELECT 
	gs.id AS "num",
	generate_pseudo_text((random() *   50 + 5)::integer) AS "title",
	generate_pseudo_text((random() *   20 + 5)::integer) AS "epic",
	generate_pseudo_text((random() * 1000 + 0)::integer) AS "description",
	(now() - mod(gs.id, 30)*interval '1 day')::date AS "created_date",
	(now() + mod(gs.id, 30)*interval '1 day')::date AS "due_date",
	CASE round(random()*4) 
		WHEN 0 THEN 'Not Started' 
		WHEN 1 THEN 'In Progress' 
		ELSE 'Completed' 
	END AS "status",
	round(random()*5)::integer AS "priority",
	round(random() * 100) AS "complexity",
	random()>0.8 AS "is_active"
FROM generate_series(1,1_000_000) AS gs(id);
```

Now fill other native examples using the generated data as a source:
```SQL
CREATE TABLE native_stories_500K AS SELECT * FROM native_stories_1M WHERE "num"<=  500_000; -- ~2 sec
CREATE TABLE native_stories_100K AS SELECT * FROM native_stories_1M WHERE "num"<=  100_000; -- ~1 sec
CREATE TABLE native_stories_50K  AS SELECT * FROM native_stories_1M WHERE "num"<=   50_000;
CREATE TABLE native_stories_25K  AS SELECT * FROM native_stories_1M WHERE "num"<=   25_000;
CREATE TABLE native_stories_10K  AS SELECT * FROM native_stories_1M WHERE "num"<=   10_000;
CREATE TABLE native_stories_5K   AS SELECT * FROM native_stories_1M WHERE "num"<=    5_000;
CREATE TABLE native_stories_1K   AS SELECT * FROM native_stories_1M WHERE "num"<=    1_000;
CREATE TABLE native_stories_500  AS SELECT * FROM native_stories_1M WHERE "num"<=      500;
CREATE TABLE native_stories_100  AS SELECT * FROM native_stories_1M WHERE "num"<=      100;
```

Native table row count and storage size statistics:

| ROWS | BYTES  | FETCH 1000 (x5 times)        | AVG MS | MS/1000 ROWS  | 
| ---: | -----: | ---------------------------: | -----: | ------------: |
|   1M | ~600MB | 3038, 2546, 3227, 2740, 2499 | 2810   |   2.810       | 
| 500K | ~300K  | 4518, 1682, 1373, 2027, 2484 | 2416   |   4.832       |
| 100K | ~60MB  |  827,  312,  555,  689,  536 |  583   |   5.830       |
|  50K | ~31MB  |  449,  207,  153,  292,  211 |  262   |   5.240       |
|  25K | ~15MB  |  168,  178,  254,  180,  109 |  177   |   7.080       |
|  10K | ~6MB   |  166,  120,   94,  163,  100 |  128   |  12.800       |
|   5K | ~3MB   |  160,  122,   86,  144,   99 |  122   |  24.400       |
|   1K | ~1MB   |  131,  146,  145,  117,   89 |  125   | 125.000       |
| 500  | ~512KB |   94,  122,   82,   93,  112 |  100   | 200.000       |
| 100  | ~64KB  |   10,   70,   81,  143,   80 |   76   | 760.000       |

So in average, a record takes ~600 bytes of native storage.

A script to calculate avg statistics:
```SQL
SELECT AVG(value) AS average FROM UNNEST(ARRAY[3038, 2546, 3227, 2740, 2499]) AS value;
SELECT AVG(value) AS average FROM UNNEST(ARRAY[4518, 1682, 1373, 2027, 2484]) AS value;
SELECT AVG(value) AS average FROM UNNEST(ARRAY[827,  312,  555,  689,  536]) AS value;
SELECT AVG(value) AS average FROM UNNEST(ARRAY[449,  207,  153,  292,  211]) AS value;
SELECT AVG(value) AS average FROM UNNEST(ARRAY[168,  178,  254,  180,  109]) AS value;
SELECT AVG(value) AS average FROM UNNEST(ARRAY[166,  120,   94,  163,  100]) AS value;
SELECT AVG(value) AS average FROM UNNEST(ARRAY[160,  122,   86,  144,   99]) AS value;
SELECT AVG(value) AS average FROM UNNEST(ARRAY[131,  146,  145,  117,   89]) AS value;
SELECT AVG(value) AS average FROM UNNEST(ARRAY[94,  122,   82,   93,  112]) AS value;
SELECT AVG(value) AS average FROM UNNEST(ARRAY[10,   70,   81,  143,   80]) AS value;
```
A script to calculate `MS/100 ROWS`:
```SQL
SELECT 
	round(2810/1_000_000.*1000,3) AS "1M", 
	round(2416/  500_000.*1000,3) AS "500K",
	round( 583/  100_000.*1000,3) AS "100K", 
	round( 262/   50_000.*1000,3) AS "50K", 
	round( 177/   25_000.*1000,3) AS "25K", 
	round( 128/   10_000.*1000,3) AS "10K", 
	round( 122/    5_000.*1000,3) AS "5K", 
	round( 125/    1_000.*1000,3) AS "1K", 
	round( 100/      500.*1000,3) AS "500=", 
	round(  76/      100.*1000,3) AS "100="
```

## Dynamic Model Example (array based)

Cleanup script:
```SQL
DROP TABLE IF EXISTS index_data;
DROP TABLE IF EXISTS index_definition;
DROP TABLE IF EXISTS table_row;
DROP TABLE IF EXISTS column_definition;
DROP TABLE IF EXISTS table_definition;
```

Logical table model:
```SQL
CREATE TABLE table_definition (
	id bigint not null primary key,
	table_name text not null
);
```

Filling with data:
```SQL
INSERT INTO table_definition (id, table_name) VALUES
			(      100, 'Stories_100'),
			(      500, 'Stories_500'),
			(    1_000, 'Stories_1K'),
			(    5_000, 'Stories_5K'),
			(   10_000, 'Stories_10K'),
			(   25_000, 'Stories_25K'),
			(   50_000, 'Stories_50K'),
			(  100_000, 'Stories_100K'),
			(  500_000, 'Stories_500K'),
			(1_000_000, 'Stories_1M');
```



