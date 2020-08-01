## 1

Find all healthcare provider types per area in Brussels

> WITH areas AS (  
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

```
"name","amenity"

"Asse","{hospital}"
"Beersel","{doctors}"
"Bruxelles / Brussel","{dentist,doctors,hospital}"
"Diegem","{dentist}"
"Erps-Kwerps","{doctors}"
"Everberg","{dentist}"
"Groot-Bijgaarden","{dentist}"
"Hamme","{dentist}"
"Laeken / Laken","{dentist,doctors,hospital}"
"Machelen","{hospital}"
"Melsbroek","{hospital}"
"Peutie","{doctors}"
"Ruisbroek","{dentist}"
"Sint-Agatha-Rode","{doctors}"
"Sint-Joris-Weert","{doctors}"
"Sint-Stevens-Woluwe","{doctors}"
"Sterrebeek","{dentist,doctors}"
"Strombeek-Bever","{dentist,doctors}"
"Veltem-Beisem","{doctors}"
```

## 2

Document design

```
{
  "area": "Asse",,
  "amenity": [
	"dentist",
	"doctors",
	"hospital"
  ]
}
```

## 3

PUT http://192.168.99.100:9200/amenity

with mapping in body

```
{
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

## 4

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

## 5

GET http://192.168.99.100:9200/amenity/_search

```
{
    "took": 33,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 19,
            "relation": "eq"
        },
        "max_score": 1.0,
        "hits": [
            {
                "_index": "amenity",
                "_type": "_doc",
                "_id": "1t1G724B90Y1Apunp3a_",
                "_score": 1.0,
                "_source": {
                    "area": "Asse",
                    "amenity": [
                        "hospital"
                    ]
                }
            },
            {
                "_index": "amenity",
                "_type": "_doc",
                "_id": "191G724B90Y1ApunqHbX",
                "_score": 1.0,
                "_source": {
                    "area": "Beersel",
                    "amenity": [
                        "doctors"
                    ]
                }
            },
	    ...
        ]
    }
}
```

## 6

Now we wanna see how many subdivisions of municipalities in Brussels have at least a single healthcare provider of given type(=amenity). We execute count aggregation as

GET http://192.168.99.100:9200/amenity/_search

with body

```
{
    "aggs" : {
        "amenity" : {
            "terms" : { 
		"field" : "amenity" 
	    } 
        }
    },
    "size" : 0
}
```

response

```
{
    "took": 33,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 19,
            "relation": "eq"
        },
        "max_score": null,
        "hits": []
    },
    "aggregations": {
        "amenity": {
            "doc_count_error_upper_bound": 0,
            "sum_other_doc_count": 0,
            "buckets": [
		{
                    "key": "doctors",
                    "doc_count": 11
                },
                {
                    "key": "dentist",
                    "doc_count": 9
                },
                {
                    "key": "hospital",
                    "doc_count": 5
                }
            ]
        }
    }
}
```
