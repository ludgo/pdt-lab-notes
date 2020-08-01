## 1

> SELECT * FROM documents WHERE supplier_ico LIKE '%5733';

Took >440 ms. (estimation based on several runs)

Naive approach would be

> CREATE INDEX index_supplier_ico ON documents (supplier_ico);

Afterwards took approx. same, so no help.

> DROP INDEX index_supplier_ico;

With more advanced feature of Postgres

> CREATE INDEX index_supplier_ico ON documents (supplier_ico varchar_pattern_ops);

Afterwards took approx. same, so no help.

Index does not help probably because it indexes from prefix. What we need is suffix to be somehow processed by index.

## 2

> SELECT * FROM documents WHERE supplier_ico LIKE '57%';

In pgAdmin we measure execution:

Without index  
439 428 471 429 453  
Mean 444 ms  
EXPLAIN displays *Filter: ((supplier_ico)::text ~~ '57%'::text)*

With index  
86 103 112 104 105  
Mean 102 ms  
EXPLAIN has additional *Index Cond: (((supplier_ico)::text \~>=\~ '57'::text) AND ((supplier_ico)::text \~<\~ '58'::text))*

Thus we speak about 4.35x improvement. As mentioned before, yes index does use order of entries alphabetically starting from beginning of word.

> SELECT * FROM documents WHERE supplier_ico LIKE 'Ahoj%';

rewrites contition like *Index Cond: (((supplier_ico)::text \~>=\~ 'Ahoj'::text) AND ((supplier_ico)::text \~<\~ 'Ahok'::text))*

## 3

Taking advantage of previous knowledge

> CREATE INDEX index_supplier_ico ON documents (reverse(supplier_ico) varchar_pattern_ops);

indexes from the end. Then we search also reversed

> SELECT * FROM documents WHERE reverse(supplier_ico) LIKE reverse('%5733');

95 91 98 126 92  
Mean 100,4 ms  
EXPLAIN shows rewriten condition *Index Cond: ((reverse((supplier_ico)::text) \~>=\~ '3375'::text) AND (reverse((supplier_ico)::text) \~<\~ '3376'::text))*

## 5

> SELECT customer FROM documents WHERE department = 'Rozhlas a televizia Slovenska';

476 479 486 480 486  
Mean 481,4 ms  
Seq Scan

Index on the condition

> CREATE INDEX index_department ON documents (department varchar_pattern_ops);

168 190 176 179 171  
Mean 176,8 ms  
Bitmap Heap Scan  
is 2.72x faster.

Covering index

> CREATE INDEX index_covering ON documents (department, customer);

159 156 156 161 179  
Mean 162,2 ms  
Index Only Scan  
is 2.97x faster.

OR

> CREATE INDEX index_covering ON documents (department varchar_pattern_ops, customer varchar_pattern_ops);

156 160 168 180 153  
Mean 163,4 ms  
Index Only Scan  
is 2.95x faster.

In conclusion covering index was faster than index on the condition.

