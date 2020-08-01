## 1

> WITH RECURSIVE factorial(n, factorial) as (  
	SELECT 0,1  
	UNION ALL  
	SELECT f.n+1, f.factorial * (f.n+1) FROM factorial f  
)  
SELECT * FROM factorial LIMIT 11;

![](https://github.com/fiit-pdt-2019/lab-notes-ludko/blob/master/images/lab6-factorial.png)

## 2

> WITH RECURSIVE fibonacci(n, previous, fibonacci) as (  
	SELECT 0,1,0  
	UNION ALL  
	SELECT f.n+1, f.fibonacci, f.fibonacci + f.previous FROM fibonacci f  
)  
SELECT n, fibonacci FROM fibonacci OFFSET 1 LIMIT 20;

![](https://github.com/fiit-pdt-2019/lab-notes-ludko/blob/master/images/lab6-fibonacci.png)

## 3

![](https://github.com/fiit-pdt-2019/lab-notes-ludko/blob/master/images/lab6-parts.png)

> WITH RECURSIVE parts(id) as (  
	SELECT id FROM product_parts WHERE name = 'chair'  
	UNION ALL  
	SELECT product_parts.id FROM product_parts, parts WHERE part_of_id = parts.id  
)  
SELECT name FROM product_parts JOIN parts ON part_of_id = parts.id;

![](https://github.com/fiit-pdt-2019/lab-notes-ludko/blob/master/images/lab6-recursion.png)

## 4

Forward

> WITH RECURSIVE hops(start_name, end_name, end_stop_id) as (  
	SELECT s.name, e.name, e.id  
	FROM connections c  
	JOIN stops s ON c.start_stop_id = s.id  
	JOIN stops e ON c.end_stop_id = e.id  
	WHERE s.name = 'Zochova' AND c.line = '39'  
	UNION  
	SELECT h.end_name, e.name, e.id  
	FROM hops h  
	JOIN connections c ON h.end_stop_id = c.start_stop_id  
	JOIN stops e ON c.end_stop_id = e.id  
	WHERE c.line = '39'  
)
SELECT start_name, end_name FROM hops;

![](https://github.com/fiit-pdt-2019/lab-notes-ludko/blob/master/images/lab6-zochova-forward.png)

Backward

> WITH RECURSIVE hops(start_name, end_name, start_stop_id) as (  
	SELECT s.name, e.name, s.id  
	FROM connections c  
	JOIN stops s ON c.start_stop_id = s.id  
	JOIN stops e ON c.end_stop_id = e.id  
	WHERE e.name = 'Zochova' AND c.line = '39'  
	UNION  
	SELECT s.name, h.start_name, s.id  
	FROM hops h  
	JOIN connections c ON h.start_stop_id = c.end_stop_id  
	JOIN stops s ON c.start_stop_id = s.id  
	WHERE c.line = '39'  
)  
SELECT start_name, end_name FROM hops;

![](https://github.com/fiit-pdt-2019/lab-notes-ludko/blob/master/images/lab6-zochova-backward.png)

Finally

> WITH RECURSIVE hops(start_name, end_name, start_stop_id, hop) as (  
	SELECT s.name, e.name, s.id, 1  
	FROM connections c  
	JOIN stops s ON c.start_stop_id = s.id  
	JOIN stops e ON c.end_stop_id = e.id  
	WHERE e.name = 'Zochova' AND c.line = '39'  
	UNION  
	SELECT s.name, h.start_name, s.id, h.hop+1  
	FROM hops h  
	JOIN connections c ON h.start_stop_id = c.end_stop_id  
	JOIN stops s ON c.start_stop_id = s.id  
	WHERE c.line = '39'  
)  
SELECT start_name as name, hop  
FROM hops  
WHERE hop < (SELECT hop FROM hops WHERE start_name = 'Zoo');

![](https://github.com/fiit-pdt-2019/lab-notes-ludko/blob/master/images/lab6-zochova-zoo.png)

