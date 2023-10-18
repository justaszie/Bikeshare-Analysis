# Bikeshare rides analysis
## 1. Project Summary
### 1.1 The What and the Why
After I completed the Google Data Analytics Certification program on [Coursera](https://www.coursera.org/professional-certificates/google-data-analytics), I was keen to use its material on practical examples. 

The program includes a guided project in which you can solve a business problem for a fictional bike-share service, called Cyclastic, by analyzing a real-life dataset. The dataset contains the ride history data and the **specific business problem** of the analysis was to:
1. find out the differences between the activity of the casual riders and members (riders holding annual membership passes)
2. based on these insights, make recommendations on how the marketing team can convert casual riders into members

Even though the project was an optional part of the certification program, I chose to complete it because I was interested in using real-life dataset to solve a realistic business problem and it allowed me to brush up on my key data analysis skills.

### 1.2. Guide to This Document
As it's my first data project, this documentation is very detailed and quite chunky. I plan to use it as a knowledge base for future projects. Feel free to skip to the parts that are most interesting to you. 

1. If you're here for a good time, not a long time (and want to go straight to the solution): [presentation with the solution to the business problem](https://github.com/justaszie/Bikeshare-Analysis/tree/main#2-business-case-solution).
2. If you're interested in what's under the hood:
- [Analysis steps](https://github.com/justaszie/Bikeshare-Analysis/tree/main#4-analysis)
- [Data preparation and cleanup steps](https://github.com/justaszie/Bikeshare-Analysis/tree/main#3-preparing-the-data)
- [SQL queries used for the analysis](https://github.com/justaszie/Bikeshare-Analysis/tree/main#5-appendix-a---sql-queries-for-analysis)
3. Finally, I listed some [ideas for improving the dataset](https://github.com/justaszie/Bikeshare-Analysis/#5-ideas-for-future-improvement) and my [lessons learned](https://github.com/justaszie/Bikeshare-Analysis/blob/main/README.md#6-lessons-learned) from this project.

### 1.3. Dataset
The idea is to solve a business problem of a fictional company. But the dataset to analyze comes from a real-life business. It is the history of rides of a bikeshare service called [Divvy](https://divvybikes.com/), operated by Lyft in Chicago, and its data is licensed out for public usage. It contains a large amount of data on rides from 2015 up to mid-2023.
- [Data license](https://divvybikes.com/data-license-agreement)
- [Original data](https://divvy-tripdata.s3.amazonaws.com/index.html) (AWS S3 bucket)
- **TODO: Update link to clean dataset** [Data after my cleanup]() (note that it's a 1GB+ .csv file). See [clean dataset format](https://github.com/justaszie/Bikeshare-Analysis/tree/main#34-final-dataset).

### 1.4. Process
I followed the analysis process provided by the Google program and adjusted it to my preferences. The overall process I followed had 4 phases:
1. **Define** - read through guidance material and refined the specific business problem for the project based on the guidance document
2. **Prepare** - collected, evaluated, and cleaned the data using SQL to get it ready for analysis
4. **Analyze** - broken the business problem down into detailed questions to be answered with data, ran SQL queries to answer them, validated my hypotheses, and gained surprising insights.
5. **Share** - created a business case presentation to share the insights with stakeholders, including using visualizations. The stakeholders in this case were the marketing director and the executive team of the fictional company

### 1.5 Data Tools
The project material did not impose any tools for the analysis, so I chose the combination of:
- **SQL**
    - The data timeframe for analysis was not defined by the project. I wanted to analyze at least 1 year of rides data (5M+ rows) and it was too large for spreadsheets
    - It allowed me to brush up on my SQL skills for data cleanup and analysis
    - To focus on the analysis itself, I used BigQuery - Google's serverless data warehouse solution which was the simplest way to set up a database for storage and analysis.
- **Google Sheets**
   - Used for some calculations which are easier to do in spreadsheets than in SQL and to build visualizations
   - I chose Sheets over Excel as it's free and, from my past experience, it is often used by startups and scaleups. The skills built on Sheets will be easily transferrable to Excel.

## 2. Business Case Solution
The final presentation that solves the business case can be accessed [here](https://docs.google.com/presentation/d/1bwYGy23ZWJG5qMf0imZLOfAincOCCUbpCrCYMcIStbE/edit?usp=sharing).

As a reminder, the business problem was to convert the casual riders to annual members of the bikeshare service. Specifically, I had to find the differences in the usage of casual riders and annual members of the service and use this knowledge to make marketing recommendations to help the conversion.

## 3. Preparing the data
I chose to work with the data on rides from a full year of 2022.
 - It's post Covid-19 restrictions, so should represent somewhat normal activity
 - A full year of data allows us to explore seasonality

 The first thing to do was to download the data from the original data source, load it into an SQL database, and perform the cleaning, necessary for my analysis. 

<details>
<summary> Click here for the detailed data preparation and cleanup steps</summary>

### 3.1. Loading Into Database
The rides data is stored in a series of .csv files. In some cases, the .csv holds a month of data, in others - a whole quarter of data. For 2022 data, I downloaded the 12 .csv files containing the data for each month of 2022. I then uploaded the 12 files to a Google bucket and created a SQL table `rides` in BigQuery by merging all 12 files together (the structure of the 12 .csv files was the same).

### 3.2. Data Dictionary
The dataset does not have any description or data dictionary. So, first, I inspected the data and built a data dictionary to help with cleanup and analysis steps later. For the "category" columns, I ran a `SELECT DISTINCT` query to find possible values.

Note the format of columns (e.g. Float) is as per BigQuery schema definition.
| Column | Description (assumed) | Format |
|---|---|---|
| ride_id | Primary key | String |
| rideable_type | Describes the type of the bike used in the ride | String <br/> Possible values: electric_bike classic_bike docked_bike |
| started_at | Date and time when the ride started | Date / time |
| ended_at | Date and time when the ride ended | Date / time |
| start_station_id | ID of the station where the ride started (bike was picked up). | String |
| start_station_name | The name of the station where the ride started | String |
| end_station_id | ID of the station where the ride ended (bike was left). | String |
| end_station_name | The name of the station where the ride ended | String |
| start_lat	 | Latitude coordinate where the ride started | Float |
| start_lng	 | Longitude coordinate where the ride started | Float |
| end_lat | Latitude coordinate where the ride ended | Float |
| end_lng | Longitude coordinate where the ride ended | Float |
| member_casual | Describes the type of the rider - either casual rider or holding annual membership | String <br /> Possible values:<br/> casual <br/> member |

It is not clear if the original source of these files was a relational DB or a Data Warehouse of some type. The schema contains multiple attributes describing the station. These attributes should not be present in the rides data in a normalized relational DB structure. It creates duplication and can lead to inconsistencies. 

### 3.3. Data Cleanup

#### 3.3.1. Process
These are the steps I took to clean the data inside the SQL table. 
1. Created a backup copy of the table in case something goes wrong.
2. Performed a set of checks to find any issues:
    1. Checking for duplicate entries
    1. Checking for null values and % of rows with null values
    1. Validating that the format of the columns
    1. Checking for incorrect values (not as expected)
3. Fixed the issues found during the above checks, where possible
4. Renamed columns for clarity
5. Wrote query to add calculated columns to facilitate the analysis later and performed the checks and fixes (Steps under (2)) on them

To ensure completeness and make data cleaning easier in the future, I created a checklist framework. I created a spreadsheet with column names in columns and the types of checks (steps 2-6) in rows. I went through each type of check and ran the check on each column, where it was relevant.

<img width="1713" alt="GDAC_Cleanup_checklist" src="https://github.com/justaszie/Bikeshare-Analysis/assets/1820805/5e6abad3-385e-4482-ab0a-9e7b8de05933">

Note that in the case of a dataset with a large number of columns, we would need to pre-select the relevant columns before the cleanup.

**---------------**
#### 3.3.2. Findings and Changes Made

**Duplicates**

I ran a query to check the number of rides per ride_id and there were no IDs having more than 1 ride. 
```sql
SELECT
ride_id, COUNT(*) as nb_rides
FROM
`phrasal-brand-398306.bikeshare_data.rides`
GROUP BY ride_id
HAVING nb_rides <> 1
```

**Null values**

I used the following steps to inspect the null values in relevant columns:
1. Does the column have any null values?
2. What % of rows have null values for that column
3. Why the null values are there - is there an explanation or pattern
4. Decide what to do with null values
    - If most values of a column are null, consider deleting the column
    - If some values are null, fill it in, if possible. If not possible, exclude the null values unless they impact the core of the analysis.

Outcomes:
- Start and end station names and IDs have ~15% of null values. Start stations are only empty in the case of electric bikes. This is most likely because there were dockless e-bikes introduced [starting from 2020](https://divvybikes.com/explore-chicago/expansion-temp) and these bikes can be picked up and left anywhere. The end station names may be null due to bikes that were not properly returned. We could look into coordinates to fill in these values but these cases represent a rather small number of the rides (15%) and it's not the core of our analysis. I left it as-is.
- End station coordinates (lat and long) have a small amount of null values. This may mean that the bikes were never returned. It's a small amount so I left it as-is.
- No other columns had issues with null values

Counting null values (repeat the query for each relevant column)
```sql
SELECT
ROUND(SUM(CASE WHEN start_station_name IS NULL THEN 1 END)
/
(SELECT COUNT(*) from `phrasal-brand-398306.bikeshare_data.rides`) * 100, 2)
 AS prc_null_values
FROM `phrasal-brand-398306.bikeshare_data.rides`
```

**Format** 

After checking a few rows of the table, the formats seemed correct. There are some inconsistencies in the station_id column but we're not joining the data with the stations' data so it should not impact the analysis.
```sql
SELECT * 
FROM `phrasal-brand-398306.bikeshare_data.rides`
ORDER BY started_at
LIMIT 5
```

| ride_id | rideable_type | started_at | ended_at | start_station_name | start_station_id | end_station_name | end_station_id | start_lat | start_lng | end_lat | end_lng | member_casual |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 98D355D9A9852BE9 | classic_bike | 2022-01-01 00:00:05.000000 UTC | 2022-01-01 00:01:48.000000 UTC | Michigan Ave & 8th St | 623 | Michigan Ave & 8th St | 623 | 41.872773 | -87.623981 | 41.872773 | -87.623981 | casual |
| 04706CA7F5BD25EE | electric_bike | 2022-01-01 00:01:00.000000 UTC | 2022-01-01 00:04:39.000000 UTC | Broadway & Waveland Ave | 13325 | Broadway & Barry Ave | 13137 | 41.94907317 | -87.64863283 | 41.93758232 | -87.64409781 | casual |
| 42178E850B92597A | electric_bike | 2022-01-01 00:01:16.000000 UTC | 2022-01-01 00:32:14.000000 UTC | Clark St & Ida B Wells Dr | TA1305000009 | Clark St & Ida B Wells Dr | TA1305000009 | 41.87591917 | -87.63119383 | 41.87593267 | -87.63058454 | casual |
| 6B93C46E8F5B114C | classic_bike | 2022-01-01 00:02:14.000000 UTC | 2022-01-01 00:31:07.000000 UTC | Michigan Ave & 8th St | 623 | Michigan Ave & 8th St | 623 | 41.872773 | -87.623981 | 41.872773 | -87.623981 | casual |
| 466943353EAC8022 | classic_bike | 2022-01-01 00:02:35.000000 UTC | 2022-01-01 00:31:04.000000 UTC | Michigan Ave & 8th St | 623 | Michigan Ave & 8th St | 623 | 41.872773 | -87.623981 | 41.872773 | -87.623981 | casual |

**Incorrect Values**

There are various checks that may be useful depending on the type of data in the column. For example, checking if the dates are in the expected range. Below I will detail the value checks I did.

Rideable type  
```sql
SELECT rideable_type, ROUND(COUNT(*)  / (SELECT COUNT(*) from `phrasal-brand-398306.bikeshare_data.rides`) * 100, 2) as prc_rides
FROM `phrasal-brand-398306.bikeshare_data.rides`
GROUP BY rideable_type
```
While classic and electric bike types are self-explanatory, it's not clear what docked bike type means, since there is no dockless bike type in the dataset. There is no definition of the values provided by the data owner. Looking at samples of older data, it seems that pre-2020 there were only docked bikes and around November 2020 it was split into "docked" and "classic" types. Since only 3% of rides were made using the docked bike type, we will assume that docked and classic bikes represent the same type of bike in this analysis. We are making changes to the table accordingly:
```sql
UPDATE `phrasal-brand-398306.bikeshare_data.rides`
SET rideable_type = 'classic_bike'
WHERE rideable_type = 'docked_bike'
```

Start, End Station Names and IDs

:warning: The dataset contains 1313 distinct `start_station_id` and 1645 `start_station_name` values. It's already an issue but it's even more concerning given that the data owner's website mentions only about 800 stations program. Although the stations are not key elements of this analysis, I decided to look into these values as it allowed me to practice data cleanup on string values. I did the following checks to detect anomalies (**Note that the same checks were done to end stations**):
- Checking distribution of start_station_id lengths and looking at IDs of various lengths to spot patterns.
```sql
SELECT LENGTH(start_station_id) AS str_length, COUNT(DISTINCT start_station_id) AS num_stations
FROM `phrasal-brand-398306.bikeshare_data.rides`
GROUP BY str_length
ORDER BY num_stations DESC
```
- Looking at cases where 1 ID is associated with multiple names (and the number of rides associated with each station name)
```sql
SELECT * FROM
(
    SELECT distinct start_station_id, start_station_name,
    COUNT(distinct start_station_name) OVER (PARTITION BY start_station_id) as num_station_names_per_ID,
    COUNT(distinct ride_id) OVER (PARTITION BY start_station_name) as num_rides_per_name
    FROM `phrasal-brand-398306.bikeshare_data.rides`
)
WHERE num_station_names_per_ID > 1
ORDER BY num_station_names_per_ID DESC, start_station_id ASC, num_rides_per_name DESC
```
- Looking at cases where 1 name is associated with multiple IDs (and the number of rides associated with each ID)
```sql
SELECT * FROM
(
    SELECT distinct start_station_id, start_station_name,
    COUNT(distinct start_station_id) OVER (PARTITION BY start_station_name) as num_station_IDs_per_name,
    COUNT(distinct ride_id) OVER (PARTITION BY start_station_id) as num_rides_per_id
    FROM `phrasal-brand-398306.bikeshare_data.rides`
)
WHERE num_station_IDs_per_name > 1
ORDER BY start_station_name ASC, num_rides_per_id DESC
```
- Looking at stations where rides with 0 or negative length have started
```sql
SELECT DISTINCT start_station_id, start_station_name
FROM `phrasal-brand-398306.bikeshare_data.rides`
WHERE ride_length <= 0 -- we added ride_length as a calculated field as part of the cleanup process
```
- Finding stations that have white space characters at the end
```sql
SELECT DISTINCT start_station_id, start_station_name
FROM `phrasal-brand-398306.bikeshare_data.rides_original`
WHERE REGEXP_CONTAINS(start_station_name, r' +$')
```

Fixing Station Names and ID values (note that these fixes were done on both start and end stations). 

First, I found that the dataset contained rides to/from "Test" stations. These could not be found in the data owner's stations map. So I removed such rides from the dataset. 
```sql
DELETE FROM `phrasal-brand-398306.bikeshare_data.rides`
WHERE start_station_id IN (
    'DIVVY 001 - Warehouse test station',
    'DIVVY 001',
    'DIVVY CASSETTE REPAIR MOBILE STATION',
    'Hastings WH 2',
    'Hubbard Bike-checking (LBS-WH-TEST)',
    'Pawel Bialowas - Test- PBSC charging station'
)
```

By looking at station IDs of various lengths, I found some stations had different IDs representing them, with some IDs having "Charging station" ID format:
```sql
SELECT distinct r1.start_station_id, r1.start_station_name, r2.start_station_id, r2.start_station_name
FROM `phrasal-brand-398306.bikeshare_data.rides` r1
JOIN `phrasal-brand-398306.bikeshare_data.rides` r2 ON REPLACE(REPLACE(r1.start_station_name, '*', ''), ' - Charging', '') = r2.start_station_name
WHERE r1.start_station_id like '%chargingst%' OR r1.start_station_id like '% - Charging%'
```
<img width="800" alt="Charging format stations" src="https://github.com/justaszie/Bikeshare-Analysis/assets/1820805/b96f5e21-5494-41c4-836a-2752f420f9c3">

I have merged them with the stations that have the "clean" version of the name.
```sql
UPDATE `phrasal-brand-398306.bikeshare_data.rides` as r1
SET r1.start_station_id = r2.start_station_id,
r1.start_station_name = r2.start_station_name
FROM (
SELECT DISTINCT start_station_id, start_station_name
FROM `phrasal-brand-398306.bikeshare_data.rides`
) r2
WHERE (r1.start_station_id LIKE '%chargingst%' OR r1.start_station_id LIKE '% - Charging%')
AND REPLACE(REPLACE(r1.start_station_name, '*', ''), ' - Charging', '') = r2.start_station_name
```

While looking at stations with multiple names for the same ID, I found a list of mistakes and temporary values added to the station name. This caused to have duplicated station values. Sample of values below ("Pubic rack", really? :sob: )

<img width="636" alt="start station value errors" src="https://github.com/justaszie/Bikeshare-Analysis/assets/1820805/eb2e48af-5ff0-401e-b05e-ee62fe91db1f">

I checked the correct station names on the data owner's website and made the updates accordingly. 

A few updates were made using `WHERE` logic
```sql
-- Replacing the station name including "vaccination site" suffix with the "clean" station name
UPDATE `phrasal-brand-398306.bikeshare_data.rides` as r1
SET
r1.start_station_name = r2.start_station_name
FROM (
SELECT DISTINCT start_station_id, start_station_name
FROM `phrasal-brand-398306.bikeshare_data.rides`
) r2
WHERE LOWER(r1.start_station_name) like '%vaccination%'
AND r1.start_station_id = r2.start_station_id AND LOWER(r2.start_station_name) NOT LIKE '%vaccination%'
```
```sql
-- Removing trailing white spaces
UPDATE `phrasal-brand-398306.bikeshare_data.rides`
SET start_station_name = TRIM(start_station_name)
WHERE REGEXP_CONTAINS(start_station_name, ' +$')
```
```sql
-- Remove ' (Temp)' from station names
UPDATE `phrasal-brand-398306.bikeshare_data.rides`
SET start_station_name = REPLACE(start_station_name, ' (Temp)', '')
WHERE start_station_name like '% (Temp)%';
```
Others had to be updated case by case (see example below).
```sql
UPDATE `phrasal-brand-398306.bikeshare_data.rides`
SET start_station_name = 'Whipple St & 26th St' -- correct value based on divvy website
WHERE start_station_id = '540'
AND start_station_name = 'Whippie St & 26th St';
```

Note that some stations have one ID and multiple names, where the name has various suffixes, such as " - midblock", "Public Rack - ", " SW", etc - see the sample below. This explains most of the cases where 1 ID has multiple name values (300+ cases).  I checked the [stations map](https://account.divvybikes.com/map) on the data owner's website and they are actually considered as different stations. So we are leaving them as different stations in the preparation step. I group them in my queries in the Analysis phase, when I'm looking at the geographical distribution of the rides. 

<img width="756" alt="station name suffixes" src="https://github.com/justaszie/Bikeshare-Analysis/assets/1820805/137f1fd9-e9d1-4199-ba32-8632e9e54d4b">

Finally, in some cases, there was nothing to be done.
- Cases, where 1 ID value has multiple names associated, with no logical association between them. This accounts for about 80 cases. On the stations map, both stations exist and are far away - see the sample below. Since no data reference is provided by the data owner, we can't make a call on which values are correct and we leave it as is. 

<img width="651" alt="Sample same ID multiple names" src="https://github.com/justaszie/Bikeshare-Analysis/assets/1820805/e4de166a-2bd7-4a73-aa34-dcf37a6add3f">

<img width="1783" alt="Screenshot 2023-10-16 at 11 35 21" src="https://github.com/justaszie/Bikeshare-Analysis/assets/1820805/c364196a-4400-4ba4-b7c9-eab3f0f68d56">

- Some station names have multiple IDs (17 such stations, representing 36k rides - i.e. only ~0.6% of the dataset). Again, with no referential information, we can't make a call here. In the analysis, I focus on the station names. There's a risk that these stations are actually different stations but it involves only a small amount of rides so it won't impact the analysis a lot.

<img width="660" alt="Sample same name multiple IDs" src="https://github.com/justaszie/Bikeshare-Analysis/assets/1820805/8dbfcedd-94ee-42ae-a7a5-4ffb6ad7b726">

Ride Started and Ended dates
- I checked that both dates are within the expected range. I selected the .csv files with 2022 rides. So the expected range for the start date is within 2022 while the end date may also be a little after the last day of 2022. The ride start dates are within range. There is a ride that ended on 02/01/2023 but it's the only one so we will ignore it.
```sql
SELECT
MIN(started_at) AS min_start_date,
MAX(started_at) AS max_start_date,
MIN(ended_at) AS min_end_date,
MAX(ended_at) AS max_end_date,
FROM
`phrasal-brand-398306.bikeshare_data.rides`
```
- I checked if there were cases where the ride end date and time equals start date/time or is before the start date/time. <br/>  :warning: It turns out there are 530 such cases. While it's a tiny percentage of the overall dataset (5M+ rows), it should not impact our analysis. But it's definitely alarming :triangular_flag_on_post: in terms of overall data quality. 
```sql
SELECT started_at, ended_at, * 
FROM `phrasal-brand-398306.bikeshare_data.rides`
WHERE ended_at <= started_at
```

Other incorrect value checks:
- The "category" fields (rideable_type and member_casual) do not contain any whitespaces, typos, or any problems. I checked it by looking at the values manually since they are few.
- No validation was done on the coordinates as it is not important to the analysis


**Adding calculated columns and renaming columns**

I renamed the `member_casual` column to `rider_type` to make it clearer.

I ran queries to add and populate 4 calculated columns which would facilitate the analysis later. The ride length allows us to explore the usage patterns of casual riders and members and the 3 other columns allow us to explore the data in time series.
| Column | Description (assumed) | Format |
|---|---|---|
| ride_length | calculated ride length in seconds | Integer |
| start_day_of_week | day of week, when the ride was started. Note that I used the `FORMAT_DATE` SQL function instead of the usual `EXTRACT` because it returns '1' for Mondays, which is what I want. The `FORMAT_DATE` function returns strings so I transformed it to an Integer for easier analysis. | Integer |
| start_hour | the hour of the day when the ride started. | Integer |
| ride_month | the month when the ride started. | Integer |

```sql
ALTER TABLE `phrasal-brand-398306.bikeshare_data.rides`
ADD COLUMN ride_length INTEGER,
ADD COLUMN start_day_of_week INTEGER,
ADD COLUMN start_hour INTEGER,
ADD COLUMN ride_month INT;


UPDATE `phrasal-brand-398306.bikeshare_data.rides`
SET
ride_length = DATETIME_DIFF(ended_at, started_at, second),
start_day_of_week = CAST(FORMAT_DATE('%u', started_at) AS INT),
start_hour = EXTRACT(HOUR from started_at),
ride_month = EXTRACT(MONTH from started_at)
WHERE TRUE;
```

After creating calculated columns, it makes sense to do cleanup checks on their values as well. The start_day_of_week, start_hour, and ride_month all have correct values and no null values. 

:warning: There are red flags with ride_length, though. 

First, I checked the minimum and maximum length values. 

```sql
SELECT 
min(ride_length) as min_length,
max(ride_length) as max_length 
FROM `phrasal-brand-398306.bikeshare_data.rides`;
```
This confirmed that there are very strange values. The minimum is -621,201s (more than -7 days) and the maximum is 2,483,235s (over 28 days). That is a second red flag :triangular_flag_on_post: in the dataset. The extremely long rides are probably rides in which the bike was not returned or the return was not registered properly but there is no way to confirm without any dataset description available. 

Next, I checked what percentage of rows fall in the area of suspicious to see if the analysis will be at all possible. I divided the length into a number of bins that represent rides of negative length, 0 length rides, suspiciously short, and suspiciously long rides. 

```sql
SELECT
CASE
WHEN ride_length < 0 THEN 'B1: <0 sec'
WHEN ride_length = 0 THEN 'B2: 0 sec'
WHEN ride_length > 0 AND ride_length <= 10 THEN 'B3: <=10 sec'
WHEN ride_length > 10 AND ride_length <= 60 THEN 'B4: 10 > X <= 60 sec'
WHEN ride_length > 60 AND ride_length <= 18000 THEN 'B5: 60 sec > X < 3 hr'
ELSE 'B6: > 5 hr'
END as bin,
COUNT(*) nb_rides,
(SELECT COUNT(*) FROM `phrasal-brand-398306.bikeshare_data.rides`) as total,
ROUND(COUNT(*)/(SELECT COUNT(*) from `phrasal-brand-398306.bikeshare_data.rides`)*100,2) as prc
FROM `phrasal-brand-398306.bikeshare_data.rides`
GROUP BY bin
ORDER BY bin;
```

<img width="588" alt="ride_length_suspicious_values" src="https://github.com/justaszie/Bikeshare-Analysis/assets/1820805/0f07bdbc-cc6a-47f9-a30d-bd9ceb5bacf9">

Although the presence of such values is alarming, the total number of suspicious values is <2.5% of the overall dataset. As the majority of the dataset seems correct, I decided to proceed with the analysis. 

:warning: Note that the outlier values will create a positive skewness in the ride length distribution. I will use median instead of mean in our analysis to negate this.
</details>

### 3.4. Final Dataset
The below table  describes the final dataset after the original dataset was cleaned up. This dataset was used in the Analysis phase. 

| Column | Description (assumed) | Format |
|---|---|---|
| ride_id | Primary key | String |
| rideable_type | Describes the type of the bike used in the ride | String   Possible values: electric_bike classic_bike |
| started_at | Date and time when the ride started | Date / time |
| ended_at | Date and time when the ride ended | Date / time |
| start_station_id | ID of the station where the ride started (bike was picked up). | String |
| start_station_name | The name of the station where the ride started | String |
| end_station_id | ID of the station where the ride ended (bike was left). | String |
| end_station_name | The name of the station where the ride ended | String |
| start_lat | Latitude coordinate where the ride started | Float |
| start_lng | Longitude coordinate where the ride started | Float |
| end_lat | Latitude coordinate where the ride ended | Float |
| end_lng | Longitude coordinate where the ride ended | Float |
| rider_type | Describes the type of the rider - either casual rider or holding annual membership | String   Possible values:  casual   member |
| ride_length | Length of the ride (from start date/time to end date/time) in seconds | Integer |
| start_day_of_week | Day of week when the ride started in numerical format (1 represents Monday) | Integer |
| start_hour | Hour (in 24h format) when the ride started | Integer |
| ride_month | Month when the ride started in numerical format (1 represents January) | Integer |

See a few rows from the table below:

| ride_id | rideable_type | started_at | ended_at | start_station_name | start_station_id | end_station_name | end_station_id | start_lat | start_lng | end_lat | end_lng | rider_type | ride_length | start_day_of_week | start_hour | ride_month |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 00000123F60251E6 | classic_bike | 2022-02-07 15:47:40.000000 UTC | 2022-02-07 15:49:28.000000 UTC | Wells St & Hubbard St | TA1307000151 | Kingsbury St & Kinzie St | KA1503000043 | 41.889906 | -87.634266 | 41.88917683258 | -87.6385057718 | member | 108 | 1 | 15 | 2 |
| 00000179CF2C4FB5 | electric_bike | 2022-07-28 09:02:27.000000 UTC | 2022-07-28 09:13:51.000000 UTC | Ellis Ave & 55th St | KA1504000076 | Ellis Ave & 55th St | KA1504000076 | 41.794237333333335 | -87.601342833333334 | 41.79430062054 | -87.6014497734 | casual | 684 | 4 | 9 | 7 |
| 0000038F578A7278 | electric_bike | 2022-12-15 15:36:06.000000 UTC | 2022-12-15 15:44:35.000000 UTC | Lakeview Ave & Fullerton Pkwy | TA1309000019 | Clark St & Elm St | TA1307000039 | 41.925825953 | -87.638823032 | 41.902973 | -87.63128 | casual | 509 | 4 | 15 | 12 |
| 0000047373295F85 | electric_bike | 2022-07-22 16:56:46.000000 UTC | 2022-07-22 17:21:58.000000 UTC | Streeter Dr & Grand Ave | 13022 | Calumet Ave & 18th St | 13102 | 41.892208833333335 | -87.611831166666661 | 41.857611 | -87.619407 | casual | 1512 | 5 | 16 | 7 |
| 000004C3185FDDE9 | electric_bike | 2022-07-17 13:01:54.000000 UTC | 2022-07-17 13:16:13.000000 UTC | Larrabee St & Webster Ave | 13193 | Honore St & Division St | TA1305000034 | 41.921853781 | -87.64402318 | 41.903119 | -87.673935 | casual | 859 | 7 | 13 | 7 |

## 4. Analysis
### 4.1. Defining the questions
As a reminder, the business task definition was broad - provide insights on how annual members and casual riders use Cyclistic bikes differently. The certification project material gave some suggestions on specific questions but they were very basic:
> looking at the total number of rows, distinct values, maximum, minimum, or mean value

So, before I started the analysis, I had to define the list of specific questions, relevant to the business task, that I would answer using the data. To solve such cases easier in the future, I created a framework that provides guidance on where to start in defining the questions. 

Note that this framework does not give an exhaustive list of relevant questions and it will need to be adjusted to larger datasets but it's a solid start.

<details>
<summary> Details of the framework </summary>
<br/>

The framework to define the specific questions consists of following a few guiding steps:
1. Classify the columns in the table: timeseries (e.g. ride month), categories (e.g. rider type, rider type), numerical (e.g. ride length), and nominal values (e.g. start station name). Note that BI tools such as Tableau automatically perform similar classification.
2. Define the list of timeseries analyses which make sense for the business task. It's a combination of the following questions: 
    - which numerical values should we analyze in the timeseries (e.g. number of rides)
    - what level timeseries (month, week, day, hour, etc.) are interesting  (e.g. number of monthly rides to explore seasonality)
    - which categories will we use to split the numerical values (e.g. the number of monthly rides by rider type to compare seasonality across different rider types).
3. List the possible combinations of (numerical value x 1 or more categories) and highlight the ones that are interesting to explore for the business task.
    - in the business task, there may be some key categories (in this case it was the rider type since we explore differences in the activity of different rider types)
    - it may be interesting to also build numerical value summary table(s) for the overall dataset as well as broken down by interesting categories. 

 Steps 2 and 3 can provide matrices that highlight the questions to be answered with data, as a starting point for the analysis. For example, for this analysis, I created 2 matrices that guided my analysis: 

**Timeseries list**

<img width="1011" alt="timeseries_questions" src="https://github.com/justaszie/Bikeshare-Analysis/assets/1820805/9794dffb-853d-4918-a2ec-56d6db8a2c24">

**Combinations of categories**

<img width="1057" alt="categories_questions" src="https://github.com/justaszie/Bikeshare-Analysis/assets/1820805/889320fa-2013-4e3c-beaa-f6c23da497c6">

This framework reminds me to consider some angles of exploration which I may not have thought about.
</details>

Using the framework mentioned above as a starting point, I decided that these queries would be a good start for our business task:
1. All summary questions<sup><b>*</b></sup> for the ride length measure, on the overall dataset and broken down by rider type
1. Total number of rides, broken down by month, and by month and rider type
1. Total number of rides, broken down by day of week
1. Total number of rides and average ride length (median), broken down by day of week and rider type
1. Total number of rides, broken down by start hour and by rider type
1. Total number of rides, broken down by rider type and by rideable type
1. Top 10 stations where most rides start for each rider type
1. Top 10 routes taken by each rider type
1. Count distinct routes taken, broken down by rider type
1. How clustered is the activity for casual riders, in terms of stations where rides start

<sup><b>*</b></sup> By summary questions, I mean finding the following values on the ride length that give a sense of the distribution (I queried them on the overall dataset and broken down by rider type): 
- Total table row count (observations)
- Max field value
- Min field value
- Average (mean)
- 50th percentile value (median)
- 25th percentile value
- 75th percentile value
- Standard deviation

### 4.2. Ride Length Summary 
Summary queries allowed me to explore the distribution of the ride lengths. Note that the ride length values are **expressed in seconds**.

[Link to queries](https://github.com/justaszie/Bikeshare-Analysis/tree/main#ride-length-summary-queries)

**Overall dataset**
<img width="1027" alt="res_summary_overall" src="https://github.com/justaszie/Bikeshare-Analysis/assets/1820805/dc6e4150-5ecd-4e92-aefb-3746cac081ae">

**Broken down by rider type**
<img width="1111" alt="res_summary_by_rider" src="https://github.com/justaszie/Bikeshare-Analysis/assets/1820805/72e73cf2-54c0-42fb-a88b-3b2871c519eb">


**:bulb: Key Insights**
- The mean is significantly higher than the median with a high standard deviation, especially for the casual riders' activity. This means that the distribution is heavily skewed due to a long tail with extremely long rides, as seen in the [Data Cleanup](https://github.com/justaszie/Bikeshare-Analysis/tree/main#3-preparing-the-data) step. We will use the median to describe the typical ride, instead of the mean.
- There are more rides taken by members but without the rider identifier attribute, we don't know if it's due to a higher number of riders or average rides per rider.
- There is a 60/40 division of data by rider type, which means the data is representative of both populations (casual riders and members).
- The casual riders are taking significantly longer rides than members (median for casual riders is 47% higher)

### 4.3. Rides by Month
Looking at the number of monthly rides allows us to see the impact of seasonality on the activity.

[Link to queries](https://github.com/justaszie/Bikeshare-Analysis/tree/main#rides-by-month-queries)

**Overall Dataset**

![viz_Monthly rides](https://github.com/justaszie/Bikeshare-Analysis/assets/1820805/d490c893-0fd5-461d-b5cd-ec2b557fd440)

**Broken down by rider type**

![viz_Monthly rides by rider type](https://github.com/justaszie/Bikeshare-Analysis/assets/1820805/03c8d47c-b31c-424b-a30e-2b22358cc8df)

**:bulb: Key Insights**
- Overall, there's a clear peak season (May - October) when 75% of the rides happen. It's predictable given the cold winters in Chicago. 
- Casual riders' activity has shorter and more intense peak. In April, casual riders reach 30% of their peak monthly rides value, while members are already at almost 60%.

### 4.4. Rides by Day of the Week
We explore the usage patterns for different rider types during the week. It can give insights into the purpose of the rides.

[Link to queries](https://github.com/justaszie/Bikeshare-Analysis/tree/main#rides-by-day-of-the-week-queries)

**Overall Dataset**

![viz_Daily rides](https://github.com/justaszie/Bikeshare-Analysis/assets/1820805/ebb7431c-b0c1-4809-a93b-53f3f06abfdc)

**Broken down by Rider Type**

![viz_Daily rides, by rider type](https://github.com/justaszie/Bikeshare-Analysis/assets/1820805/c54d1be2-67ac-4385-b50c-61643444408c)

We also look into the difference in weekend rides vs weekday rides for both types of riders to see how significant the preferences are. 

![viz_Average rides on a weekday and on a weekend day, by rider type](https://github.com/justaszie/Bikeshare-Analysis/assets/1820805/71e0a373-ee6b-40e2-8cf3-e34837efa1c2)

We also explore the differences in ride lengths on various days.

![viz_Median ride length, by rider type](https://github.com/justaszie/Bikeshare-Analysis/assets/1820805/87667293-df57-4a45-bd43-00f0bca8ae54)


**:bulb: Key Insights**
- Overall, the number of rides taken steadily increases as the week goes on, reaching its peak on Saturday.
- The members' rides seem to have a daily commute pattern versus more leisure-centered activity for casual riders. On an average weekday, members take 21% more rides than on a weekend day. Casual riders, on average, take 48% more rides on weekends. 

### 4.5. Rides by Starting Hour
Looking at the “busiest times” for each rider type allows us to find patterns and potentially validate the commute vs leisure hypothesis. 

[Link to queries](https://github.com/justaszie/Bikeshare-Analysis/tree/main#rides-by-starting-hour-queries)

**Weekday Rides**

![viz_Weekday rides - hourly breakdown](https://github.com/justaszie/Bikeshare-Analysis/assets/1820805/10c907ab-1d1b-4bb3-bc0c-5a4ec1972074)


**Weekend Rides**

![viz_Weekend rides - hourly breakdown](https://github.com/justaszie/Bikeshare-Analysis/assets/1820805/f3c1df03-db68-4c9c-b383-69a1f20c4689)

**:bulb: Key Insights**
- Activity on weekends is similar for both types of customers. But the activity is clearly different on weekdays - members have spikes in activity during "commuter" hours (7-8 am and 4-6 pm) and casual riders increase their activity steadily until 5 pm. This pattern signals daily commute habits of members.

### 4.6. Rides by Rideable Type
We are exploring if casual riders and members have any preferences in terms of bike types (classic or electric bikes).

[Link to queries](https://github.com/justaszie/Bikeshare-Analysis/tree/main#rides-by-rideable-type-queries)

<img width="491" alt="res_rideable_type" src="https://github.com/justaszie/Bikeshare-Analysis/assets/1820805/943a67d0-5271-4766-ad8f-12da62bba789">

The results are pretty equally distributed in both cases so there are no conclusions to draw here.

### 4.7. Ride Start Stations
The next set of queries allows us to see if casual riders and members start their rides in different or similar locations. Since there are over a thousand stations in the dataset, we look at the TOP 10 stations, where most of the rides start, and look for any patterns there.

[Link to queries](https://github.com/justaszie/Bikeshare-Analysis/tree/main#ride-start-stations-queries)

**Top 10 stations for Casual Riders**

![viz_Top 10 stations for casual riders](https://github.com/justaszie/Bikeshare-Analysis/assets/1820805/2a60fa2e-aa68-43e8-ac80-79d3a18313d7)

For better visibility, we mark a few of the most popular stations on the map to look for geographical patterns. As we can see, casual riders mostly start their rides in leisure sports: near beaches, piers, parks, etc.

<img width="1089" alt="res_top_stations_casual_map" src="https://github.com/justaszie/Bikeshare-Analysis/assets/1820805/7a004861-dc14-4331-8272-92dd672d7a98">

**Top 10 stations for Members**

![viz_Top 10 stations for members](https://github.com/justaszie/Bikeshare-Analysis/assets/1820805/78648eb9-be10-4a5b-871b-ab59c7483f16)

Members start their rides mostly in the middle of the city, presumably in business areas. One of the most popular stations is also near the main university in the city. 

<img width="1169" alt="res_top_stations_members_map" src="https://github.com/justaszie/Bikeshare-Analysis/assets/1820805/b24fbea8-2181-40b2-873d-553cff94edad">

**Stations clustering for Casual Riders**

While looking at the distribution of rides starting at various stations, I noticed a lot of stations had only 1 or 2 rides that started there. Inspired by the 80/20 principle, I decided to check if it applies here - i.e. if a small subset of starting stations cover most of the rides. 
If it's confirmed, it could help focus the physical marketing assets on a small number of areas. 

![viz_casual_station_cluster](https://github.com/justaszie/Bikeshare-Analysis/assets/1820805/0d509e92-68d6-4c44-b096-7a5b6170aec0)

**:bulb: Key Insights**
- Casual riders seem to use the service for leisure and members are using it for their commute. 
    - Casual riders start their rides mostly around leisure spots and scenic areas
    - Members start their rides in the middle of the city, near universities, etc.
- The activity of casual riders is clustered. 90% of the rides start in 350 stations (out of 1k+ total).

### 4.8. Routes Taken
Similar to start stations, we are checking for any patterns in what routes the casual riders and members are taking.

[Link to queries](https://github.com/justaszie/Bikeshare-Analysis/tree/main#routes-taken-queries)

**Top 10 routes for Casual Riders**

![Top 10 routes for casual riders](https://github.com/justaszie/Bikeshare-Analysis/assets/1820805/a2f7e422-12ca-49a2-89a3-e6a520d5101d)

**Top 10 routes for Members**

![viz_Top 10 routes for members](https://github.com/justaszie/Bikeshare-Analysis/assets/1820805/9d3b1f41-fbf6-4f7c-b4e7-9e2c673f22a0)

**Number of Distinct Routes**

Finally, I decided to check if there are differences in the variety of routes taken by casual riders and by members. 

<img width="251" alt="res_distinct_routes_num" src="https://github.com/justaszie/Bikeshare-Analysis/assets/1820805/9f4cda45-3465-40b2-977f-8c7bdad2521c">

There is no significant difference in the number of different routes taken by different riders.

**:bulb: Key Insights**
- There are very different patterns in routes taken by casual riders and by members. This is another argument for the theory of casual riders using the service for leisure and members using it for their commute.
    - The majority of the most popular routes for casual riders start and end in the same station. This suggests that the purpose of the ride is not transportation but the ride itself.
    - The most popular routes for members have a clear "round-trip" pattern.  

## 5. Ideas for Dataset Improvement
It's surprising to see so many data quality issues in a dataset provided by a tech-savvy company such as Lyft. 

There are multiple ways to improve the analysis by improving the data quality of the dataset.

1. Rider ID should be added. This would help with the business problem in multiple ways:
    1. Analyze the volume and frequency of rides per rider to enhance rider profile for targeting.
    1. Create and track KPIs such as the (casual riders/members) ratio. This will help measure the performance of conversion marketing campaigns.
1. Data dictionary should be provided to help understand the following:
    1. How start and end station values are populated and in what cases will they be Null?
    1. Are all electric bikes considered dockless? Why in most situations, rides using electric bikes have start and end station values, and in some cases, they don't.
    1. How is it possible to have a start date > end date? In what cases there will be very short rides and very long rides (multiple days long)
    1. How is it possible to have a large number of different coordinates associated with the same start/end station?
    1. Why does the dataset contain 1k+ station IDs, while the website mentions 800 stations?
1. Data quality should be improved:
    1. The data around station IDs and names data is not very reliable. The values should be reviewed against referential data and values should be corrected to fix the issues raised in this document.
    1. The rideable type data should be reviewed against referential and updated based on the current inventory.

## 6. Lessons Learned
Since it was my first data project, I learned a lot from it. Some of my thoughts below.

**Business side**
1. Markdown is not the easiest format to publish your projects :sweat_smile: A tool such as Jupyter or R notebook would save tons of time. 
2. In real life, I would follow up with the business stakeholders to better refine the problem. The main question I had was: should the marketing recommendations also include potential changes to the membership plan (to make it more attractive to casual riders), or is it out of the question?
3. It's very useful to build a framework for the cleanup checklist or to come up with analysis queries but we should always aim to be more creative and not limit the scope to that. As we go through the analysis, new questions come up. For example, we found that members use the service mostly for their short commutes. But there is a good number of weekend rides too. How do those rides look, what patterns do they follow?
4. The 80/20 principle is a useful angle for analysis. I had a theory that most of the rides would be taken from a small number of stations (e.g. 10-50 stations). It was not true but then I found out that less than 20% of stations covered 90% of rides.
5. Pretty obvious but having a business understanding helps a lot in the analysis. In this case, understanding what types of bikes are used, how the bikes are picked up and parked, etc., would explain a lot of unusual things in the data (such as varied coordinates for the same station name.

**Technical**
1. One of the key goals for the project was to practice SQL but Python or R would be 100x more efficient for such analysis. 
2. Cleaning string values with no referential data is always painful but a manual look-through, combined with REGEXP functions is super helpful. For example, finding leading or trailing whitespace characters, replacing strings in certain patterns, etc. 
1. I made a classic rookie mistake by assuming that `EXTRACT DAYOFWEEK` in SQL was returning 1 for Mondays when 1 actually means Sunday. When I realized it rather late, I had to redo a bunch of queries and visualizations. `FORMAT_DATE` is a better option for us, Europeans.
1. I had forgotten to check if end_date values were always after start_date values during cleanup. I only realized it when looking at the ride_lenght value distribution during analysis.
1. When structuring the output of our queries, we should think about the story that the visualization will tell. For example, we query aggregated number of rides per member type and per bike type and we want to use the column chart to show contrast. It’s a different story if the member type is in the X axis (ie. the story is casual riders use electrical bikes more often) versus the bike type in the X axis (electrical bikes are more often used by casual riders than by members).
1. When we aggregate data by multiple variables using SQL, it's useful to do it in a way where one of the aggregation variables is in columns (e.g. `total_rides_member`, `total_rides_casual`). The key dimension of analysis (e.g. rider type in this case) should be in the rows. This way we can build charts immediately on the results data - either in spreadsheets or in a BI tool such as Metabase. However, in the case of many distinct values, the column approach may be inefficient.
1. I learned some useful methods to look at distribution in SQL: percentile functions, and partition of values into bins.
1. I had a lot of opportunities to use SQL window functions which are really powerful.
1. I need to dig deeper into statistical analysis, particularly in measuring seasonality. I struggled to find the right metric to express the magnitude of seasonal peaks for different types of riders.
1. I need to do technical research on the benefits of different pre-processing approaches in SQL: adding calculated columns, using CTEs, views, or subqueries.

## Appendix A - SQL Queries for Analysis
This section lists all the SQL queries used in the analysis and the results when they were run. 

### Ride Length Summary Queries
**Overall dataset**
```sql
-- All summary questions for ride length
SELECT
COUNT(*) AS total_rides,
MIN(ride_length) AS min_ride_length,
MAX(ride_length) AS max_ride_length,
ROUND(AVG(ride_length),0) AS mean_ride_length,
(SELECT PERCENTILE_CONT(ride_length, 0.25) OVER() FROM `phrasal-brand-398306.bikeshare_data.rides` LIMIT 1) AS perc_25_ride_lengh,
(SELECT PERCENTILE_CONT(ride_length, 0.5) OVER() FROM `phrasal-brand-398306.bikeshare_data.rides` LIMIT 1) AS median_ride_length,
(SELECT PERCENTILE_CONT(ride_length, 0.75) OVER() FROM `phrasal-brand-398306.bikeshare_data.rides` LIMIT 1) AS perc_75_ride_lenght,
STDDEV(ride_length) AS std_dev_ride_length,
FROM `phrasal-brand-398306.bikeshare_data.rides`;
```

<img width="1027" alt="res_summary_overall" src="https://github.com/justaszie/Bikeshare-Analysis/assets/1820805/c67465b2-ce93-49f4-899f-07d5a7ec3c14">

**Broken down by Rider Type**
```sql
-- All summary questions for ride length by rider type
-- Step 1. Return median ride length values by rider type
WITH percentiles AS (
SELECT DISTINCT
rider_type,
PERCENTILE_CONT(ride_length, 0.25) OVER(PARTITION BY rider_type) AS perc_25_ride_lengh,
PERCENTILE_CONT(ride_length, 0.5) OVER(PARTITION BY rider_type) AS median_ride_length,
PERCENTILE_CONT(ride_length, 0.75) OVER(PARTITION BY rider_type) AS perc_75_ride_lenght
FROM
`phrasal-brand-398306.bikeshare_data.rides`
),


-- Step 2. Return summary table (except percentiles)
summary AS (
SELECT
rider_type,
COUNT(*) AS total_rides,
MIN(ride_length) AS min_ride_length,
MAX(ride_length) AS max_ride_length,
ROUND(AVG(ride_length),0) AS mean_ride_length,
STDDEV(ride_length) AS std_dev_ride_length,
FROM `phrasal-brand-398306.bikeshare_data.rides`
GROUP BY rider_type
)

-- Step 3. Join the summary table with percentile values
SELECT summary.rider_type, total_rides, min_ride_length, max_ride_length, mean_ride_length, median_ride_length, perc_25_ride_lengh, perc_75_ride_lenght, std_dev_ride_length
FROM
summary
JOIN
percentiles
ON summary.rider_type = percentiles.rider_type;
```

<img width="1111" alt="res_summary_by_rider" src="https://github.com/justaszie/Bikeshare-Analysis/assets/1820805/c6e259f1-ccee-4883-9ea3-0946e6e4e25d">

### Rides by Month Queries
**Overall Dataset**

```sql
-- Total rides by month
SELECT
ride_month,
COUNT(*) AS total_rides
FROM `phrasal-brand-398306.bikeshare_data.rides`
GROUP BY ride_month
ORDER BY ride_month;
```

<img width="251" alt="res_monthly_overall" src="https://github.com/justaszie/Bikeshare-Analysis/assets/1820805/ac8e1ecf-86e9-49a5-bee4-735fd716d58e">

**Broken down by Rider Type**

Note that in this query I chose to have rider type breakdown in separate columns. A simpler way would be to add just the `rider_type` column in the query and have separate rows for each rider type. However the advantage of splitting it into separate columns allows us to create charts on that data without additional processing. And in this case, the rider type only has 2 values so it makes sense. 

```sql
-- Total rides by month and rider type
SELECT
ride_month,
COUNT(CASE rider_type WHEN 'casual' THEN 1 END) as rides_casual,
COUNT(CASE rider_type WHEN 'member' THEN 1 END) as rides_member,
COUNT(*) AS total_rides
FROM `phrasal-brand-398306.bikeshare_data.rides`
GROUP BY ride_month
ORDER BY ride_month;
```

<img width="468" alt="res_monthly_by_rider" src="https://github.com/justaszie/Bikeshare-Analysis/assets/1820805/6fea7e78-a73f-40e6-b205-d05094efe341">

### Rides by Day of the Week Queries
**Overall Dataset**
```sql
-- Total rides by day of the week
SELECT
start_day_of_week,
COUNT(*) AS total_rides
FROM `phrasal-brand-398306.bikeshare_data.rides`
GROUP BY start_day_of_week
ORDER BY start_day_of_week;
```

<img width="253" alt="res_day_of_week_total" src="https://github.com/justaszie/Bikeshare-Analysis/assets/1820805/c3fc5634-35c9-4ed3-b402-d9426a95be3d">

**Broken down by Rider Type**

```sql
-- Total rides and median by day of week and rider type
-- Step 1. Return median ride length values by rider type and day of the week.
WITH medians AS (
SELECT DISTINCT
start_day_of_week,
rider_type,
PERCENTILE_CONT(ride_length, 0.5) OVER(PARTITION BY rider_type, FORMAT_DATE('%u', started_at)) AS median_ride_length
FROM
`phrasal-brand-398306.bikeshare_data.rides`
),

-- Step 2. Return total ride count by rider type and day of the week
totals AS (
SELECT
start_day_of_week,
rider_type,
COUNT(*) AS total_rides
FROM
`phrasal-brand-398306.bikeshare_data.rides`
GROUP BY start_day_of_week, rider_type
)
-- Step 3. Create the main query that pulls relevant values from the 2 separate queries. Since there are only 2 rider types, we define a column for each measure for each rider type (4 columns total).
SELECT DISTINCT t.start_day_of_week,
(SELECT total_rides FROM totals WHERE start_day_of_week = t.start_day_of_week AND rider_type = 'casual') AS total_rides_casual,
(SELECT total_rides FROM totals WHERE start_day_of_week = t.start_day_of_week AND rider_type = 'member') AS total_rides_member,
(SELECT median_ride_length FROM medians WHERE start_day_of_week = t.start_day_of_week AND rider_type = 'casual') AS median_ride_length_casual,
(SELECT median_ride_length FROM medians WHERE start_day_of_week = t.start_day_of_week AND rider_type = 'member') AS median_ride_length_member
FROM
totals t
ORDER BY t.start_day_of_week;
```

<img width="623" alt="res_day_of_week_rider" src="https://github.com/justaszie/Bikeshare-Analysis/assets/1820805/8221c1b2-a0c5-4d72-8960-1fd833dff961">

### Rides by Starting Hour Queries
```sql
SELECT start_hour,
count(*) AS total_rides,
SUM(CASE rider_type WHEN 'casual' THEN 1 END) AS casual_rides,
SUM(CASE rider_type WHEN 'member' THEN 1 END) AS member_rides,
FROM `phrasal-brand-398306.bikeshare_data.rides`
WHERE start_day_of_week NOT IN (6,7) -- Change this condition to IN (6,7) for weekend rides
GROUP BY start_hour
ORDER BY start_hour
```
**Weekday Rides**

<img width="370" alt="res_hourly_rider_weekday" src="https://github.com/justaszie/Bikeshare-Analysis/assets/1820805/308d019d-c808-4e2e-a282-a45d476ebf10">

**Weekend Rides**

<img width="365" alt="res_hourly_rider_weekend" src="https://github.com/justaszie/Bikeshare-Analysis/assets/1820805/4f4ea371-ecc1-44eb-a6a3-409914f1df75">

### Rides by Rideable Type Queries
```sql
-- Total rides by the rider type and rideable type
SELECT
rider_type,
COUNT(CASE rideable_type WHEN 'electric_bike' THEN 1 END) as rides_electric,
COUNT(CASE rideable_type WHEN 'classic_bike' THEN 1 END) as rides_classic,
COUNT(*) AS total_rides
FROM `phrasal-brand-398306.bikeshare_data.rides`
GROUP BY rider_type;
```

<img width="491" alt="res_rideable_type" src="https://github.com/justaszie/Bikeshare-Analysis/assets/1820805/780bb31e-3d70-44b3-b679-e14476cac7df">

### Ride Start Stations Queries

```sql
-- Group the stations that have the same name, except for suffix - these stations are pretty much in the same location
WITH processed_rides AS (
SELECT *,
REGEXP_REPLACE(
REPLACE(REPLACE(REPLACE(REPLACE(REPLACE(REPLACE(start_station_name, 'Public Rack - ', ''),' - midblock', ''), ' - SW', ''), ' - NW', ''), ' - SE', ''), ' - NE', '')
, r'( -)? (?:(?:N|E|W|S)|(?:North|West|East|South)|(?:(?:north|south|east|west) corner)|(?:NW|NE|SW|SE))$', '')
AS grouped_start_station_name
FROM `phrasal-brand-398306.bikeshare_data.rides`
)
, counts AS(
    SELECT
    rider_type,
    grouped_start_station_name,
    COUNT(*) AS total_rides,
    RANK() OVER (PARTITION BY rider_type ORDER BY COUNT(*) DESC) AS station_rank
    FROM processed_rides
    -- Ignoring stations with null values
    WHERE grouped_start_station_name IS NOT NULL
    GROUP BY rider_type, grouped_start_station_name
)

SELECT *
FROM counts
WHERE station_rank <= 10;
```

**Top 10 stations for Casual Riders**

<img width="653" alt="res_top10_stations_casual" src="https://github.com/justaszie/Bikeshare-Analysis/assets/1820805/c0134603-ef85-49d5-91ba-a3d024ae7d16">

**Top 10 stations for Members**

<img width="652" alt="res_top10_stations_member" src="https://github.com/justaszie/Bikeshare-Analysis/assets/1820805/ba3c9af9-fa18-4989-bb31-420f28e52072">

**Stations clustering for Casual Riders**

```sql
WITH processed_rides AS (
SELECT *,
REGEXP_REPLACE(
REPLACE(REPLACE(REPLACE(REPLACE(REPLACE(REPLACE(start_station_name, 'Public Rack - ', ''),' - midblock', ''), ' - SW', ''), ' - NW', ''), ' - SE', ''), ' - NE', '')
, r'( -)? (?:(?:N|E|W|S)|(?:North|West|East|South)|(?:(?:north|south|east|west) corner)|(?:NW|NE|SW|SE))$', '')
AS grouped_start_station_name
FROM `phrasal-brand-398306.bikeshare_data.rides`
)

, station_ride_counts AS (
    SELECT
    grouped_start_station_name,
    COUNT(*) AS station_rides,
    RANK() OVER (ORDER BY count(*) DESC) AS station_rank
    FROM processed_rides
    -- Ignoring stations with null values
    WHERE grouped_start_station_name IS NOT NULL
    AND rider_type = 'casual'
    GROUP BY grouped_start_station_name
    ORDER BY 3 ASC
)

SELECT
grouped_start_station_name,
station_rides,
SUM(station_rides) OVER (ORDER BY station_rank ASC) AS cumul_rides, -- Number of cumulative rides - i.e. sum of rides started in stations from TOP 1 station to the current station, included
ROUND(SUM(station_rides) OVER (ORDER BY station_rank ASC) / SUM(station_rides) OVER() * 100, 2) as prc_total_rides, -- Number of cumulative rides as a % of total rides
station_rank,
ROUND(station_rank / (SELECT MAX(station_rank) FROM station_ride_counts) * 100, 2) as prc_stations -- Number of stations (from TOP 1 station to the current station, included) as a % of the total number of stations
FROM station_ride_counts
ORDER BY station_rank
```

<img width="873" alt="res_station_cluster_casual_sample" src="https://github.com/justaszie/Bikeshare-Analysis/assets/1820805/2d363e45-4b50-4e1f-a216-96c0c5c927af">

### Routes Taken Queries
```sql
WITH processed_rides AS (
  SELECT *,
  REGEXP_REPLACE(
  REPLACE(REPLACE(REPLACE(REPLACE(REPLACE(REPLACE(start_station_name, 'Public Rack - ', ''),' - midblock', ''), ' - SW', ''), ' - NW', ''), ' - SE', ''), ' - NE', '')
  , r'( -)? (?:(?:N|E|W|S)|(?:North|West|East|South)|(?:(?:north|south|east|west) corner)|(?:NW|NE|SW|SE))$', '')
  AS grouped_start_station_name,
  REGEXP_REPLACE(
  REPLACE(REPLACE(REPLACE(REPLACE(REPLACE(REPLACE(end_station_name, 'Public Rack - ', ''),' - midblock', ''), ' - SW', ''), ' - NW', ''), ' - SE', ''), ' - NE', '')
  , r'( -)? (?:(?:N|E|W|S)|(?:North|West|East|South)|(?:(?:north|south|east|west) corner)|(?:NW|NE|SW|SE))$', '')
  AS grouped_end_station_name
  FROM `phrasal-brand-398306.bikeshare_data.rides`
)
-- Total rides by rider and route
, counts AS(
    SELECT
    rider_type,
    CONCAT(grouped_start_station_name, " - ", grouped_end_station_name) AS route,
    -- After the first look, I noticed routes that start and end in the same station. In this V2 of the query, I added a specific column for better visibility.
    (grouped_start_station_name = grouped_end_station_name) AS same_station,
    COUNT(*) AS total_rides,
    RANK() OVER (PARTITION BY rider_type ORDER BY COUNT(*) DESC) AS route_rank
    FROM processed_rides
    --  Ignoring routes that have either start or end station as null
    WHERE grouped_start_station_name IS NOT NULL AND grouped_end_station_name IS NOT NULL
    GROUP BY 1, 2, 3
)

SELECT *
FROM counts
WHERE route_rank <= 10;
```
**Top 10 routes for Casual Riders**

<img width="978" alt="res_top10_routes_casual" src="https://github.com/justaszie/Bikeshare-Analysis/assets/1820805/f3646e33-e610-4daf-b9b4-87b3ccbc41ba">

**Top 10 routes for Members**

<img width="825" alt="res_top10_routes_member" src="https://github.com/justaszie/Bikeshare-Analysis/assets/1820805/dd43f31e-e6ca-4ffc-93b9-7544bf16f3b0">

**Number of Distinct Routes**

```sql
WITH processed_rides AS (
  SELECT *,
  REGEXP_REPLACE(
  REPLACE(REPLACE(REPLACE(REPLACE(REPLACE(REPLACE(start_station_name, 'Public Rack - ', ''),' - midblock', ''), ' - SW', ''), ' - NW', ''), ' - SE', ''), ' - NE', '')
  , r'( -)? (?:(?:N|E|W|S)|(?:North|West|East|South)|(?:(?:north|south|east|west) corner)|(?:NW|NE|SW|SE))$', '')
  AS grouped_start_station_name,
  REGEXP_REPLACE(
  REPLACE(REPLACE(REPLACE(REPLACE(REPLACE(REPLACE(end_station_name, 'Public Rack - ', ''),' - midblock', ''), ' - SW', ''), ' - NW', ''), ' - SE', ''), ' - NE', '')
  , r'( -)? (?:(?:N|E|W|S)|(?:North|West|East|South)|(?:(?:north|south|east|west) corner)|(?:NW|NE|SW|SE))$', '')
  AS grouped_end_station_name
  FROM `phrasal-brand-398306.bikeshare_data.rides`
)
SELECT
rider_type,
COUNT(DISTINCT CONCAT(grouped_start_station_name, " - ", grouped_end_station_name)) AS num_routes,
FROM processed_rides
-- Ignoring routes that have either start or end station as null
WHERE grouped_start_station_name IS NOT NULL AND grouped_end_station_name is not null
GROUP BY rider_type
```

<img width="251" alt="res_distinct_routes_num" src="https://github.com/justaszie/Bikeshare-Analysis/assets/1820805/0440996c-c524-417e-9e7d-fffe2aaef8b1">
