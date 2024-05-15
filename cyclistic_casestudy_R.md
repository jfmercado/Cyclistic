<a href="https://www.kaggle.com/code/jfmercado/cyclistic-case-study?scriptVersionId=176274764" target="_blank"><img align="left" alt="Kaggle" title="Open in Kaggle" src="https://kaggle.com/static/images/open-in-kaggle.svg"></a>

Prepared by Joshua Mercado on 04/25/2024

# Introduction

This sample case study was provided through the Google Data Analytics Professional Certificate course. I will be analyzing real world data from the bikeshare company, Divvy, based in Chicago, using their 2023 ride data. The datasets are publically available [here](https://divvy-tripdata.s3.amazonaws.com/index.html) (The data has been made available by Motivate International Inc. under this [license](https://divvybikes.com/data-license-agreement)). With over 5 million total rows, I will be using R to handle the large dataset rather than a spreadsheet.

## Scenario

Cyclistic is a (fictional) bikeshare company in Chicago with two types of customers: **annual members** that pay an annual fee for unlimited rides throughout the year and **casual riders** who pay either per ride or per day. The Cyclistic marketing analytics team is looking to convert casual riders into annual members as the latter are deemed to be much more profitable and they believe that converting casual riders is a better strategy than targeting all-new customers.

My objective is to analyze the data to answer three key questions:

1. How do annual members and casual riders use Cyclistic bikes differently?
2. Why would casual riders buy Cyclistic annual memberships?
3. How can Cyclistic use digital media to influence casual riders to become members?

# Environment Set-Up


```python
# Load the packages
library(tidyverse)
library(janitor)
options(dplyr.summarise.inform = FALSE)
```

# 1. Collect the data

We will be using Cyclistic's trip data from January 2023 to Deember 2023.


```python
#Import the data
Jan2023 <- read_csv("/kaggle/input/divvy-trip-data-2023/202301-divvy-tripdata.csv")
Feb2023 <- read_csv("/kaggle/input/divvy-trip-data-2023/202302-divvy-tripdata.csv")
Mar2023 <- read_csv("/kaggle/input/divvy-trip-data-2023/202303-divvy-tripdata.csv")
Apr2023 <- read_csv("/kaggle/input/divvy-trip-data-2023/202304-divvy-tripdata.csv")
May2023 <- read_csv("/kaggle/input/divvy-trip-data-2023/202305-divvy-tripdata.csv")
Jun2023 <- read_csv("/kaggle/input/divvy-trip-data-2023/202306-divvy-tripdata.csv")
Jul2023 <- read_csv("/kaggle/input/divvy-trip-data-2023/202307-divvy-tripdata.csv")
Aug2023 <- read_csv("/kaggle/input/divvy-trip-data-2023/202308-divvy-tripdata.csv")
Sep2023 <- read_csv("/kaggle/input/divvy-trip-data-2023/202309-divvy-tripdata.csv")
Oct2023 <- read_csv("/kaggle/input/divvy-trip-data-2023/202310-divvy-tripdata.csv")
Nov2023 <- read_csv("/kaggle/input/divvy-trip-data-2023/202311-divvy-tripdata.csv")
Dec2023 <- read_csv("/kaggle/input/divvy-trip-data-2023/202312-divvy-tripdata.csv")
```

# 2. Wrangle the data and combine into a single file

We must check the structure of each dataframe to ensure consistency before merging


```python
str(Jan2023)
str(Feb2023)
str(Mar2023)
str(Apr2023)
str(May2023)
str(Jun2023)
str(Jul2023)
str(Aug2023)
str(Sep2023)
str(Oct2023)
str(Nov2023)
str(Dec2023)
```

#### Since all of the columns seem to match in name and datatype, now we merge the dataframes


```python
# Merge dataframes and print
print(uncleaned_cyclistic_tripdata_2023 <- bind_rows(Jan2023, Feb2023, Mar2023, Apr2023, May2023, Jun2023, Jul2023, Aug2023, Sep2023, Oct2023, Nov2023, Dec2023), n=13)
```

<div class="alert alert-block alert-info">
5,719,877 rows returned
</div>



# 3. Clean and transform data prior to analysis


```python
# Clean names to remove spaces, parenthesis, camelCase, etc.
cyclistic_tripdata_2023 <- clean_names(uncleaned_cyclistic_tripdata_2023)

# Remove duplicate records based on ride_id
cyclistic_tripdata_2023 <- distinct(cyclistic_tripdata_2023, ride_id, .keep_all=TRUE)

# Remove rides with any of the following fields missing: ride_id, start_lat, start_lng, end_lat, end_lng, member_casual
cyclistic_tripdata_2023 <- cyclistic_tripdata_2023[!(is.na(cyclistic_tripdata_2023$ride_id) | is.na(cyclistic_tripdata_2023$start_lat) | 
                                                       is.na(cyclistic_tripdata_2023$start_lng) | is.na(cyclistic_tripdata_2023$end_lat) | 
                                                       is.na(cyclistic_tripdata_2023$end_lng) | is.na(cyclistic_tripdata_2023$member_casual)),]

# N.B. Some records are missing station IDs/names because some bikes do not have to be returned to a specific dock and can be locked to any public bike rack. Those rides are still considered good data.

# Remove empty rows
drop_na(cyclistic_tripdata_2023)
```

<div class="alert alert-block alert-info">
6,990 rows removed, 5,712,887 rows remaining
</div>

#### Next, we will create a few columns that will allow for more granular analysis as well as deeper data cleaning


```python
# Add a ride_length by subtracting started_at column from ended_at column
cyclistic_tripdata_2023$ride_length <- difftime(cyclistic_tripdata_2023$ended_at, cyclistic_tripdata_2023$started_at)

# Add a day of the week column using the wday() function on the started_at column
cyclistic_tripdata_2023$day_of_week <- format(as.Date(cyclistic_tripdata_2023$started_at), "%a")

# Add a month column by extracting the month from the started_at column
cyclistic_tripdata_2023$month <- format(as.Date(cyclistic_tripdata_2023$started_at), "%B")

# Add an hour column by extracting the hour from the started_at column
cyclistic_tripdata_2023$hour <- format(as.POSIXct(cyclistic_tripdata_2023$started_at), "%H")
```

#### Using the newly calculated ride_length data, we can remove any rides lasting less than 0 seconds, which are obviously bad data, and any rides lasting longer than 24 hours, which are assumed to have been stolen or lost


```python
# Include only rides that are greater than 0 seconds and less than 24 hours
cyclistic_tripdata_2023 <- cyclistic_tripdata_2023[!(cyclistic_tripdata_2023$ride_length <= 0 | cyclistic_tripdata_2023$ride_length > 86400),]
```

<div class="alert alert-block alert-info">
1,488 rows removed, 5,711,399 rows remaining
</div>

# 4. Descriptive Analysis

#### Now that our data is sufficiently cleaned and prepared for analysis, we can compare the mean, median, max, and min of the rides by customer type (all units are seconds)


```python
# Calculate the mean, median, max, and min rides by customer type
aggregate(cyclistic_tripdata_2023$ride_length ~ cyclistic_tripdata_2023$member_casual, FUN = mean)
aggregate(cyclistic_tripdata_2023$ride_length ~ cyclistic_tripdata_2023$member_casual, FUN = median)
aggregate(cyclistic_tripdata_2023$ride_length ~ cyclistic_tripdata_2023$member_casual, FUN = max)
aggregate(cyclistic_tripdata_2023$ride_length ~ cyclistic_tripdata_2023$member_casual, FUN = min)
```

#### We can see that casual riders go for longer rides on average than annual members. Nearly twice as long.We can also confirm that all rides fall between 0 seconds and 24 hours.

#### Next, we'll dive deeper into that data by comparing the number of rides as well as the average length of rides taken by customer type AND day of the week.


```python
# Aggregate the average length and number of rides by the day of the week as well as by the customer type and then sort by weekday
cyclistic_tripdata_2023 %>% 
  mutate(weekday = wday(started_at, label = TRUE)) %>%
  group_by(member_casual, weekday) %>%
  summarise(number_of_rides = n()
            ,average_duration = mean(ride_length), .groups = NULL) %>%
  arrange(weekday)
```

#### It looks like casual riders tend to ride more often on weekends, while annual members tend to see ride usage spike in the middle of the week. Let's visualize that same data using ggplot.


```python
# Number of rides visualization
cyclistic_tripdata_2023 %>% 
  mutate(weekday = wday(started_at, label = TRUE)) %>% 
  group_by(member_casual, weekday) %>% 
  summarise(number_of_rides = n()
            ,average_duration = mean(as.numeric(ride_length))) %>% 
  arrange(member_casual, weekday)  %>% 
  ggplot(aes(x = weekday, y = number_of_rides, fill = member_casual)) +
  geom_col(position = "dodge")

# Average ride length visualization
cyclistic_tripdata_2023 %>% 
  mutate(weekday = wday(started_at, label = TRUE)) %>% 
  group_by(member_casual, weekday) %>% 
  summarise(number_of_rides = n()
            ,average_duration = mean(as.numeric(ride_length))) %>% 
  arrange(member_casual, weekday)  %>% 
  ggplot(aes(x = weekday, y = average_duration, fill = member_casual)) +
  geom_col(position = "dodge")
```

### From the graph, we can make several more observations:

It seems that **casual rider** usage steadily increases from Monday until the weekend, suggesting that *casual rides are not generally being used for commuting purposes*. At the same time, **annual member** usage spikes in the middle of the week with weekends being the least utilized days, suggesting *strong commuter usage of bikes by annual members*.

We can also see that **annual members** spend a fairly similar amount of time per ride on average regardless of the day, with slight spikes on the weekends. **Casual riders** have sharp spikes in average ride duration on the weekend. Even on their lowest average ride duration day (Wednesday), casual riders still spend significantly more average time on each ride than annual members, further suggesting that *casual riders are using the bikes for leisure rides as opposed to annual members, who could be using the bikes for purposeful A to B trips like commutes.*

# 5. Export data for further visualization analysis

We could generate even more visualizations using ggplot, but it would be much more efficient to output the dataframe for analysis using a more powerful viz tool like Tableau.


```python
#Export to CSV
write.csv(cyclistic_tripdata_2023, file = 'cyclistic_tripdata_2023.csv')
```

# 6. Tableau Visualizations

Now that we've migrated our data from RStudio to Tableau, we can create cleaner and more descriptive visualizations. I've compiled all of the above viz data into an interactive [dashboard](https://public.tableau.com/app/profile/joshua.mercado2815/viz/CyclisticCaseStudy_17138915127790/CyclisticBikeshareDatabyCustomerType2023) that allows users to highlight the data by customer type:

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

# 6. Conclusion

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

