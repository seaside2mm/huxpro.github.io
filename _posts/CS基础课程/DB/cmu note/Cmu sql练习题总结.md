# Cmu sql练习题总结

# 2017 

Q1 [5 points]:** Count the number of attorneys in the system. The output should look like this (only a single number):

```sql
--- Count the number of attorneys

select count(distinct(name)) 
from attorneys
;
```

Details:

 Ensure that there are no duplicates.

**Q2 [5 points]:** Repeated phone calls can sometimes land you in jail. Count the number of charges related to phone calls by examining their description. The output should look like this:

```sql
--- Count the number of cases related to repeated phone calls.

select count(case_id)
from charges
where description like '%PHONE%' ;
```

Details:

 The search string for the appropriate column is  %PHONE%

**Q3 [5 points]:** Reckless endangerment is another reason why one can end up in a courthouse. Count the number of cases related to reckless endangerment in each county. The output should look like this:

```
County A|500
County B|400
County C|300
```

```sql
--- Count the number of cases related to reckless endangerment in each county: 
--- Print county name and number of cases
--- Sort by number of cases (descending),
--- and break ties by county name (ascending),
--- and report only the top 3 counties.

select cases.violation_county, count(cases.case_id) as cnt 
from cases, charges
where cases.case_id = charges.case_id
and cases.violation_county <> ''
and charges.description like '%RECKLESS%' group by cases.violation_county
order by cnt desc, cases.violation_county asc limit 3
;
```

Details:

 Print the county name and number of cases in that particular county. Sort the counties by the number of cases in descending order, and break ties by ordering them in ascending order with respect to the county name. Report only the top 3 counties with the maximum number of cases. The search string for the appropriate column is %RECKLESS%. Ensure that you only fetch the cases whose county name is not empty.

**Q4 [5 points]:** Let's now go back in time and look at the cases filed in the 1950s. The output should look like this:

```
CASE1|1950-03-12
CASE2|1951-01-01
CASE3|1952-01-01
```

```sql
--- Cases filed in the 1950s

select case_id, filing_date
from cases
where filing_date >= '1950-01-01' and filing_date < '1960-01-01' 
order by filing_date
limit 3
;
```

Details:

 Print the case id and filing date for the cases filed in the 1950s. List the oldest cases first, and report only the earliest 3 cases.

**Q5 [10 points]:** It looks like a lot of cases got filed in the 1950s. In which decades did the most number of cases get filed? The output should look like this:

```
10000|1970s
5000|2000s
1000|1980s
```

```sql
--- In which 3 decades did the most number of cases get filed?

select count(case_id) as case_count, 
substr(filing_date, 1, 3) || '0s' as decade from cases
where filing_date <> ''
group by decade
order by case_count desc 
limit 3
;
```

Details:

 Print the number of cases and the relevant decade. We will print the relevant decade in a fancier format by constructing a string that looks like this 1970s. Sort the decades in decreasing order with respect to the number of cases. Report only the top 3 decades wherein the most number of cases got filed. Ensure that you only fetch the cases whose filing date is not empty.

**Q6 [5 points]:** Cases can often be closed for bizzare reasons. One such reason is statistics. Determine the percentage of cases that were statistically closed. The output should look like this:

```
5.123456789
```

```sql
--- Fraction of cases closed statistically

select statistically_closed_cases.cnt * 100.0 / all_cases.cnt as percentage from
(
select count(case_id) as cnt
from cases
where status = 'Case Closed Statistically' )
as statistically_closed_cases,
(
select count(case_id) as cnt
from cases
)
as all_cases
;
```

Details:

 Print the percentage of cases. To compute the percentage, you will need to multiply the numerator of the fraction by 100.0. While searching the case's status, use the following string: Case Closed Statistically. To keep things simple, there is no need to truncate or round numbers. To compute the percentage, simply multiply the numerator by 100.0 and divide by the appropriate denominator.

