# Google Data Analytics Capstone Project: Cyclistic Bike-Share

![Cyclistic Bike-Share_Logo](images/cyclistic_logo.png)

## Introduction

In this case study, I perform the work of a data analyst working for a fictional company, Cyclistic, which is a succesful bike-sharing company based in Chicago and which was started in 2016. In the brief, we are told that they have a fleet of more than 5,000 bikes and 600 docking stations dotted around the city. With this large number of bikes and docking stations, users can take a bike from their starting station and return it to any other station in the network, making it incredibly easy and convenient.

Furthermore, we are told that their offering is unique in that, apart from traditional two-wheel bikes, they also offer reclining bikes, hand tricycles, and cargo bikes, thus making bike-share more inclusive to people with disabilities and riders who can’t use a standard two-wheeled bike. The majority of riders opt for traditional bikes; about 8% of riders use the assistive options. We are also told that Cyclistic users are more likely to ride for leisure, but about 30% use the bikes to commute to work each day.

They have two types of customers: **casual riders** who purchase single-ride or full-day passes, and **members** who purchase annual memberships. Cyclistic’s finance analysts have concluded that **annual members are much more profitable than casual riders**, and the company's marketing director believes that the company's future success depends on **maximizing the number of annual memberships**. Therefore, my team and I have been tasked with understanding how casual riders and annual members use Cyclistic bikes differently. **Using the insights that we gain, our team will formulate a new marketing strategy to convert casual riders into annual members**.

In order to conduct my analysis of the data, I will be using Google's data analysis methodology which follows these six steps:
1. Ask
2. Prepare
3. Process
4. Analyze
5. Share
6. Act

### Ask

In this phase of the data analysis process, the key aim is to identify the **business task**, which in this case is:  

***Analyze historical bike trip data for the past 12 months to identify trends in how annual members and casual riders use Cyclistic bikes differently, in order to provide actionable insights for developing a marketing strategy to convert casual riders into annual members.***

My goal is thus to provide actionable insights for the marketing team by identifying distinct usage patterns between these two customer segments.

### Prepare

**Data Source**  

In order to complete my data analysis, I made use of Cyclistic's historical trip data to analyze usage trends for the past twelve months which, in this case, was from June 2025 to May 2026. The original dataset can be found [here](https://divvy-tripdata.s3.amazonaws.com/index.html). Although Cyclistic is a fictional company, the data is real data for a company called Divvy Bikes that is based in Chicago. The data has kindly been made available to us by Motivate International Inc. for the purposes of these capstone projects under this [license](https://divvybikes.com/data-license-agreement).

**Data Structure**  

The data has been provided in the form of CSV files, one file for each month, and with the naming structure of 'YYYYMM-divvy-tripdata.csv'. My combined dataset has thus been composed from 12 of these files in consecutive date order. I used Excel to conduct an initial data assessment on the files, checking for things such as schema consistency and general data quality. Each row in these CSV files represents a single bike trip, with each of the trips having been anonymized and identified with a unique `ride_id` value. The primary key for each record is thus `ride_id`. The table below gives a description of the 13 column names, their data types, and a description of each column contained in the original CSV files:

| Column Number | Column Name | Data Type | Description |
| -: | - | - | - |
| 1 | ride_id | STRING | Ride ID (unique to each trip) |
| 2 | rideable_type | STRING | Bike type |
| 3 | started_at | TIMESTAMP | Trip start date and time |
| 4 | ended_at | TIMESTAMP | Trip end date and time |
| 5 | start_station_name | STRING | Trip start station name |
| 6 | start_station_id | STRING | Trip start station ID |
| 7 | end_station_name | STRING | Trip end station name |
| 8 | end_station_id | STRING | Trip end station ID |
| 9 | start_lat | FLOAT64 | Trip start latitude |
| 10 | start_lng | FLOAT64 | Trip start longitude |
| 11 | end_lat | FLOAT64 | Trip end latitude |
| 12 | end_lng | FLOAT64 | Tripe end longitude |
| 13 | member_casual | STRING | Rider type (casual rider or member) |

**Data Limitations**  

While the dataset provided by Divvy Bikes is overall comprehensive and reliable, it is subject to several limitations that must be taken into account when performing an analysis on the data. After a brief initial exploratory data analysis using Excel, these are the key observations that I made:

1. **Lack of Personally Identifiable Information**  
Data privacy regulations prohibit the use of riders' personal information. It is thus impossible to create profiles for individual riders, this includes both casual riders and annual members. While it can almost safely be assumed that most annual members are local Chicago residents, the implication of the lack of personally identifiable information is that we can deduce even less about casual riders, and we cannot determine if they are local Chicago residents, tourists or daily commuters. We are also unable to track if an individual has purchased multiple single-ride passes over time. This makes is impossible to distinguish between frequent casual riders and one-time riders.

2. **Missing Station and GPS Coordinate Data**  
A large portion of the dataset contains missing `start_station_name`, `end_station_name`, or GPS coordinate fields. This represents a significant portion of the dataset. While we do mostly have the GPS coordinates for these trips with missing station fields, the lack of station names makes it harder to analyze station-specific usage patterns without extra data-cleaning steps, such as mapping the available GPS coordinates back to known stations in the bike-share network.

