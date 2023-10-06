# Bikeshare rides analysis
## 1. Project - the What and the Why
After I completed the Google Data Analytics Certification program on [Coursera](https://www.coursera.org/professional-certificates/google-data-analytics), I was keen to use its material on practical examples. 

The program includes an optional guided project in which you can solve a business problem for a fictional bike-share service, called Cyclastic, by analyzing a real-world dataset. The dataset contained the ride history data and the **specific business task** of the analysis was to:
1. explore the differences between the activity of the casual riders and members (riders holding annual membership pass)
2. use these insights to make recommendations on how marketing efforts can convert casual riders into members

Even though the project was an optional part of the certification program, I chose to complete it because I was interested in using real-world dataset and solve a realistic business problem and it allowed me to brush up on my key data analysis skills.

This document will describe:
1. The summary of the project
2. Final solution for the business case - a slide presentation for the stakeholders with the business case insights and recommendations
3. Details of my analysis process, including the queries, lessons learned and ideas for future improvements

## 2. Summary

### 2.1. Process
I followed the analysis process provided by Google program and adjusted it to my preferences. The overall process I followed had 6 phases:
1. **Ask** - refined the specific business task for the project
2. **Prepare** - collected and evaluated the data needed for the analysis
3. **Process** - cleaned the data using SQL to get it ready for analysis
4. **Analyze** - defined relevant questions, ran SQL queries to answer them using the data, validated my hypotheses, and gained surprising insights
5. **Share** - created a business case presentation to share the insights with stakeholders, including using visualizations. The stakeholders in this case were the marketing director and the executive team of the fictional company
6. **Act** - updated the presentation to include recommendations to solve the business problem using the insights

### 2.2 Data Tools
- **SQL**
    - Timeframe for analysis was not defined by the project. I wanted to analyze at least 1 year of rides data (5M+ rows) and it was too large for spreadsheets
    - It allowed me to brush up on my SQL skills for data cleanup and analysis
    - To focus on the analysis itself, I used BigQuery - Google's serverless data warehouse solution which was the simplest way to set up a database.
- **Google Sheets**
   - Used for some calculations which are easier to do in spreadsheets than in SQL and to build visualizations
   - I chose Sheets over Excel as it's free and, from my past experience, it is often used by startups and scaleups. The skills built on Sheets will be easily transferrable to Excel.

## 3. Business Case Solution
The final presentation that solves the business case can be accessed [here](https://docs.google.com/presentation/d/1bwYGy23ZWJG5qMf0imZLOfAincOCCUbpCrCYMcIStbE/edit?usp=sharing)

**TODO: decide (1) how much info about the business case context (company, chicago etc) to include and (2) where should it be - here (if yes, which section) or the presentation itself?

## 3. Preparing the data
The project is to solve the business problem of a fictional company. But the dataset to analyze comes from a real business. It is history of rides of a bikeshare service called Divvy, operated by Lyft in Chicago and is licensed out to public. 
- [Data](https://divvy-tripdata.s3.amazonaws.com/index.html) (AWS S3 bucket)
- [Data License](https://divvybikes.com/data-license-agreement)

### 3.1. Loading Into Database
The rides data is stored in a series of .csv files. In some cases, the .csv holds a month of data, in others - a whole quarter of data. I chose to analyze the data from a whole year 2022 because it should represent normal activity, unaffected by Covid-19 and it would allow to analyze seasonality. To do this, I uploaded the 12 .csv files to a Google bucket and created a SQL table *rides* in BigQuery by merging all 12 files together, since the data structure of the .csv files was the same.

### 3.2. Data Cleanup
The dataset does not have any description or data dictionary. So, first, after observing the data, I built a data dictionary to help with cleanup and analysis later.

| Column | Description (assumed) | Format | Comments |
|---|---|---|---|
| ride_id | Primary key | String | Alphanumeric value |
| rideable_type | Describes the typo of the bike used in the ride | String  Possible values: electric_bike classic_bike docked_bike | I filtered for unique values in excel.  |
| started_at | Date and time when the ride started | Date / time |  |
| ended_at | Date and time when the ride ended | Date / time |  |
| start_station_id | ID of the station where the ride started (bike was picked up). | String | Assuming this is the foreign key - link to stations table. Note that formats for different rows are different. Some have format “TAXXX”, others just have number value.  |
| start_station_name | Describes the name of the station | String |  |
| end_station_id | ID of the station where the ride started (bike was picked up). | String | Assuming this is the foreign key - link to stations table. Note that formats for different rows are different. Some have format “TAXXX”, others just have number value.  |
| end_station_name | Describes the name of the station | String |  |
| start_lat	 | Latitude coordinate where the ride started | Float |  |
| start_lng	 | Longitude coordinate where the ride started | Float |  |
| end_lat | Latitude coordinate where the ride ended | Float |  |
| end_lng | Longitude coordinate where the ride ended | Float |  |
| member_casual | Describes the type of the rider - either casual rider or holding annual membership | String Possible values: casual member |  |

<br /><br />
