## accent

> **CREATE EXTENSION unaccent;**  
SELECT unaccent('unaccent', 'Išiel pštros s pštrosicou a pštrosíčatami Pštrosou ulicou.');

    Isiel pstros s pstrosicou a pstrosicatami Pstrosou ulicou.

## Slovak

> **CREATE TEXT SEARCH CONFIGURATION sk(copy = simple);  
ALTER TEXT SEARCH CONFIGURATION sk ALTER MAPPING FOR word WITH unaccent, simple;**  
SELECT to_tsvector('sk', 'Išiel pštros s pštrosicou a pštrosíčatami Pštrosou ulicou.');

    'a':5 'isiel':1 'pstros':2 'pstrosicatami':6 'pstrosicou':4 'pstrosou':7 's':3 'ulicou':8

## fulltext search

> **ALTER TABLE contracts ADD searchcol tsvector;  
UPDATE contracts SET searchcol = to_tsvector('sk',  
    name || ' ' || department || ' ' || customer || ' ' || supplier);**  
SELECT name, department, supplier, customer  
FROM contracts  
WHERE searchcol @@ to_tsquery('sk', 'oracle & socialna & poistovna');

2 rows, ~1 s

> **CREATE INDEX index_searchcol ON contracts USING gin(searchcol);**  
SELECT name, department, supplier, customer  
FROM contracts  
WHERE searchcol @@ to_tsquery('sk', 'oracle & socialna & poistovna');

2 rows, ~100 ms  
(approx. 10x faster)

## alphanumeric search

> SELECT DISTINCT supplier_ico  
FROM contracts  
WHERE cast(supplier_ico AS varchar) LIKE '3569%';

94 rows, ~850 ms

> **CREATE EXTENSION pg_trgm;  
CREATE INDEX index_supplier_ico ON contracts USING gin(cast(supplier_ico AS varchar) gin_trgm_ops);**  
SELECT DISTINCT supplier_ico  
FROM contracts  
WHERE cast(supplier_ico AS varchar) LIKE '3569%';

94 rows, ~120 ms  
(approx. 7x faster)

> **CREATE INDEX index_identifier ON contracts USING gin(cast(identifier AS varchar) gin_trgm_ops);**  
SELECT DISTINCT identifier  
FROM contracts  
WHERE cast(identifier AS varchar) LIKE '%2100%';

7384 rows, ~150 ms

> SELECT DISTINCT identifier  
FROM contracts  
WHERE cast(identifier AS varchar) LIKE '%OIaMIS%';

3 rows, ~90 ms

## boost

We define rank as

_(fulltext ts_rank) / MAX(fulltext ts_rank) + (days from oldest) / MAX(days from oldest)_

so that both parameters have got same weight.

> **WITH published_on_max_query AS(  
	SELECT published_on AS published_on_max  
	FROM contracts  
	WHERE searchcol @@ to_tsquery('sk', 'oracle') AND published_on IS NOT NULL  
	GROUP BY published_on  
	ORDER BY published_on ASC  
	LIMIT 1  
),  
published_on_min_query AS(  
	SELECT published_on AS published_on_min  
	FROM contracts  
	WHERE searchcol @@ to_tsquery('sk', 'oracle') AND published_on IS NOT NULL  
	GROUP BY published_on  
	ORDER BY published_on DESC  
	LIMIT 1  
),  
fulltext_score_max_query AS (  
	SELECT ts_rank(searchcol, to_tsquery('sk', 'oracle')) AS fulltext_score_max  
	FROM contracts  
	WHERE searchcol @@ to_tsquery('sk', 'oracle')  
	GROUP BY fulltext_score_max  
	ORDER BY fulltext_score_max DESC  
	LIMIT 1  
)  
SELECT id, name, searchcol,  
	ts_rank(searchcol, to_tsquery('sk', 'oracle')) / fulltext_score_max AS fulltext_score,  
	(CASE WHEN published_on IS NULL THEN published_on_max ELSE published_on END - published_on_max)\*1. / (published_on_min - published_on_max) AS published_on_score,  
	ts_rank(searchcol, to_tsquery('sk', 'oracle')) / fulltext_score_max  
	+ (CASE WHEN published_on IS NULL THEN published_on_max ELSE published_on END - published_on_max)\*1. / (published_on_min - published_on_max) AS boosted_score  
FROM contracts, published_on_max_query, published_on_min_query, fulltext_score_max_query  
WHERE searchcol @@ to_tsquery('sk', 'oracle') --AND published_on IS NOT NULL  
ORDER BY boosted_score DESC;**

![](https://github.com/fiit-pdt-2019/lab-notes-ludko/blob/master/images/lab7-boost.png)

## limitations

1. provide search by accronym  
For example would be nice to accept

> ...WHERE searchcol @@ to_tsquery('sk', 'oracle & mdv')

not only

> ...WHERE searchcol @@ to_tsquery('sk', 'oracle & ministerstvo & dopravy & vystavby')

2. handling special characters

> ...WHERE cast(identifier AS varchar) LIKE '271/§51/2012%'

and

> ...WHERE cast(identifier AS varchar) LIKE '271/51/2012%'

need to be searched by two distinct queries but in reality mean the same, thus should be united.
