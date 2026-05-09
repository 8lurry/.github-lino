## Installing GeoSpetialDB extensions

In Ubuntu noble install the following (which already installs GEOS, PROJ, GDAL & PostGIS):

```
$ sudo apt install gdal-bin libgdal-dev libgeos-dev libproj-dev proj-bin
$ sudo apt install postgresql postgresql-contrib postgis postgresql-16-postgis-3
```

Downloading bangladeshi Union level polygons:

- Download polygon data from
  https://data.humdata.org/dataset/geoboundaries-admin-boundaries-for-bangladesh
- Download `geoBoundaries-BGD-ADM4_simplified.geojson` file.
- Download `geoBoundaries-BGD-ADM3_simplified.geojson` file.
- Download `geoBoundaries-BGD-ADM2_simplified.geojson` file.
- Download `geoBoundaries-BGD-ADM1_simplified.geojson` file.

Install the downloaded data into `polygon` plugin models:

- run `goto pronto1 && pm import_geoboundaries path/to/data_dir --country bd`.
