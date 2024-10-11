# How to create vector tiles from geometries in Oracle Database 23ai?

Duration: 15 minutes

**Vector tiles** are the preferred method for **modern map data delivery** with **full flexibility on design**.  
As of **Oracle Database version 23ai**, developers can easily create vector tiles directly from spatial data managed in the Oracle Database 23ai. With a simple SQL call, large amounts of spatial data are efficiently streamed to web or GIS clients.  
Vector tiles enable

* dynamic styling,
* fast performance,
* smooth map interactions,
* and dynamic map queries.

## Prerequisites

### Lab environment

This LiveLabs Sprint uses an **Autonomous Database 23ai** (Free Tier) in the Oracle Cloud. If you don´t have an Oracle Cloud account and an Autonomous Database (ADB instance) provisioned, use [this LiveLabs workshop](https://livelabs.oracle.com/pls/apex/r/dbpm/livelabs/view-workshop?wid=928) via the **LiveLabs Sandbox** ("Green button") option to set up the environment.  
For simplicity, we also use the ADB user `ADMIN`. If you create and use another database user, you need to **REST Enable** it.

### Lab data

If you don´t have a sample spatial data set available, you can download [this USGS Earthquakes data set](https://corgis-edu.github.io/corgis/csv/earthquakes/) (`earthquakes.csv`) containing point geometries.

## Load the data set into your Autonomous Database instance

1. Load the data using `Database Actions` > `Load Data` into a new table called `USGS_EARTHQUAKES`.

    <img src="./images/load-earthquake-data-using-database-actions.png" alt="Upload earthquake data" width="800"/>

2. Verify the uploaded data.

    Open a `SQL Worksheet` in `Database Actions` and run the following SQL statement:

    ```sql
    <copy>SELECT * FROM usgs_earthquakes;</copy>
    ```

## Convert LON/LAT values into point geometries

Next, perform the following simple steps within the already opened `SQL Worksheet`.

1. Add a new column to the table to store the geometries.

    ```sql
    <copy>ALTER TABLE usgs_earthquakes ADD (geometry SDO_GEOMETRY );
    </copy>
    ```

2. Convert LON/LAT values into geometries.

    ```sql
    <copy>UPDATE usgs_earthquakes
    SET geometry = SDO_GEOMETRY(location_longitude, location_latitude);
    </copy>
    ```

3. Verify the updated data.

    ```sql
    <copy>SELECT location_longitude, location_latitude, g.geometry.sdo_point.x AS lon, g.geometry.sdo_point.y AS lat, g.geometry.sdo_srid AS srid
    FROM usgs_earthquakes g;
    </copy>
    ```

4. Create a spatial index.

    ```sql
    <copy>CREATE INDEX usgs_earthquakes_sidx
    ON usgs_earthquakes (geometry)
    INDEXTYPE IS MDSYS.SPATIAL_INDEX_V2;
    </copy>
    ```

## Create Vector Tiles using SQL

The SQL function `GET_VECTORTILE` to create the Vector Tiles is included in the PL/SQL Package `SDO_UTIL`. Here is how you can use it:

```sql
<copy>SELECT SDO_UTIL.GET_VECTORTILE(
    TABLE_NAME => 'USGS_EARTHQUAKES',
    GEOM_COL_NAME => 'GEOMETRY',
    ATT_COL_NAMES => SDO_STRING_ARRAY('LOCATION_NAME','IMPACT_MAGNITUDE','LOCATION_LATITUDE','LOCATION_LONGITUDE'),
    TILE_X => :x,
    TILE_Y_PBF => :y,
    TILE_ZOOM => :z) AS vtile
FROM DUAL;
</copy>
```

Providing the following values for the binding variables:

```txt
x = 2
y = 2.pbf
z = 2
```

should return a BLOB in column `VTILE`. Well done.

## Set up a REST endpoint for the Vector Tiles

To access the Vector Tiles in a web application we need to set up a REST endpoint with a GET handler that specifies as source the Vector Tiles.  
Create a new REST endpoint by executing the next three SQL statements.

```sql
-- Create a REST Module
<copy>BEGIN
    ORDS.DEFINE_MODULE(
        P_MODULE_NAME => 'earthquakes',
        P_BASE_PATH => '/usgs/',
        P_ITEMS_PER_PAGE => 25,
        P_STATUS => 'PUBLISHED',
        P_COMMENTS => ''
    );
    COMMIT;
END;
/
</copy>
```

```sql
-- Create a Module Template
<copy>BEGIN
    ORDS.DEFINE_TEMPLATE(
        P_MODULE_NAME => 'earthquakes',
        P_PATTERN => 'vt/:z/:x/:y',
        P_PRIORITY => 0,
        P_ETAG_TYPE => 'HASH',
        P_COMMENTS => ''
    );
    COMMIT;
END;
/
</copy>
```

```sql
-- Create a Handler
<copy>BEGIN
    ORDS.DEFINE_HANDLER(
        P_MODULE_NAME => 'earthquakes',
        P_PATTERN => 'vt/:z/:x/:y',
        P_METHOD => 'GET',
        P_SOURCE_TYPE => ords.source_type_media,
        P_SOURCE => 'SELECT
                ''application/vnd.mapbox-vector-tile'' as mediatype,
                SDO_UTIL.GET_VECTORTILE(
                TABLE_NAME => ''USGS_EARTHQUAKES'',
                GEOM_COL_NAME => ''GEOMETRY'',
                ATT_COL_NAMES => sdo_string_array(''LOCATION_NAME'',''LOCATION_FULL'',''IMPACT_MAGNITUDE'',''LOCATION_LATITUDE'',''LOCATION_LONGITUDE''),
                TILE_X => :x,
                TILE_Y_PBF => :y,
                TILE_ZOOM => :z) AS vtile
            FROM dual',
        P_ITEMS_PER_PAGE => 25,
        P_COMMENTS => ''
    );
    COMMIT;
END;
/
</copy>
```

Congratulations. You have successfully defined a new REST Endpoint.

## Access the Vector Tiles via the REST endpoint

You can now copy the REST endpoint to get the Vector Tiles and use it in a sample web application.

In your Autonomous Database instance, go to `Database Actions` > `Developer` > `REST` > `Modules`. Click on module `earthquakes`. Then click on template `vt/:z/:x/:x`. And finally, click on the `GET` handler.  
You can now copy the URL for the REST endpoint. It has the following pattern:  

`https://it1ryzir5xuwlpe-<adb_name>.adb.<region>.oraclecloudapps.com/ords/<username>/usgs/vt/:z/:x/:y`

Now, download [this sample HTML file](./src/display-xyz-vector-tiles-points.html). Open it with an editor of your choice. Replace the placeholder text in the map source definition with the URL of your REST endpoint. Make sure to also change the pattern `/:z/:x/:y` with `/{z}/{x}/{y}.pbf`.

```html
// USGS Earthquakes
map.addSource('earthquakes', {
    type: 'vector',
    tiles: [
        "<<Insert your REST endpoint URL here and replace the /:z/:x/:y with /{z}/{x}/{y}.pbf>>"
    ]
});
```

You can now run this single page HTML web app. If you don´t have, for this purpose, a web server installed and running, then you can use, for example, **Visual Studio Code** and the **Live Server** extension. What you will see, is the following image:

<img src="./images/display-earthquake-vector-tiles.png" alt="Display earthquake vector tiles" width="800"/>

## Learn More

* [Oracle Spatial Developer´s Guide - Vector Tiles](https://docs.oracle.com/en/database/oracle/oracle-database/23/spatl/vector-tiles.html)
* [ORDS 101: Enabling Oracle Schemas for HTTPS/REST](https://www.thatjeffsmith.com/archive/2023/09/ords-101-enabling-oracle-schemas-for-https-rest/)
* [MapLibre GL JS](https://maplibre.org/maplibre-gl-js/docs/examples/vector-source/)

## Acknowledgements

* **Author** - Karin Patenge, Senior Principal Product Manager, Oracle Spatial and Graph
* **Contributors** -  ..., Oracle Spatial and Graph
* **Last Updated By/Date** - Karin Patenge, Oct 2024
