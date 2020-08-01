## 1 

> WITH fiit AS (  
        SELECT way FROM planet_osm_polygon WHERE name = 'Fakulta informatiky a informačných technológií STU'  
     ), restaurants AS (  
		SELECT name, way FROM planet_osm_point WHERE amenity = 'restaurant'  
	 )  
SELECT restaurants.name AS restaurant, ST_Distance_Sphere(fiit.way, restaurants.way) AS meters,  
	fiit.way AS fiit_way, restaurants.way AS restaurant_way  
FROM fiit, restaurants  
WHERE ST_Distance_Sphere(fiit.way, restaurants.way) < 1000  
ORDER BY meters;

![](https://github.com/fiit-pdt-2019/lab-notes-ludko/blob/master/images/lab5-table.png)

![Faculty](https://github.com/fiit-pdt-2019/lab-notes-ludko/blob/master/images/lab5-faculty.png)
![restaurants](https://github.com/fiit-pdt-2019/lab-notes-ludko/blob/master/images/lab5-restaurants.png)

## 2

EXPLAIN ANALYZE > Execution time [ms]  
71.912 75.222 74.490 75.974 77.502  
Mean 75.020

Index on _amenity_

> CREATE INDEX index_amenity ON planet_osm_point (amenity text_pattern_ops);

EXPLAIN ANALYZE > Execution time [ms]  
61.698 61.774 68.153 67.800 67.015  
Mean 65.288

Geoindex  

> CREATE INDEX index_point ON planet_osm_point USING gist((way::geography));

Based on the [lecture](https://github.com/fiit-pdt-2019/lectures/blob/master/05a%20-%20Spatial%20indices.pdf) we replace ST_Distance with ST_DWithin. Only ST_DWithin out of two [can use geoindex](https://postgis.net/2013/08/26/tip_ST_DWithin).

> WITH fiit AS (  
        SELECT way FROM planet_osm_polygon WHERE name = 'Fakulta informatiky a informačných technológií STU'  
     ), restaurants AS (  
		SELECT name, way FROM planet_osm_point WHERE amenity = 'restaurant'  
	 )  
SELECT restaurants.name AS restaurant, ST_Distance_Sphere(fiit.way, restaurants.way) AS meters,  
	fiit.way AS fiit_way, restaurants.way AS restaurant_way  
FROM fiit, restaurants  
WHERE ST_DWithin(fiit.way, restaurants.way, 1000, false) = true  
ORDER BY meters;

EXPLAIN ANALYZE > Execution time [ms]  
56.313 53.518 45.536 47.613 54.785  
Mean 51.553

We can see how, by indexing, **we saved arround 1/3 search time.**

## 3

> SELECT array_to_json(array_agg(row_to_json(row)))  
FROM (  
	WITH fiit AS (  
			SELECT way FROM planet_osm_polygon WHERE name = 'Fakulta informatiky a informačných technológií STU'  
		 ), restaurants AS (  
			SELECT name, way FROM planet_osm_point WHERE amenity = 'restaurant'  
		 )  
	SELECT restaurants.name AS restaurant, ST_Distance_Sphere(fiit.way, restaurants.way) AS meters,  
		ST_AsGeoJSON(restaurants.way)::json AS restaurant_way  
	FROM fiit, restaurants  
	WHERE ST_DWithin(fiit.way, restaurants.way, 1000, false) = true  
	ORDER BY meters  
) row;

[

   {

      "restaurant":"Bastion - Slovenská Koliba",

      "meters":35.722522494,

      "restaurant_way":{

         "type":"Point",

         "coordinates":[

            17.0722601,

            48.1547198

         ]

      }

   },

   {

      "restaurant":"Drag",

      "meters":365.692576368,

      "restaurant_way":{

         "type":"Point",

         "coordinates":[

            17.0672632,

            48.1509517

         ]

      }

   },

   {

      "restaurant":"Idyla",

      "meters":412.815058102,

      "restaurant_way":{

         "type":"Point",

         "coordinates":[

            17.0779226,

            48.1538106

         ]

      }

   },

   {

      "restaurant":"Družba",

      "meters":622.932728343,

      "restaurant_way":{

         "type":"Point",

         "coordinates":[

            17.0701073,

            48.147537

         ]

      }

   },

   {

      "restaurant":"Venza (študentská jedáleň)",

      "meters":674.029480863,

      "restaurant_way":{

         "type":"Point",

         "coordinates":[

            17.0690275,

            48.1605604

         ]

      }

   },

   {

      "restaurant":"Kamel",

      "meters":747.483590105,

      "restaurant_way":{

         "type":"Point",

         "coordinates":[

            17.0618877,

            48.150218

         ]

      }

   },

   {

      "restaurant":"Eat and Meet (študentská jedáleň)",

      "meters":771.380311496,

      "restaurant_way":{

         "type":"Point",

         "coordinates":[

            17.0672253,

            48.1610779

         ]

      }

   },

   {

      "restaurant":"Riviera",

      "meters":824.271029866,

      "restaurant_way":{

         "type":"Point",

         "coordinates":[

            17.0629032,

            48.1480213

         ]

      }

   },

   {

      "restaurant":"Seoul garden",

      "meters":832.934121791,

      "restaurant_way":{

         "type":"Point",

         "coordinates":[

            17.0772523,

            48.1610729

         ]

      }

   }

]