3. **Negative/Zero Ride Lengths**  
There are several trips where the `ended_at` time is before ot equal to the `started_at` time. We can assume that these instances can be related to factors such as system maintenance, bike checks, or technical glitches in the docking sensor.

4. **Extremely Short/Long Rides**  
Trips that last for only a few seconds or that are extremely long (more than 24 hours) should be considered to be anomalies. Very short rides could be assumed to be "false starts", where a user immediately re-docks a bike due to a mechanical issue or a change of mind. Very long rides could be assumed to be a stolen bike or one that wasn't docked correctly.

5. **Limited Context**  
The dataset provides only "what happened" (the ride), but not "why it happened" (the motivation). We thus lack qualitative data about the rides and have to infer intent from behaviour, which is an assumption, not a fact. To be more specific, the data does not tell us why any of the bikes were used, nor have we been supplied with any contextual data such as weather conditions, demographics of the riders, etc.

### Process

**Importing and Combining the Data**

Due to the large size of the CSV datasets (millions of rows), I decided to use SQL via BigQuery in order to process the data as Excel would've be an inefficient tool for this job due to the limitations of how many rows it can handle. BigQuery is the ideal tool for this job as it can easily and efficiently handle very large datasets containing millions of rows.

However, as I am using the Sandbox version of BigQuery and do not have billing enabled on my account, I was unable to create a bucket in Google Cloud Services in order to upload the original CSV files to BigQuery, which would've been the preferred way of doing this. Many of these CSV files exceed 100MB in size, which is the local file upload limit in BigQuery, so this option was not available to me either. My workaround to this problem was to use the Pandas library in Python to 'chunk' the CSV files into smaller files each containing 50,000 rows or less, making the size of the files to be around 10MB each. I achieved this by running the following Python script in the folder containing the CSV source files:

```
import pandas as pd
import os

# Configuration
input_folder = './'  # Files are in the current directory
output_folder = './split_files/'
chunk_size = 50000  # Adjust rows per file to ensure < 100MB
os.makedirs(output_folder, exist_ok=True)

# List all CSV files in directory
files = [f for f in os.listdir(input_folder) if f.endswith('.csv')]

for file in files:
    print(f"Processing {file}...")
    # Use chunksize to read large files without overloading RAM
    reader = pd.read_csv(file, chunksize=chunk_size)
    
    for i, chunk in enumerate(reader):
        chunk.to_csv(f"{output_folder}{file.split('.')[0]}_part_{i}.csv", index=False)

print("Splitting complete.")
```

After the 'chunking' process was complete, I was left with 113 CSV files that now needed to be uploaded to BigQuery. BigQuery would not allow me to select multiple files using the 'Upload' option, and trying to upload them individually would've been time-consuming and inefficient, so I decided to use the 'bq' tool in the Google Cloud CLI suite of programs to automate the process of uploading those smaller CSV files directly into BigQuery from my local storage device.

However, before I could do that, I had to create a table inside of BigQuery and populate it with the first of the chunked CSV files in order to create the table schema so that the remaining 112 files could be imported correctly. As this first file was a single file of only about 10MB in size, it was possible to upload it using the 'Upload' option in the BiqQuery web interface. I then removed this CSV file from the directory containing the other CSV files as I did not want to duplicate its data in the table by re-importing it into the table during the batch upload using the 'bq' tool. Further to this, as each of the chunked files each contained a header, I had to insert the `skip_leading_rows=1` parameter into the command line prompt to remove the headers so that they would not be imported into the table I was creating. This was the command line prompt that I used to import the remaining 112 files into BigQuery:

```
for %f in (*.csv) do bq load --source_format=CSV --skip_leading_rows=1 --noreplace cyclistic_capstone_project.cyclistic_12_months_dataset "%f"
```

**Ingestion Verification**

After I performed the bulk ingestion of chunked CSV data into a single BigQuery relational table called `cyclistic_12_months_dataset`, I validated the result via this `COUNT(*)` query, confirming a total of 5,848,703 rows, representing an entire year of data:

```
SELECT
  COUNT(*)
FROM `course-493609.cyclistic_capstone_project.cyclistic_12_months_dataset`
```

I verified this by using Excel to reconcile the total row count of the 12 CSV source files against the imported BigQuery table. The validation confirmed 5,848,703 records across both datasets, ensuring no data loss occurred during the upload from local storage to the cloud.

**Data Cleaning and Transformation**

*Checked for Duplicate `ride_id` records*

As `ride_id` is intended to be the unique identifier (primary key) for each individual bike ride, we can assume that there should be no duplicate `ride_id` entries in our dataset. I thus performed a validation in BigQuery using the following code:

```
SELECT 
  ride_id, 
  COUNT(*) AS total_occurrences
FROM `course-493609.cyclistic_capstone_project.cyclistic_12_months_dataset`
GROUP BY ride_id
HAVING COUNT(*) > 1;
```

The query identified 35 duplicate `ride_id` entries which needed to be removed. To ensure data integrity, I created a cleaned version of the dataset called `cyclistic_clean_dataset` by performing a `DISTINCT` selection, thus creating a new table:

```
CREATE OR REPLACE TABLE `course-493609.cyclistic_capstone_project.cyclistic_clean_dataset` AS
SELECT DISTINCT *
FROM `course-493609.cyclistic_capstone_project.cyclistic_12_months_dataset`
```

