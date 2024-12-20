# Data cleaning process

Our first lines of code allow us to preview the data to visualise data cleaning requirements: 

```sql
SELECT *
FROM `careful-striker-437421-g8.divvy.202409_tripdata` 
LIMIT 1000
```

We clean rows with missing start_station_id OR end_station_id

```sql
DELETE FROM `careful-striker-437421-g8.divvy.202409_tripdata` 
WHERE start_station_id IS NULL OR
  end_station_id IS NULL
```

We ensure that start_station_id and start_station_name are consistent 

```sql
SELECT 
  start_station_name, 
  COUNT (DISTINCT start_station_id) AS duplicates
FROM `careful-striker-437421-g8.divvy.202409_tripdata` 
GROUP BY start_station_name
ORDER BY duplicates DESC
```

We identified two stations with duplicate values. To make sure that we have a unique station_id for each station_name, we create a separate table (’stations’) with clean station_id per station_name. 

```sql
SELECT DISTINCT
  start_station_name, 
  start_station_id, 
  start_lat, 
  start_lng
FROM careful-striker-437421-g8.divvy.202409_tripdata
ORDER BY start_station_name
```

In the ‘stations_v2’ table, we compute average latitude and longitude per station, clean column names and group by station_names. We add a new column for the clean_station_id and assign clean_station_id by row number. We save the results in a new table ‘stations_v2’

```sql
SELECT DISTINCT
  start_station_name AS station_name, 
  ROUND(AVG(start_lat),2) AS latitude,
  ROUND(AVG(start_lng),2) AS longitude, 
  ROW_NUMBER() OVER (ORDER BY start_station_name) AS station_id_new
FROM `careful-striker-437421-g8.divvy.stations`
GROUP BY start_station_name
```

We JOIN the two tables on start_station_name AND end_station_name with station_id_new and latitude and longitude. We repeat the process for end_stations:

```sql
SELECT
  trips.ride_id, 
  trips.rideable_type,
  trips.started_at,
  trips.ended_at,
  trips.end_station_name,
  trips.member_casual,
  stations.station_name AS start_station,
  stations.latitude AS start_latitude,
  stations.longitude AS start_longitude, 
  stations.station_id_new AS start_id
FROM `careful-striker-437421-g8.divvy.202409_tripdata` AS trips
JOIN careful-striker-437421-g8.divvy.stations_v2 AS stations
  ON stations.station_name = trips.start_station_name 
```

We compute ride_length in seconds and ride_distance in meters. We clean our table from rides where length is 0 OR distance is 0. We save results in a ‘202409_tripsdata_clean’

```sql
WITH new_table AS (
  SELECT *,
  TIMESTAMP_DIFF(ended_at, started_at, SECOND) AS ride_length, 
  ST_GEOGPOINT(start_longitude, start_latitude) AS start_geopoint, 
  ST_GEOGPOINT(end_longitude, end_latitude) AS end_geopoint
FROM `careful-striker-437421-g8.divvy.202409_tripdata_v3`)

SELECT *,
  ST_DISTANCE(start_geopoint, end_geopoint, TRUE) AS ride_distance
FROM new_table
WHERE ride_length > 0 AND ST_DISTANCE(start_geopoint, end_geopoint, TRUE) > 0 
```
