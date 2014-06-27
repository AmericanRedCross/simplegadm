#**Simple GADM**

This is the step-by-step method American Red Cross (ARC) GIS used to build a topologically-correct simplified administrative boundary stack based on GADM v. 2. This was developed to provide global administrative boundaries for the world so that we can run database queries on a specific boundary levels that are appropriate for the analysis.

![alt text](/images/world.png)

Using basic simplification tools didn’t work (PostGIS’s ST_Simplify, ST_SimplifyPreserveTopology, QGIS’s Simplify geometries) due to topological errors in the original data. To prepare the data for the stack the topology was corrected, encoding converted, simplified while preserving topology, and then dissolved into separate layers. The software is open-source, stable, and scalable.

All methods were implemented with Mac OS X and Ubuntu 14.04 Trusty Tahr.
The software used:
* [QGIS 2.2 Valmiera](www.qgis.org/en/site/)
* [GRASS 6.4.3](grass.osgeo.org) (QGIS GRASS Plugin 0.1)
* [PPRepair](https://github.com/tudelft-gist/pprepair)
* [MapShaper 0.1.18](www.mapshaper.org)
* [PostGIS 2.1.0](www.postgis.net)
* [Homebrew 0.9.3](brew.sh)

Each application needs certain dependencies and plug-ins. Using a package manager like Homebrew or apt-get helps with proper installation.

>###Note
>
>These processes are stable but computationally heavy. An Amazon EC2 server was used for the big number crunching.

#**Instructions**

##**Topology Correction:**

We used the [GADM v. 2 boundaries](http://www.gadm.org) from UC Davis for our administrative boundary stack. Simplifying the original dataset caused a loss of features due to encoding and numerous topology errors due to poor initial topology.

Original and simplified France Admin 2 without topology correction:

---

![alt text](/images/original.png)
Original geometries, dissolved for the stack

---

![alt text](/images/bad_simp.png)
Original simplification (without topological considerations)

---

The following method can also be implemented with NaturalEarth and other boundary datasets for alternatives within the ARC boundary database.

###Check Topology

* Load the administrative boundaries into QGIS
* Download TopologyChecker Plugin
* Setup TopologyChecker:
  * Click Configure
  * Select your boundaries and the topological relationships you want to check (overlaps, gaps, and invalid geometries are recommended- FYI checking gaps can sometimes freeze up QGIS)

---

![alt text](/images/topochecker.png)
GADM v. 2 has 90,259 overlaps. France has 641; errors in red from TopologyChecker:

---

###Fix Topology

Fix topology with PPRepair and/or GRASS. GADM required using both, because of the size of the dataset and the extent of the errors. We encountered errors when using the programs independently, but by using both and correcting as few errors as possible in GRASS and the remainder in PPRepair, we found that all visible errors are accounted for and properly corrected.

For smaller datasets, PPRepair is usually sufficient.

####PPRepair

Install dependencies:

Linux:
```
$ sudo apt-get install build-essential cmake libcgal-dev libgdal-dev -y
```
Mac:
```
$ brew install cmake gdal cgal
```

Install Git:

Linux
```
$ sudo apt-get install git
```
Mac:
```
$ brew install git
```

Clone PPRepair from Git:
```
$ git clone https://github.com/tudelft-gist/pprepair.git
```

Each time you run PPRepair, you must follow these steps:
```
$ cmake .
$ make
$ ./pprepair -i ~/Desktop/GADM/gadm2.shp -o ~/Desktop/GADM/gadm2_ppr_output.shp -fix
```

Check the boundary data with TopologyChecker. If there are still errors, there are two options to correct them:
1. Correct by hand
* Edit the geometries and fix the errors that still exist by hand
* It's usually a good idea to re-run PPRepair to perfectly fit the boundaries

2. Use GRASS
* For larger datasets with lots of errors that PPRepair can't fix (like GADM) GRASS is very helpful in understanding the extent of the errors and correcting some of them. GRASS uses a topologic data model, and tries to correct errors at the import with an automated snapping tool. It's a simple process so it's not the best tool to use exclusively for topological correction.  
  *The output from GRASS sometimes contains other errors (invalid geometries) that are consistent with its own data model but not with shapefiles. Using PPRepair to correct these errors after using GRASS is necessary.

---

![alt text](/images/pprepair_corsica-01-01.png)

Two errors are highlighted that PPRepair didn't fix. When coupled with GRASS, the errors were fixed.

---

####GRASS


* Load GRASS (easiest to use through the QGIS Plugin)

![alt text](/images/grass_menu-01.png)

* Import boundary data with v.in.ogr

   * Select import thresholds:

      * Minimun Area set to 0

      * Snapping set to -1 (for no snapping)  
      OR  
      Set Snapping to an extremely low number (1E-13) to snap some, but not all, the errors. After an initial import with no snapping, the process output suggests a low threshold - this is a good starting point. A longer discussion of the reasons behind thresholding the snapping error correction can be found [here](/tutorial_draft/GADM_Correction_Doc.pdf) (PDF of tutorial rough draft).

* Export boundary data with v.out.ogr

   * Select shapefile output data format

---

![alt text](/images/grass_out1-01.png)
GRASS menu and v.out.ogr window.

---

* Dissolve into multipolygons and join attributes:

  * GRASS breaks multipart features into separate features, and it tags each feature with an ID value (cat) that designates the original multipart source feature. Use this value to dissolve the data. This can be done in ArcGIS, but we used MapShaper.


Install Node.js and npm

Linux:		
```
$ sudo apt-get install nodejs
```
Mac:		
```
$ brew install nodejs
```

Install MapShaper:
```
$ npm install -g mapshaper
```

Dissolve the shapefile:
```
$ mapshaper --dissolve cat boundariesFromGRASS.shp
```

Join original attributes:
```
$ mapshaper --join boundariesFromGRASS.dbf --join-keys cat,cat dissolvedBoundariesFromGRASS.shp
```

* Run PPRepair (see above directions)


---

##**Convert Encoding:**

Preserving the encoding of the text attributes is needed if the text is being used to label or relate to the geometry in a function. If the encoding is wrong, weird characters like ⚄ occur, and worse, if an SQL function calls a geometry by the text attribute that’s improperly encoded and starts with a ⚄, the geometry gets dropped.

* Open QGIS, select “Import Vector Layer”
* Import the layer with the proper source encoding.  
   Latin1 helps with foreign characters, ISO-8859-1 is the typical for Esri shapefiles)  
* Right-click the layer and select “Save as…”
* Select shapefile for data type and then select a new encoding in the pull-down menu (eg. UTF-8).
* Save shapefile.


---

##**Simplify:**

Load your shapefile into www.mapshaper.org to visually simplify or use the command-line tool installed earlier to power through it much faster.

The following example maintains small shapes and reduces the entire file to half its original size. [Full documentation](https://github.com/mbloch/mapshaper/wiki/Command-Reference) for MapShaper is available on GitHub.

```
$ mapshaper --keep-shapes -p 0.5 gadm_level.shp -o simp_gadm_level.shp
```
`
Original and simplified France Admin 2 geometries with all the errors corrected:

---

![alt text](/images/clean_topo.png)
Original without errors

---

![alt text](/images/simplified.png)
Simplified without errors

---

##**Build the Stack:**

###MapShaper Dissolve

Here we will dissolve the same file multiple times to get each stack level.

For GADM, there are six stack layers:

```
$ mapshaper --dissolve ID_0 --copy-fields ID_0,NAME_0 france.shp

$ mapshaper -filter "ID_1>0" --expression "CAT=(NAME_0+NAME_1)" --filter "$.partCount>0" --dissolve CAT --copy-fields ID_0,NAME_0,ID_1,NAME_1 france.shp

$ mapshaper -filter "ID_2>0" --expression "CAT=(NAME_0+NAME_1+NAME_2)" --dissolve CAT --copy-fields ID_0,NAME_0,ID_1,NAME_1,ID_2,NAME_2 france.shp

$ mapshaper -filter "ID_3>0" --expression "CAT=(NAME_0+NAME_1+NAME_2+NAME_3)" --dissolve CAT --copy-fields ID_0,NAME_0,ID_1,NAME_1,ID_2,NAME_2,ID_3,NAME_3 france.shp

$ mapshaper -filter "ID_4>0" --expression "CAT=(NAME_0+NAME_1+NAME_2+NAME_3+NAME_4)" --dissolve CAT --copy-fields ID_0,NAME_0,ID_1,NAME_1,ID_2,NAME_2,ID_3,NAME_3,ID_4,NAME_4 france.shp

$ mapshaper --filter "ID_5>0" --expression “CAT=(NAME_0+NAME_1+NAME_2+NAME_3+NAME_4+NAME_5)" --dissolve CAT --copy-fields ID_0,NAME_0,ID_1,NAME_1,ID_2,NAME_2,ID_3,NAME_3,ID_4,NAME_4,ID_5,NAME_5 france.shp
```


###PostGIS Input


Install PostGIS:

Linux
```
sudo apt-get install postgis
```
Mac
```
brew install postgis
```

Create and enable database:
```
$ createdb gadm_v2
$ psql gadm_v2
```
```
# create extension postgis;
# \q
```

Import the Shapefile:
```
$ shp2pgsql -I -s 4326 france_gadm0.shp france_gadm0 | psql -d gadm_v2
```