I then used a `COUNT(*)` query to verify that the new table had 35 rows less than the original table, which it did:

```
SELECT
  COUNT(*)
FROM `course-493609.cyclistic_capstone_project.cyclistic_clean_dataset`
```

*Validated Ride Durations*

The next step in the data cleaning came in the form of culling bike trips with durations that were of an impossible or impractical length to be considered valid. I decided to cull all trips with the following ride durations:
1. Ride duration < 0
2. Ride duration = 0
3. Ride duration < 1 minute
4. Ride duration > 18 hours

My reasonings behind the culling criteria are as follows:
1. Ride durations that are negative are logically impossible and could be the result of a data-recording issue, system maintenance, or a docking sensor issue.
2. Ride durations that have a duration of zero seconds have thus travelled zero distance and cannot be considered to be valid trips, regardless of the reason.
3. Ride durations that are only a few seconds long could be due to system maintenance, a re-docking attempt, or a user undocking a bike and then changing their mind and re-docking it, but either way, they are too short to tell us anything useful about the trip. While these are technically 'trips', they don't tell us anything useful about usage behaviour as they are trips too short to be purposeful.
4. Ride durations that are longer than 18 hours in duration represent a physical impossibility for most riders, even if they stopped to have breaks along the way. Ride lengths exceeding 18 hours should thus be considered to be either multiple rides, bikes having being removed from their docking stations for maintenance, or possibly bikes that have been stolen.

In order to faciliate the cleaning process, and also to make the data more human-readable, I added the `ride_duration_mins` column to the dataset, which shows ride duration rounded off to the nearest minute. Being a calculated field, it was created by calculating the difference between the `started_at` and `ended_at` times, dividing by 60, and then rounding the result to the nearest integer. This is the code that I used to achieve this:

```
CREATE OR REPLACE TABLE `course-493609.cyclistic_capstone_project.cyclistic_clean_dataset` AS
SELECT 
  *,
  -- Calculate duration in seconds, divide by 60, and round to the nearest integer
  SAFE_CAST(ROUND(TIMESTAMP_DIFF(ended_at, started_at, SECOND) / 60) AS INT64) AS ride_duration_mins
FROM `course-493609.cyclistic_capstone_project.cyclistic_clean_dataset`
```

After engineering the `ride_duration_mins` feature, I performed a quality audit using an anomaly threshold query. This enabled me to quantify non-representative data, ensuring that the final analysis focuses exclusively on valid, intentional trips:

```
SELECT
  COUNTIF(ride_duration_mins < 0) AS negative_duration,
  COUNTIF(ride_duration_mins = 0) AS zero_duration,
  COUNTIF(ride_duration_mins < 1) AS less_than_1_minute,
  COUNTIF(ride_duration_mins > 1080) AS eighteen_plus_hours,
  COUNTIF(
    ride_duration_mins < 0 
    OR ride_duration_mins = 0 
    OR ride_duration_mins < 1 
    OR ride_duration_mins > 1080
  ) AS total_anomalies
FROM `course-493609.cyclistic_capstone_project.cyclistic_clean_dataset`
```

My audit revealed the following:

| Anomaly | Number of Occurences |
|:---|---:|
| Ride duration < 0 | 29 |
| Ride duration = 0 | 115,688 |
| Ride duration < 1 minute | 115,717 |
| Ride duration > 18 hours | 6,982 |
| Total anomalies | 122,609 |

For the purposes of interpretation, it must be noted that the total number of anomalies was calculated by adding together trips with ride durations less than one minute and longer than 18 hours, and that trips with ride durations less than zero and equal to zero are merely a subset of trips with ride durations less than one minute.

The 122,609 ride duration anomalies represent only 2.09% of our dataset, so I felt confident in culling them as this reduction was statistically insignificant, but enhances the reliability of our results. The culling of these outliers was performed with the following code:

```
CREATE OR REPLACE TABLE `course-493609.cyclistic_capstone_project.cyclistic_durations_culled` AS
SELECT *
FROM `course-493609.cyclistic_capstone_project.cyclistic_clean_dataset`
WHERE ride_duration_mins >= 1 
  AND ride_duration_mins <= 1080;
```

This produced a new table named 'cyclistic_durations_culled' containing 5,726,059 entries, which I verified the integrity of using a `COUNT(*)` query.

To validate my decision to remove these records from the dataset, and to check if these anomalies were skewing the results, I conducted a comparative statistical audit by calculating the mean, minimum and maximum ride durations before and after the removal of these anomalies using variations of this code:
```
SELECT 
  AVG(ride_duration_mins) AS average_ride_duration_mins,
  MIN(ride_duration_mins) AS shortest_ride,
  MAX(ride_duration_mins) AS longest_ride
FROM `course-493609.cyclistic_capstone_project.cyclistic_clean_dataset`
```

As we can see from the table below, my filtering logic successfully isolated the intended outlier records without impacting the integrity of the dataset: 

