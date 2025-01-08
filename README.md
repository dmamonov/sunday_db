# sundayDB

This is an ellistrative example of key concepts sufficient to build a low latency logical database on top of the relational model.

This example is based on PostgreSQL, but it could be adapted to MySQL as well.

## Setting up the schema

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

```SQL
-- Insert 1,000,000 rows into the table (SQLite alternative)
WITH RECURSIVE sequence(id) AS (
    SELECT 1
    UNION ALL
    SELECT id + 1
    FROM sequence
    WHERE id < 1000000
)
INSERT INTO native_stories_1M (
    num,
    title,
    epic,
    description,
    created_date,
    due_date,
    status,
    priority,
    complexity,
    is_active
)
SELECT
    id AS num,
    (SELECT GROUP_CONCAT(
                    CASE
                        WHEN RANDOM() % 10 = 0 THEN ' '
                        ELSE CHAR(65 + (ABS(RANDOM()) % 26))
                        END,
                    ''
            )
     FROM sequence seq
     WHERE seq.id <= (ABS(RANDOM()) % 50 + 5)) AS title,
    (SELECT GROUP_CONCAT(
                    CASE
                        WHEN RANDOM() % 10 = 0 THEN ' '
                        ELSE CHAR(65 + (ABS(RANDOM()) % 26))
                        END,
                    ''
            )
     FROM sequence seq
     WHERE seq.id <= (ABS(RANDOM()) % 20 + 5)) AS epic,
    (SELECT GROUP_CONCAT(
                    CASE
                        WHEN RANDOM() % 10 = 0 THEN ' '
                        ELSE CHAR(65 + (ABS(RANDOM()) % 26))
                        END,
                    ''
            )
     FROM sequence seq
     WHERE seq.id <= (ABS(RANDOM()) % 1000)) AS description,
    DATETIME('now', printf('-%d days', id % 30)) AS created_date,
    DATETIME('now', printf('+%d days', id % 30)) AS due_date,
    CASE ROUND(RANDOM() % 4)
        WHEN 0 THEN 'Not Started'
        WHEN 1 THEN 'In Progress'
        ELSE 'Completed'
        END AS status,
    ROUND(RANDOM() % 5) AS priority,
    ROUND(RANDOM() % 100) AS complexity,
    RANDOM() > 0.8 AS is_active
FROM "sequence";
-- 1804ms
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

```SQL
-- same done in SQLite
CREATE TABLE native_stories_500K AS SELECT * FROM native_stories_1M WHERE "num"<=  500000; -- ~960ms
CREATE TABLE native_stories_100K AS SELECT * FROM native_stories_1M WHERE "num"<=  100000; -- ~330ms
CREATE TABLE native_stories_50K  AS SELECT * FROM native_stories_1M WHERE "num"<=   50000; -- ~263ms
CREATE TABLE native_stories_25K  AS SELECT * FROM native_stories_1M WHERE "num"<=   25000; -- ~208ms
CREATE TABLE native_stories_10K  AS SELECT * FROM native_stories_1M WHERE "num"<=   10000; -- ~198ms
CREATE TABLE native_stories_5K   AS SELECT * FROM native_stories_1M WHERE "num"<=    5000; -- ~203ms
CREATE TABLE native_stories_1K   AS SELECT * FROM native_stories_1M WHERE "num"<=    1000; -- ~185ms
CREATE TABLE native_stories_500  AS SELECT * FROM native_stories_1M WHERE "num"<=     500; -- ~177ms
CREATE TABLE native_stories_100  AS SELECT * FROM native_stories_1M WHERE "num"<=     100; -- ~183ms
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

## Indexed native table

For 5k items:
```SQL
CREATE TABLE native_stories_5k_indexed AS SELECT * FROM native_stories_5k;
CREATE INDEX i_native_stories_5k_indexed__complexity ON native_stories_5k_indexed(complexity);

SELECT * FROM native_stories_5k WHERE complexity=99 ORDER BY "num";
-- 115, 130, 108, 97, 163, 71

SELECT * FROM native_stories_5k_indexed WHERE complexity=99 ORDER BY "num";
-- 112, 97, 179, 95, 81, 171
```

