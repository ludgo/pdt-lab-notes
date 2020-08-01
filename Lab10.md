## Setup

```
Windows 10 Pro
Docker Toolbox
VM 5120 MB
2 CPU

ES_JAVA_OPTS=-Xms512m -Xmx512m

docker-machine ssh default
sudo sysctl -w vm.max_map_count=262144
```

## 1

> GET http://192.168.99.100:9200/_cat/nodes?v

```
ip         heap.percent ram.percent cpu load_1m load_5m load_15m node.role master name
172.18.0.2           36          98  75    6.88   10.25     5.95 dil       -      es01
172.18.0.4           14          98  75    6.88   10.25     5.95 ilm       *      esmaster02
172.18.0.3           35          98  75    6.88   10.25     5.95 ilm       -      esmaster01
172.18.0.6           30          98  75    6.88   10.25     5.95 ilm       -      esmaster03
172.18.0.7           30          98  74    6.88   10.25     5.95 dil       -      es03
172.18.0.5           33          98  75    6.88   10.25     5.95 dil       -      es02
```

All the nodes joined the cluster. Master is at esmaster02. Tried 3 times, the master stayed the same.

## 2

> docker stop esmaster02

```
ip         heap.percent ram.percent cpu load_1m load_5m load_15m node.role master name
172.18.0.2           15          84  11    0.50    2.38     3.69 dil       -      es01
172.18.0.3           36          84  11    0.50    2.38     3.69 ilm       -      esmaster01
172.18.0.6           16          84  11    0.50    2.38     3.69 ilm       *      esmaster03
172.18.0.7           31          84  11    0.50    2.38     3.69 dil       -      es03
172.18.0.5           35          84  11    0.50    2.38     3.69 dil       -      es02
```

esmaster03 became new master.

> GET http://192.168.99.100:9200/_cluster/health?pretty

```
{
  "cluster_name" : "es-docker-cluster",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 5,
  "number_of_data_nodes" : 3,
  "active_primary_shards" : 0,
  "active_shards" : 0,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 0,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 100.0
}
```

My cluster status is **green**, thus is healthy.

## 3

PUT http://192.168.99.100:9200/amenity

with mapping in body

```
{
    "settings": {
        "index.number_of_shards": 2,
        "index.number_of_replicas": 1
    },
	"mappings" : {
	    "properties" : {
		  	"area" : {
		  		"type" : "text",
		  		"fields" : {
					"keyword" : {
						"type" : "keyword",
						"ignore_above" : 64
					}
				}
			},
			"amenity" : {
				"type" : "text",
				"fielddata": true
			}
		} 
	}
}
```

response

```
{
    "acknowledged": true,
    "shards_acknowledged": true,
    "index": "amenity"
}
```

We used following python script

```
from sqlalchemy import create_engine, text
import requests
import json

DB_URL = 'postgresql://{}:{}@{}:{}/{}'.format('postgres', 'password', 'localhost', '5433', 'gisproject')
engine = create_engine(DB_URL, pool_size=10, max_overflow=0)
engine.connect()

result = engine.execute(text('''
	WITH areas AS (  
		SELECT name, way  
		FROM planet_osm_polygon  
		WHERE boundary='administrative' AND admin_level='9'  
		-- for Belgium admin level 8 is municipality and 9 subdivision of municipality  
		-- https://wiki.openstreetmap.org/wiki/Tag:boundary%3Dadministrative#11_admin_level_values_for_specific_countries  
	)  
	SELECT areas.name, ARRAY_AGG(DISTINCT pp.amenity) AS amenity  
	FROM planet_osm_point pp  
	CROSS JOIN areas  
	WHERE amenity IN ('dentist', 'doctors', 'hospital') AND st_contains(areas.way, pp.way)  
	GROUP BY areas.name;
''')).fetchall()

print(result)
amenities = list(map(lambda x: { 'area': x[0], 'amenity': x[1] }, result))
print(amenities)
for a in amenities:
	print(json.dumps(a))
	requests.post('http://192.168.99.100:9200/amenity/_doc/', 
		headers={'Content-Type': 'application/json'}, 
		data=json.dumps(a))
```

## 4

> docker stop esmaster01

> GET http://192.168.99.100:9200/_cat/nodes?v

```
{"error":{"root_cause":[{"type":"master_not_discovered_exception","reason":null}],"type":"master_not_discovered_exception","reason":null},"status":503}
```

## 5

```
import requests
import json

result = [('area1', ['hospital']), ('area2', ['doctors','hospital'])]
print(result)
amenities = list(map(lambda x: { 'area': x[0], 'amenity': x[1] }, result))
print(amenities)
for a in amenities:
	print(json.dumps(a))
	requests.post('http://192.168.99.100:9200/amenity/_doc/', 
		headers={'Content-Type': 'application/json'}, 
		data=json.dumps(a))
```

> GET http://192.168.99.100:9200/amenity/_search

