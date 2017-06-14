% SQL and Databases for Research
% Eric Seymour, PhD
% June 14, 2017

# Spatial Databases

---

## Spatial Databases

* We can use coordinate/geometry data to make spatial tables
* SQLite -> Spatialite
* Special spatial SQL syntax for querying for relationships based on location
* QGIS makes it easy to create and work w/ Spatialite DBs

---

## Spatial DB Fundamentals

* Spatial data have three types of geometry: points, lines, polygons
* Polygons may contains points
* Points may be within polygons
* Census geographies are polygons
* Properties with simple X, Y coordinates are points

---

## Spatialite Tables from Shapefiles

* Open QGIS
* Navigate to Census Tracts 2010 folder
* Drag 'geo\_export\ldots.shp' into QGIS window
* Right-click layer name in table of contents in QGIS, select SAVE AS
* Under FORMAT, select SPATIALITE
* Change CRS (coordinate reference system) to 2898
* Rename table 'detroit\_tracts10'
* Save to workshop folder

---

## Coordinate Reference Systems

* Many different CRSs
* Each suited to different locations on the Earth's surface
* Each suited to different spatial scales, e.g., continents and states
* CRS 2898 is tailored to SE Michigan
* Distance in this CRS measured in __feet__

---

## Connecting to Databases

* Select feather icon
* Find database you created in last step
* Select tracts table
* Tracts should now be displayed in QGIS

---

## Import MCM into Spatialite

1. Navigate to Motor City Mapping folder
2. Drag 'Motor\_City\_Mapping\ldots.shp' into QGIS window
3. Right-click layer name in table of contents in QGIS, select SAVE AS
4. Under FORMAT, select SPATIALITE
5. Change CRC (coordinate reference system) to 2898
6. Rename table 'MCM'
7. Save into BD created when saving tract table

---

## QGIS DB Manager

* From the top menu, select Database, then DB Manager
* Find and connect to our new spatial DB
* Expand the DB and select any one table
* Press F2 to open the SQL Window

---

## Spatial SQL

We can query Spatialite DBs w/ special spatial parameters. 'ST' means Space-Time

* ST\_Contains(geom1, geom2)
* ST\_Within(geom1, geom2)
* ST\_Distance(geom1, geom2)
* ST\_Intersects(geom1, geom2)

----

## Spatial SQL, Cont.