**Q7 [5 points]:** Let's look at some prolific defendants. List the top 3 parties who have been charged in the most number of distinct counties. The output should look like this:

```
A|100
B|50
C|10
```

```sql
--- List the top 3 parties who have been
--- charged in the most number of distinct counties.

select parties.name, count(distinct(cases.violation_county)) as cnt 
from cases, parties
where cases.case_id = parties.case_id
and parties.type = 'Defendant'
and parties.name <> '' group by parties.name order by cnt desc
limit 3
;
```



Details:

 Print the name of the party along with the number of distinct counties. Sort the parties by the number of distinct counties in descending order, and report only the top   3  parties. Ensure that you only fetch the parties who are defendants (Defendant) and whose name is not empty.

**Q8 [20 points]:** How does the average age of guilty criminals vary over time? The output should look like this:

```
2017|50.123456789
2016|55.123456789
2015|60.123456789
2014|55.123456789
2013|50.123456789
```

```sql
--- Average age of guilty criminals over time

with filing_year_table as
(
select cases.case_id, strftime('%Y', filing_date) as filing_year
from cases, charges
where cases.case_id = charges.case_id
and charges.disposition = 'Guilty'
and cases.filing_date <> ''
),
age_table as
(
select cases.case_id,
strftime('%Y.%m%d', cases.filing_date) - strftime('%Y.%m%d', parties.dob) as age from cases, parties
where cases.case_id = parties.case_id
and parties.type = 'Defendant'
and parties.name <> ''
and parties.dob is not null
and age > 0
and age < 100
)
select filing_year_table.filing_year, avg(age_table.age) as average_age
from cases, filing_year_table, age_table
where cases.case_id = filing_year_table.case_id
and cases.case_id = age_table.case_id
group by filing_year_table.filing_year
order by filing_year desc
limit 5
;
```

Details:

 Print the filing year and the average age of the criminals that were found guilty in cases filed in that particular year. To compute the average age, first determine the age of the criminal by using the case's filing date and the party's date of birth. Use the strftime('%Y.%m%d',...) function for this purpose. To determine the filing year from the filing date, again make use of the strftime('%Y',...)function. Look at the disposition to pick only Guilty parties (charges.disposition = 'Guilty'). Ensure that you only fetch the cases whose filing date is not empty. Also, ensure that you only fetch the parties who are defendants (parties.type = Defendant), whose name is not empty, whose date of birth is not empty, and whose computed age is greater than 0 and less than 100 years. List the tuples in descending order with respect to the filing year and only display 5 tuples. You might want to leverage common table expressions (CTEs) in this query. Here's more information on using CTEs in SQLite

**Q9 [15 points]:** Let's next look at case disposition by race to see if there is any inherent bias in the system. The output should look like this:

```
African American|Guilty|60.000
African American|Not Guilty|40.000
Caucasian|Guilty|50.000
Caucasian|Not Guilty|50.000
```

```sql
--- Disposition by race

with disposition_table as
(
select race, charges.disposition, parties.case_id as case_id
from charges, parties
where charges.case_id = parties.case_id
and race <> ''
and disposition in ('Guilty', 'Not Guilty')
and race in ('African American', 'Caucasian')
),
disposition_aggregate_table as
(
select disposition, count(case_id) as case_count
from disposition_table
group by disposition
),
race_aggregate_table as
(
select race, disposition, count(case_id) as case_count
from disposition_table
group by race, disposition
)
select race, race_aggregate_table.disposition,
(race_aggregate_table.case_count * 100.0) / disposition_aggregate_table.case_count 
from race_aggregate_table, disposition_aggregate_table
where race_aggregate_table.disposition = disposition_aggregate_table.disposition
;
```