```
{"took":8,"timed_out":false,"_shards":{"total":2,"successful":2,"skipped":0,"failed":0},"hits":{"total":{"value":19,"relation":"eq"},"max_score":1.0,"hits":[{"_index":"amenity","_type":"_doc","_id":"iaEM-m4BF7cvgIr9ilhr","_score":1.0,"_source":{"area": "Bruxelles / Brussel", "amenity": ["dentist", "doctors", "hospital"]}},{"_index":"amenity","_type":"_doc","_id":"jKEM-m4BF7cvgIr9ilj2","_score":1.0,"_source":{"area": "Everberg", "amenity": ["dentist"]}},{"_index":"amenity","_type":"_doc","_id":"jaEM-m4BF7cvgIr9i1ga","_score":1.0,"_source":{"area": "Groot-Bijgaarden", "amenity": ["dentist"]}},{"_index":"amenity","_type":"_doc","_id":"kKEM-m4BF7cvgIr9i1iX","_score":1.0,"_source":{"area": "Machelen", "amenity": ["hospital"]}},{"_index":"amenity","_type":"_doc","_id":"kaEM-m4BF7cvgIr9i1jP","_score":1.0,"_source":{"area": "Melsbroek", "amenity": ["hospital"]}},{"_index":"amenity","_type":"_doc","_id":"lKEM-m4BF7cvgIr9jFhi","_score":1.0,"_source":{"area": "Sint-Agatha-Rode", "amenity": ["doctors"]}},{"_index":"amenity","_type":"_doc","_id":"lqEM-m4BF7cvgIr9jFjI","_score":1.0,"_source":{"area": "Sint-Stevens-Woluwe", "amenity": ["doctors"]}},{"_index":"amenity","_type":"_doc","_id":"l6EM-m4BF7cvgIr9jVgO","_score":1.0,"_source":{"area": "Sterrebeek", "amenity": ["dentist", "doctors"]}},{"_index":"amenity","_type":"_doc","_id":"h6EM-m4BF7cvgIr9iFjG","_score":1.0,"_source":{"area": "Asse", "amenity": ["hospital"]}},{"_index":"amenity","_type":"_doc","_id":"iKEM-m4BF7cvgIr9ilgN","_score":1.0,"_source":{"area": "Beersel", "amenity": ["doctors"]}}]}}
```

Search still works but showing previously inserted data only. Data inserted after esmaster01 removal are not present.

## 6

> docker start esmaster01

> GET http://192.168.99.100:9200/_cat/nodes?v

```
ip         heap.percent ram.percent cpu load_1m load_5m load_15m node.role master name
172.18.0.2           23          89   6    1.10    1.03     1.22 dil       -      es01
172.18.0.3           18          89   7    1.10    1.03     1.22 ilm       *      esmaster01
172.18.0.6           22          89   7    1.10    1.03     1.22 ilm       -      esmaster03
172.18.0.7           32          89   9    1.10    1.03     1.22 dil       -      es03
172.18.0.5           24          89   8    1.10    1.03     1.22 dil       -      es02
```

We back and green.

```
import requests
import json

result = [('area1', ['hospital']), ('area2', ['doctors','hospital'])]
print(result)
amenities = list(map(lambda x: { 'area': x[0], 'amenity': x[1] }, result))
print(amenities)
for a in amenities:
	print(json.dumps(a))
	requests.post('http://192.168.99.100:9200/amenity/_doc/', 
		headers={'Content-Type': 'application/json'}, 
		data=json.dumps(a))
```

> GET http://192.168.99.100:9200/amenity/_search

```
{"took":649,"timed_out":false,"_shards":{"total":2,"successful":2,"skipped":0,"failed":0},"hits":{"total":{"value":21,"relation":"eq"},"max_score":1.0,"hits":[{"_index":"amenity","_type":"_doc","_id":"iaEM-m4BF7cvgIr9ilhr","_score":1.0,"_source":{"area": "Bruxelles / Brussel", "amenity": ["dentist", "doctors", "hospital"]}},{"_index":"amenity","_type":"_doc","_id":"jKEM-m4BF7cvgIr9ilj2","_score":1.0,"_source":{"area": "Everberg", "amenity": ["dentist"]}},{"_index":"amenity","_type":"_doc","_id":"jaEM-m4BF7cvgIr9i1ga","_score":1.0,"_source":{"area": "Groot-Bijgaarden", "amenity": ["dentist"]}},{"_index":"amenity","_type":"_doc","_id":"kKEM-m4BF7cvgIr9i1iX","_score":1.0,"_source":{"area": "Machelen", "amenity": ["hospital"]}},{"_index":"amenity","_type":"_doc","_id":"kaEM-m4BF7cvgIr9i1jP","_score":1.0,"_source":{"area": "Melsbroek", "amenity": ["hospital"]}},{"_index":"amenity","_type":"_doc","_id":"lKEM-m4BF7cvgIr9jFhi","_score":1.0,"_source":{"area": "Sint-Agatha-Rode", "amenity": ["doctors"]}},{"_index":"amenity","_type":"_doc","_id":"lqEM-m4BF7cvgIr9jFjI","_score":1.0,"_source":{"area": "Sint-Stevens-Woluwe", "amenity": ["doctors"]}},{"_index":"amenity","_type":"_doc","_id":"l6EM-m4BF7cvgIr9jVgO","_score":1.0,"_source":{"area": "Sterrebeek", "amenity": ["dentist", "doctors"]}},{"_index":"amenity","_type":"_doc","_id":"mqEh-m4BF7cvgIr9I1g5","_score":1.0,"_source":{"area": "area1", "amenity": ["hospital"]}},{"_index":"amenity","_type":"_doc","_id":"m6Eh-m4BF7cvgIr9I1hy","_score":1.0,"_source":{"area": "area2", "amenity": ["doctors", "hospital"]}}]}}
```