| Statistic | Before Cleaning | After Cleaning | Change Value | Change Percent |
| :--- | ---: | ---: | ---: | ---: |
| Row Count | 5,848,668 | 5,726,059 | -122,609 | -2.09% | 
| Avg ride duration (mins) | 15.98 | 14.57 | -1.41 | -8.82% |
| Min ride duration (mins) | -55 | 1 | 56 | -101.82% |
| Max ride duration (mins) | 1,575 | 1,080 | -495 | -31.43% |

We have confirmed that these ride duration anomalies were skewing our results to a significant degree and had the potential to bias any potential insights we could gain from the data, so their removal was necessary and justified.

It must be pointed out the negative percentage change in the 'Min ride duration (mins)' statistic is in fact a *positive* improvement of 101.82% relative to the absolute magnitude of the original baseline. The number is negative because of the instances of logically impossible negative ride durations present in the dataset.

*Checked for Trips with Missing Start and End Point Data*

For a bike trip (ride) record to be considered complete and useful for the purposes of our analysis, it must contain complete start and end point data. As stated earlier, my initial exploration of the monthly datasets revealed that a significant portion of the datasets contains missing `start_station_name` and `end_station_name` data, as well as missing GPS coordinate data for bike trips, so this had to be investigated in detail. First, I checked how many bike trip entries lack station names, so I ran this query in BigQuery to get the exact figures:

```
SELECT
  COUNTIF(start_station_name IS NULL) AS missing_start_station,
  COUNTIF(end_station_name IS NULL) AS missing_end_station,
  COUNTIF(start_station_name IS NULL AND end_station_name IS NULL) AS missing_both,
  COUNTIF(start_station_name IS NULL OR end_station_name IS NULL) AS missing_either
FROM `course-493609.cyclistic_capstone_project.cyclistic_durations_culled`
```

The query showed the following:

| Anomaly | Number of Occurences |
| :--- | ---: |
| Missing `start_station_name` | 1,179,653 |
| Missing `end_station_name` | 1,218,780 |
| Missing `start_station_name` and `end_station_name` | 525,042 |
| Missing `start_station_name` or `end_station_name` | 1,873,391 |

In the project brief, we are told that Cyclistic's business model is station-based and uses fixed docking systems located at each station from which a user must remove a bike for use and then either return it to the same station or any other station in the network. It is thus a fair assumption that the primary source of 'location truth' for each bike trip *should* be the station names, as bikes need to be physically docked, or at the very least be present in the GPS-ring-fenced station locations, in order for station names to be recorded in the trip records, assuming that no technical glitch or data collection error occurred during this process.

However, as our query revealed, 1,873,291 trips out of our current dataset of 5,726,059 trips are missing either the start or end station names, making these records incomplete and problematic to use for our analysis. These incomplete entries makes up a staggering 32.7% of our dataset and could not simply be removed without leading to catastrophic data losses that could severely skew the analysis, destroy statistical power, and introduce significant selection bias. A decision had to be made about how to handle these trips with missing station names.

A high frequency of missing station names could suggest a systemic data collection issue, or possible user non-compliance with docking protocols, but the assumption could not be made that these trips were automatically invalid. Before making a decision on how to handle the missing station names, I conducted a data availability assessment to determine how complete the GPS coordinate data was for our bike trip records:

```
CREATE OR REPLACE TABLE `course-493609.cyclistic_capstone_project.spatial_completeness_report` AS
WITH TotalCount AS (
  SELECT COUNT(*) AS total_rows 
  FROM `course-493609.cyclistic_capstone_project.cyclistic_durations_culled`
),
MissingData AS (
  SELECT
    SUM(CASE WHEN start_lng IS NULL THEN 1 ELSE 0 END) AS missing_start_lng,
    SUM(CASE WHEN start_lat IS NULL THEN 1 ELSE 0 END) AS missing_start_lat,
    SUM(CASE WHEN end_lng IS NULL THEN 1 ELSE 0 END) AS missing_end_lng,
    SUM(CASE WHEN end_lat IS NULL THEN 1 ELSE 0 END) AS missing_end_lat
  FROM `course-493609.cyclistic_capstone_project.cyclistic_durations_culled`
)
SELECT
  -- Count of present data
  t.total_rows - m.missing_start_lng AS start_lng_present,
  t.total_rows - m.missing_start_lat AS start_lat_present,
  t.total_rows - m.missing_end_lng AS end_lng_present,
  t.total_rows - m.missing_end_lat AS end_lat_present,
  
  -- Percentages of present data
  ROUND(SAFE_DIVIDE(t.total_rows - m.missing_start_lng, t.total_rows) * 100, 4) AS start_lng_pct,
  ROUND(SAFE_DIVIDE(t.total_rows - m.missing_start_lat, t.total_rows) * 100, 4) AS start_lat_pct,
  ROUND(SAFE_DIVIDE(t.total_rows - m.missing_end_lng, t.total_rows) * 100, 4) AS end_lng_pct,
  ROUND(SAFE_DIVIDE(t.total_rows - m.missing_end_lat, t.total_rows) * 100, 4) AS end_lat_pct,
  
  -- Percentages of missing data
  ROUND(SAFE_DIVIDE(m.missing_start_lng, t.total_rows) * 100, 4) AS missing_start_lng_pct,
  ROUND(SAFE_DIVIDE(m.missing_start_lat, t.total_rows) * 100, 4) AS missing_start_lat_pct,
  ROUND(SAFE_DIVIDE(m.missing_end_lng, t.total_rows) * 100, 4) AS missing_end_lng_pct,
  ROUND(SAFE_DIVIDE(m.missing_end_lat, t.total_rows) * 100, 4) AS missing_end_lat_pct
FROM TotalCount t, MissingData m;
```

