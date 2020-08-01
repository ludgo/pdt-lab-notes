We deduce spatial functions (i.e. ST_*) further from [Spatial Relationships](http://postgis.net/workshops/postgis-intro/spatial_relationships.html).

## 1

Let's identify both location objects first.

> SELECT name, operator, railway FROM planet_osm_point WHERE railway LIKE '%station%';

![](https://github.com/fiit-pdt-2019/lab-notes-ludko/blob/master/images/lab3-trainstation.png)

_Bratislava hlavná stanica_

> SELECT name, building FROM planet_osm_polygon WHERE building LIKE '%university%' AND name IS NOT NULL;

![](https://github.com/fiit-pdt-2019/lab-notes-ludko/blob/master/images/lab3-fiitstu.png)

> SELECT name, operator, ref FROM planet_osm_polygon WHERE name LIKE '%Fakulta%';

![](https://github.com/fiit-pdt-2019/lab-notes-ludko/blob/master/images/lab3-fiitstu2.png)

_Fakulta informatiky a informačných technológií STU_

With help of [this](https://gis.stackexchange.com/a/19600/151717) and [this](https://stackoverflow.com/a/13224812)

> SELECT ST_Distance_Sphere(planet_osm_point.way, planet_osm_polygon.way)  
FROM planet_osm_point, planet_osm_polygon  
WHERE planet_osm_point.name = 'Bratislava hlavná stanica' AND planet_osm_polygon.name = 'Fakulta informatiky a informačných technológií STU';

2598.567243387

Google maps measure (air) distance check: 2.60 km - confirms we correct.

## 2

> SELECT name, boundary, admin_level, way_area, way FROM planet_osm_polygon WHERE name LIKE '%Karlova%';

![](https://github.com/fiit-pdt-2019/lab-notes-ludko/blob/master/images/lab3-karlovaves.png)

We consider 2 relevant (geographically equivalent) rows with different [admin_level](https://wiki.openstreetmap.org/wiki/Key:admin_level)

![](https://github.com/fiit-pdt-2019/lab-notes-ludko/blob/master/images/lab3-karlovaves-map.png)

> SELECT l.name, l.admin_level, l.way_area, l.way, r.name  
 FROM planet_osm_polygon l  
 CROSS JOIN planet_osm_polygon r  
 WHERE r.name = 'Karlova Ves' AND r.admin_level = '9'  
 AND l.boundary = 'administrative' AND l.admin_level = '9'  
 AND ST_Touches(l.way, r.way) = true;
 
![](https://github.com/fiit-pdt-2019/lab-notes-ludko/blob/master/images/lab3-neighbours9.png)
![](https://github.com/fiit-pdt-2019/lab-notes-ludko/blob/master/images/lab3-neighbours9-map.png)

Replacing admin_level 9 by 10 we get

![](https://github.com/fiit-pdt-2019/lab-notes-ludko/blob/master/images/lab3-neighbours10.png)
![](https://github.com/fiit-pdt-2019/lab-notes-ludko/blob/master/images/lab3-neighbours10-map.png)

We observe the key difference - at level 10 _Staré mesto_ is divided into several _Oblasť_. Taking level 9, the answer is:

_Devín_  
_Dúbravka_  
_Bratislava - mestská časť Staré Mesto_

We can also see how the river below prevents the area below the river from bordering with _Karlova Ves_

Note: The word "districts" in task is quite misleding because based on [admin_level specification](https://wiki.openstreetmap.org/wiki/Tag:boundary%3Dadministrative#10_admin_level_values_for_specific_countries) for Slovakia:  
**8:   district borders** (SK: hranica okresu)  
9:   LAU 2 Obec (Town/Village), autonomous towns in Bratislava and Košice  
10:  Katastrálne územie (Cadastral place) (SK: katastrálne územie obce)  
Solving the task with 8 does not make any sense though.

## 3

> SELECT osm_id, name, boundary, "natural", water, waterway FROM planet_osm_line WHERE name ILIKE '%Danube%';

0 results. Better try Slovak

> SELECT osm_id, name, boundary, "natural", water, waterway, way FROM planet_osm_line WHERE name ILIKE '%Dunaj%';

![](https://github.com/fiit-pdt-2019/lab-notes-ludko/blob/master/images/lab3-dunaj.png)

We conclude from pgAdmin visialization: Austrian part of Danube river is named _Donau - Dunaj_ (30852710). The river then continues with _Dunaj_ (30459655) and finally with _Dunaj_ (30463028).

For bridges, first explore roughly

> SELECT DISTINCT name, bridge, bicycle, foot, highway, oneway, operator, railway, ref, surface, toll  
 FROM planet_osm_roads  
 WHERE bridge = 'yes' AND name IS NOT NULL;

![](https://github.com/fiit-pdt-2019/lab-notes-ludko/blob/master/images/lab3-bridge.png)

> SELECT l.name, l.way, r.name, r.operator, r.construction, r.railway, r.way  
 FROM planet_osm_line l  
 CROSS JOIN planet_osm_roads r  
 WHERE l.osm_id IN (30852710, 30459655, 30463028)  
 AND r.bridge = 'yes'  
 AND ST_Crosses(l.way, r.way) = true;
 
![](https://github.com/fiit-pdt-2019/lab-notes-ludko/blob/master/images/lab3-crossings.png)
![](https://github.com/fiit-pdt-2019/lab-notes-ludko/blob/master/images/lab3-crossings-map.png)

_Most Lafranconi_  
_Most Apollo_  
_Prístavný most_  
_Most SNP_  
_Starý most_

Doubled due to 2 crossings (lines in database). Missing name (tram lines) corresponds to _Starý most_. [List of bridges](https://sk.wikipedia.org/wiki/Zoznam_mostov_cez_Dunaj_v_Bratislave) check. _D4_ is coming soon, our data will be outdated.

## 4

Similarly to _Karlova Ves_ above, we identified polygon name _Dlhé diely_ at admin_level 10.

These are existing highway types

> SELECT array_to_string(array_agg(DISTINCT highway),';') from planet_osm_line;

bridleway;construction;cycleway;elevator;footway;living_street;motorway;motorway_link;path;pedestrian;platform;primary;primary_link;proposed;raceway;residential;road;secondary;secondary_link;service;steps;tertiary;tertiary_link;track;unclassified

Results can be filtered by them if necessary. However, **important is to take non-null highway results, so that district borders (which are also contained in planet_osm_line) are eliminated.**

> SELECT l.name, l.way  
 FROM planet_osm_line l  
 CROSS JOIN planet_osm_polygon r  
 WHERE r.name = 'Dlhé diely'  
 AND l.highway IS NOT NULL  
 AND ST_Intersects(l.way, r.way) = true;

![](https://github.com/fiit-pdt-2019/lab-notes-ludko/blob/master/images/lab3-dlhediely-correct.png)

> SELECT array_to_string(array_agg(DISTINCT l.name),';')  
 FROM planet_osm_line l  
 CROSS JOIN planet_osm_polygon r  
 WHERE r.name = 'Dlhé diely'  
 AND l.name IS NOT NULL  
 AND l.highway IS NOT NULL  
 AND ST_Intersects(l.way, r.way) = true;

Albína Brunovského;Beniakova;Blyskáčová;Cikkerova;Dlhé diely I;Dlhé diely II;Dlhé diely III;Ferdiša Kostku;Hany Meličkovej;Hlaváčikova;Iskerníková;Jamnického;Jána Stanislava;Kolískova;Komonicová;Kresánkova;Kuklovská;Ľudovíta Fullu;Majerníkova;Matejkova;Nad lúčkami;Nad Sihoťou;Na Kampárke;Pribišova;Stoklasová;Sumbalova;Svíbová;Tománkova;Veternicová;Vincenta Hložníka;Vyhliadka

Incorrect null highway result

![](https://github.com/fiit-pdt-2019/lab-notes-ludko/blob/master/images/lab3-dlhediely-null.png)

**ST_Within is also incorrect since it limits result to lines fully within district.** It result only in 394 rows compared to intersect's 444. (Well then depends what our aim is, for example with visualization)

![](https://github.com/fiit-pdt-2019/lab-notes-ludko/blob/master/images/lab3-dlhediely-within.png)

Another mistake is excluding lines without name from results

![](https://github.com/fiit-pdt-2019/lab-notes-ludko/blob/master/images/lab3-dlhediely-namedonly.png)