```SQL
-- same for SQLite:

CREATE TABLE native_stories_5k_indexed AS SELECT * FROM native_stories_5k; --29ms
CREATE INDEX i_native_stories_5k_indexed__complexity ON native_stories_5k_indexed(complexity); --18ms

SELECT * FROM native_stories_5k WHERE complexity=99 ORDER BY "num"; --253ms

SELECT * FROM native_stories_5k_indexed WHERE complexity=99 ORDER BY "num"; --~100ms
```

For 100k items:
```SQL
CREATE TABLE native_stories_100k_indexed AS SELECT * FROM native_stories_100k;
CREATE INDEX i_native_stories_100k_indexed__complexity ON native_stories_100k_indexed(complexity);

SELECT * FROM native_stories_100k WHERE complexity=99 ORDER BY "num";
-- 191, 148 108, 167, 95, 136

SELECT * FROM native_stories_100k_indexed WHERE complexity=99 ORDER BY "num";
-- 73, 107, 96, 84, 139 82
```

```SQL
-- same for SQLite
CREATE TABLE native_stories_100k_indexed AS SELECT * FROM native_stories_100k; --194ms
CREATE INDEX i_native_stories_100k_indexed__complexity ON native_stories_100k_indexed(complexity); --72ms

SELECT * FROM native_stories_100k WHERE complexity=99 ORDER BY "num"; --407ms
-- 191, 148 108, 167, 95, 136

SELECT * FROM native_stories_100k_indexed WHERE complexity=99 ORDER BY "num";--131 ms
-- 73, 107, 96, 84, 139 82
```


## Logical Tables

Logical model (in contrast to native) is a model of a fixed tables structure 
desinged to represent various (dynamic) table models. 

Cleanup script:
```SQL
DROP TABLE IF EXISTS logical_index_data;
DROP TABLE IF EXISTS logical_index_model;
DROP TABLE IF EXISTS logical_row_data;
DROP TABLE IF EXISTS logical_column_model;
DROP TABLE IF EXISTS logical_table_model;
```

Logical table model:
```SQL
CREATE TABLE logical_table_model (
	id bigserial not null primary key,
	table_name text not null,
	aux_size integer default 0
);
```

Filling with data:
```SQL
INSERT INTO logical_table_model (table_name, aux_size) VALUES 
			('Stories_100',       100),
			('Stories_500',       500),
			('Stories_1K',      1_000),
			('Stories_5K',      5_000),
			('Stories_10K',    10_000),
			('Stories_25K',    25_000),
			('Stories_50K',    50_000),
			('Stories_100K',  100_000),
			('Stories_500K',  500_000),
			('Stories_1M',  1_000_000);
```

Check content:
```SQL
SELECT * FROM logical_table_model ORDER BY id;
```

Creating logical column model:
```SQL
CREATE TABLE logical_column_model (
	id bigserial not null primary key,
	table_id bigint not null references logical_table_model (id),
	column_name text not null,
	storage_offset integer not null -- starting from 1.
);
```

Creating 10 logical columns per each logical table:
```SQL
INSERT INTO logical_column_model (table_id, column_name, storage_offset)
SELECT "table".id, "column"."name", "column"."offset"
  FROM logical_table_model AS "table"
  JOIN (VALUES 
	       ('NUM',          1), -- Ordinal mumber of the story (integer)
	       ('TITLE',        2), -- Title of the story (text)
	       ('EPIC',         3), -- Epic category associated with the story (text)
	       ('DESCRIPTION',  4), -- Detailed description of the story (text)
	       ('CREATED_DATE', 5), -- Date when the story was created (date)
	       ('DUE_DATE',     6), -- Due date for the story (date)
	       ('STATUS',       7), -- Status of the story (e.g., "In Progress") (text)
	       ('PRIORITY',     8), -- Priority level of the story (integer)
	       ('COMPLEXITY',   9), -- Numeric estimate of complexity (numeric)
	       ('IS_ACTIVE',   10)  -- Flag indicating if the story is active (boolean)
	   ) AS "column"("name", "offset") ON 1=1;
```

Creating logical row:
```SQL
CREATE TABLE logical_row_data (
	id bigserial not null primary key,
	table_id bigint not null references logical_table_model (id),
	cells text[] not null -- alternatively you may use JSON/JSONB
);
CREATE INDEX i_logical_row_data__table_id ON logical_row_data(table_id);
```