The results were as follows:

| Anomaly | Count | Percentage Missing |
| :---    | ---:  | ---:               |
| Start Longitude Missing | 0 | 0% |
| Start Latitude Missing | 0 | 0% |
| End Longitude Missing | 290 | 0.004% |
| End Latitude Missing | 290 | 0.004% |

The query showed that the end coordinates were missing in only 290 of bike trip records, representing only 0.004%. I then checked to see if these records with missing end coordinates had end station names, but they didn't, so it was not possible to complete the records using station names as we didn't have that data either. As these incomplete records make up only a miniscule and statistically insignificant part of the dataset, I decided that these records should be removed rather than wasting time exploring these anomalies further:

```
CREATE OR REPLACE TABLE `course-493609.cyclistic_capstone_project.cyclistic_gps_complete` AS
SELECT *
FROM `course-493609.cyclistic_capstone_project.cyclistic_durations_culled`
WHERE start_lat IS NOT NULL 
  AND start_lng IS NOT NULL 
  AND end_lat IS NOT NULL 
  AND end_lng IS NOT NULL;
```

A new table called `cyclistic_gps_complete` was created and I used a `COUNT(*)` query to verify that the 290 records with missing GPS coordinate data were removed.

At this point, it became clear to me that a decision needed to be made on what my source of location truth should be for these bike trips. I could not use start and end station names solely as a third of the records are missing this complete data. Although the brief tells us that Cyclistic's business model is station-to-station, the data told another story. My earlier inspection of the dataset revealed something unexpected that was not mentioned in the brief: apart from the station names that we expected, there was another type of station name data that started with the prefix 'Public Rack'. I investigated the extent of these entries by running the following query:

```
SELECT
  COUNT(*) AS total_public_rack_trips
FROM `course-493609.cyclistic_capstone_project.cyclistic_gps_complete`
WHERE start_station_name LIKE '%Public Rack%' 
   OR end_station_name LIKE '%Public Rack%';
```

The results were that 45,195 of the trips (0.79%) either started or ended at a public rack rather than an official station. This is a tiny percentage and statistically insignificant to my analysis, but it revealed that Cyclistic's bike-share network has evolved and now includes docking points that lie outside of the official station network. This is most likely due to the network having being expanded to include these public racks as time has gone by, but the brief not having been updated, or because it was an error of omission when the brief was drafted. As these records constitute valid trips by the virtue of them having a start and end point, I decided to retain these records in the dataset.

Considering that 32.7% of the dataset contains missing station names, yet 100% of the updated dataset contains complete GPS coordinate data, I decided to shift my approach and use solely GPS coordinate data as the source of location truth for each bike trip as valid GPS coordinates provide sufficient confirmation of point-to-point transit. This decision ensured the analysis remained representative of the entire network’s activity rather than only a subset of perfectly logged records.

The alternative that I had to the GPS-coordinate-only approach was to use geospatial imputation to impute the missing station names using the available GPS coordinates, but this approach was not chosen due to the following limitations:
1. Stations do not exist as a single fixed GPS coordinate point as they occupy lateral space, and without knowing their boundary GPS coordinates that define them (a perfectly square station would require four sets of coordinates to define its location accurately), it is very difficult to perform this kind of imputation.
2. Without knowing the exact boundary GPS coordinates for each station, the easiest viable approach would be to define the station location as a single fixed GPS coordinate and then use a circular radius to define the capture zone, matching the available GPS coordinates to the nearest known station.
3. However, a capture radius that is too small would exclude stations, and one too large would capture more than one station due to overlap, resulting in null or false mappings.

Even though I decided to use GPS coordinate data as the only source of location truth for these bike trip records, it was a significant compromise as GPS geolocation is itself subject to the following limitations that have bearing on our analysis:
1. Due to phenomena such as 'GPS drift', 'urban canyoning' and 'GPS sensor noise', which can result in GPS coordinates being recorded that are not the exact true location of the bike.
2. Bad weather can interfere with GPS satellite signals and return false values.
3. GPS sensors can themselves be faulty, require calibration, and thus provide inaccurate location information.

In a perfect world, we would have station name data for every record in our dataset, but as we don't, this compromise of using only GPS coordinate data was reached. However, assuming that the GPS coordinate data that we have in our dataset is accurate enough for purposes of this analysis, it has the benefit of more accurately showing us true bike trip behaviour of users as it shows us exactly where they pick up and the bike and drop it off, regardless of whether it was at an offical station, public rack, or on the side of the road.

*Checked for Null Values in the `member_casual` Column*

As the stated goal of this analysis is to determine the difference in bike use behaviour between members and casual riders, I checked to for any null values in the `member_casual` field. If null values exists, then those records need to be removed from the dataset as no inferences can be made without knowing the membership status of a user:

```
SELECT
  COUNT(*) AS null_member_casual_count
FROM `course-493609.cyclistic_capstone_project.cyclistic_gps_complete`
WHERE member_casual IS NULL;
```

No null values were found.