Details:

 Print the race, the case disposition, and the percentage of cases disposed with that verdict. Let's restrict our focus to 2 races (African American, Caucasian), and 2 types of case disposition (Guilty, Not Guilty). To compute the percentage, you will need to multiply the numerator of the fraction by 100.0. Ensure that the race of the party is not empty. You might want to leverage common table expressions (CTEs) in this query. Here's more information on using CTEs in SQLite.

**Q10 [5 points]:** Certain zip codes might have -- ahem -- more interesting citizens than Squirrel Hill. Retrieve the top 3 zip codes in Maryland where the most number of cases were filed. The output should look like this:

```
21000|500
21001|400
21002|300
```

```sql
--- Top zip codes with most criminals

select parties.zip, count(parties.case_id) as case_count 
from parties, charges
where parties.case_id = charges.case_id
and parties.zip <> ''
group by parties.zip
order by case_count desc 
limit 3
;
```

Details:

 Print the zip code along with the number of cases filed in that particular zip code. List them in decreasing order with respect to the number of cases. Display only the top 3 zip codes. Ensure that the zip code is not empty.



**Q11 [15 points]:** Some attorneys are awesome at their job. List the top 5 attorneys in Maryland by examining the number of cases that an attorney handles and the percentage of cases wherein they were successful. The output should look like this:

```
A|100|60.123
B|200|45.123
C|100|30.123
D|500|20.123
E|100|10.123
```

```sql
--- Attorney performance

with attorney_table as
(
select charges.case_id as case_id, disposition, name from charges, attorneys
where charges.case_id = attorneys.case_id
and name <> ''
),
aggregate_table as
(
select count(case_id) as total_count, name
from attorney_table
group by name
),
success_table as
(
select count(case_id) as success_count, name
from attorney_table
where disposition = 'Not Guilty'
group by name
)
select aggregate_table.name, total_count, (success_count * 100.0/total_count) as success_percent 
from aggregate_table, success_table
where aggregate_table.name = success_table.name and total_count > 100
order by success_percent desc, total_count desc
limit 5
;
```

Details:

 Print the attorney's name, number of cases handled, and the percentage of cases won (i.e. the disposition was Not Guilty). Examine only attorneys who have handled more than 100 cases. List the attorneys in decreasing order with respect to their success percentage and number of cases handled, respectively. Display only the top 5 attorneys. Ensure that the attorney's name is not empty. You might want to leverage common table expressions (CTEs) in this query. Here's more information on using CTEs in SQLite.

**Q12 [5 points]:** Find the attorney with the seventh highest success percentage (by extending the previous query). The output should look like this:

```
G|50|5.123
```

```sql
--- Attorney with seventh highest success percentage

with attorney_table as
(
select charges.case_id as case_id, disposition, name from charges, attorneys
where charges.case_id = attorneys.case_id
and name <> ''
),
aggregate_table as
(
select count(case_id) as total_count, name
from attorney_table
group by name
),
success_table as
(
select count(case_id) as success_count, name
from attorney_table
where disposition = 'Not Guilty'
group by name
),
percent_table as
(
select aggregate_table.name as name, total_count, (success_count * 100.0/total_count) as success_percent 
from aggregate_table, success_table
where aggregate_table.name = success_table.name and total_count > 100
order by success_percent desc, total_count desc
)
select name, total_count, success_percent
from percent_table
limit 1 
offset 6
;
```



**Details:** Print the attorney's name, number of cases handled, and the percentage of cases won (i.e. the disposition was `Not Guilty`). Examine only attorneys who have handled more than `100` cases. Ensure that the attorney's name is not empty.



## 2018 

### Q3 [10 POINTS] (Q3_POPULAR_CITY):

Find the percentage of trips in each city. A trip belongs to a city as long as its start station or end station is in the city. For example, if a trip started from station A in city P and ended in station B in city Q, then the trip belongs to both city P and city Q. If P equals to Q, the trip is only counted once.