Fill logical tables with example data:
```SQL
INSERT INTO logical_row_data (table_id, cells)
SELECT 
	model.id AS table_id,
	ARRAY[
		"num"::text,
		"title",
		"epic",
		"description",
		"created_date"::text,
		"due_date"::text,
		"status",
		"priority"::text,
		"complexity"::text,
		"is_active"::text
	] AS "cells"
 FROM logical_table_model AS model 
 JOIN native_stories_1M AS stories ON stories."num" <=model.aux_size
ORDER BY stories."num";
```

Example of fetching data from a logical table:
```SQL
SELECT 
 	cells[1] as "num", 
	cells[2] as "title", 
	cells[3] as "epic", 
	LEFT(cells[4],50)||'...' as "description_brief", 
	cells[5] as "created_date",
	cells[6] as "due_date", 
	cells[7] as "status", 
	cells[8] as "priority", 
	cells[9] as "complexity", 
	cells[10] as "is_active"
  FROM logical_row_data 
 WHERE table_id=(SELECT id FROM logical_table_model WHERE table_name='Stories_100')
 ORDER BY "num";
```

Addind straight filter:
```SQL
SELECT 
 	cells[1] as "num", 
	cells[2] as "title", 
	cells[3] as "epic", 
	LEFT(cells[4],50)||'...' as "description_brief", 
	cells[5] as "created_date",
	cells[6] as "due_date", 
	cells[7] as "status", 
	cells[8] as "priority", 
	cells[9] as "complexity", 
	cells[10] as "is_active"
  FROM logical_row_data 
 WHERE table_id=(SELECT id FROM logical_table_model WHERE table_name='Stories_100K')
   AND cells[9]='99' -- complexity
 ORDER BY "num";
```

Create logical index model:
```SQL
CREATE TABLE logical_index_model (
	id bigserial not null primary key,
	table_id bigint not null references logical_table_model (id),
	column_id bigint not null references logical_column_model (id)
);
```

Create example logical index:
```SQL
INSERT INTO logical_index_model (table_id, column_id)
SELECT 
	table_id, 
	id AS column_id 
  FROM logical_column_model 
 WHERE table_id=(SELECT id FROM logical_table_model WHERE table_name='Stories_100K')
   AND column_name='COMPLEXITY';

-- check data:
SELECT * FROM logical_index_model;
```

Create logicl index data storage (actual values that will be indexed):
```SQL
CREATE TABLE logical_index_data (
	index_id bigint not null,
	row_id bigint not null references logical_row_data(id),
	cell text null,
	PRIMARY KEY(index_id, row_id)
);
CREATE INDEX i_logical_index_data__index_id__cell ON logical_index_data(index_id, cell);
```

Index actual data:
```SQL
INSERT INTO logical_index_data("index_id", "row_id", "cell")
SELECT 
    "index".id AS "index_id",
    "row".id AS "row_id",
    "row".cells["column".storage_offset] AS "cell"
  FROM logical_index_model AS "index"
  JOIN logical_column_model AS "column" ON "column".id="index".column_id
  JOIN logical_row_data AS "row" ON "row".table_id="index".table_id;
```

Selecting data using index:
```SQL
WITH 
	"table" AS (SELECT id FROM logical_table_model WHERE table_name='Stories_100K'),
	"column" AS (SELECT id FROM logical_column_model WHERE table_id=(SELECT id FROM "table") AND column_name='COMPLEXITY'),
	"index" AS (SELECT id FROM logical_index_model WHERE table_id=(SELECT id FROM "table") AND column_id=(SELECT id FROM "column"))
SELECT 
 	cells[1] as "num", 
	cells[2] as "title", 
	cells[3] as "epic", 
	LEFT(cells[4],50)||'...' as "description_brief", 
	cells[5] as "created_date",
	cells[6] as "due_date", 
	cells[7] as "status", 
	cells[8] as "priority", 
	cells[9] as "complexity", 
	cells[10] as "is_active"
  FROM logical_row_data
 WHERE table_id=(SELECT id FROM logical_table_model WHERE table_name='Stories_100K')
   AND id IN (
		SELECT row_id 
		  FROM logical_index_data 
	     WHERE index_id=(SELECT id FROM "index")
	       AND cell='99')
   -- AND cells[9]='99' -- complexity
 ORDER BY "num";
```