[Follow link for complete list of spatial functions](http://www.gaia-gis.it/spatialite-2.4.0/spatialite-sql-2.4.html)

---

# Find Properties in Tracts

---


```sql
SELECT A.tractce_10, COUNT(*), A.geometry
FROM detroit_tracts10 AS A, mcm AS B
WHERE st_contains(A.geometry, B.geometry)
AND B.ROWID IN ( 
    SELECT ROWID 
    FROM SpatialIndex 
    WHERE f_table_name="mcm"
    AND search_frame=A.geometry)
GROUP BY A.tractce_10;
```

* line 3 is spatial join syntax
* lines 4-8 invoke spatial index, not mandatory, but speeds processing time

---

# Find Average Sale Price by Tract

---

```sql
SELECT A.geoid_10, AVG(C.SALEPRICE) AS avg_price, A.geometry
FROM detroit_tracts10 AS A JOIN mcm AS B
ON A.geoid_10 = B.geoid10_tr
JOIN detroit_sales AS C ON B.cityparcel = C.PARCELNO
WHERE SUBSTR(C.SALEDATE,1,4) IN ('2013','2014','2015')
AND SUBSTR(C.PROPCLASS,1,1)='4' /* residential only */
AND C.SalesInstr='WD' /* warranty deeds */
GROUP BY A.geoid_10
HAVING COUNT(*) >= 5 /* only tracts w/ 5+ sales */ ;
```
* line 5 IN allows selection of multiple values
* line 9 HAVING filters GROUP BY results


---

# Spatial analysis using census data

---

## Detroit tract level tenure 

* Collect 5-year ACS data for all tracts in Wayne County from [American Fact Finder](https://factfinder.census.gov/bkmk/table/1.0/en/ACS/11_5YR/B25003/0500000US26163.14000)
* Import into 'workshop_spatal.sqlite'
* Replace '.' in field names with '_'

```sql
CREATE TABLE "ACS_11_5YR_B25003" (
"GEO_id" INTEGER, 
"GEO_id2" INTEGER,
"GEO_label" TEXT,
"HD01_VD01" INTEGER,
"HD02_VD01" INTEGER
"HD01_VD02" INTEGER
"HD02_VD02" INTEGER
"HD01_VD03" INTEGER
"HD02_VD03" INTEGER)
```
---

## Join to census tract polygons

* Calculate homeownership rate
* Divide homeowner households by total households
* Metadata in 'ACS\_11\_5YR\_B25003\_metadata.csv' 

```sql
SELECT A.geoid_10, 
( B.HD01_VD02 * 1.0 / B.HD01_VD01 ) * 100 AS HO_rate,
A.geometry
FROM detroit_tracts10 AS A JOIN ACS_11_5YR_B25003 AS B
ON A.geoid_10 = B.GEO_id2
WHERE B.HD01_VD01 > 0; /* remove tracts w/ 0 households */
```

---

## Make choropleth map of homeownership rate

* Layers created from DB Manager in QGIS return all values as *string*
* Need to use the expression dialog in QGIS to convert to *real*

```sql
 to_real( "HO_rate" )
```

---

![Homeownership Rate](figures/ho_rate.png)

---

# Tax foreclosure choropleth map

---

## Steps

* [Download shapefile of dataset from Data Driven Detroit](http://portal.datadrivendetroit.org/datasets/9438afd734d348a694c42a28c4103731_0)
* Save data into 'workshop_spatial.db'
* Set CRS 2898 like rest of spatial tables in db
* Note these are *point* data

---

## Count foreclosures by tract

```sql
SELECT A.tractce_10, COUNT(*) AS foreclosure, A.geometry
FROM detroit_tracts10 AS A, tax_foreclosures AS B
WHERE st_contains(A.geometry, B.geometry)
AND B.ROWID IN ( 
    SELECT ROWID 
    FROM SpatialIndex 
    WHERE f_table_name="tax_foreclosures"
    AND search_frame=A.geometry)
GROUP BY A.tractce_10;
```

---

### Tax foreclosure concentration

* Join count of parcels per tract to normalize count
* Multiply by 1,000 to get rate

```sql
SELECT A.geoid_10, 
COUNT(*) * 1.0 / C.parcels * 1000 AS rate,
A.geometry
FROM detroit_tracts10 AS A, tax_foreclosures AS B
JOIN 
    (SELECT geoid10_tr, COUNT(*) AS parcels /* subquery of MCM */
    FROM mcm
    GROUP BY geoid10_tr) AS C
ON A.geoid_10 = C.geoid10_tr
WHERE st_contains(A.geometry, B.geometry)
AND B.ROWID IN ( 
    SELECT ROWID 
    FROM SpatialIndex 
    WHERE f_table_name="tax_foreclosures"
    AND search_frame=A.geometry)
GROUP BY A.geoid_10
HAVING C.parcels > 0;
```

---

![Tax Foreclosure Rate](figures/fc_rate.png)

---

# Property Ownership Analysis

---

## Detroit Property Ownership

* Shapefile from [Detroit Open Data Portal](https://data.detroitmi.gov/Property-Parcels/Parcel-Map/fxkw-udwf)
* Contains polygon for every parcel in Detroit 
* Time period reported to be March 2016
* Drag .shp file into QGIS and save into spatialite db
* We will find largest property owners
* And location of Fannie Mae's properties

---

## Largest Property Owners

```sql
SELECT taxpayer1, COUNT(*)
FROM ownership
GROUP BY taxpayer1
ORDER BY COUNT(*) DESC
LIMIT 7;
```

---


| taxpayer                     | COUNT |
|:-----------------------------|------:|
| DETROIT LAND BANK AUTHORITY  | 85479 |
| CITY OF DETROIT-P&DD         |  5964 |
| MI LAND BANK FAST TRACK AUTH |  5878 |
| HANTZ WOODLANDS LLC          |  1917 |
| TAXPAYER                     |  1235 |
| CITY OF DETROIT - P&DD       |  1058 |
| CITY OF DETROIT              |   950 |
| HUD                          |   782 |


---