```sql
---my solution, need to improve

with start_city_count as (
select station.city as city, count(station.city) as per_city_count
from trip, station
where trip.start_station_id = station.station_id
group by station.city
),
end_city_count as (
select station.city as city, count(station.city) as per_city_count
from trip, station
where trip.end_station_id = station.station_id
and trip.end_station_id <> trip.start_station_id
group by station.city
),
city_trip_count as (
select start_city_count.city as city, start_city_count.per_city_count + end_city_count.per_city_count as per_city_count
from start_city_count, end_city_count
where start_city_count.city = end_city_count.city
), 
trip_count as(
select  sum(per_city_count) as all_city_count
from city_trip_count
)
select city_trip_count.city,  round((city_trip_count.per_city_count * 100.0 /trip_count.all_city_count ),4) as ratio
from city_trip_count, trip_count
group by city_trip_count.city
order by ratio desc;
```

**Details:** Print city name and ratio between the number of trips that belong to that city against the total number of trips (a decimal between 0-1, round to `four` decimal places using `ROUND()`). Sort by ratio (decreasing), and break ties by city name (increasing).

### Q4 [15 POINTS] (Q4_MOST_POPULAR_STATION):

For each city, find the most popular station in that city. "Popular" means that the station has the highest count of visits. As above, either starting a trip or finishing a trip at a station, the trip is counted as one "visit" to that station. The trip is only counted once if the start station and the end station are the same.

```sql
with start_station_count as (
select station.city as city, station.station_name as station_name, 
count(station.station_name) as per_city_station_count
from trip, station
where trip.start_station_id = station.station_id
group by station.city, station.station_name
),
end_station_count as (
select station.city as city, station.station_name as station_name, 
count(station.station_name) as per_city_station_count
from trip, station
where trip.end_station_id = station.station_id
and trip.end_station_id <> trip.start_station_id
group by station.city, station.station_name
),
station_count as (
select start_station_count.city as city, start_station_count.station_name as station_name, start_station_count.per_city_station_count + end_station_count.per_city_station_count as per_station_city_count
from start_station_count, end_station_count
where start_station_count.station_name = end_station_count.station_name
and start_station_count.city = end_station_count.city
group by start_station_count.city, start_station_count.station_name
)
select station_count.city, station_count.station_name, max(per_station_city_count)  as most_popular_station_of_city
from station_count
group by station_count.city
order by per_station_city_count asc;
```



**Details:** For each station, print city name, most popular station name and its visit count. Sort by city name, ascending.

### Q5 [15 POINTS] (Q5_DAYS_MOST_BIKE_UTILIZATION):

Find the top 10 days that have the highest average bike utilization. For simplicity, we only consider trips that use bikes with id <= 100. The average bike utilization on date D is calculated as the sum of the durations of all the trips that happened on date D divided by the total number of bikes with id <= 100, which is a constant. If a trip overlaps with date D, but starts before date D or ends after date D, then only the interval that overlaps with date D (from 0:00 to 24:00) will be counted when calculating the average bike utilization of date D. And we only calculate the average bike utilization for the date that has been either a start or an end date of a trip. You can assume that no trip has negative time (i.e., for all trips, start time <= end time).

```sql

```



**Details:** For the dates with the top 10 average duration, print the date and the average bike duration on that date (in seconds, round to four decimal places using the `ROUND()` function). Sort by the average duration, decreasing. `Please refer to the updated note before Q1 when calculating the duration of a trip.`

**Hint:** All timestamps are stored as text after loaded from csv in sqlite. You can use `datetime(timestamp string)` to get the timestamp out of the string and `date(timestamp string)` to get the date out of the string. You may also find the funtion `strftime()` helpful in computing the duration between two timestamps.

### Q6 [10 POINTS] (Q6_OVERLAPPING_TRIPS):

One of the possible data-entry errors is to record a bike as being used in two different trips, at the same time. Thus, we want to spot pairs of overlapping intervals (start time, end time). To keep the output manageable, we ask you to do this check for bikes with id between 100 and 200 (both inclusive). Note: Assume that no trip has negative time, i.e., for all trips, start time <= end time.

