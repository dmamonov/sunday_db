# sunday_db

Create and activate playground schema:
```SQL
CREATE SCHEMA IF NOT EXISTS sunday_db;

SET search_path TO sunday_db;
```

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

| ROWS | BYTES  |
| ---: | -----: |
|   1M | ~600MB |
| 500K | ~300K  |
| 100K | ~100MB |
|  50K | ~31MB  |
|  25K | ~15MB  |
|  10K | ~6MB   |
|   5K | ~3MB   |
|   1K | ~1MB   |
| 500  | ~512KB |
| 100  | ~64KB  |