*Checked for null values in `started_at` and `ended_at` Columns`

In order to perform a thorough analysis of the data, we need complete timestamp data for when each trip started and ended. I ran the following query to check for null values in these respective fields:

```
SELECT
  COUNT(*) AS null_timestamp_count,
  COUNTIF(started_at IS NULL) AS missing_started_at,
  COUNTIF(ended_at IS NULL) AS missing_ended_at
FROM `course-493609.cyclistic_capstone_project.cyclistic_gps_complete`
WHERE started_at IS NULL 
   OR ended_at IS NULL;
```

No null values were found.

*Created New Columns Called `day_of_week` and `month`*

Part of my analysis will involve analyzing usage patterns depending on the day of the week and month of the year that each trip was begun, so these new columns were created to make it easier to perform granular trend analysis of the data. The new columns were created by extracting the information from the timestamps in the `started_at` column, thus transforming the raw timestamp data into actionable insights for understanding seasonal rider behaviour:

```
CREATE OR REPLACE TABLE `course-493609.cyclistic_capstone_project.cyclistic_final_dataset` AS
SELECT
  *,
  -- Extracting day of week (1 = Sunday, 7 = Saturday)
  EXTRACT(DAYOFWEEK FROM started_at) AS day_of_week,
  -- Extracting month (1 = January, 12 = December)
  EXTRACT(MONTH FROM started_at) AS month
