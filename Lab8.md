We used following python script

 ```
import time
from sqlalchemy import create_engine, text

DB_URL = 'postgresql://{}:{}@{}:{}/{}'.format('postgres', '', '192.168.99.100', '5432', 'oz')
engine = create_engine(DB_URL, pool_size=10, max_overflow=0)
engine.connect()

N = 20000
start = time.time()
for ii in range(N):
   engine.execute(text('''
   	insert into documents(name, type, created_at, department, contracted_amount) 
   	values ('Generated doc', 'MyType', now(), 'Department', 100);
   	'''))
finish = time.time()
duration = finish - start

with open('./lab8.txt', 'a') as f:
	f.write('{} inserts, {} s\n'.format(N, duration))
	f.write('{} inserts/s\n'.format(N / duration))
	f.write('\n')
  ```

> docker run -p 5432:5432 fiitpdt/postgres

20000 inserts, 84.31908512115479 s
237.19422443048077 inserts/s

20000 inserts, 83.87826800346375 s
238.44078419900256 inserts/s

> docker run -p 5432:5432 fiitpdt/postgres postgres -c 'synchronous_commit=off'

20000 inserts, 35.740904331207275 s
559.5829309371152 inserts/s

20000 inserts, 35.07769536972046 s
570.1628852522698 inserts/s

## Conclusion

The standard command performed mean 237.817 inserts/s, the one with synchronous commit switched off mean 564.873 inserts/s, which means synchronous commit reduced time by 58%.
