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

---- data exploration 

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


SELECT DISTINCT
*
FROM Cyclistic_Data.start_stations;

--- stations repeat that have various start_lats and start_lngs
  --- WHY??? and what to do

SELECT DISTINCT
start_station_name
FROM Cyclistic_Data.start_stations
ORDER BY 
start_station_name;

--- 713 results.... should only be ~692 looks like all have 2 with diff station ids

SELECT
COUNT(DISTINCT(start_station_name))
FROM Cyclistic_Data.start_stations
WHERE CONTAINS_SUBSTR(start_station_name, "(*)") = TRUE;

--- 349 instances where "(*)" found in station name, 9 distinct
--- 21300 instances where "Temp" is found in name, 7 distinct
  --- we dont know what the * means, temp maybe should be kept bc functions the same as regular. should we change the name to regular
  --- 5 other, looked like special temp stations

SELECT
CASE
  WHEN CONTAINS_SUBSTR(start_station_name, "(Temp)") = TRUE THEN REPLACE(start_station_name, "(Temp)", "")
  WHEN CONTAINS_SUBSTR(start_station_name, "(*)") = TRUE THEN NULL
  WHEN CONTAINS_SUBSTR(start_station_name, "test") = TRUE THEN NULL
  ELSE start_station_name
  END
FROM Cyclistic_Data.start_stations;

--- rename temp stations to regular station name; takes us down to 707 unique stations
---- gonna leave (*) alone bc idk what it means - is it an error? maybe just remove the stations 
--- 3 'test' stations removed/changed to null
      --- when changed to null gets us down to 696
--- when non nulls dropped get 141,898 less rides
```


3. Data was segmented, aggregated, and further analyzed.

```sql
DROP TABLE IF EXISTS Cyclistic_Data.start_stations;

CREATE TABLE Cyclistic_Data.start_stations AS (
SELECT
  CASE
  WHEN CONTAINS_SUBSTR(start_station_name, "(Temp)") = TRUE THEN REPLACE(start_station_name, "(Temp)", "")
  WHEN CONTAINS_SUBSTR(start_station_name, "(*)") = TRUE THEN NULL
  WHEN CONTAINS_SUBSTR(start_station_name, "test") = TRUE THEN NULL
  ELSE start_station_name
  END AS start_station_name,
start_station_id,
start_lat,
start_lng
FROM Cyclistic_Data.combined_trips
WHERE
start_station_name IS NOT NULL
);


DROP TABLE IF EXISTS Cyclistic_Data.end_stations;

CREATE TABLE Cyclistic_Data.end_stations AS (
SELECT
CASE
  WHEN CONTAINS_SUBSTR(end_station_name, "(Temp)") = TRUE THEN REPLACE(end_station_name, "(Temp)", "")
  WHEN CONTAINS_SUBSTR(end_station_name, "(*)") = TRUE THEN NULL
  WHEN CONTAINS_SUBSTR(end_station_name, "test") = TRUE THEN NULL
  ELSE end_station_name
  END AS end_station_name,
end_station_id,
end_lat,
end_lng
FROM Cyclistic_Data.combined_trips
WHERE
end_station_name IS NOT NULL
);

---- before eliminating those stations 711 stations
--- after, 700

SELECT DISTINCT
  start_station_name,
  CONCAT(AVG(start_lat),  " ,", AVG(start_lng)) AS avg_start_coords
FROM Cyclistic_Data.start_stations
WHERE
  start_station_name IS NOT NULL
GROUP BY start_station_name
ORDER BY start_station_name;
--- 551506 start coords for ~ 700 start stations
--- avg start coords to fix

SELECT DISTINCT
CASE
  WHEN start_station_name = end_station_name THEN start_station_name
  END AS station_name,
  COUNT(start_station_name) AS rides_from_station,
  COUNT(end_station_name) AS rides_to_station
FROM Cyclistic_Data.combined_trips
GROUP BY station_name
ORDER BY station_name ASC

SELECT
  start_station_name,
  CONCAT(AVG(start_lat),  ", ", AVG(start_lng)) AS avg_station_coords,
  COUNT(ride_id) AS no_rides
FROM Cyclistic_Data.combined_trips
WHERE
  start_station_name IS NOT NULL
GROUP BY start_station_name
ORDER BY no_rides DESC;

SELECT
  end_station_name,
  CONCAT(AVG(end_lat),  ", ", AVG(end_lng)) AS avg_station_coords,
  COUNT(ride_id) AS no_rides
FROM Cyclistic_Data.combined_trips
WHERE
  end_station_name IS NOT NULL
GROUP BY end_station_name
ORDER BY no_rides DESC;

SELECT
  EXTRACT(MONTH from started_at) month,
  (AVG(TIMESTAMP_DIFF(ended_at, started_at, MINUTE))) AS avg_duration_min,
  COUNT(ride_id) AS no_rides
FROM
  Cyclistic_Data.combined_trips
GROUP BY month
ORDER BY avg_duration_min DESC;
--- ride length and count by month

SELECT
  EXTRACT(DAY from started_at) AS day,
  (AVG(TIMESTAMP_DIFF(ended_at, started_at, MINUTE))) AS avg_duration_min,
  COUNT(ride_id) AS no_rides
FROM
  Cyclistic_Data.combined_trips
GROUP BY day
ORDER BY avg_duration_min DESC;
--- ride length and count by day of month

SELECT
  day_of_week,
  (AVG(TIMESTAMP_DIFF(ended_at, started_at, MINUTE))) AS avg_duration_min,
  COUNT(ride_id) AS no_rides
FROM
  Cyclistic_Data.combined_trips
GROUP BY day_of_week
ORDER BY no_rides DESC, avg_duration_min DESC;
--- ride length and count by day of month

SELECT
  EXTRACT(HOUR from started_at) AS hr_of_day,
  (AVG(TIMESTAMP_DIFF(ended_at, started_at, MINUTE))) AS avg_duration_min,
  COUNT(ride_id) AS no_rides
FROM
  Cyclistic_Data.combined_trips
GROUP BY hr_of_day
ORDER BY no_rides, avg_duration_min DESC;
--- ride length and count by hr of day


```

4. Data was visualized in Tableau
