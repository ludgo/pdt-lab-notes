## 2

First check whether not indexed yet
> SELECT indexname,indexdef
FROM pg_indexes
WHERE tablename = 'documents';

Let our example supplier be *SPP* and 
> SELECT * FROM documents WHERE supplier='SPP';

Obtaining N=5 values [msec] from pgAdmin execution  
438
429
407
409
417  
Mean 420 msec

> CREATE INDEX index_supplier ON documents (supplier);

The same query with index  
92
97
106
94
97  
Mean 97,2 msec

This means here we talk about approx. 4,32x speedup with the index  

> SELECT * FROM documents WHERE total_amount > 100000 and total_amount <= 999999999;

1308
1379
1385
1382
1395  
1369,8 msec

> CREATE INDEX index_total_amount ON documents (total_amount);

874
937
929
915
905  
Mean 912 msec

approx. 1,50x speedup with the index

Running *EXPLAIN ANALYZE* before query tells us plans:  
without index **Seq Scan**  
with index **Bitmap Heap Scan**

## 3

> DROP INDEX index_supplier;

> DROP INDEX index_total_amount;

Check all column names
> SELECT column_name
FROM INFORMATION_SCHEMA.COLUMNS 
WHERE table_name = 'documents';

N=500

> INSERT INTO documents  
(name,type,created_at,department,customer,supplier,supplier_ico,contracted_amount,published_on,effective_from,expires_on,note,pocet_stran,source)  
(SELECT    name,type,created_at,department,customer,supplier,supplier_ico,contracted_amount,published_on,effective_from,expires_on,note,pocet_stran,source  
FROM documents LIMIT 500);

without indices  
97
104
81
75
74  
Mean 86,2 msec

1 index
> CREATE INDEX index_supplier ON documents (supplier);

134
105
129
113
110
Mean 118,2 msec

2 indices
> CREATE INDEX index_total_amount ON documents (total_amount);

135
113
123
129
115  
Mean 123 msec

3 indices
> CREATE INDEX index_department ON documents (department);

147
127
127
111
116  
125,6 msec

Indexed is approx. 1,4x slower compared to no index. The observation is INSERT slows down with the 1st index but difference between 1st, 2nd or 3rd is not striking - while still additive. (Based empirically on recent experiment only.)

> DROP INDEX index_supplier;

> DROP INDEX index_total_amount;

> DROP INDEX index_department;

Low-cardinality column *department* because <100 results of

> SELECT DISTINCT(department) FROM documents;

High-cardinality column is *supplier* because thousands results of

> SELECT DISTINCT(supplier) FROM documents;

Without index, Mean 86,2 msec (see above)  
Index on *supplier* only, Mean 118,2 msec (see above)  
Index on *department* only

> CREATE INDEX index_department ON documents (department);

86
92
78
77
90  
Mean 84,6 msec

Conclusion is low-cardinality index had almost no effect on INSERT slow-down, whereas high-cardinality did. However in [the literature I found](https://www.ibm.com/developerworks/data/library/techarticle/dm-1309cardinal/index.html) it is suggested to avoid low-cardinality indices because they provide no gain and sometimes has the same negative impact on INSERT performance.

--------------------
Read further:  
(https://use-the-index-luke.com/sql/dml/insert)