FROM `course-493609.cyclistic_capstone_project.cyclistic_gps_complete`;
```

*Removed Columns Not Needed For My Analysis*

In order to prepare the dataset for the next steps in the process, I removed the following columns that I would not be requiring for my analysis, and created a new table called `cyclistic_tableau`:
* `start_station_name`
* `start_station_id`
* `end_station_name`
* `end_station_id`

This culling process also helped to reduce the size of the dataset, making it more efficient to process in the next steps.

*Summary of Data Cleaning and Transformation Activities*

The data cleaning and transformation phase prioritized data quality without compromising the representative volume of the dataset, reducing selection bias. Following a comprehensive audit of spatial and temporal integrity, 122,853 records (2.1%) were removed due to irreconcileable data gaps, thus eliminating noise. The resulting high-fidelity dataset `cyclistic_tableau` consists of 5,725,850 records. This high preservation rate (97.9%) ensures that the final analytical model remains unbiased, robust, and fully aligned with the requirements for the discovery of core insights, particularly those concerning member versus casual rider behavior, and are based on a comprehensive and representative sample of the overall bike-share system usage.

### Analyze

TALK ABOUT HOW THE DATA WAS IMPORTED INTO TABLEAU, SMALLER TABLES, STATIC RENDERINGS, ETC. OPTIMIZATION, SPEED


**Annual Bike Rides By User Type**

IMAGE

* In the 12 months of data that were analyzed, Members were the dominant user segment, accounting for 64.44% of bike rides, compared to Casual Riders at 35.56%.
* While Casual Riders represent over a third of the total ride volume, the membership subscription model is the primary driver of engagement, capturing nearly two-thirds of all trips.
* This large gap between the two represents the core busines opportunity for Cyclistic, and there is a clear opportunity to investigate conversion strategies aimed at transitioning casual riders into the annual membership program.

**User Type Breakdown by Month**

IMAGE

* While the usage of Casual Riders follows a strong seasonal curve peaking mid-year during the warmer months, Members maintain a higher baseline throughout the year, with a smaller peak during the warmer summer months, indicating that their need for transportation is relatively inelastic. This demand inelasticity strongly suggests usage that is utilitatian in nature, and is indicative of a commuter demographic utilizing the service for daily, recurring trips between home, work, or transit hubs.
* Conversely, the usage by Casual Riders appears to be more discretionary in nature, influenced by seasonal factors, such as the weather. Hence, we make the assumption that Casual Riders are predominantly comprised of users whose bike usage is recreation-focused.

**Monthly Ridership Deviation from Mean**

While the previous breakdown by raw ride counts offered an essential quantitative breakdown of absolute demand, a secondary analysis was required to quantify the behavioral differences between user groups. To move beyond simple ride volume and confirm my suspicions about demand elasticity, I normalized the data to represent the monthly ridership deviation from mean.

IMAGE

* This visualisation exposes the elasticity profile of each group, showing that Casual Riders exhibit significantly higher volatility (elasticity), with monthly deviations swinging from approximately -85% to +90% relative to their annual mean.
* Conversely, Members maintain a much tighter range of deviation (approximately -64% to +45%), providing empirical evidence that their demand is more inelastic and resistant to seasonal shifts.
* During the peak of winter, we see that Casual Rider all but disappears, whereas Member demand retains a hard floor of activity. The November to March decline thus represents a threshold of feasability for both rider groups.
* This qualitative contrast in user behaviour confirms that while Members provide the stable, essential baseline of the business (they are mostly commuters), Casual Riders are the primary drivers of seasonal capacity fluctuations (they are mostly recreational users), making them the ideal target for conversion strategies.
* Even for utlitatrian users, such as commuters, the demand for bike trips is rarely perfectly inelastic. In cities like Chicago, which experiences severe winters, the sharp decline in Member ridership between November and March is driven by environmental constraints that override even the most routine commute, resulting in transport mode substitution. This doesn't invalidate my theory, but rather adds context.

**Day of Week Analysis**

While seasonal analyses give us a broad overview of *why* people ride during different times of the year (i.e. commuter vs recreational), day- and time-of-use analyses give us a better idea of rider intent regardless of the season. We would expect that commuter rides would be more concentrated on the working days (i.e. Monday to Friday), and also on working times (i.e. early mornings and late afternoons).

IMAGE

* As we can see from the chart, Member ridership is heavily concentrated on weekdays, consistent with what we would expect from non-discretionary, utilitarian commuting, where riders rely on Cyclistic for their work commutes.
* In contrast, Casual Rider volume is lowest during the early workweek and exhibits a surge towards the weekend, peaking on Saturday. This weekend-dominant pattern reinforces the qualitative assessment that Casual Rider usage is primarily driven by recreational, leisure-based motivations rather routine transit needs.
* This chart confirms that the two user groups operate on fundamentally different temporal schedules.

**Time of Day Analysis**

To finalize the behavioural characterization of Cyclistic users, I performed an hourly trend analysis, visualized in the line chart below. This data provides the clearest temporal distinction between the two user segments:

IMAGE

* As we expected, the line chart shows sharp and distinct peaks around 08h00 and 17h00 in the Members group, which aligns closely with standard Chucago workday schedules. It confirms the utilitarian nature of these trips, i.e. that they are work commutes. This bimodal pattern is strong quantitative evidence that Members rely on Cyclistic's bike-share service as a primary transit method for the workday.
* Also, as expected,  we see a more unimodal distribution with the Casual Riders group, with a gradual increase in trips that peak in the late afternoon, characteristic of recreational or tourist usage. The absence of a morning rush-hour peak indicates that this segment is not constrained by traitional workplace arrival times, further validating their profile as recreational or leisure-based users.
* These temporal fingerprints provide proof that the two user segments are not just different in volume, but exhibit two different usage patterns.

**Average Ride Duration by Month**

The visualization below illustrates a clear divergence in ride duration between the two user groups based on the season, providing further empirical evidence of the intent behind rider behaviour:

IMAGE

* Members maintain a stable, lower average ride duration throughout the year, typically around 10 to 12 minutes, which shows their intent to engage in efficient travel. This consistency is indicative of purposeful, utilitarian transit where riders prioritize the most direct route to their destination (work).
* Casual Riders exhibit significantly longer ride durations, peaking at over 20 minutes during the warm summer months. The higher average duration and increased seasonal volatility confirm a leisure-based usage patterns, where the experience and duration of the ride itself are central to the user's engagement.

**Average Ride Duration by Month**

The line chart below confirms the divergence in ride duration between the two user groups:

IMAGE

* Members maintain a consistently stable and shorter average ride duration, regardless of the day of the week. There is a slight increase in trip duration for Member over the weekend, suggesting some recreational bike usage on their part.
* Again, Casual Riders demonstrate significant variability, with ride durations escalating sharply as the weekend approached, and peaking on the weekend. This weekend-dominant increase confirms a leisure-based intent, where the duration of the ride is extended for recreational exploration rather than functional transportation.

**Usage by Bike Type**

My cleaned dataset `cyclistic_tableau` contains only two type of bikes: 'classic bike' and 'electric bike'. I doubled-checked the original uncleaned dataset `cyclistic_12_months_dataset` to make sure that no bike types were lost from this analysis due to data cleaning - none were. We can speculate that the choice of bike might serve as an indicator of user intent, so to investigate this, I created the following visualization:

IMAGE

* Electric bikes are the most utilized equipment across both user segments, demontrating that both sets of riders value the efficiency and ease provided by electric-assist technology.
* We can also see that Members utilize electric bikes at the highest volume, reinforcing their profile as utilitatian commuters, as they prioritize the faster, most reliable transport method for their daily work-related trips.
* While Casual Riders also use electric bikes to a significant degree, their sustained use of classic bikes is consistent with a recreational profile, not constrained the the time pressures of a daily commute.

**Comparative Spatial Flow Analysis**

As my dataset is huge - more than five million rows - and the only complete spatial infomation we have for each bike trip is GPS coordinates, I had to decided on an efficient way to visualize this data in Tableau. In order to do so, I created a new summary table in BigQuery called `cyclistic_spatial` that applies spatial binning protocols to the GPS coordinate data found in my cleaned dataset. GPS coordinates were rounded off to four decimal places, resulting in a spatial accuracy of roughly 11m, which is perfect for providing us with a granular level of information whilst keeping the processing in Tableau fast and efficient.

This visualization uses a comparative spatial flow matrix to contrast the bike-share utilization patterns of the two user segments. By isolating the start and end coordinates of everytrip into a 2x2 grid, the visualization removes visual occlusion and reveals the underlying pulse of the Cyclistic network:

IMAGE

* The Member panels in the right column demonstrate a high degree of spatial alignment between start and end locations, indicating a predictable, utilitarian communter loop, where riders utilize the service for round-trips, whereas for Casual Riders, there is a lower degree of spatial alignment, indicating that these trips are not return trips, but recreational in nature.
* For Members, the relative evenness in size of the circles (representing the number of trips) in the start and end locations further indicates that these are mostly round-trips, whereas for Casual Riders, the larger circle size variance is suggestive of predominantly one-way trips that are recreational in nature.
* The Member panels also show a greater geographic spread of ride start and end locations, with many rides going between residential suburbs and the central business district, whereas for Casual Riders, the trips are concentrated in known tourist areas of the city, most very near the lakefront.
* This comparative spatial flow analysis clear shows the difference in usage patterns between the two user groups.

**Net Flow Intensity Analysis**

This analysis uses a net flow intensity matrix to diagnose inefficiencies within the Cyclistic bike-share network. By quantifying the imbalance between trip start and end points, this visualization identifies specific nodes where the system faces potential inventory pressures, either through bike depletion (a net departure shortage - red squares) or surplus accumulation (a net arrival surplus - blue squares). Those nodes which are balanced are represented by the white squares.

IMAGE

* The comparison reveals that Members and Casual Riders impose different logistical loads on the network, with imbalances being present in each respective group's characteristic start and end trip points.
* Distinct clusters of imbalances in the Casual Riders panel highlight that recreational usage often drives demand towards specific attractor nodes, such as parks, the waterfront, and tourist hotspots.
* Conversely, we see that the imbalances present in the Members panel are predominant in locations that are associated with work commutes.
* This visualization provides concrete, data-backed evidence for where Cyclistic should deploy its maintenance and rebalancing teams to maintain bike availability.

### Share & Act

**Key Findings Summary**

The analysis of 5.7 million records over the past 12 months reveals two distinct customer segments operating within the Cyclistic bike-share ecosystem. To move beyond descriptive statistics and into strategic action, we have categorized these segments into two core personas:

* **The Efficient Commuter (Member):** This segment represents the backbone of the system (64.4% of riders). Their behavior is characterized by inelastic, high-frequency, short-duration trips concentrated on weekdays during peak commuting hours (08h00 and 17h00). Their usage is utilitarian, focused on efficiency, and highly consistent throughout the year.
* **The "Leisure Explorer" (Casual Rider):** This segment accounts for 35.56% of riders. Their behavior is highly elastic and seasonal, peaking during warmer months and weekends. They favor longer ride durations and exhibit spatial patterns concentrated around leisure hotspots rather than the central business district.

**Strategic Recommendations**

Based on these behavioral fingerprints that I have identified, I propose the following three-pronged marketing and operational strategy to drive the conversion of Casual Riders into annual Members:

**1. The "Weekend Warrior" Membership Tier**

Casual Riders exhibit a massive spike in usage on weekends, yet they currently pay per-ride or for day passes.

**Action:** Introduce a "Weekend-Only" annual membership that is significantly cheaper than the full annual membership.

**The Logic:** This acts as a bridge product. It lowers the barrier to entry for frequent weekend users who are hesitant to commit to a full-week annual membership due to the higher cosst. By providing a cost-effective, high-value alternative to day passes, we capture their loyalty and normalize the subscriber relationship with Cyclistic.

**2. Off-Season Retention Campaigns**

My analysis showed that Casual Riders virtually disappear during the Chicago winter (November–March).

**Action:** Launch a "Winter Warm-up" promotion targeting high-frequency casual riders from the previous season.

**The Logic:** Offer a trial month of annual membership at a significantly reduced rate during the shoulder months. By encouraging usage during the dead of winter, we can habituate casual riders to the service, potentially keeping them as active members once the warmer weather returns.

**3. Logistical Rebalancing Incentives**

I identified net-flow imbalances at key attractor nodes.

**Action:** Implement a "System Balancer" reward program.

**The Logic:** Leverage the casual rider base to assist in bike rebalancing. Offer loyalty points or small discounts toward an annual membership to casual riders who end their trips at high-demand stations where there is a net-departure deficit. This transforms a logistical challenge into a data-driven conversion tactic that rewards the user for helping the network's efficiency.

**Future Considerations & Data Enhancements**

While I believe this analysis provides a robust foundation for strategic planning, it is constrained by the data currently available. A more comprehensive dataset is thus required. To further refine these conversion strategies in the future, I recommend the following:

**The Inclusion of Weather Data:** Correlating ride trip data with daily local weather records (including temperature and precipitation) would allow us to model the exact weather threshold where casual usage drops, enabling more targeted, timely, and weather-triggered marketing notifications as well as incentive offerings.

**Survey-Based Qualitative Data:** Since we lack personally identifiable information, we cannot directly survey our riders. We recommend implementing an in-app, anonymized opt-in survey for all users at the end of their trips. This would allow us to capture data on intent (e.g., "Are you a tourist, local, or work commuter?") and perceived barriers to membership.

**Refinement of Spatial Binning:** As the network evolves, future analysis should move beyond basic spatial binning to incorporate station zone modeling. This would allow us to more precisely quantify how specific neighborhoods act as conversion gateways where Casual Riders eventually transition into Members.

**Conclusion**

The data confirms that the opportunity to grow Cyclistic lies in the behavioral divergence between the two user groups. Members provide the stable demand required for business sustainability, while Casual Riders represent the growth opportunity. By shifting our marketing focus from broad-spectrum outreach to the targeted, incentive-based strategies outlined above, Cyclistic can successfully transition a significant portion of our Casual Rider base into the more sustainable and profitable Member segment.


