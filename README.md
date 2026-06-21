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

The data has been provided in the form of CSV files, one file for each month, and with the naming structure of YYYYMM-divvy-tripdata. My combined dataset has thus been composed from 12 of these files in consecutive date order. Each row in these CSV files represents a single bike trip, with each of the trips having been anonymized and identified with a unique `ride_id` value. The primary key for each record is thus `ride_id`. The table below gives a description of the 13 column names, their data types, and a description of each column contained in the original CSV files:

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

While the dataset provided by Divvy Bikes is overall comprehensive and reliable, it is subject to a few limitations that must be taken into account when performing an analysis on the data. After a brief initial exploratory data analysis using Excel, these are the key observations that I have made:

1. **Lack of Personally Identifiable Information**  
Data privacy regulations prohibit the use of riders' personal information. It is thus impossible to create profiles for individual riders, this includes both casual riders and annual members. While it can almost safely be assumed that most annual members are local Chicago residents, the implication of the lack of personally identifiable information is that we can deduce even less about casual riders, and we cannot determine is they are local Chicago residents, tourists or daily commuters. We are also unable to track if an individual has purchased multiple single-ride passes over time. This makes is impossible to distinguish between frequent casual riders and one-time tourists.

2. **Missing Station and GPS Coordinate Data**  
Approximately 30% of the dataset contains missing `start_station_name`, `end_station_name` or GPS coordinate fields. This represents a significant portion of the dataset. However, due to the very large size of the dataset, we still have sufficient complete data to work with. While we do mostly have the GPS coordinates for these trips with missing station fields, the lack of station names makes it harder to analyze station-specific usage patterns without extra data-cleaning steps, such as mapping the available GPS coordinates back to known stations in the bike-share network.

    There are, however, a limitations when it comes to trying to map GPS coordinates to the known stations. There is the phenomenon of 'GPS drift'. This occurs when a GPS sensor records a location that deviates from its true physical position. This often happens due to environmental noise such as signal blockage caused by trees, building structures or bad weather, but can also be caused by tall buildings reflecting the satellite signals, making it look like a bike is a block away from where it actually is. In a built-up city such as Chicago, these effects are even more pronounced. The end result of GPS drift is that bikes can be incorrectly mapped to the wrong station (such as a nearby neighbouring station), or no station at all, depending on how severe the GPS drift was. In the latter case, this would introduce ghost trips, distorting our analysis completely.

4. **Negative/Zero Ride Lengths**  
There are several trips where the `ended_at` time is before ot equal to the `started_at` time. We can assume that these instances can be related to system maintenance, bike checks, or technical glitches in the docking sensor.

5. **Extremely Short/Long Rides**  
Trips that last for only a few seconds or that are extremely long (more than 24 hours) should be considered to be anomalies. Very short rides can be assumed to be "false starts", where a user immediately re-docks a bike due to a mechanical issue or a change of mind. Very long rides could be assumed to be a stolen bike or one that wasn't docked correctly.

6. **Limited Context**  
The dataset provides only "what happened" (the ride), but not "why it happened" (the motivation). We thus lack qualitative data about the rides and have to infer intent from behaviour, which is an assumption, not a fact.

### Process

