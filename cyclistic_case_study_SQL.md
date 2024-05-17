<a href="https://www.kaggle.com/code/jfmercado/cyclistic-case-study?scriptVersionId=176274764" target="_blank"><img align="left" alt="Kaggle" title="Open in Kaggle" src="https://kaggle.com/static/images/open-in-kaggle.svg"></a>

Prepared by Joshua Mercado on 04/25/2024

The following report utilizes SQL (BigQuery). Another version of the case study utilizing R for the data cleaning is also available under cyclistic_casestudy_R.md.

# Introduction

This sample case study was provided through the Google Data Analytics Professional Certificate course. I will be analyzing real world data from the bikeshare company, Divvy, based in Chicago, using their 2023 ride data. The datasets are publically available [here](https://divvy-tripdata.s3.amazonaws.com/index.html) (The data has been made available by Motivate International Inc. under this [license](https://divvybikes.com/data-license-agreement)). With over 5 million total rows, I will be using R to handle the large dataset rather than a spreadsheet.

## Scenario

Cyclistic is a (fictional) bikeshare company in Chicago with two types of customers: **annual members** that pay an annual fee for unlimited rides throughout the year and **casual riders** who pay either per ride or per day. The Cyclistic marketing analytics team is looking to convert casual riders into annual members as the latter are deemed to be much more profitable and they believe that converting casual riders is a better strategy than targeting all-new customers.

My objective is to analyze the data to answer three key questions:

1. How do annual members and casual riders use Cyclistic bikes differently?
2. Why would casual riders buy Cyclistic annual memberships?
3. How can Cyclistic use digital media to influence casual riders to become members?

# 1. Collect the data

We will be using Cyclistic's trip data from January 2023 to December 2023. Using Google BigQuery, the project name and dataset prefix for tables is "cyclistic-case-study-422918.divvy_2023".

Upload each month of trip data as its own table using an easier naming convention.

jan2023 <- 202301-divvy-tripdata.csv
feb2023 <- 202302-divvy-tripdata.csv
mar2023 <- 202303-divvy-tripdata.csv
apr2023 <- 202304-divvy-tripdata.csv
may2023 <- 202305-divvy-tripdata.csv
jun2023 <- 202306-divvy-tripdata.csv
jul2023 <- 202307-divvy-tripdata.csv
aug2023 <- 202308-divvy-tripdata.csv
sep2023 <- 202309-divvy-tripdata.csv
oct2023 <- 202310-divvy-tripdata.csv
nov2023 <- 202311-divvy-tripdata.csv
dec2023 <- 202312-divvy-tripdata.csv

# 2. Wrangle the data and combine into a single file

First, we must check the schema of each table to ensure consistency before merging

![Screen Shot 2024-05-15 at 1 43 07 PM](https://github.com/jfmercado/Cyclistic/assets/25253461/0c2e28ec-4856-4dc8-83b7-5f9a972ec281)

#### Since all of the tables have the same number of columns as well as matching names and datatypes, we can now merge the tables using the UNION ALL operator into a single table

The UNION ALL operator allows us to merge the columns rather than creating new columns for each added table. We will create a new table called tripdata_2023 which will contain all of the merged tables.

```
DROP TABLE IF EXISTS `cyclistic-case-study-422918.divvy_2023.tripdata_2023`;

CREATE TABLE IF NOT EXISTS`cyclistic-case-study-422918.divvy_2023.tripdata_2023` AS
  SELECT * FROM (
    SELECT * FROM `cyclistic-case-study-422918.divvy_2023.jan2023`
    UNION ALL
    SELECT * FROM `cyclistic-case-study-422918.divvy_2023.feb2023`
    UNION ALL
    SELECT * FROM `cyclistic-case-study-422918.divvy_2023.mar2023`
    UNION ALL
    SELECT * FROM `cyclistic-case-study-422918.divvy_2023.apr2023`
    UNION ALL
    SELECT * FROM `cyclistic-case-study-422918.divvy_2023.may2023`
    UNION ALL
    SELECT * FROM `cyclistic-case-study-422918.divvy_2023.jun2023`
    UNION ALL
    SELECT * FROM `cyclistic-case-study-422918.divvy_2023.jul2023`
    UNION ALL
    SELECT * FROM `cyclistic-case-study-422918.divvy_2023.aug2023`
    UNION ALL
    SELECT * FROM `cyclistic-case-study-422918.divvy_2023.sep2023`
    UNION ALL
    SELECT * FROM `cyclistic-case-study-422918.divvy_2023.oct2023`
    UNION ALL
    SELECT * FROM `cyclistic-case-study-422918.divvy_2023.nov2023`
    UNION ALL
    SELECT * FROM `cyclistic-case-study-422918.divvy_2023.dec2023`
);

SELECT COUNT(*)
  FROM `cyclistic-case-study-422918.divvy_2023.tripdata_2023`;
```

<div class="alert alert-block alert-info">
5,719,877 rows returned
</div>

# 3. Explore data

```
# Check for duplicate records based on ride_id
SELECT COUNT(ride_id) - COUNT(DISTINCT ride_id) AS duplicate_rows
FROM `cyclistic-case-study-422918.divvy_2023.tripdata_2023`;
```
<div class="alert alert-block alert-info">
duplicate_rows: 0
</div>

```
# Check for rides with any of the following fields missing: ride_id, start_lat, start_lng, end_lat, end_lng, member_casual
SELECT COUNT(*) - COUNT(ride_id) ride_id,
 COUNT(*) - COUNT(started_at) started_at,
 COUNT(*) - COUNT(ended_at) ended_at,
 COUNT(*) - COUNT(start_lat) start_lat,
 COUNT(*) - COUNT(start_lng) start_lng,
 COUNT(*) - COUNT(end_lat) end_lat,
 COUNT(*) - COUNT(end_lng) end_lng,
 COUNT(*) - COUNT(member_casual) member_casual
FROM `cyclistic-case-study-422918.divvy_2023.tripdata_2023`;

# N.B. Some records are missing station IDs/names because some bikes do not have to be returned to a specific dock and can be locked to any public bike rack. Those rides are still considered good data.
```
|ride_id|started_at|ended_at|start_lat|start_lng|end_lat|end_lng|member_casual|
|-|-|-|-|-|-|-|-|
|0|0|0|0|0|6990|6,990|0|

```
# Check for rides that are longer than 24 hours, which are assumed to have been stolen or lost
SELECT COUNT(*) AS longer_than_a_day
FROM `cyclistic-case-study-422918.divvy_2023.tripdata_2023`
WHERE (TIMESTAMP_DIFF(ended_at, started_at, SECOND)) > 86400;
```
<div class="alert alert-block alert-info">
longer_than_a_day: 6,418
</div>

```
# Check for rides that are shorter than 1 second, which are obviously bad data
SELECT COUNT(*) AS less_than_a_second
FROM `cyclistic-case-study-422918.divvy_2023.tripdata_2023`
WHERE (TIMESTAMP_DIFF(ended_at, started_at, SECOND)) < 1;
```
<div class="alert alert-block alert-info">
less_than_a_second: 1,269
</div>

### So after exploring our data a bit, we have some cleaning to do. There are no duplicate ride IDs which is great, but there are 6990 rides without ending GPS information, which we will assume are bad data. There are also 6,418 rides that lasted longer than 24 hours and 1,269 rides that were less than a second long, so we must eliminate those rides as well. Some of the bad ride_length rides might overlap with the rides that have bad ending GPS information.

# 4. Clean and transform data

We will create a new table for the cleaned table since we don't want to use destructive commands on the original dataset. We will also create a few columns that will allow for more granular analysis as well as deeper data cleaning.

```
DROP TABLE IF EXISTS `cyclistic-case-study-422918.divvy_2023.cleaned_tripdata_2023`;

CREATE TABLE IF NOT EXISTS `cyclistic-case-study-422918.divvy_2023.cleaned_tripdata_2023` AS (
  SELECT 
    a.ride_id, rideable_type, started_at, ended_at, start_station_name, 
    end_station_name, start_lat, start_lng, end_lat, end_lng, member_casual, ride_length,
    EXTRACT(DAYOFWEEK FROM started_at) AS day_of_week, -- create a day_of_week column
    EXTRACT(MONTH FROM started_at) AS month, -- create a month column 
    EXTRACT(HOUR FROM started_at) AS hour -- create an hour column
  FROM `cyclistic-case-study-422918.divvy_2023.tripdata_2023` a
  JOIN ( -- create a ride_length column
    SELECT ride_id, (
      TIMESTAMP_DIFF(ended_at, started_at, SECOND)) AS ride_length,
    FROM `cyclistic-case-study-422918.divvy_2023.tripdata_2023`
  ) b 
  ON a.ride_id = b.ride_id
  WHERE 
    a.ride_id IS NOT NULL AND -- filter out rides missing crucial data
    start_lat IS NOT NULL AND
    start_lng IS NOT NULL AND
    end_lat IS NOT NULL AND
    end_lng IS NOT NULL AND
    member_casual IS NOT NULL AND
    ride_length > 0 AND ride_length < 86400 -- filter out rides shorter than 1 second and longer than 24 hours
);

SELECT COUNT(*) AS total_rides
FROM `cyclistic-case-study-422918.divvy_2023.cleaned_tripdata_2023`;
```
<div class="alert alert-block alert-info">
total_rides: 5,711,399
8,478 rows removed
</div>

# 4. Analyze data

#### Now that our data is sufficiently cleaned and prepared for analysis, we can compare the total, mean, max, and min of the rides by customer type (all units are seconds)

```
SELECT member_casual,
  COUNT(ride_id) AS total_rides,
  ROUND(AVG(ride_length)) AS average_ride_length,
  MAX(ride_length) AS max_ride_length,
  MIN(ride_length) AS min_ride_length
FROM `cyclistic-case-study-422918.divvy_2023.cleaned_tripdata_2023`
GROUP BY member_casual
ORDER BY member_casual;
```
|member_casual|total_rides|average_ride_length|max_ride_length|min_ride_length|
|-|-|-|-|-|
|casual|2052587|1232.0|86261|1|
|member|3658812|723.0|86392|1|

#### We can see that casual riders go for longer rides on average than annual members. Nearly twice as long.We can also confirm that all rides fall between 0 seconds and 24 hours.

#### Next, we'll dive deeper into that data by comparing the number of rides as well as the average length of rides taken by customer type AND day of the week.

```
# Aggregate the average length and number of rides by the day of the week as well as by the customer type and then sort by weekday
SELECT day_of_week, member_casual, COUNT(ride_id) AS total_trips, ROUND(AVG(ride_length)) AS average_ride_length
FROM `cyclistic-case-study-422918.divvy_2023.cleaned_tripdata_2023`
GROUP BY day_of_week, member_casual
ORDER BY day_of_week, member_casual;
```
|day_of_week|member_casual|total_trips|average_ride_length|
|-|-|-|-|
|1|casual|334393|1434.0|
|1|member|408612|806.0|
|2|casual|234118|1213.0|
|2|member|494328|687.0|
|3|casual|245529|1102.0|
|3|member|576459|695.0|
|4|casual|248479|1055.0|
|4|member|586189|690.0|
|5|casual|269828|1075.0|
|5|member|589309|694.0|
|6|casual|310965|1195.0|
|6|member|531327|719.0|
|7|casual|409275|1393.0|
|7|member|472588|804.0|

#### It looks like casual riders tend to ride more often on weekends, while annual members tend to see ride usage spike in the middle of the week. Diving deeper into the data, we can make several more observations:

It seems that **casual rider** usage steadily increases from Monday until the weekend, suggesting that *casual rides are not generally being used for commuting purposes*. At the same time, **annual member** usage spikes in the middle of the week with weekends being the least utilized days, suggesting *strong commuter usage of bikes by annual members*.

We can also see that **annual members** spend a fairly similar amount of time per ride on average regardless of the day, with slight spikes on the weekends. **Casual riders** have sharp spikes in average ride duration on the weekend. Even on their lowest average ride duration day (Wednesday), casual riders still spend significantly more average time on each ride than annual members, further suggesting that *casual riders are using the bikes for leisure rides as opposed to annual members, who could be using the bikes for purposeful A to B trips like commutes.*

#### We could continue querying for more insights, but without specific directives, it might be a better use of our time to switch over to a powerful visualization tool like Tableau for general data exploration.

# 5. Tableau Visualizations

Now that we've migrated our data to Tableau, we can create cleaner and more descriptive visualizations. I've compiled all of the above viz data into an interactive [dashboard](https://public.tableau.com/app/profile/joshua.mercado2815/viz/CyclisticCaseStudy_17138915127790/CyclisticBikeshareDatabyCustomerType2023) that allows users to highlight the data by customer type:

<div class='tableauPlaceholder' id='viz1715283208917' style='position: relative'><noscript><a href='#'><img alt='Cyclistic Bikeshare Data by Customer Type 2023 ' src='https:&#47;&#47;public.tableau.com&#47;static&#47;images&#47;Cy&#47;CyclisticCaseStudy_17138915127790&#47;CyclisticBikeshareDatabyCustomerType2023&#47;1_rss.png' style='border: none' /></a></noscript><object class='tableauViz'  style='display:none;'><param name='host_url' value='https%3A%2F%2Fpublic.tableau.com%2F' /> <param name='embed_code_version' value='3' /> <param name='site_root' value='' /><param name='name' value='CyclisticCaseStudy_17138915127790&#47;CyclisticBikeshareDatabyCustomerType2023' /><param name='tabs' value='no' /><param name='toolbar' value='yes' /><param name='static_image' value='https:&#47;&#47;public.tableau.com&#47;static&#47;images&#47;Cy&#47;CyclisticCaseStudy_17138915127790&#47;CyclisticBikeshareDatabyCustomerType2023&#47;1.png' /> <param name='animate_transition' value='yes' /><param name='display_static_image' value='yes' /><param name='display_spinner' value='yes' /><param name='display_overlay' value='yes' /><param name='display_count' value='yes' /><param name='language' value='en-US' /><param name='filter' value='publish=yes' /></object></div>                <script type='text/javascript'>                    var divElement = document.getElementById('viz1715283208917');                    var vizElement = divElement.getElementsByTagName('object')[0];                    if ( divElement.offsetWidth > 800 ) { vizElement.style.width='800px';vizElement.style.height='827px';} else if ( divElement.offsetWidth > 500 ) { vizElement.style.width='800px';vizElement.style.height='827px';} else { vizElement.style.width='100%';vizElement.style.height='1827px';}                     var scriptElement = document.createElement('script');                    scriptElement.src = 'https://public.tableau.com/javascripts/api/viz_v1.js';                    vizElement.parentNode.insertBefore(scriptElement, vizElement);                </script>

## Key Metrics: Total Rides, Average Ride Length, and Total Ride Length

This single viz reiterates the previous observations that **annual members** ride nearly *twice as often* as **casual riders**, but spend *nearly half the time per ride on average*. These two metrics combined show that the total time spent on bikes is *nearly equal between the two customer types*.

## Popular [Docking] Stations

While the map is slightly cluttered due to the density of docking stations, we can still see a clear trend of **casual riders** starting their trips closer to the downtown/shoreline area, further suggesting that casual rides are generally for leisure/sightseeing. We do have the ability to zoom in and see that **Annual members** have some very dense clusters near universities and hospitals. We can infer that *students and hospital workers would serve as a great demographic to target for conversion to annual membership.*

## Daily Rides and Average Daily Ride Length

These are the same visualizations that we created in RStudio earlier, recreated here for a complete picture.

## Hourly Rides and Hourly Ride Lengths on **weekends**

The hourly data from the weekends show that **annual members** spend pretty much the same amount of time on their rides no matter the time of day, while **casual riders** extend their ride time as the morning progresses, peaking in the early afternoon, and then gradually declining for the rest of the day. **Annual members** are taking "inelastic" rides, meaning that *their ride length is not governed by the time of day*. We can infer that this is because *their rides are objective based rather than for leisure*, as is shown in the **casual riders'** "elastic" ride length curve. 

## Hourly Rides Lengths **weekdays**:

The hourly data from weekdays shows extremely clear peaks during the traditional "rush hour" periods, which all but confirms *a significant percentage of annual members draw their value from being able to use Cyclistic bikes for commuting purposes*.

# 7. Conclusion

### As a reminder, the business task was to answer three specific questions:

1. How do annual members and casual riders use Cyclistic bikes differently?
2. Why would casual riders buy Cyclistic annual memberships?
3. How can Cyclistic use digital media to influence casual riders to become members?

**How do annual members and casual riders use Cyclistic bikes differently?**

|Casual Riders|Annual Members|
|------|------|
|Ride longer on average|Annual members ride more often|
|Bike usage spikes in the afternoon, regardless of day|Bike usage spikes during weekday rush hour commutes|
|Tend to start their rides near downtown or shoreside locations|Tend to start their rides near medical centers and universities|
|Average ride lengths are highly determined by the time of day|Average ride lengths are fairly static regardless of the time of day|

**Why would casual riders buy Cyclistic annual memberships?**

The easier question to ask is why wouldn't a casual rider buy a Cyclistic annual membership? The simple answer is value. We can infer that many casual rides are taken by tourists or infrequent sightseers, and are therefore very unlikely to glean value from an annual membership. The casual riders that would make up the target segment for annual memberships are customers whose behavior mimics all or even part of the behavior of annual members: purpose driven, frequent rides between regular docking stations, including frequent rides during peak commuting times. Without individual user data, it is hard for me to identify that customer segment, but if I were working for the real life company with access to that data, then it would be a simple matter of isolating the top percentage of casual riders that match any aspect of that behavior.

**How can Cyclistic use digital media to influence casual riders to become members?**

Digital media is an incredibly powerful marketing tool when combined with high level, data driven insights like the ones that we've gleaned above.
* A monthly and/or annual "wrapped" statistics summary could prove useful to point out to casual riders how their riding habits could benefit from an annual membership. This could also incentivize less frequent riders to try and augment their riding numbers when contextualized by carbon offsets or "adventure" styled marketing (e.g. "You rode X miles last month! That's like biking the length of X"). Top casual riders could also be presented with an opportunity for a discounted first year membership, thereby gamifying rides.
* Digital advertisements or even discount codes through tactical partnerships with hospitals and universities would effectively target a population that has demonstrated its outsized use of Cyclistic bikes with the goal of Cyclistic bikes being the de facto mode of transportation for patrons of those institutions.
* Not necessarily a digital media form of marketing, but advertisements targeting non-cycling commuting avenues can present Cyclistic as a more fun, possibly more eco friendly option for commuting. This could include high traffic streets, congested subway stations, and popular walking paths.

### Thanks for taking the time to read all of this! If you've made it this far, you might be:
* considering me for a job position (thank you again for your time and consideration)
* my mom, incredibly proud of her son's work (whether it be macaroni art or data analysis)
* super interested in Chicago bikeshare data and/or data cleaning! (cool!)

