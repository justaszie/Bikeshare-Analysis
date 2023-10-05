# Bikeshare rides analysis
# My SQL CODE
```SQL
-- 5 - All summary questions for ride length
WITH rides_processed AS (
SELECT
DATETIME_DIFF(ended_at, started_at, second) as ride_length,
r.*
FROM `phrasal-brand-398306.bikeshare_data.rides` r
)


SELECT
COUNT(*) as total_rides,
MIN(ride_length) as min_ride_length,
MAX(ride_length) as max_ride_length,
ROUND(AVG(ride_length),0) as mean_ride_length,
(SELECT PERCENTILE_CONT(ride_length, 0.25) OVER() FROM rides_processed LIMIT 1) as perc_25_ride_lengh,
(SELECT PERCENTILE_CONT(ride_length, 0.5) OVER() FROM rides_processed LIMIT 1) as median_ride_length,
(SELECT PERCENTILE_CONT(ride_length, 0.75) OVER() FROM rides_processed LIMIT 1) as perc_75_ride_lenght,
STDDEV(ride_length) as std_dev_ride_length,
FROM rides_processed;
```
