# Google Data Analytics Capstone Project: Cyclistic Bike-Share

## Introduction
In this case study, I perform the work of a data analyst working for a fictional company, Cyclistic, which is a succesful bike-sharing company based in Chicago and which was started in 2016. They have a fleet of more than 5,000 bikes and 600 docking stations dotted around the city. With this large number of bikes and docking stations, users can take a bike from their starting station and return it to any other station in the network, making it incredibly easy and convenient.

Their offering is unique in that, apart from traditional two-wheel bikes, they also offer reclining bikes, hand tricycles, and cargo bikes, making bike-share more inclusive to people with disabilities and riders who can’t use a standard two-wheeled bike. The majority of riders opt for traditional bikes; about 8% of riders use the assistive options. Cyclistic users are more likely to ride for leisure, but about 30% use the bikes to commute to work each day.

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

In order to complete my data analysis, I made use of Cyclistic's historical trip data to analyze usage trends for the past twelve months which, in this case, was from June 2025 to May 2026. The original dataset can be found [here](https://divvy-tripdata.s3.amazonaws.com/index.html). Although Cyclistic is a fictional company, the data is real data for a company called Divvy Bikes that is based in Chicago. The data has kindly been made available to us for the purposes of these capstone projects by them under this [license](https://divvybikes.com/data-license-agreement).

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
Data privacy regulations prohibit the use of riders' personal information. It is thus impossible to create profiles for individual riders, this includes both casual riders and annual members. While it can almost safely be assumed that most annual members are local Chicago residents, the implication of the lack of personally identifiable information is that we can deduce even less about casual riders, and we cannot determine is they are local Chicago residents, tourists or daily commuters. We are also unable to track if an individual has purchased multiple single-ride passes over time. This makes is impossible to distinguish between frequent casual riders and one-time tourists.

2. **Missing Station and GPS Coordinate Data**  
A large portion of the dataset contains missing `start_station_name`, `end_station_name`, or GPS coordinate fields. This represents a significant portion of the dataset. While we do mostly have the GPS coordinates for these trips with missing station fields, the lack of station names makes it harder to analyze station-specific usage patterns without extra data-cleaning steps, such as mapping the available GPS coordinates back to known stations in the bike-share network.

3. **Negative/Zero Ride Lengths**  
There are several trips where the `ended_at` time is before ot equal to the `started_at` time. We can assume that these instances can be related to system maintenance, bike checks, or technical glitches in the docking sensor.

4. **Extremely Short/Long Rides**  
Trips that last for only a few seconds or that are extremely long (more than 24 hours) should be considered to be anomalies. Very short rides can be assumed to be "false starts", where a user immediately re-docks a bike due to a mechanical issue or a change of mind. Very long rides could be assumed to be a stolen bike or one that wasn't docked correctly.

5. **Limited Context**  
The dataset provides only "what happened" (the ride), but not "why it happened" (the motivation). We thus lack qualitative data about the rides and have to infer intent from behaviour, which is an assumption, not a fact. To be more specific, the data does not tell us why any of the bikes were used, nor do we have any contextual data such as weather conditions, demographics of the riders, etc.

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

However, before I could do that, I had to create a table inside of BigQuery and populate it with the first of the chunked CSV files in order to create the table schema so that the remaining 112 files could be imported correctly. As this first file was a single file of only about 10MB in size, it was easy to upload it using the 'Upload' option in the BiqQuery web interface. I then removed this CSV file from the directory containing the other CSV files as I did not want to duplicate its data in the table by re-importing it into the table during the batch upload using the 'bq' tool. Further to this, as each of the chunked files each contained a header, I had to insert the `skip_leading_rows=1` parameter into the command line prompt to remove the headers so that they would not be imported into the table I was creating. This was the command line prompt that I used to import the remaining 112 files into BigQuery:

```
for %f in (*.csv) do bq load --source_format=CSV --skip_leading_rows=1 --noreplace cyclistic_capstone_project.cyclistic_12_months_dataset "%f"
```

**Ingestion Verification**

After I performed the bulk ingestion of chunked CSV data into a single BigQuery relational table, I validated the result via this `COUNT(*)` query, confirming a total of 5,848,703 rows, representing an entire year of data:

```
SELECT
  COUNT(*)
FROM `course-493609.cyclistic_capstone_project.cyclistic_12_months_dataset`
```

I verified the data ingestion by using Excel to reconcile the total row count of the 12 CSV source files against the imported BigQuery table. The validation confirmed 5,848,703 records across both datasets, ensuring no data loss occurred during the upload from local storage to the cloud.

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

The query identified 35 duplicate `ride_id` entries which needed to be removed. To ensure data integrity, I created a cleaned version of the dataset by performing a `DISTINCT` selection, thus creating a new table:

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

The next cleaning step in the data came in the form of culling bike trips with durations that were of an impossible or impractical length to be considered valid. I decided to cull all trips with the following ride durations:
1. Ride duration < 0
2. Ride duration = 0
3. Ride duration < 1 minute
4. Ride duration > 18 hours