Now search displays also newly inserted items.

> docker start esmaster01

> GET http://192.168.99.100:9200/_cat/nodes?v

```
ip         heap.percent ram.percent cpu load_1m load_5m load_15m node.role master name
172.18.0.5           16          98  69    3.34    1.33     1.20 dil       -      es02
172.18.0.2           17          98  69    3.34    1.33     1.20 dil       -      es01
172.18.0.3           29          98  69    3.34    1.33     1.20 ilm       *      esmaster01
172.18.0.6           19          98  69    3.34    1.33     1.20 ilm       -      esmaster03
172.18.0.7           17          98  69    3.34    1.33     1.20 dil       -      es03
172.18.0.4           23          98  84    3.34    1.33     1.20 ilm       -      esmaster02
```

## 7

> GET http://192.168.99.100:9200/_cat/shards

```
amenity 1 p STARTED 11  8.2kb 172.18.0.7 es03
amenity 1 r STARTED 11  8.2kb 172.18.0.5 es02
amenity 0 p STARTED 10 11.6kb 172.18.0.2 es01
amenity 0 r STARTED  8  8.1kb 172.18.0.5 es02
```

> docker stop es02

```
amenity 1 p STARTED    11  8.2kb 172.18.0.7 es03
amenity 1 r UNASSIGNED                      
amenity 0 p STARTED    10 11.7kb 172.18.0.2 es01
amenity 0 r UNASSIGNED      
```

After a while

```
amenity 1 r STARTED 11  8.2kb 172.18.0.2 es01
amenity 1 p STARTED 11  8.2kb 172.18.0.7 es03
amenity 0 p STARTED 10 11.7kb 172.18.0.2 es01
amenity 0 r STARTED 10 11.7kb 172.18.0.7 es03
```

> GET http://192.168.99.100:9200/_cat/indices

```
green open amenity RVXntSl_R-OFXSDKeOUv8Q 2 1 21 0 39.9kb 19.9kb
```

Status is **green**.

## 8

> docker stop es03

> GET http://192.168.99.100:9200/_cat/shards

```
amenity 1 p STARTED    11  8.2kb 172.18.0.2 es01
amenity 1 r UNASSIGNED                      
amenity 0 p STARTED    10 11.7kb 172.18.0.2 es01
amenity 0 r UNASSIGNED 
```

> GET http://192.168.99.100:9200/_cat/indices

```
yellow open amenity RVXntSl_R-OFXSDKeOUv8Q 2 1 21 0 19.9kb 19.9kb
```

Status is **yellow**.

## 9

```
import requests
import json

result = [('area3', ['hospital']), ('area4', ['doctors','hospital'])]
print(result)
amenities = list(map(lambda x: { 'area': x[0], 'amenity': x[1] }, result))
print(amenities)
for a in amenities:
	print(json.dumps(a))
	requests.post('http://192.168.99.100:9200/amenity/_doc/', 
		headers={'Content-Type': 'application/json'}, 
		data=json.dumps(a))
```

Search again displays previously indexed data only.

## 10

> docker stop es01

> GET http://192.168.99.100:9210/_cat/shards

```
amenity 1 p UNASSIGNED    
amenity 1 r UNASSIGNED    
amenity 0 p UNASSIGNED    
amenity 0 r UNASSIGNED 
```

> GET http://192.168.99.100:9210/_cat/indices

```
red open amenity RVXntSl_R-OFXSDKeOUv8Q 2 1  
```

Status is **red**.

> GET http://192.168.99.100:9210/_cluster/health?pretty

```
{
  "cluster_name" : "es-docker-cluster",
  "status" : "red",
  "timed_out" : false,
  "number_of_nodes" : 3,
  "number_of_data_nodes" : 0,
  "active_primary_shards" : 0,
  "active_shards" : 0,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 4,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 0.0
}
```

Status is **red**.


## 11

> docker start es01  
docker start es02  
docker start es03

No, we did not lose anything, everything is in the original state.
