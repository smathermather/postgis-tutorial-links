postgis-tutorial-links
======================

http://bl.ocks.org/anonymous/raw/ac04c76460a874998783/

command for getting data:


git clone https://github.com/smathermather/postgis-tutorial-data.git
cd postgis-tutorial-data
ls
cd data/st_area
( if you are in data/disaggregation, do a 'cd ../st_area' )

 shp2pgsql -s 4326 waterstat_pacific.shp waterstat_pacific | \
 psql -U <your-username> -d <your-dbname>
 
(-s 4326 means WGS84 Latitude/Longitude)
Now we calculate area inside this table.To do so, we use ST_Area as our function

Code in SQL window:
-- Create column
ALTER TABLE waterstat_pacific
   ADD COLUMN area numeric;
-- Calculate area of geography in that column
UPDATE waterstat_pacific SET area = ST_Area(geom::geography) / 1000000;

---
Depending on where you are at, you'll switch directories to pgRouting
data directory... .

For example
cd postgis-tutorial-data/data/pgRouting
or
cd ../pgRouting
ls

I should see cleveland.osm in the directory.
cleveland.osm is an OpenStreetMap export.

We'll use osm2pgsql as an importer. This will take care of translating
a lot of our OpenStreetMap tags (the important ones we really need)
and it will convert to geometry

/usr/share/bin/osm2pgrouting -file cleveland.osm -conf \ 
/usr/share/osm2pgrouting/mapconfig.xml -dbname test \ 
-user SankAnHub -host localhost -prefixtables cleveland_ -clean

Also, if you want to copy and paste any (most) of these commands, see your
github repository... 

SELECT pgr_dijkstra( 'SELECT osm_id AS id, source, target,
length AS cost, x1, x2, y1, y2 FROM cleveland_ways',
3736,
3224,
false,
false);

But, this doesn't give us what we were expecting. I want geometries,
man!

CREATE TABLE public.route AS

WITH dijkstra AS (
	SELECT pgr_dijkstra( 'SELECT osm_id AS id, source, target, length AS cost, x1, x2, y1, y2 FROM cleveland_ways',
		3736,
		3224,
		false,
		false)
	)
SELECT osm_id, the_geom
	FROM cleveland_ways cw, dijkstra d
	WHERE cw.id2 = (d.pgr_dijkstra).id2;
	
---

cd ../pointcloud

In that directory there is a zip file and and xml file

unzip 18001190PAS.zip

that should give us 18001190PAS.las

We want this to be in a PostGIS database, so we're going to use two
1) PointCloud Extension
2) pdal (pointcloud data abstraction layer)

Pdal can dump las files into PostGIS.

Edit 18001190PAS.xml for database connection info:

<?xml version="1.0" encoding="utf-8"?>
<Pipeline version="1.0">
  <Writer type="drivers.pgpointcloud.writer">
    <Option name="connection">dbname='AAnkhBuns' user='AAnkhBuns' host='localhost'</Option>
    <Option name="table">a18001190PAS</Option>
    <Option name="srid">3362</Option>
    <Filter type="filters.chipper">
      <Option name="capacity">400</Option>
      <Filter type="filters.cache">
        <Reader type="drivers.las.reader">
          <Option name="filename">18001190PAS.las</Option>
          <Option name="spatialreference">EPSG:3362</Option>
        </Reader>
      </Filter>
    </Filter>
  </Writer>
</Pipeline>

pdal pipeline pc_config.xml

If you're on a Windows machine, set your shell to PowerShell

DROP TABLE IF EXISTS a18001190PAS_ph1;

CREATE TABLE a18001190PAS_ph1 AS
SELECT
  pa::geometry(Polygon, 3362) AS geom,
  PC_PatchAvg(pa, 'Z') AS elevation,
  PC_PatchMax(pa, 'Z') - PC_PatchMin(pa, 'Z') AS height
FROM a18001190PAS;

Recommended reading: http://workshops.boundlessgeo.com/tutorial-lidar/
also see:
python laspy + FLANN

--- 

DROP TABLE IF EXISTS patchexplode;

CREATE TABLE patchexplode AS
SELECT PC_Explode(pa)::geometry(PointZ, 3362) AS pt
	FROM a18001190pas;
	

-- So the code to explode actually removes our efficient storage of the points in
points in patches and breaks them out as points again. This could be
useful if we need individual access to the points, say for doing a 
height calculation agains the points, or analyzing some aspect of the
individuals.

PC_Explode is essentially the opposite operation from what the pdal 
command did when we put the data into the database.