My reasonings behind the culling criteria are as follows:
1. Ride durations that are negative are logically impossible and could be the result of a data-recording issue, system maintenance, or a docking sensor issue.
2. Ride durations that have a duration of zero seconds have thus travelled zero distance and cannot be considered to be valid trips, regardless of the reason.
3. Ridge durations that are only a few seconds long could be due to system maintenance, a re-docking attempt, or a user undocking a bike and then changing their mind, but either way, they are too short to tell us anything useful about the trip. While these are technically 'trips', they don't tell us anything useful about usage behaviour as they are trips too short to be purposeful.
4. Ride durations that are longer than 18 hours in duration represent a physical impossibility for most riders, even if they stopped to have breaks along the way. Ride lengths exceeding 18 hours should thus be considered to be either multiple rides, bikes having being removed from their docking stations for maintenance, or possibly bikes that have been stolen.

In order to faciliate the culling process, and also to make the data more human-readable, I added the `ride_duration_mins` column to the dataset, which shows ride duration rounded off to the nearest minute. Being a calculated field, it was created by calculating the difference between the `started_at` and `ended_at` times, dividing by 60, and the rounding the result to the nearest integer. This is the code that I used to achieve this:

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

To validate the data cleaning process, and to check if these anomalies were skewing the results, I conducted a comparative statistical audit by calculating the mean, minimum and maximum ride durations before and after the removal of these anomalies using variations of this code:
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

*Checked for Missing Station Names and/or GPS Coordinates*

As stated earlier, my initial exploration of the monthly datasets revealed that a significant portion of the datasets contains missing `start_station_name` and `end_station_name` data, as well as missing GPS coordinate data for bike trips, so this had to be investigated in detail. First, I checked how many bike trip entries lack station names, so I ran this query in BigQuery to get the exact figures:

```
SELECT
  COUNTIF(start_station_name IS NULL) AS missing_start_station,
  COUNTIF(end_station_name IS NULL) AS missing_end_station,
  COUNTIF(start_station_name IS NULL AND end_station_name IS NULL) AS missing_both,
  COUNTIF(start_station_name IS NULL OR end_station_name IS NULL) AS missing_either
FROM `course-493609.cyclistic_capstone_project.cyclistic_clean_dataset`
```

The query showed the following:

| Anomaly | Number of Occurences |
| :--- | ---: |
| Missing `start_station_name` | 1,249,661 |
| Missing `end_station_name` | 1,314,320 |
| Missing `start_station_name` and `end_station_name` | 593,702 |
| Missing `start_station_name` or `end_station_name` | 1,970,279 |

As we are told in the project brief, Cyclistic's business model is station-based and uses fixed docking systems located at each station from which a user must remove a bike for use and then either return it to the same station or any other station in the network. It is thus a fair assumption that the primary source of 'location truth' for each bike trip should be the station names, as bikes need to be physically docked, or at the very least be present in the GPS-ring-fenced station locations, in order for station names to be recorded in the trip records, assuming that no technical glitch or data collection error occurred.

However, as our query revealed, 1,970,279 trips out of our current dataset of 5,726,059 trips are missing either the start or end station names, making these records incomplete and problematic to use for our analysis. These incomplete entries makes up a staggering 34.4% of our dataset and could not simply be removed without leading to catastrophic data losses that could severely skew the analysis, destroy statistical power, and introduce significant selection bias. A decision had to be made about how to handle these trips with missing station names.

A high frequency of missing station names could suggest a systemic data collection issue, or possible non-compliance with docking protocols, but the assumption could not be made that these trips with missing data were automatically invalid. To maintain the statistical integrity of the analysis, I opted to retain records with missing station names that possessed valid GPS coordinates, as valid GPS coordinates provide sufficient confirmation of point-to-point transit. This decision ensured the analysis remained representative of the entire network’s activity rather than only a subset of perfectly logged records.

Before I could perform coordinate-based imputation to populate the fields with missing station names, I needed to first devise a decision matrix to decide which records should be retained (some requiring imputation) and which should be removed:

| Record Status | Criteria | Action | Logic |
| :---     | :---     | :---   | :---  |
| Complete | Both `start_station_name` and `end_station_name` values exist for the record, although GPS coordinates may or may not be available. | Retain record. | Compliant with the station-based business model. GPS coordinate data not required. |
| Recoverable | Either `start_station_name` and/or `end_station_name` missing for the record, but GPS coordinates are available in lieu of the missing data and matches the coordinates of a known station within a radius of 100m. | Impute station name(s) and retain record. | Sufficient data exists to impute station name(s) from available GPS coordinate data. |
| Unrecoverable | Either `start_station_name` and/or `end_station_name` missing for the record, but insufficient GPS coordinate data available to impute station name matches to complete the record, or there is GPS coordinate data available, but it does not map to known station locations within a radius of 100m. | Remove record. | Insufficient data available to impute station name(s) to complete the record. |

For the purposes of my geospatial matching, a radius of 100m of known station coordinates was used as the spatial tolerance theshold to determine valid matches. This decision was made for the following reasons:
1. It is the optimal balance between precision (matching to the correct station) and recall (recovering as much data as possible).
2. A larger radius could not be used as there might be overlap between neigbouring stations.
3. A smaller radius was deemed not robust enough due to the phenomena of 'GPS drift', 'urban canyoning' and 'GPS sensor noise', which can result in GPS coordinates being recorded that are far from the true location of the bike.





**Avoiding Selection Bias**



The entire data cleaning process removed XXX records from the dataset, leaving us with a total of XXX records, this representing XX% of the original source data.



