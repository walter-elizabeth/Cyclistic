In Excel, the following actions were performed for each of the 12 excel sheets:

1. Filter to get a look of the data. 
2. Add a row called day_of_week, using started_at column, where 1 = Sunday, 2 = Monday, etc. 
3. Added a row called ride_length by subtracting ended_at column from started_at column.
4. Ride length was validated with conditional formatting using day_of_week and temp column day_of_week_end, where if day_of_week != day_of_week_end, ride_length was checked to be over 24 hrs
5. Ride length was properly formatted to account for rides that lasted over several days that were not being properly added up by the default formatting.

In SQL, the following queries were run to clean and manipulate the data:

1. After loading the data, the tables were combined. There was an issue with inconsistent data types that had to be fixed before all 12 tables could be joined. 
```sql
ALTER TABLE Cyclistic_Data.12_2020
RENAME column ride_length TO ride_length_string;

SELECT *,
SAFE_CAST(ride_length_string AS TIME) AS ride_length
FROM Cyclistic_Data.12_2020;

DROP TABLE IF EXISTS Cyclistic_Data.12_2020_cleaned;

CREATE TABLE Cyclistic_Data.12_2020_cleaned AS
(
SELECT
ride_id,
rideable_type,
started_at,
ended_at,
ride_length,
start_station_name,
start_station_id,
end_station_name,
end_station_id,
start_lat,
start_lng,
end_lat,
end_lng,
member_casual,
day_of_week
FROM Cyclistic_Data.12_2020
)

DROP TABLE IF EXISTS Cyclistic_Data.2020_combined_trips_temp;

CREATE TABLE Cyclistic_Data.2020_combined_trips_temp AS
(
  SELECT * FROM Cyclistic_Data.04_2020
  UNION ALL
  SELECT * FROM Cyclistic_Data.05_2020
  UNION ALL
  SELECT * FROM Cyclistic_Data.06_2020
  UNION ALL
  SELECT * FROM Cyclistic_Data.07_2020
  UNION ALL
  SELECT * FROM Cyclistic_Data.08_2020
  UNION ALL
  SELECT * FROM Cyclistic_Data.09_2020
  UNION ALL
  SELECT * FROM Cyclistic_Data.10_2020
  UNION ALL
  SELECT * FROM Cyclistic_Data.11_2020
);

DROP TABLE IF EXISTS Cyclistic_Data.2020_combined_trips;

CREATE TABLE Cyclistic_Data.2020_combined_trips AS
(
SELECT
  ride_id,
  rideable_type,
  started_at,
  ended_at,
  ride_length,
  start_station_name,
  SAFE_CAST(start_station_id AS STRING) AS start_station_id,
  end_station_name,
  SAFE_CAST(end_station_id AS STRING) AS end_station_id,
  start_lat,
  start_lng,
  end_lat,
  end_lng,
  member_casual,
  day_of_week

FROM Cyclistic_Data.2020_combined_trips_temp
);

DROP TABLE IF EXISTS Cyclistic_Data.combined_trips;

CREATE TABLE Cyclistic_Data.combined_trips AS
(
  SELECT * FROM Cyclistic_Data.2020_combined_trips
  UNION ALL
  SELECT * FROM Cyclistic_Data.12_2020_cleaned
  UNION ALL
  SELECT * FROM Cyclistic_Data.01_2021
    UNION ALL
  SELECT * FROM Cyclistic_Data.02_2021
    UNION ALL
  SELECT * FROM Cyclistic_Data.03_2021
    UNION ALL
  SELECT * FROM Cyclistic_Data.04_2021
)

```
2. Data was explored. I investigated the data, looked for nulls and errors and their possible causes and effects on the data.

```sql

SELECT
count(ride_id)
FROM Cyclistic_Data.combined_trips;
--- output: 3756620 rows

SELECT
count(member_casual)
FROM 
  Cyclistic_Data.combined_trips
WHERE
member_casual = "casual";
--- 2213971 members - 59% of users
--- 1542649 casual riders - 41%

SELECT
distinct(day_of_week)
FROM Cyclistic_Data.combined_trips;
--- seven days confirmed


--- create geography data types of long & lat
SELECT
  st_geogpoint(start_lng,start_lat) AS start_coords,
FROM Cyclistic_Data.combined_trips;

SELECT
  st_geogpoint(end_lng,end_lat) AS end_coords,
FROM Cyclistic_Data.combined_trips;



SELECT DISTINCT
  CONCAT(start_lat, start_lng) AS start_coords
FROM Cyclistic_Data.combined_trips
WHERE 
  start_lat IS NULL
  OR start_lng IS NULL;

--- no missing start coords

SELECT
  start_station_name
FROM Cyclistic_Data.combined_trips
  WHERE start_station_name IS NULL;

SELECT
  start_station_id
FROM Cyclistic_Data.combined_trips
  WHERE start_station_id IS NULL;

--- 552137 unique start coordinates
--- they say they only have 692 stations
--- 141898 entries where start station name is NULL
--- 142479 entries where start station id is NULL
      ---- discrepancy interesting. Cause? relevant?
--- no entries where start coordinates are NULL
--- those rides were started NOT at stations


SELECT DISTINCT
  end_station_name
FROM Cyclistic_Data.combined_trips
  WHERE end_station_name IS NOT NULL;

--- 711 stations listed - dont have the formal list of stations to compare to for errors; 
---   several "(Temp)" names may account for doubles. Do we keep?

SELECT DISTINCT
  start_station_name
FROM Cyclistic_Data.combined_trips
  WHERE start_station_name IS NOT NULL;

--- 713 start stations; presence of "(Temp)" stations

SELECT
  end_station_id
FROM Cyclistic_Data.combined_trips
  WHERE end_station_id IS NULL;

--- 158787 where end station name is NULL
--- 159201 where end station id is NULL
```
3. Data was segmented, aggregated, and further analyzed.
   