```sql
---查询出重复数据

with former_table as (
select trip.id as id , trip.bike_id as bike_id, trip.start_time as start_time , trip.end_time as end_time
	from trip
	where trip.bike_id > 100
	and trip.bike_id < 200
	order by trip.bike_id
),	
latter_table as (
select trip.id as id , trip.bike_id as bike_id , trip.start_time as start_time, trip.end_time as end_time
	from trip
	where trip.bike_id > 100
	and trip.bike_id < 200
	order by trip.bike_id
)
	
select former_table.bike_id,  former_table.id as former_id, former_table.start_time as former_start_time, former_table.end_time as former_end_time,
 latter_table.id as latter_id, latter_table.start_time as latter_start_time, latter_table.end_time as latter_end_time
	from former_table, latter_table
	where former_table.bike_id = latter_table.bike_id
	and former_table.id < latter_table.id
	and former_table.start_time < latter_table.end_time
	and former_table.end_time > latter_table.start_time
	order by former_table.bike_id;
```

**Details:** For each conflict (a pair of conflict trips), print the bike id, former trip id, former start time, former end time, latter trip id, latter start time, latter end time. Sort by bike id (increasing), break ties with former trip id (increasing) and then latter trip id (increasing).

**Hint:** (1) Report each conflict pair only once, so that `former trip id < latter trip id`. (2) We give you the (otherwise tricky) condition for conflicts: start1 < end2 AND end1 > start2

### Q7 [10 POINTS] (Q7_MULTI_CITY_BIKES):

Find all the bikes that have been to more than one city. A bike has been to a city as long as the start station or end station in one of its trips is in that city.

```sql
select bike_id, city
```



**Details:** For each bike that has been to more than one city, print the bike id and the number of cities it has been to. Sort by the number of cities (decreasing), then bike id (increasing).

### Q8 [10 POINTS] (Q8_BIKE_POPULARITY_BY_WEATHER):

Find what is the average number of trips made per day on each type of weather day. The type of weather on a day is specified by weather.events, such as 'Rain', 'Fog' and so on. For simplicity, we consider all days that does not have a weather event (

```
weather.events = '\N'
```

) as a single type of weather. Here a trip belongs to a date only if its start time is on that date. We use the weather at the starting position of that trip as its weather type as well. There are also 'Rain' and 'rain' in weather.events. For simplicity, we consider them as different types of weathers. When counting the total number of days for a weather, we consider a weather happened on a date as long as it happened in at least one region on that date.

**Details:** Print the name of the weather and the average number of trips made per day on that type of weather (round to `four` decimal places using `ROUND()`). Sort by the average number of trips (decreasing), then weather name (increasing).

### Q9 [10 POINTS] (Q9_TEMPERATURE_SHORTER_TRIPS):

A short trip is a trip whose duration is <= 60 seconds

. Compute the average temperature that a short trip starts versus the average temperature that a non-short trip starts. We use weather.mean_temp on the date of the start time as the Temperature measurement.

**Details:** Print the average temperature that a short trip starts and the average temperature that a non-short trip starts. (on the same row, and both round to `four` decimal places using `ROUND()`) `Please refer to the updated note before Q1 when calculating the duration of a trip.`

### Q10 [15 POINTS] (Q10_RIDING_IN_STORM):

For each zip code that has experienced 'Rain-Thunderstorm' weather, find the station that has the most number of trips in that zip code under the storm weather. For simplicity, we only consider the start time of a trip when deciding the station and the weather for that trip.

**Details:** Print the zip code that has experienced the 'Rain-Thunderstorm' weather, the name of the station that has the most number of trips under the strom weather in that zip code, and the total number of trips that station has under the storm weather. Sort by the zip code (increasing). You do not need to print the zip code that has experienced 'Rain-Thunderstorm' weather but no trip happens on any storm day in that zip code.