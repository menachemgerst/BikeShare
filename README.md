# BikeShare

# Executive Summary
This analysis was done for the business and marketing division in the company in order to identify ways to convert casual riders to annual riders. The analysis focuses on the characteristics of the casual users vs. the annual users showing that casual users ride the bikes mostly for leisure during the spring, summer on weekends and at prime tourist locations in the city. Annual users ride the bikes for regular day to day needs, ride all year round, ride mostly on weekdays and during the work day hours (not including around noon) and when they ride on the weekends or during the summer vacation the popular locations are similar to the casual rides and in touristy locations.

They data analysis team recommendations are to focus on two methods, 

First method – special annual packages for short time periods (weeklong, seasonal) to convert casual users to short time annual members (leading to them returning to using the package the flowing season or visit)

Second method – advertise in the suburbs targeting the casual riders that may not use the bikes as frequently as the annual riders but also don't have the same characteristics as the average casual riders. 

### Limitations 

-	The data does not include any type of distinct users (via paying methods) and demographic information about the users (sex, age occupation etc.) that can shed light about the characteristics of the different users. 


### Further analysis

-	Calculation of ride distance, routes, and station clusters for creating location packages (using python pandas)
-	With taking into consideration the method of creating packages a further analysis will try to include data about weather, distance from areas of interest, yearly events (Christmas, sporting events etc.) to deeply understand the characteristics of the rides.






## Full Analysis Report 

## Introduction 
### About
In 2016, Cyclistic launched a successful bike-share program that features more than 5,800 bicycles and 600 docking stations.
Cyclistic sets itself apart by also offering reclining bikes, hand tricycles, and cargo bikes, 
making bike-share more inclusive to people with disabilities and riders who can’t use a standard two-wheeled bike.
the majority of riders opt for traditional bikes; about 8% of riders use the assistive options
Cyclistic users are more likely to ride for leisure, but about 30% use them to commute to work each day

Since 2016, the program has grown to a fleet of 5,824 bicycles that are geotracked 
and locked into a network of 692 stations across Chicago. 
The bikes can be unlocked from one station and returned to any other station in the system anytime.

Cyclistic’s marketing strategy relied on building general awareness and appealing to broad consumer segments. 
One approach that helped make these things possible was the flexibility of its pricing plans: single-ride passes, full-day passes, and annual memberships. 
Customers who purchase single-ride or full-day passes are referred to as casual riders. 
Customers who purchase annual memberships are Cyclistic members.

Cyclistic’s finance analysts have concluded that annual members are much more profitable than casual riders. 
Although the pricing flexibility helps Cyclistic attract more customers, 
the finance analysts believe that maximizing the number of annual members will be key to future growth. 
Rather than creating a marketing campaign that targets all-new customers, the finance analysts believe 
there is a very good chance to convert casual riders into members. 

### Analysis Goals
1. To identify the differences between the types of riders membership - casual vs. annual
2. To identify the high and low usage of the bikes during the day, week and year (including differences between membership)
3. To identify the popular (and not popular) stations and riding ereas (including differences between membership)
Answering these three questions can help build a new stratgy to convert casual to annual riders

### The Data
The data contains all the rides recorded in 12 month period. the data of each use of a bike is recorderd and stored in the DWH. 
The data is then uploaded to a public server and can be accesed at this site: https://divvy-tripdata.s3.amazonaws.com/index.html
This analysis will refer to the rides taken between July 2020 and June 2021. 
In further analysis will analyze longer time periods and try to identify and predict types of riders, bikes and other information to improve the service and plans.

## Analysis Process
### Data modifying and cleaning

1. what is in the date?
	
	query for the tables and data types in the db(using a pivot table to identify same column with different data types):

		SELECT *
		FROM 
		(	SELECT	 tab.name as table_name
 				,col.name as column_name
 				,t.name as data_type    
			FROM sys.tables as tab
			INNER JOIN sys.columns as col
			ON tab.object_id = col.object_id
			LEFT JOIN sys.types as t
			ON col.user_type_id = t.user_type_id
		)
		AS TBL
		PIVOT (COUNT(table_name) FOR data_type IN ([nvarchar],[float],[datetime]))AS PVT


the query shows us that not all the columns in the db have the same data type and they need to be altered in order to ba able to inserte all the tables into one main table.

2. data altering

	the columns with differnet data types are - end_station_id: nvarchar/float, start_station_id: nvarchar/float
	the stations ID's are mostly numbers, but in some month stored also with letters, there for we will alter all stations ID's into NVARCHAR and not into 		FLOAT
	(also cheack to see if it is possibale to change station ID's with existing FLOAT ID's using this query:
	
		SELECT * FROM
		(
			SELECT	 end_station_name
				,end_station_id
				,COUNT(DISTINCT end_station_name) AS 'station'
			FROM trips202106
			GROUP BY end_station_name ,end_station_id
		) AS tbl
		 WHERE station > 1
	
sinch the quey returned zero rows we can understand that stations ID's with letters or distinct and can't be replaced with existing FLOAT ID's)
	
Query for altering start_station_id data type to all be the same (varchar):
	
		ALTER TABLE trips202007 ALTER COLUMN start_station_id varchar(15)
		ALTER TABLE trips202008 ALTER COLUMN start_station_id varchar(15)
		ALTER TABLE trips202009 ALTER COLUMN start_station_id varchar(15)
		ALTER TABLE trips202010 ALTER COLUMN start_station_id varchar(15)
		ALTER TABLE trips202011 ALTER COLUMN start_station_id varchar(15)
		ALTER TABLE trips202012 ALTER COLUMN start_station_id varchar(15)
		ALTER TABLE trips202101 ALTER COLUMN start_station_id varchar(15)
		ALTER TABLE trips202102 ALTER COLUMN start_station_id varchar(15)
		ALTER TABLE trips202103 ALTER COLUMN start_station_id varchar(15)
	
(did not include month 202104,05,06 because altering returned error - 
Msg 8152, Level 16, State 13, Line 50
String or binary data would be truncated.
The statement has been terminated.)
	
and for altering end_station_id data type to all be the same (varchar):
	
		ALTER TABLE trips202007 ALTER COLUMN end_station_id varchar(15)
		ALTER TABLE trips202008 ALTER COLUMN end_station_id varchar(15)
		ALTER TABLE trips202009 ALTER COLUMN end_station_id varchar(15)
		ALTER TABLE trips202010 ALTER COLUMN end_station_id varchar(15)
		ALTER TABLE trips202011 ALTER COLUMN end_station_id varchar(15)
		ALTER TABLE trips202104 ALTER COLUMN end_station_id varchar(15)
		

3. Omitted data from the analysis
		
	the following data is irrelevant for the analysis
	
	a. test rides
	
		SELECT *
		FROM trips202007
		WHERE 	start_station_name LIKE '%test%' 
			OR start_station_name LIKE '%CHECK%' 
			OR end_station_name LIKE '%test%' 
			OR end_station_name LIKE '%CHECK%'
			
		
	b. rides that didn't happen (or bad data was collocted)
	
		WITH CTE AS
		(
			 SELECT  *,
			 CASE WHEN start_station_id = end_station_id AND DATEDIFF(mi,started_at,ended_at) <= 0 THEN 1
			      WHEN ended_at < started_at THEN 2
				 ELSE 0 
			 END AS 'no_ride_code'
			 FROM trips202007
     		 )
		  SELECT *
		  FROM CTE
		  WHERE no_ride_code > 0
		
	
4. Inserting all tabels into one main table
	
	into the main table will add as many indicators and information that will help the final analysis
	
	a. ride time
	
		
		DATEDIFF(mi,started_at,ended_at) AS ride_time
		
		
	b. hour of the ride (to idetify riding "rush hours")
		
		DATEPART(HOUR, started_at) AS hour_ride
		
	c. weekday
		
		 DATEPART(WEEKDAY, started_at) AS ride_weekday
		,DATENAME(WEEKDAY, started_at) AS ride_weekday_name
		
	d. month
		
		DATEPART(MONTH, started_at) AS month
		,DATENAME(MONTH, started_at) AS month_name
		
	e. season
		
		CASE 
		WHEN (MONTH(started_at) = 3 AND DAY(started_at) >= 20)
		 OR (MONTH(started_at) = 4)
		 OR (MONTH(started_at) = 5)
		 OR (MONTH(started_at) = 6 AND DAY(started_at) <= 20) THEN 'Spring'
		WHEN (MONTH(started_at) = 6 AND DAY(started_at) >= 21)
		 OR (MONTH(started_at) = 7)
		 OR (MONTH(started_at) = 8)
		 OR (MONTH(started_at) = 9 AND DAY(started_at) <= 21) THEN 'Summer'
		WHEN (MONTH(started_at) = 9 AND DAY(started_at) >= 22)
		 OR (MONTH(started_at) = 10)
		 OR (MONTH(started_at) = 11)
		 OR (MONTH(started_at) = 12 AND DAY(started_at) <= 20) THEN 'Fall'
		HEN  (MONTH(started_at) = 12 AND DAY(started_at) >= 21)
		 OR (MONTH(started_at) = 1)
		 OR (MONTH(started_at) = 2)
		 OR (MONTH(started_at) = 3 AND DAY(started_at) <= 19) THEN 'Winter'
		END AS Season
		
### Inserting all the data into the main table

using UNION ALL for each one of the twelve tables and creating all the indicators

		
	WITH CTE AS
	(
	SELECT	 ride_id
			,rideable_type
			,started_at
			,ended_at
			,start_station_name
			,start_station_id
			,LEFT(start_lat, 6) + ',' + LEFT(start_lng, 7) AS start_lat_lng
			,end_station_name
			,end_station_id
			,LEFT(end_lat, 6) + ',' + LEFT(end_lng, 7) AS end_lat_lng
			,member_casual
			,DATEDIFF(mi,started_at,ended_at) AS ride_time
			,DATEPART(HOUR, started_at) AS hour_ride
			,DATEPART(WEEKDAY, started_at) AS ride_weekday
			,DATENAME(WEEKDAY, started_at) AS ride_weekday_name
			,DATEPART(MONTH, started_at) AS month
			,DATENAME(MONTH, started_at) AS month_name
			,CASE WHEN DATEPART(HOUR, started_at) BETWEEN 5 AND 6 THEN '1.Early Morning'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 7 AND 9 THEN '2.Morning'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 10 AND 11 THEN '3.Late Morning'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 12 AND 15 THEN '4.Afternoon'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 16 AND 17 THEN '5.Late Afternoon'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 18 AND 20 THEN '6.Evening'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 21 AND 23 THEN '7.Night'
				  WHEN DATEPART(HOUR, started_at) = 00 THEN '7.Night'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 1 AND 4 THEN '0.Middle of the night'
			 END AS 'day_part'
			,CASE WHEN (MONTH(started_at) = 3 AND DAY(started_at) >= 20)
					 OR (MONTH(started_at) = 4)
					 OR (MONTH(started_at) = 5)
					 OR (MONTH(started_at) = 6 AND DAY(started_at) <= 20) THEN 'Spring'
				  WHEN (MONTH(started_at) = 6 AND DAY(started_at) >= 21)
					 OR (MONTH(started_at) = 7)
					 OR (MONTH(started_at) = 8)
					 OR (MONTH(started_at) = 9 AND DAY(started_at) <= 21) THEN 'Summer'
				  WHEN (MONTH(started_at) = 9 AND DAY(started_at) >= 22)
					 OR (MONTH(started_at) = 10)
					 OR (MONTH(started_at) = 11)
					 OR (MONTH(started_at) = 12 AND DAY(started_at) <= 20) THEN 'Fall'
				  WHEN  (MONTH(started_at) = 12 AND DAY(started_at) >= 21)
					 OR (MONTH(started_at) = 1)
					 OR (MONTH(started_at) = 2)
					 OR (MONTH(started_at) = 3 AND DAY(started_at) <= 19) THEN 'Winter'
			 END AS 'season'
			,CASE WHEN start_station_id = end_station_id AND DATEDIFF(mi,started_at,ended_at) <= 0 THEN 1
	 			  WHEN ended_at < started_at THEN 2
				  WHEN start_station_name LIKE '%test%' 
	 				OR start_station_name LIKE '%CHECK%' 
	 				OR end_station_name LIKE '%test%' 
	 				OR end_station_name LIKE '%CHECK%' THEN 3
	 			  ELSE 0 
	 		  END AS 'no_ride'
			 ,DATEDIFF(dy,started_at,ended_at) AS same_day_ride
	FROM trips202007
	UNION ALL
	SELECT	 ride_id
			,rideable_type
			,started_at
			,ended_at
			,start_station_name
			,start_station_id
			,LEFT(start_lat, 6) + ',' + LEFT(start_lng, 7) AS start_lat_lng
			,end_station_name
			,end_station_id
			,LEFT(end_lat, 6) + ',' + LEFT(end_lng, 7) AS end_lat_lng
			,member_casual
			,DATEDIFF(mi,started_at,ended_at) AS ride_time
			,DATEPART(HOUR, started_at) AS hour_ride
			,DATEPART(WEEKDAY, started_at) AS ride_weekday
			,DATENAME(WEEKDAY, started_at) AS ride_weekday_name
			,DATEPART(MONTH, started_at) AS month
			,DATENAME(MONTH, started_at) AS month_name
			,CASE WHEN DATEPART(HOUR, started_at) BETWEEN 5 AND 6 THEN '1.Early Morning'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 7 AND 9 THEN '2.Morning'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 10 AND 11 THEN '3.Late Morning'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 12 AND 15 THEN '4.Afternoon'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 16 AND 17 THEN '5.Late Afternoon'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 18 AND 20 THEN '6.Evening'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 21 AND 23 THEN '7.Night'
				  WHEN DATEPART(HOUR, started_at) = 00 THEN '7.Night'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 1 AND 4 THEN '0.Middle of the night'
			 END AS 'day_part'
			,CASE WHEN (MONTH(started_at) = 3 AND DAY(started_at) >= 20)
					 OR (MONTH(started_at) = 4)
					 OR (MONTH(started_at) = 5)
					 OR (MONTH(started_at) = 6 AND DAY(started_at) <= 20) THEN 'Spring'
				  WHEN (MONTH(started_at) = 6 AND DAY(started_at) >= 21)
					 OR (MONTH(started_at) = 7)
					 OR (MONTH(started_at) = 8)
					 OR (MONTH(started_at) = 9 AND DAY(started_at) <= 21) THEN 'Summer'
				  WHEN (MONTH(started_at) = 9 AND DAY(started_at) >= 22)
					 OR (MONTH(started_at) = 10)
					 OR (MONTH(started_at) = 11)
					 OR (MONTH(started_at) = 12 AND DAY(started_at) <= 20) THEN 'Fall'
				  WHEN  (MONTH(started_at) = 12 AND DAY(started_at) >= 21)
					 OR (MONTH(started_at) = 1)
					 OR (MONTH(started_at) = 2)
					 OR (MONTH(started_at) = 3 AND DAY(started_at) <= 19) THEN 'Winter'
			 END AS 'season'
			,CASE WHEN start_station_id = end_station_id AND DATEDIFF(mi,started_at,ended_at) <= 0 THEN 1
	 			  WHEN ended_at < started_at THEN 2
				  WHEN start_station_name LIKE '%test%' 
	 				OR start_station_name LIKE '%CHECK%' 
	 				OR end_station_name LIKE '%test%' 
	 				OR end_station_name LIKE '%CHECK%' THEN 3
	 			  ELSE 0 
	 		  END AS 'no_ride'
			 ,DATEDIFF(dy,started_at,ended_at) AS same_day_ride
	FROM trips202008
	UNION ALL
	SELECT	 ride_id
			,rideable_type
			,started_at
			,ended_at
			,start_station_name
			,start_station_id
			,LEFT(start_lat, 6) + ',' + LEFT(start_lng, 7) AS start_lat_lng
			,end_station_name
			,end_station_id
			,LEFT(end_lat, 6) + ',' + LEFT(end_lng, 7) AS end_lat_lng
			,member_casual
			,DATEDIFF(mi,started_at,ended_at) AS ride_time
			,DATEPART(HOUR, started_at) AS hour_ride
			,DATEPART(WEEKDAY, started_at) AS ride_weekday
			,DATENAME(WEEKDAY, started_at) AS ride_weekday_name
			,DATEPART(MONTH, started_at) AS month
			,DATENAME(MONTH, started_at) AS month_name
			,CASE WHEN DATEPART(HOUR, started_at) BETWEEN 5 AND 6 THEN '1.Early Morning'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 7 AND 9 THEN '2.Morning'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 10 AND 11 THEN '3.Late Morning'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 12 AND 15 THEN '4.Afternoon'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 16 AND 17 THEN '5.Late Afternoon'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 18 AND 20 THEN '6.Evening'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 21 AND 23 THEN '7.Night'
				  WHEN DATEPART(HOUR, started_at) = 00 THEN '7.Night'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 1 AND 4 THEN '0.Middle of the night'
			 END AS 'day_part'
			,CASE WHEN (MONTH(started_at) = 3 AND DAY(started_at) >= 20)
					 OR (MONTH(started_at) = 4)
					 OR (MONTH(started_at) = 5)
					 OR (MONTH(started_at) = 6 AND DAY(started_at) <= 20) THEN 'Spring'
				  WHEN (MONTH(started_at) = 6 AND DAY(started_at) >= 21)
					 OR (MONTH(started_at) = 7)
					 OR (MONTH(started_at) = 8)
					 OR (MONTH(started_at) = 9 AND DAY(started_at) <= 21) THEN 'Summer'
				  WHEN (MONTH(started_at) = 9 AND DAY(started_at) >= 22)
					 OR (MONTH(started_at) = 10)
					 OR (MONTH(started_at) = 11)
					 OR (MONTH(started_at) = 12 AND DAY(started_at) <= 20) THEN 'Fall'
				  WHEN  (MONTH(started_at) = 12 AND DAY(started_at) >= 21)
					 OR (MONTH(started_at) = 1)
					 OR (MONTH(started_at) = 2)
					 OR (MONTH(started_at) = 3 AND DAY(started_at) <= 19) THEN 'Winter'
			 END AS 'season'
			,CASE WHEN start_station_id = end_station_id AND DATEDIFF(mi,started_at,ended_at) <= 0 THEN 1
	 			  WHEN ended_at < started_at THEN 2
				  WHEN start_station_name LIKE '%test%' 
	 				OR start_station_name LIKE '%CHECK%' 
	 				OR end_station_name LIKE '%test%' 
	 				OR end_station_name LIKE '%CHECK%' THEN 3
	 			  ELSE 0 
	 		  END AS 'no_ride'
			 ,DATEDIFF(dy,started_at,ended_at) AS same_day_ride
	FROM trips202009
	UNION ALL
	SELECT	 ride_id
			,rideable_type
			,started_at
			,ended_at
			,start_station_name
			,start_station_id
			,LEFT(start_lat, 6) + ',' + LEFT(start_lng, 7) AS start_lat_lng
			,end_station_name
			,end_station_id
			,LEFT(end_lat, 6) + ',' + LEFT(end_lng, 7) AS end_lat_lng
			,member_casual
			,DATEDIFF(mi,started_at,ended_at) AS ride_time
			,DATEPART(HOUR, started_at) AS hour_ride
			,DATEPART(WEEKDAY, started_at) AS ride_weekday
			,DATENAME(WEEKDAY, started_at) AS ride_weekday_name
			,DATEPART(MONTH, started_at) AS month
			,DATENAME(MONTH, started_at) AS month_name
			,CASE WHEN DATEPART(HOUR, started_at) BETWEEN 5 AND 6 THEN '1.Early Morning'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 7 AND 9 THEN '2.Morning'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 10 AND 11 THEN '3.Late Morning'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 12 AND 15 THEN '4.Afternoon'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 16 AND 17 THEN '5.Late Afternoon'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 18 AND 20 THEN '6.Evening'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 21 AND 23 THEN '7.Night'
				  WHEN DATEPART(HOUR, started_at) = 00 THEN '7.Night'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 1 AND 4 THEN '0.Middle of the night'
			 END AS 'day_part'
			,CASE WHEN (MONTH(started_at) = 3 AND DAY(started_at) >= 20)
					 OR (MONTH(started_at) = 4)
					 OR (MONTH(started_at) = 5)
					 OR (MONTH(started_at) = 6 AND DAY(started_at) <= 20) THEN 'Spring'
				  WHEN (MONTH(started_at) = 6 AND DAY(started_at) >= 21)
					 OR (MONTH(started_at) = 7)
					 OR (MONTH(started_at) = 8)
					 OR (MONTH(started_at) = 9 AND DAY(started_at) <= 21) THEN 'Summer'
				  WHEN (MONTH(started_at) = 9 AND DAY(started_at) >= 22)
					 OR (MONTH(started_at) = 10)
					 OR (MONTH(started_at) = 11)
					 OR (MONTH(started_at) = 12 AND DAY(started_at) <= 20) THEN 'Fall'
				  WHEN  (MONTH(started_at) = 12 AND DAY(started_at) >= 21)
					 OR (MONTH(started_at) = 1)
					 OR (MONTH(started_at) = 2)
					 OR (MONTH(started_at) = 3 AND DAY(started_at) <= 19) THEN 'Winter'
			 END AS 'season'
			,CASE WHEN start_station_id = end_station_id AND DATEDIFF(mi,started_at,ended_at) <= 0 THEN 1
	 			  WHEN ended_at < started_at THEN 2
				  WHEN start_station_name LIKE '%test%' 
	 				OR start_station_name LIKE '%CHECK%' 
	 				OR end_station_name LIKE '%test%' 
	 				OR end_station_name LIKE '%CHECK%' THEN 3
	 			  ELSE 0 
	 		  END AS 'no_ride'
			 ,DATEDIFF(dy,started_at,ended_at) AS same_day_ride
	FROM trips202010
	UNION ALL
	SELECT	 ride_id
			,rideable_type
			,started_at
			,ended_at
			,start_station_name
			,start_station_id
			,LEFT(start_lat, 6) + ',' + LEFT(start_lng, 7) AS start_lat_lng
			,end_station_name
			,end_station_id
			,LEFT(end_lat, 6) + ',' + LEFT(end_lng, 7) AS end_lat_lng
			,member_casual
			,DATEDIFF(mi,started_at,ended_at) AS ride_time
			,DATEPART(HOUR, started_at) AS hour_ride
			,DATEPART(WEEKDAY, started_at) AS ride_weekday
			,DATENAME(WEEKDAY, started_at) AS ride_weekday_name
			,DATEPART(MONTH, started_at) AS month
			,DATENAME(MONTH, started_at) AS month_name
			,CASE WHEN DATEPART(HOUR, started_at) BETWEEN 5 AND 6 THEN '1.Early Morning'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 7 AND 9 THEN '2.Morning'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 10 AND 11 THEN '3.Late Morning'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 12 AND 15 THEN '4.Afternoon'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 16 AND 17 THEN '5.Late Afternoon'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 18 AND 20 THEN '6.Evening'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 21 AND 23 THEN '7.Night'
				  WHEN DATEPART(HOUR, started_at) = 00 THEN '7.Night'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 1 AND 4 THEN '0.Middle of the night'
			 END AS 'day_part'
			,CASE WHEN (MONTH(started_at) = 3 AND DAY(started_at) >= 20)
					 OR (MONTH(started_at) = 4)
					 OR (MONTH(started_at) = 5)
					 OR (MONTH(started_at) = 6 AND DAY(started_at) <= 20) THEN 'Spring'
				  WHEN (MONTH(started_at) = 6 AND DAY(started_at) >= 21)
					 OR (MONTH(started_at) = 7)
					 OR (MONTH(started_at) = 8)
					 OR (MONTH(started_at) = 9 AND DAY(started_at) <= 21) THEN 'Summer'
				  WHEN (MONTH(started_at) = 9 AND DAY(started_at) >= 22)
					 OR (MONTH(started_at) = 10)
					 OR (MONTH(started_at) = 11)
					 OR (MONTH(started_at) = 12 AND DAY(started_at) <= 20) THEN 'Fall'
				  WHEN  (MONTH(started_at) = 12 AND DAY(started_at) >= 21)
					 OR (MONTH(started_at) = 1)
					 OR (MONTH(started_at) = 2)
					 OR (MONTH(started_at) = 3 AND DAY(started_at) <= 19) THEN 'Winter'
			 END AS 'season'
			,CASE WHEN start_station_id = end_station_id AND DATEDIFF(mi,started_at,ended_at) <= 0 THEN 1
	 			  WHEN ended_at < started_at THEN 2
				  WHEN start_station_name LIKE '%test%' 
	 				OR start_station_name LIKE '%CHECK%' 
	 				OR end_station_name LIKE '%test%' 
	 				OR end_station_name LIKE '%CHECK%' THEN 3
	 			  ELSE 0 
	 		  END AS 'no_ride'
			 ,DATEDIFF(dy,started_at,ended_at) AS same_day_ride
	FROM trips202011
	UNION ALL
	SELECT	 ride_id
			,rideable_type
			,started_at
			,ended_at
			,start_station_name
			,start_station_id
			,LEFT(start_lat, 6) + ',' + LEFT(start_lng, 7) AS start_lat_lng
			,end_station_name
			,end_station_id
			,LEFT(end_lat, 6) + ',' + LEFT(end_lng, 7) AS end_lat_lng
			,member_casual
			,DATEDIFF(mi,started_at,ended_at) AS ride_time
			,DATEPART(HOUR, started_at) AS hour_ride
			,DATEPART(WEEKDAY, started_at) AS ride_weekday
			,DATENAME(WEEKDAY, started_at) AS ride_weekday_name
			,DATEPART(MONTH, started_at) AS month
			,DATENAME(MONTH, started_at) AS month_name
			,CASE WHEN DATEPART(HOUR, started_at) BETWEEN 5 AND 6 THEN '1.Early Morning'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 7 AND 9 THEN '2.Morning'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 10 AND 11 THEN '3.Late Morning'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 12 AND 15 THEN '4.Afternoon'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 16 AND 17 THEN '5.Late Afternoon'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 18 AND 20 THEN '6.Evening'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 21 AND 23 THEN '7.Night'
				  WHEN DATEPART(HOUR, started_at) = 00 THEN '7.Night'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 1 AND 4 THEN '0.Middle of the night'
			 END AS 'day_part'
			,CASE WHEN (MONTH(started_at) = 3 AND DAY(started_at) >= 20)
					 OR (MONTH(started_at) = 4)
					 OR (MONTH(started_at) = 5)
					 OR (MONTH(started_at) = 6 AND DAY(started_at) <= 20) THEN 'Spring'
				  WHEN (MONTH(started_at) = 6 AND DAY(started_at) >= 21)
					 OR (MONTH(started_at) = 7)
					 OR (MONTH(started_at) = 8)
					 OR (MONTH(started_at) = 9 AND DAY(started_at) <= 21) THEN 'Summer'
				  WHEN (MONTH(started_at) = 9 AND DAY(started_at) >= 22)
					 OR (MONTH(started_at) = 10)
					 OR (MONTH(started_at) = 11)
					 OR (MONTH(started_at) = 12 AND DAY(started_at) <= 20) THEN 'Fall'
				  WHEN  (MONTH(started_at) = 12 AND DAY(started_at) >= 21)
					 OR (MONTH(started_at) = 1)
					 OR (MONTH(started_at) = 2)
					 OR (MONTH(started_at) = 3 AND DAY(started_at) <= 19) THEN 'Winter'
			 END AS 'season'
			,CASE WHEN start_station_id = end_station_id AND DATEDIFF(mi,started_at,ended_at) <= 0 THEN 1
	 			  WHEN ended_at < started_at THEN 2
				  WHEN start_station_name LIKE '%test%' 
	 				OR start_station_name LIKE '%CHECK%' 
	 				OR end_station_name LIKE '%test%' 
	 				OR end_station_name LIKE '%CHECK%' THEN 3
	 			  ELSE 0 
	 		  END AS 'no_ride'
			 ,DATEDIFF(dy,started_at,ended_at) AS same_day_ride
	FROM trips202012
	UNION ALL
	SELECT	 ride_id
			,rideable_type
			,started_at
			,ended_at
			,start_station_name
			,start_station_id
			,LEFT(start_lat, 6) + ',' + LEFT(start_lng, 7) AS start_lat_lng
			,end_station_name
			,end_station_id
			,LEFT(end_lat, 6) + ',' + LEFT(end_lng, 7) AS end_lat_lng
			,member_casual
			,DATEDIFF(mi,started_at,ended_at) AS ride_time
			,DATEPART(HOUR, started_at) AS hour_ride
			,DATEPART(WEEKDAY, started_at) AS ride_weekday
			,DATENAME(WEEKDAY, started_at) AS ride_weekday_name
			,DATEPART(MONTH, started_at) AS month
			,DATENAME(MONTH, started_at) AS month_name
			,CASE WHEN DATEPART(HOUR, started_at) BETWEEN 5 AND 6 THEN '1.Early Morning'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 7 AND 9 THEN '2.Morning'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 10 AND 11 THEN '3.Late Morning'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 12 AND 15 THEN '4.Afternoon'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 16 AND 17 THEN '5.Late Afternoon'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 18 AND 20 THEN '6.Evening'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 21 AND 23 THEN '7.Night'
				  WHEN DATEPART(HOUR, started_at) = 00 THEN '7.Night'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 1 AND 4 THEN '0.Middle of the night'
			 END AS 'day_part'
			,CASE WHEN (MONTH(started_at) = 3 AND DAY(started_at) >= 20)
					 OR (MONTH(started_at) = 4)
					 OR (MONTH(started_at) = 5)
					 OR (MONTH(started_at) = 6 AND DAY(started_at) <= 20) THEN 'Spring'
				  WHEN (MONTH(started_at) = 6 AND DAY(started_at) >= 21)
					 OR (MONTH(started_at) = 7)
					 OR (MONTH(started_at) = 8)
					 OR (MONTH(started_at) = 9 AND DAY(started_at) <= 21) THEN 'Summer'
				  WHEN (MONTH(started_at) = 9 AND DAY(started_at) >= 22)
					 OR (MONTH(started_at) = 10)
					 OR (MONTH(started_at) = 11)
					 OR (MONTH(started_at) = 12 AND DAY(started_at) <= 20) THEN 'Fall'
				  WHEN  (MONTH(started_at) = 12 AND DAY(started_at) >= 21)
					 OR (MONTH(started_at) = 1)
					 OR (MONTH(started_at) = 2)
					 OR (MONTH(started_at) = 3 AND DAY(started_at) <= 19) THEN 'Winter'
			 END AS 'season'
			,CASE WHEN start_station_id = end_station_id AND DATEDIFF(mi,started_at,ended_at) <= 0 THEN 1
	 			  WHEN ended_at < started_at THEN 2
				  WHEN start_station_name LIKE '%test%' 
	 				OR start_station_name LIKE '%CHECK%' 
	 				OR end_station_name LIKE '%test%' 
	 				OR end_station_name LIKE '%CHECK%' THEN 3
	 			  ELSE 0 
	 		  END AS 'no_ride'
			 ,DATEDIFF(dy,started_at,ended_at) AS same_day_ride
	FROM trips202101
	UNION ALL
	SELECT	 ride_id
			,rideable_type
			,started_at
			,ended_at
			,start_station_name
			,start_station_id
			,LEFT(start_lat, 6) + ',' + LEFT(start_lng, 7) AS start_lat_lng
			,end_station_name
			,end_station_id
			,LEFT(end_lat, 6) + ',' + LEFT(end_lng, 7) AS end_lat_lng
			,member_casual
			,DATEDIFF(mi,started_at,ended_at) AS ride_time
			,DATEPART(HOUR, started_at) AS hour_ride
			,DATEPART(WEEKDAY, started_at) AS ride_weekday
			,DATENAME(WEEKDAY, started_at) AS ride_weekday_name
			,DATEPART(MONTH, started_at) AS month
			,DATENAME(MONTH, started_at) AS month_name
			,CASE WHEN DATEPART(HOUR, started_at) BETWEEN 5 AND 6 THEN '1.Early Morning'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 7 AND 9 THEN '2.Morning'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 10 AND 11 THEN '3.Late Morning'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 12 AND 15 THEN '4.Afternoon'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 16 AND 17 THEN '5.Late Afternoon'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 18 AND 20 THEN '6.Evening'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 21 AND 23 THEN '7.Night'
				  WHEN DATEPART(HOUR, started_at) = 00 THEN '7.Night'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 1 AND 4 THEN '0.Middle of the night'
			 END AS 'day_part'
			,CASE WHEN (MONTH(started_at) = 3 AND DAY(started_at) >= 20)
					 OR (MONTH(started_at) = 4)
					 OR (MONTH(started_at) = 5)
					 OR (MONTH(started_at) = 6 AND DAY(started_at) <= 20) THEN 'Spring'
				  WHEN (MONTH(started_at) = 6 AND DAY(started_at) >= 21)
					 OR (MONTH(started_at) = 7)
					 OR (MONTH(started_at) = 8)
					 OR (MONTH(started_at) = 9 AND DAY(started_at) <= 21) THEN 'Summer'
				  WHEN (MONTH(started_at) = 9 AND DAY(started_at) >= 22)
					 OR (MONTH(started_at) = 10)
					 OR (MONTH(started_at) = 11)
					 OR (MONTH(started_at) = 12 AND DAY(started_at) <= 20) THEN 'Fall'
				  WHEN  (MONTH(started_at) = 12 AND DAY(started_at) >= 21)
					 OR (MONTH(started_at) = 1)
					 OR (MONTH(started_at) = 2)
					 OR (MONTH(started_at) = 3 AND DAY(started_at) <= 19) THEN 'Winter'
			 END AS 'season'
			,CASE WHEN start_station_id = end_station_id AND DATEDIFF(mi,started_at,ended_at) <= 0 THEN 1
	 			  WHEN ended_at < started_at THEN 2
				  WHEN start_station_name LIKE '%test%' 
	 				OR start_station_name LIKE '%CHECK%' 
	 				OR end_station_name LIKE '%test%' 
	 				OR end_station_name LIKE '%CHECK%' THEN 3
	 			  ELSE 0 
	 		  END AS 'no_ride'
			 ,DATEDIFF(dy,started_at,ended_at) AS same_day_ride
	FROM trips202102
	UNION ALL
	SELECT	 ride_id
			,rideable_type
			,started_at
			,ended_at
			,start_station_name
			,start_station_id
			,LEFT(start_lat, 6) + ',' + LEFT(start_lng, 7) AS start_lat_lng
			,end_station_name
			,end_station_id
			,LEFT(end_lat, 6) + ',' + LEFT(end_lng, 7) AS end_lat_lng
			,member_casual
			,DATEDIFF(mi,started_at,ended_at) AS ride_time
			,DATEPART(HOUR, started_at) AS hour_ride
			,DATEPART(WEEKDAY, started_at) AS ride_weekday
			,DATENAME(WEEKDAY, started_at) AS ride_weekday_name
			,DATEPART(MONTH, started_at) AS month
			,DATENAME(MONTH, started_at) AS month_name
			,CASE WHEN DATEPART(HOUR, started_at) BETWEEN 5 AND 6 THEN '1.Early Morning'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 7 AND 9 THEN '2.Morning'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 10 AND 11 THEN '3.Late Morning'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 12 AND 15 THEN '4.Afternoon'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 16 AND 17 THEN '5.Late Afternoon'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 18 AND 20 THEN '6.Evening'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 21 AND 23 THEN '7.Night'
				  WHEN DATEPART(HOUR, started_at) = 00 THEN '7.Night'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 1 AND 4 THEN '0.Middle of the night'
			 END AS 'day_part'
			,CASE WHEN (MONTH(started_at) = 3 AND DAY(started_at) >= 20)
					 OR (MONTH(started_at) = 4)
					 OR (MONTH(started_at) = 5)
					 OR (MONTH(started_at) = 6 AND DAY(started_at) <= 20) THEN 'Spring'
				  WHEN (MONTH(started_at) = 6 AND DAY(started_at) >= 21)
					 OR (MONTH(started_at) = 7)
					 OR (MONTH(started_at) = 8)
					 OR (MONTH(started_at) = 9 AND DAY(started_at) <= 21) THEN 'Summer'
				  WHEN (MONTH(started_at) = 9 AND DAY(started_at) >= 22)
					 OR (MONTH(started_at) = 10)
					 OR (MONTH(started_at) = 11)
					 OR (MONTH(started_at) = 12 AND DAY(started_at) <= 20) THEN 'Fall'
				  WHEN  (MONTH(started_at) = 12 AND DAY(started_at) >= 21)
					 OR (MONTH(started_at) = 1)
					 OR (MONTH(started_at) = 2)
					 OR (MONTH(started_at) = 3 AND DAY(started_at) <= 19) THEN 'Winter'
			 END AS 'season'
			,CASE WHEN start_station_id = end_station_id AND DATEDIFF(mi,started_at,ended_at) <= 0 THEN 1
	 			  WHEN ended_at < started_at THEN 2
				  WHEN start_station_name LIKE '%test%' 
	 				OR start_station_name LIKE '%CHECK%' 
	 				OR end_station_name LIKE '%test%' 
	 				OR end_station_name LIKE '%CHECK%' THEN 3
	 			  ELSE 0 
	 		  END AS 'no_ride'
			 ,DATEDIFF(dy,started_at,ended_at) AS same_day_ride
	FROM trips202103
	UNION ALL
	SELECT	 ride_id
			,rideable_type
			,started_at
			,ended_at
			,start_station_name
			,start_station_id
			,LEFT(start_lat, 6) + ',' + LEFT(start_lng, 7) AS start_lat_lng
			,end_station_name
			,end_station_id
			,LEFT(end_lat, 6) + ',' + LEFT(end_lng, 7) AS end_lat_lng
			,member_casual
			,DATEDIFF(mi,started_at,ended_at) AS ride_time
			,DATEPART(HOUR, started_at) AS hour_ride
			,DATEPART(WEEKDAY, started_at) AS ride_weekday
			,DATENAME(WEEKDAY, started_at) AS ride_weekday_name
			,DATEPART(MONTH, started_at) AS month
			,DATENAME(MONTH, started_at) AS month_name
			,CASE WHEN DATEPART(HOUR, started_at) BETWEEN 5 AND 6 THEN '1.Early Morning'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 7 AND 9 THEN '2.Morning'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 10 AND 11 THEN '3.Late Morning'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 12 AND 15 THEN '4.Afternoon'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 16 AND 17 THEN '5.Late Afternoon'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 18 AND 20 THEN '6.Evening'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 21 AND 23 THEN '7.Night'
				  WHEN DATEPART(HOUR, started_at) = 00 THEN '7.Night'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 1 AND 4 THEN '0.Middle of the night'
			 END AS 'day_part'
			,CASE WHEN (MONTH(started_at) = 3 AND DAY(started_at) >= 20)
					 OR (MONTH(started_at) = 4)
					 OR (MONTH(started_at) = 5)
					 OR (MONTH(started_at) = 6 AND DAY(started_at) <= 20) THEN 'Spring'
				  WHEN (MONTH(started_at) = 6 AND DAY(started_at) >= 21)
					 OR (MONTH(started_at) = 7)
					 OR (MONTH(started_at) = 8)
					 OR (MONTH(started_at) = 9 AND DAY(started_at) <= 21) THEN 'Summer'
				  WHEN (MONTH(started_at) = 9 AND DAY(started_at) >= 22)
					 OR (MONTH(started_at) = 10)
					 OR (MONTH(started_at) = 11)
					 OR (MONTH(started_at) = 12 AND DAY(started_at) <= 20) THEN 'Fall'
				  WHEN  (MONTH(started_at) = 12 AND DAY(started_at) >= 21)
					 OR (MONTH(started_at) = 1)
					 OR (MONTH(started_at) = 2)
					 OR (MONTH(started_at) = 3 AND DAY(started_at) <= 19) THEN 'Winter'
			 END AS 'season'
			,CASE WHEN start_station_id = end_station_id AND DATEDIFF(mi,started_at,ended_at) <= 0 THEN 1
	 			  WHEN ended_at < started_at THEN 2
				  WHEN start_station_name LIKE '%test%' 
	 				OR start_station_name LIKE '%CHECK%' 
	 				OR end_station_name LIKE '%test%' 
	 				OR end_station_name LIKE '%CHECK%' THEN 3
	 			  ELSE 0 
	 		  END AS 'no_ride'
			 ,DATEDIFF(dy,started_at,ended_at) AS same_day_ride
	FROM trips202104
	UNION ALL
	SELECT	 ride_id
			,rideable_type
			,started_at
			,ended_at
			,start_station_name
			,start_station_id
			,LEFT(start_lat, 6) + ',' + LEFT(start_lng, 7) AS start_lat_lng
			,end_station_name
			,end_station_id
			,LEFT(end_lat, 6) + ',' + LEFT(end_lng, 7) AS end_lat_lng
			,member_casual
			,DATEDIFF(mi,started_at,ended_at) AS ride_time
			,DATEPART(HOUR, started_at) AS hour_ride
			,DATEPART(WEEKDAY, started_at) AS ride_weekday
			,DATENAME(WEEKDAY, started_at) AS ride_weekday_name
			,DATEPART(MONTH, started_at) AS month
			,DATENAME(MONTH, started_at) AS month_name
			,CASE WHEN DATEPART(HOUR, started_at) BETWEEN 5 AND 6 THEN '1.Early Morning'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 7 AND 9 THEN '2.Morning'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 10 AND 11 THEN '3.Late Morning'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 12 AND 15 THEN '4.Afternoon'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 16 AND 17 THEN '5.Late Afternoon'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 18 AND 20 THEN '6.Evening'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 21 AND 23 THEN '7.Night'
				  WHEN DATEPART(HOUR, started_at) = 00 THEN '7.Night'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 1 AND 4 THEN '0.Middle of the night'
			 END AS 'day_part'
			,CASE WHEN (MONTH(started_at) = 3 AND DAY(started_at) >= 20)
					 OR (MONTH(started_at) = 4)
					 OR (MONTH(started_at) = 5)
					 OR (MONTH(started_at) = 6 AND DAY(started_at) <= 20) THEN 'Spring'
				  WHEN (MONTH(started_at) = 6 AND DAY(started_at) >= 21)
					 OR (MONTH(started_at) = 7)
					 OR (MONTH(started_at) = 8)
					 OR (MONTH(started_at) = 9 AND DAY(started_at) <= 21) THEN 'Summer'
				  WHEN (MONTH(started_at) = 9 AND DAY(started_at) >= 22)
					 OR (MONTH(started_at) = 10)
					 OR (MONTH(started_at) = 11)
					 OR (MONTH(started_at) = 12 AND DAY(started_at) <= 20) THEN 'Fall'
				  WHEN  (MONTH(started_at) = 12 AND DAY(started_at) >= 21)
					 OR (MONTH(started_at) = 1)
					 OR (MONTH(started_at) = 2)
					 OR (MONTH(started_at) = 3 AND DAY(started_at) <= 19) THEN 'Winter'
			 END AS 'season'
			,CASE WHEN start_station_id = end_station_id AND DATEDIFF(mi,started_at,ended_at) <= 0 THEN 1
	 			  WHEN ended_at < started_at THEN 2
				  WHEN start_station_name LIKE '%test%' 
	 				OR start_station_name LIKE '%CHECK%' 
	 				OR end_station_name LIKE '%test%' 
	 				OR end_station_name LIKE '%CHECK%' THEN 3
	 			  ELSE 0 
	 		  END AS 'no_ride'
			 ,DATEDIFF(dy,started_at,ended_at) AS same_day_ride
	FROM trips202105
	UNION ALL
	SELECT	 ride_id
			,rideable_type
			,started_at
			,ended_at
			,start_station_name
			,start_station_id
			,LEFT(start_lat, 6) + ',' + LEFT(start_lng, 7) AS start_lat_lng
			,end_station_name
			,end_station_id
			,LEFT(end_lat, 6) + ',' + LEFT(end_lng, 7) AS end_lat_lng
			,member_casual
			,DATEDIFF(mi,started_at,ended_at) AS ride_time
			,DATEPART(HOUR, started_at) AS hour_ride
			,DATEPART(WEEKDAY, started_at) AS ride_weekday
			,DATENAME(WEEKDAY, started_at) AS ride_weekday_name
			,DATEPART(MONTH, started_at) AS month
			,DATENAME(MONTH, started_at) AS month_name
			,CASE WHEN DATEPART(HOUR, started_at) BETWEEN 5 AND 6 THEN '1.Early Morning'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 7 AND 9 THEN '2.Morning'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 10 AND 11 THEN '3.Late Morning'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 12 AND 15 THEN '4.Afternoon'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 16 AND 17 THEN '5.Late Afternoon'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 18 AND 20 THEN '6.Evening'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 21 AND 23 THEN '7.Night'
				  WHEN DATEPART(HOUR, started_at) = 00 THEN '7.Night'
				  WHEN DATEPART(HOUR, started_at) BETWEEN 1 AND 4 THEN '0.Middle of the night'
			 END AS 'day_part'
			,CASE WHEN (MONTH(started_at) = 3 AND DAY(started_at) >= 20)
					 OR (MONTH(started_at) = 4)
					 OR (MONTH(started_at) = 5)
					 OR (MONTH(started_at) = 6 AND DAY(started_at) <= 20) THEN 'Spring'
				  WHEN (MONTH(started_at) = 6 AND DAY(started_at) >= 21)
					 OR (MONTH(started_at) = 7)
					 OR (MONTH(started_at) = 8)
					 OR (MONTH(started_at) = 9 AND DAY(started_at) <= 21) THEN 'Summer'
				  WHEN (MONTH(started_at) = 9 AND DAY(started_at) >= 22)
					 OR (MONTH(started_at) = 10)
					 OR (MONTH(started_at) = 11)
					 OR (MONTH(started_at) = 12 AND DAY(started_at) <= 20) THEN 'Fall'
				  WHEN  (MONTH(started_at) = 12 AND DAY(started_at) >= 21)
					 OR (MONTH(started_at) = 1)
					 OR (MONTH(started_at) = 2)
					 OR (MONTH(started_at) = 3 AND DAY(started_at) <= 19) THEN 'Winter'
			 END AS 'season'
			,CASE WHEN start_station_id = end_station_id AND DATEDIFF(mi,started_at,ended_at) <= 0 THEN 1
	 			  WHEN ended_at < started_at THEN 2
				  WHEN start_station_name LIKE '%test%' 
	 				OR start_station_name LIKE '%CHECK%' 
	 				OR end_station_name LIKE '%test%' 
	 				OR end_station_name LIKE '%CHECK%' THEN 3
	 			  ELSE 0 
	 		  END AS 'no_ride'
			 ,DATEDIFF(dy,started_at,ended_at) AS same_day_ride
	FROM trips202106
	)
	SELECT *
	INTO trips
	FROM CTE
			
	
## Some Numbers - number of riders and average ride time

1. total rides

		SELECT COUNT (*) AS 'total rides'
		FROM trips
		WHERE no_ride = 0
		
	![image](https://user-images.githubusercontent.com/73856609/209859670-c7822b8a-3952-424b-a833-9e48c187e344.png)

2. Average ride time

		SELECT AVG (ride_time) AS 'ride_avg'
		FROM trips
		WHERE no_ride = 0
		
	![image](https://user-images.githubusercontent.com/73856609/209872781-2c5462dd-ab48-4ed1-8e1b-df5f2a6b32ef.png)


		SELECT   member_casual
			,AVG (ride_time) AS 'ride_avg'
		FROM trips
		WHERE no_ride = 0
		GROUP BY member_casual
		
	![image](https://user-images.githubusercontent.com/73856609/210017858-f718e027-e8c2-42bc-9294-0e6fbbb2fad2.png)

3. total rides by membership
	
		SELECT 	 member_casual
			,COUNT(*) AS rider_type
			,FORMAT(COUNT(*)*1.0 / SUM(COUNT(*)) OVER (), 'P') AS pct
		FROM trips
		WHERE no_ride = 0
		GROUP BY member_casual
	
	![image](https://user-images.githubusercontent.com/73856609/209859448-0d97bb97-30d5-4333-b59a-f568313cdc55.png)

	
4. total rides by bike type
	
		
		WITH  CTE AS
		(
			SELECT	 *
			,CASE WHEN rideable_type IN ('classic_bike','docked_bike') THEN 'regular_bike' 
			      ELSE 'electric_bike'
		 	 END AS bike_type
			 FROM trips
			 WHERE no_ride = 0
		)
		SELECT 	 bike_type
			,COUNT(*) AS rider_type
			,FORMAT(COUNT(*)*1.0 / SUM(COUNT(*)) OVER (), 'P') AS pct
		FROM CTE
		GROUP BY bike_type
		
		
	![image](https://user-images.githubusercontent.com/73856609/209859607-3b8f10ad-f23d-460e-b151-55f3c3b224da.png)

	
5. total rides by membership and bike type
	
		WITH  CTE AS
		(
			SELECT	 *
				,CASE WHEN rideable_type IN ('classic_bike','docked_bike') THEN 'regular_bike' 
				      ELSE 'electric_bike'
				 END AS bike_type
			FROM trips
			WHERE no_ride = 0
		)
		SELECT	member_casual
			,bike_type
			,COUNT(*) AS 'rider_type'
			,FORMAT(COUNT(*)*1.0 / SUM(COUNT(*)) OVER (), 'P') AS 'pctfrom_total'
			,FORMAT(COUNT(*) * 1.0 / SUM(COUNT(*)) OVER (PARTITION BY member_casual), 'P') AS 'pct_from_membership'
			,FORMAT(COUNT(*) * 1.0 / SUM(COUNT(*)) OVER (PARTITION BY bike_type), 'P') AS 'pct_from_bike_type'
		FROM CTE
		GROUP BY member_casual
			,bike_type 
		ORDER BY member_casual
			,bike_type
		
		
	![image](https://user-images.githubusercontent.com/73856609/209859745-4ae618bd-6b20-4832-9241-009a51791c53.png)

		
6. days of the week
	
	
		SELECT	 ride_weekday_name
			,COUNT(*) AS total_day
			,FORMAT(COUNT(*)*1.0 / SUM(COUNT(*)) OVER (), 'P') AS pct
		FROM trips
		WHERE no_ride = 0
		GROUP BY ride_weekday
			,ride_weekday_name
		ORDER BY ride_weekday
			,ride_weekday_name
			
	![image](https://user-images.githubusercontent.com/73856609/209867534-ea6ee5ab-9386-4a94-9b5b-713d195f01fe.png)
		
		
		SELECT	 member_casual
			,ride_weekday_name
			,COUNT(*) AS total_day
			,AVG(ride_time) AS 'avg'
			,FORMAT(COUNT(*)*1.0 / SUM(COUNT(*)) OVER (), 'P') AS pct
			,FORMAT(COUNT(*) * 1.0 / SUM(COUNT(*)) OVER (PARTITION BY member_casual), 'P') AS 'pct_from_membership'
			,FORMAT(COUNT(*) * 1.0 / SUM(COUNT(*)) OVER (PARTITION BY ride_weekday), 'P') AS 'pct_from_ride_weekday'
		FROM trips
		WHERE no_ride = 0
		GROUP BY member_casual
			,ride_weekday
			,ride_weekday_name
		ORDER BY member_casual
			,ride_weekday
			,ride_weekday_name
		
	![image](https://user-images.githubusercontent.com/73856609/210068262-2c0d74bf-b267-483b-b700-787676c400f4.png)

		
7. month
	
		SELECT	 month_name
			,COUNT(*) AS total_day
			,FORMAT(COUNT(*)*1.0 / SUM(COUNT(*)) OVER (), 'P') AS pct
		FROM trips
		WHERE no_ride = 0
		GROUP BY  month
		 	 ,month_name
		ORDER BY  month
		 	 ,month_name
			 
	![image](https://user-images.githubusercontent.com/73856609/209867672-e4ab3cb9-6e84-4fcf-a61c-e1d5b0d43d00.png)

	
8. seasons
	
		WITH CTE AS
		(
			SELECT *
				,CASE WHEN season = 'spring' THEN 1
				      WHEN season = 'summer' THEN 2
				      WHEN season = 'fall' THEN 3
				      WHEN season = 'winter' THEN 4
			END AS 'season_n'
			FROM trips
			WHERE no_ride = 0
		)
		SELECT	 season
			,COUNT(*) AS total_day
			,FORMAT(COUNT(*)*1.0 / SUM(COUNT(*)) OVER (), 'P') AS pct
		FROM CTE
		GROUP BY  season_n
			 ,season
		ORDER BY  season_n
			 ,season
				
	
	![image](https://user-images.githubusercontent.com/73856609/209867895-74b29f01-5928-4448-9c3c-7315946250b1.png)

	
10. part of the day

		WITH CTE AS
		(
			SELECT *
				,CASE WHEN day_part = '0.Middle of the night' THEN '01:00-04:59'
			  		WHEN day_part = '1.Early Morning' THEN '05:00-06:59'
			  		WHEN day_part = '2.Morning' THEN '07:00-09:59'
			  		WHEN day_part = '3.Late Morning' THEN '10:00-11:59'
			  		WHEN day_part = '4.Afternoon' THEN '12:00-15:59'
			  		WHEN day_part = '5.Late Afternoon' THEN '16:00-17:59'
			  		WHEN day_part = '6.Evening' THEN '18:00-20:59'
			  		WHEN day_part = '7.Night' THEN '21:00-00:59'
		 		END AS 'day_part_h'
			FROM trips
			WHERE no_ride = 0
		)
		SELECT	 day_part
			,day_part_h
			,COUNT(*) AS total_day
			,FORMAT(COUNT(*)*1.0 / SUM(COUNT(*)) OVER (), 'P') AS pct
		FROM CTE
		GROUP BY day_part
			,day_part_h
		ORDER BY day_part

	![image](https://user-images.githubusercontent.com/73856609/209868175-fe0a5a50-d2bb-4b14-9b79-e15da2b3f7ff.png)


### Quick Analysis

over 12 month 4,460,151 rides were recorded, 38,383 are not in the analysis (test rides, zero minutes or bad data) and 4,421,768 rides are in the final analysis (column name = 'no_ride')
- annual members are 56% of total rides
- annual riders ride an average of 15 minutes to an average of 40 minutes by casual riders
- riders (both annual and casual riders) prefer the regualr bike (75%) over the electric bikes (25%)
- casual riders ride more on weekends  - 57% of casual rides are on Fri-Sun. 
- annual riders ride a little bit more on weekends but not significantly
- when it's cold the number of rides drop - 73% of the rides or in the spring and summer
- the number of casual riders drops in the winter, only 25% of the rides in the winter are by casual riders
- between 10:00 and 21:00 the casual and annual riders use the bikes the same. members ride more in the earler morning hours (05:00 - 10:00, 70% + of the rides), and casual riders like the late hours more (21:00-05:00, 60% +/- of the rides)


## Further Analysis - Stations

in the next step we will analyze the characteristics of the riders by stations - top stations by membership, days of the week, seasons etc.

the number of stations is:

		SELECT	 COUNT(DISTINCT start_station_name) AS 'Total_start_stations'
			,COUNT(DISTINCT end_station_name) AS 'Total_end_stations'
		FROM 	 trips
		
		SELECT	 COUNT(DISTINCT start_station_name) AS 'Total_start_stations'
			,COUNT(DISTINCT end_station_name) AS 'Total_end_stations'
		FROM 	 trips
		WHERE no_ride = 0



all atations:	
![image](https://user-images.githubusercontent.com/73856609/211014914-dd206105-a99e-43d4-81f5-4d65336e8380.png)
stations:
![image](https://user-images.githubusercontent.com/73856609/213883862-27a2160c-510a-48fa-8c35-2b72c6aec6fb.png)


a. top stations
		
		SELECT	 start_station_name 
			,COUNT(*) AS 'rides_per _station'
			,FORMAT(COUNT(*)*1.0 / SUM(COUNT(*)) OVER (), 'P') AS pct
		FROM trips
		WHERE start_station_name IS NOT NULL
		AND no_ride = 0
		GROUP BY start_station_name
		ORDER BY rides_per_station DESC
	
	

![image](https://user-images.githubusercontent.com/73856609/213883920-5041a601-240c-4fed-a8ff-572cda376b3c.png)
	
b. stations by membership

		SELECT	 s.start_station_name
			,s.station_rnk
			,s.station
			,c.casual_rnk
			,c.casual
			,m.member_rnk
			,m.member
		FROM
		(
		SELECT	 start_station_name
			,ROW_NUMBER() OVER (ORDER BY COUNT(ride_id) DESC) AS 'station_rnk'
			,COUNT(ride_id) AS 'station'
		FROM trips
		WHERE start_station_name IS NOT NULL 
		      AND no_ride = 0
		GROUP BY start_station_name
		)s 
		JOIN
		(
		SELECT	 start_station_name
			,COUNT(ride_id) AS 'casual'
			,ROW_NUMBER() OVER (ORDER BY COUNT(ride_id) DESC) AS 'casual_rnk'
		FROM trips
		WHERE start_station_name IS NOT NULL 
		      AND member_casual = 'casual'
		      AND no_ride = 0
		GROUP BY start_station_name
		) c ON s.start_station_name = c.start_station_name
		JOIN
		(
		SELECT	 start_station_name
			,COUNT(ride_id) AS 'member'
			,ROW_NUMBER() OVER (ORDER BY COUNT(ride_id) DESC) AS 'member_rnk'
		FROM trips
		WHERE start_station_name IS NOT NULL 
		      AND member_casual = 'member'
		      AND no_ride = 0
		GROUP BY start_station_name
		) m
		ON c.start_station_name = m.start_station_name
		ORDER BY s.station DESC
	

![image](https://user-images.githubusercontent.com/73856609/213884011-0ce4f53d-eecc-41f8-9a4e-ffe6c414ee77.png)



c. stations by weekday - weekend

		SELECT	 s.start_station_name
			,s.station_rnk
			,s.station
			,d.weekday_rnk
			,d.weekday
			,e.weekend_rnk
			,e.weekend
			,d.weekday_rnk - e.weekend_rnk AS 'weekend_change'
		FROM
		(
		SELECT start_station_name
			,COUNT(ride_id) AS 'station'
			,ROW_NUMBER() OVER (ORDER BY COUNT(ride_id) DESC) AS 'station_rnk'
		FROM trips
		WHERE start_station_name IS NOT NULL 
		      AND no_ride = 0
		GROUP BY start_station_name
		) s
		JOIN
		(
		SELECT	  start_station_name
			,COUNT(ride_id) AS 'weekday'
			,ROW_NUMBER() OVER (ORDER BY COUNT(ride_id) DESC) AS 'weekday_rnk'
		FROM trips
		WHERE start_station_name IS NOT NULL 
		AND ride_weekday  IN (2,3,4,5)
		AND no_ride = 0
		GROUP BY start_station_name
		) d ON s.start_station_name = d.start_station_name
		JOIN
		(
		SELECT	  start_station_name
			,COUNT(ride_id) AS 'weekend'
			,ROW_NUMBER() OVER (ORDER BY COUNT(ride_id) DESC) AS 'weekend_rnk'
		FROM trips
		WHERE start_station_name IS NOT NULL 
		AND ride_weekday  IN (6,7,1)
		AND no_ride = 0
		GROUP BY start_station_name
		) e ON d.start_station_name = e.start_station_name
		ORDER BY station DESC


![image](https://user-images.githubusercontent.com/73856609/213884417-051853f7-483d-4e9e-b08c-08cf79956cbb.png)


d. stations by weekday casual
		
			SELECT	s.start_station_name
			,s.station_rnk
			,s.station
			,c.casual_rnk
			,d.weekday_rnk
			,e.weekend_rnk
			,d.weekday_rnk - e.weekend_rnk AS 'weekend_change'
			FROM
			(
			SELECT start_station_name
				,COUNT(ride_id) AS 'station'
				,ROW_NUMBER() OVER (ORDER BY COUNT(ride_id) DESC) AS 'station_rnk'
			FROM trips
			WHERE start_station_name IS NOT NULL
			AND no_ride = 0
			GROUP BY start_station_name
			) s
			JOIN
			(
				SELECT start_station_name
				,COUNT(ride_id) AS 'casual'
				,ROW_NUMBER() OVER (ORDER BY COUNT(ride_id) DESC) AS 'casual_rnk'
			FROM trips
			WHERE start_station_name IS NOT NULL 
			AND member_casual = 'casual'
			AND no_ride = 0
			GROUP BY start_station_name
			) c ON s.start_station_name = c.start_station_name
			JOIN
			(
			SELECT	  start_station_name
				,COUNT(ride_id) AS 'weekday'
				,ROW_NUMBER() OVER (ORDER BY COUNT(ride_id) DESC) AS 'weekday_rnk'
			FROM trips
			WHERE start_station_name IS NOT NULL 
			AND ride_weekday  IN (2,3,4,5)
			AND member_casual = 'casual'
			AND no_ride = 0
			GROUP BY start_station_name
			) d ON c.start_station_name = d.start_station_name
			JOIN
			(
			SELECT	  start_station_name
				,COUNT(ride_id) AS 'weekend'
				,ROW_NUMBER() OVER (ORDER BY COUNT(ride_id) DESC) AS 'weekend_rnk'
			FROM trips
			WHERE start_station_name IS NOT NULL 
			AND ride_weekday  IN (6,7,1)
			AND member_casual = 'casual'
			AND no_ride = 0
			GROUP BY start_station_name
			) e ON d.start_station_name = e.start_station_name
			ORDER BY casual DESC
			

![image](https://user-images.githubusercontent.com/73856609/213884473-439ea870-2387-45d3-a445-650ce181672b.png)

d. stations by weekday member


			SELECT	s.start_station_name
				,s.station_rnk
				,s.station
				,c.member_rnk
				,d.weekday_rnk
				,e.weekend_rnk
				,d.weekday_rnk - e.weekend_rnk AS 'weekend_change'
			FROM
			(
			SELECT start_station_name
				,COUNT(ride_id) AS 'station'
				,ROW_NUMBER() OVER (ORDER BY COUNT(ride_id) DESC) AS 'station_rnk'
			FROM trips
			WHERE start_station_name IS NOT NULL 
			AND no_ride = 0
			GROUP BY start_station_name
			) s
			JOIN
			(
				SELECT start_station_name
				,COUNT(ride_id) AS 'member'
				,ROW_NUMBER() OVER (ORDER BY COUNT(ride_id) DESC) AS 'member_rnk'
			FROM trips
			WHERE start_station_name IS NOT NULL 
			AND member_casual = 'member'
			AND no_ride = 0
			GROUP BY start_station_name
			) c ON s.start_station_name = c.start_station_name
			JOIN
			(
			SELECT	  start_station_name
				,COUNT(ride_id) AS 'weekday'
				,ROW_NUMBER() OVER (ORDER BY COUNT(ride_id) DESC) AS 'weekday_rnk'
			FROM trips
			WHERE start_station_name IS NOT NULL 
			AND ride_weekday  IN (2,3,4,5)
			AND member_casual = 'member'
			AND no_ride = 0
			GROUP BY start_station_name
			) d ON c.start_station_name = d.start_station_name
			JOIN
			(
			SELECT	  start_station_name
				,COUNT(ride_id) AS 'weekend'
				,ROW_NUMBER() OVER (ORDER BY COUNT(ride_id) DESC) AS 'weekend_rnk'
			FROM trips
			WHERE start_station_name IS NOT NULL 
			AND ride_weekday  IN (6,7,1)
			AND member_casual = 'member'
			AND no_ride = 0
			GROUP BY start_station_name
			) e ON d.start_station_name = e.start_station_name
			ORDER BY member DESC
			
			
![image](https://user-images.githubusercontent.com/73856609/213884542-914d9f69-3c94-47a4-a183-31010f3ffedc.png)


e. stations by season
			
			SELECT	s.start_station_name
				,s.station_rnk
				,s.station
				,sp.spring_rnk
				,FORMAT(sp.spring*1.0 / s.station, 'P') AS 'spring_p'
				,su.summer_rnk
				,FORMAT(su.summer*1.0 / s.station, 'P') AS 'summer_p'
				,f.fall_rnk
				,FORMAT(f.fall*1.0 / s.station, 'P') AS 'fall_p'
				,w.winter_rnk
				,FORMAT(w.winter*1.0 / s.station, 'P') AS 'winter_p'
			FROM
			(
			SELECT start_station_name
				,COUNT(ride_id) AS 'station'
				,ROW_NUMBER() OVER (ORDER BY COUNT(ride_id) DESC) AS 'station_rnk'
			FROM trips
			WHERE start_station_name IS NOT NULL
			AND no_ride = 0
			GROUP BY start_station_name
			) s
			LEFT JOIN
			(
			SELECT start_station_name
				,COUNT(ride_id) AS 'spring'
				,ROW_NUMBER() OVER (ORDER BY COUNT(ride_id) DESC) AS 'spring_rnk'
			FROM trips
			WHERE start_station_name IS NOT NULL 
			AND season = 'spring'
			AND no_ride = 0
			GROUP BY start_station_name
			) sp ON s.start_station_name = sp.start_station_name
			LEFT JOIN
			(
			SELECT	  start_station_name
				,COUNT(ride_id) AS 'summer'
				,ROW_NUMBER() OVER (ORDER BY COUNT(ride_id) DESC) AS 'summer_rnk'
			FROM trips
			WHERE start_station_name IS NOT NULL 
			AND season = 'summer'
			AND no_ride = 0
			GROUP BY start_station_name
			) su ON sp.start_station_name = su.start_station_name
			LEFT JOIN
			(
			SELECT	  start_station_name
				,COUNT(ride_id) AS 'fall'
			,ROW_NUMBER() OVER (ORDER BY COUNT(ride_id) DESC) AS 'fall_rnk'
			FROM trips
			WHERE start_station_name IS NOT NULL 
			AND season = 'fall'
			AND no_ride = 0
			GROUP BY start_station_name
			) f ON su.start_station_name = f.start_station_name
			LEFT JOIN
			(
			SELECT	  start_station_name
				,COUNT(ride_id) AS 'winter'
				,ROW_NUMBER() OVER (ORDER BY COUNT(ride_id) DESC) AS 'winter_rnk'
			FROM trips
			WHERE start_station_name IS NOT NULL 
			AND season = 'winter'
			AND no_ride = 0
			GROUP BY start_station_name
			) w ON f.start_station_name = w.start_station_name
			ORDER BY station DESC
			
			
![image](https://user-images.githubusercontent.com/73856609/213884603-eadae394-c32c-46d1-a410-66c89eeabbca.png)


f. stations by season - casual


			
			SELECT	s.start_station_name
				,s.station_rnk
				,s.station
				,c.casual_rnk
				,c.casual
				,sp.spring_rnk
				,FORMAT(sp.spring*1.0 / c.casual, 'P') AS 'spring_p'
				,su.summer_rnk
				,FORMAT(su.summer*1.0 / c.casual, 'P') AS 'summer_p'
				,f.fall_rnk
				,FORMAT(f.fall*1.0 / c.casual, 'P') AS 'fall_p'
				,w.winter_rnk
				,FORMAT(w.winter*1.0 / c.casual, 'P') AS 'winter_p'
			FROM
			(
			SELECT start_station_name
				,COUNT(ride_id) AS 'casual'
				,ROW_NUMBER() OVER (ORDER BY COUNT(ride_id) DESC) AS 'casual_rnk'
			FROM trips
			WHERE start_station_name IS NOT NULL 
			AND member_casual = 'casual'
			AND no_ride = 0
			GROUP BY start_station_name
			) c
			JOIN
			(
			SELECT start_station_name
				,COUNT(ride_id) AS 'station'
				,ROW_NUMBER() OVER (ORDER BY COUNT(ride_id) DESC) AS 'station_rnk'
			FROM trips
			WHERE start_station_name IS NOT NULL 
			AND no_ride = 0
			GROUP BY start_station_name
			) s ON c.start_station_name = s.start_station_name
			JOIN
			(
			SELECT start_station_name
				,COUNT(ride_id) AS 'spring'
				,ROW_NUMBER() OVER (ORDER BY COUNT(ride_id) DESC) AS 'spring_rnk'
			FROM trips
			WHERE start_station_name IS NOT NULL 
			AND season = 'spring'
			AND member_casual = 'casual'
			AND no_ride = 0
			GROUP BY start_station_name
			) sp ON s.start_station_name = sp.start_station_name
			JOIN
			(
			SELECT	  start_station_name
				,COUNT(ride_id) AS 'summer'
				,ROW_NUMBER() OVER (ORDER BY COUNT(ride_id) DESC) AS 'summer_rnk'
			FROM trips
			WHERE start_station_name IS NOT NULL 
			AND season = 'summer'
			AND member_casual = 'casual'
			AND no_ride = 0
			GROUP BY start_station_name
			) su ON sp.start_station_name = su.start_station_name
			JOIN
			(
			SELECT	  start_station_name
				,COUNT(ride_id) AS 'fall'
				,ROW_NUMBER() OVER (ORDER BY COUNT(ride_id) DESC) AS 'fall_rnk'
			FROM trips
			WHERE start_station_name IS NOT NULL 
			AND season = 'fall'
			AND member_casual = 'casual'
			AND no_ride = 0
			GROUP BY start_station_name
			) f ON su.start_station_name = f.start_station_name
			JOIN
			(
			SELECT	  start_station_name
				,COUNT(ride_id) AS 'winter'
				,ROW_NUMBER() OVER (ORDER BY COUNT(ride_id) DESC) AS 'winter_rnk'
			FROM trips
			WHERE start_station_name IS NOT NULL 
			AND season = 'winter'
			AND member_casual = 'casual'
			AND no_ride = 0
			GROUP BY start_station_name
			) w ON f.start_station_name = w.start_station_name
			ORDER BY casual DESC
			

![image](https://user-images.githubusercontent.com/73856609/213565850-78c7118e-40b9-4f47-8d21-dc34e2dd9d35.png)


g. stations by season - member

			
			SELECT	s.start_station_name
				,s.station_rnk
				,s.station
				,c.member_rnk
				,c.member
				,sp.spring_rnk
				,FORMAT(sp.spring*1.0 / c.member, 'P') AS 'spring_p'
				,su.summer_rnk
				,FORMAT(su.summer*1.0 / c.member, 'P') AS 'summer_p'
				,f.fall_rnk
				,FORMAT(f.fall*1.0 / c.member, 'P') AS 'fall_p'
				,w.winter_rnk
				,FORMAT(w.winter*1.0 / c.member, 'P') AS 'winter_p'
			FROM
			(
			SELECT start_station_name
				,COUNT(ride_id) AS 'member'
				,ROW_NUMBER() OVER (ORDER BY COUNT(ride_id) DESC) AS 'member_rnk'
			FROM trips
			WHERE start_station_name IS NOT NULL 
			AND member_casual = 'member'
			AND no_ride = 0
			GROUP BY start_station_name
			) c
			JOIN
			(
			SELECT start_station_name
				,COUNT(ride_id) AS 'station'
				,ROW_NUMBER() OVER (ORDER BY COUNT(ride_id) DESC) AS 'station_rnk'
			FROM trips
			WHERE start_station_name IS NOT NULL 
			AND no_ride = 0
			GROUP BY start_station_name
			) s ON c.start_station_name = s.start_station_name
			JOIN
			(
			SELECT start_station_name
				,COUNT(ride_id) AS 'spring'
				,ROW_NUMBER() OVER (ORDER BY COUNT(ride_id) DESC) AS 'spring_rnk'
			FROM trips
			WHERE start_station_name IS NOT NULL 
			AND season = 'spring'
			AND member_casual = 'member'
			AND no_ride = 0
			GROUP BY start_station_name
			) sp ON s.start_station_name = sp.start_station_name
			JOIN
			(
			SELECT	  start_station_name
				,COUNT(ride_id) AS 'summer'
				,ROW_NUMBER() OVER (ORDER BY COUNT(ride_id) DESC) AS 'summer_rnk'
			FROM trips
			WHERE start_station_name IS NOT NULL 
			AND season = 'summer'
			AND member_casual = 'member'
			AND no_ride = 0
			GROUP BY start_station_name
			) su ON sp.start_station_name = su.start_station_name
			JOIN
			(
			SELECT	  start_station_name
				,COUNT(ride_id) AS 'fall'
				,ROW_NUMBER() OVER (ORDER BY COUNT(ride_id) DESC) AS 'fall_rnk'
			FROM trips
			WHERE start_station_name IS NOT NULL 
			AND season = 'fall'
			AND member_casual = 'member'
			AND no_ride = 0
			GROUP BY start_station_name
			) f ON su.start_station_name = f.start_station_name
			JOIN
			(
			SELECT	  start_station_name
				,COUNT(ride_id) AS 'winter'
				,ROW_NUMBER() OVER (ORDER BY COUNT(ride_id) DESC) AS 'winter_rnk'
			FROM trips
			WHERE start_station_name IS NOT NULL 
			AND season = 'winter'
			AND member_casual = 'member'
			AND no_ride = 0
			GROUP BY start_station_name
			) w ON f.start_station_name = w.start_station_name
			ORDER BY member DESC


![image](https://user-images.githubusercontent.com/73856609/213885026-eec2c4fc-b748-4ed5-8610-2ab6b00ffbf2.png)


### Part of Day

a. station by part of the day

			SELECT	s.start_station_name
				,s.station_rnk
				,s.station
				,mn.Middle_of_the_night_rnk
				,em.Early_Morning_rnk
				,m.Morning_rnk
				,lm.Late_Morning_rnk
				,a.Afternoon_rnk
				,la.Late_Afternoon_rnk
				,e.Evening_rnk
				,n.Night_rnk
			FROM
			(
			SELECT start_station_name
				,COUNT(ride_id) AS 'station'
				,ROW_NUMBER() OVER (ORDER BY COUNT(ride_id) DESC) AS 'station_rnk'
			FROM trips
			WHERE start_station_name IS NOT NULL 
			AND no_ride = '0'
			GROUP BY start_station_name
			) s
			LEFT JOIN
			(
			SELECT start_station_name
				,COUNT(ride_id) AS 'Middle_of_the_night'
				,ROW_NUMBER() OVER (ORDER BY COUNT(ride_id) DESC) AS 'Middle_of_the_night_rnk'
			FROM trips
			WHERE start_station_name IS NOT NULL 
			AND day_part = '0.Middle of the night'
			AND no_ride = '0'
			GROUP BY start_station_name
			) mn ON s.start_station_name = mn.start_station_name
			LEFT JOIN
			(
			SELECT start_station_name
				,COUNT(ride_id) AS 'Early_Morning'
				,ROW_NUMBER() OVER (ORDER BY COUNT(ride_id) DESC) AS 'Early_Morning_rnk'
			FROM trips
			WHERE start_station_name IS NOT NULL 
			AND day_part = '1.Early Morning'
			AND no_ride = '0'
			GROUP BY start_station_name
			) em ON mn.start_station_name = em.start_station_name
			LEFT JOIN
			(
			SELECT start_station_name
				,COUNT(ride_id) AS 'Morning'
				,ROW_NUMBER() OVER (ORDER BY COUNT(ride_id) DESC) AS 'Morning_rnk'
			FROM trips
			WHERE start_station_name IS NOT NULL 
			AND day_part = '2.Morning'
			AND no_ride = '0'
			GROUP BY start_station_name
			) m ON em.start_station_name = m.start_station_name
			LEFT JOIN
			(
			SELECT start_station_name
				,COUNT(ride_id) AS 'Late_Morning'
				,ROW_NUMBER() OVER (ORDER BY COUNT(ride_id) DESC) AS 'Late_Morning_rnk'
			FROM trips
			WHERE start_station_name IS NOT NULL 
			AND day_part = '3.Late Morning'
			AND no_ride = '0'
			GROUP BY start_station_name
			) lm ON m.start_station_name = lm.start_station_name
			LEFT JOIN
			(
			SELECT start_station_name
				,COUNT(ride_id) AS 'Afternoon'
				,ROW_NUMBER() OVER (ORDER BY COUNT(ride_id) DESC) AS 'Afternoon_rnk'
			FROM trips
			WHERE start_station_name IS NOT NULL 
			AND day_part = '4.Afternoon'
			AND no_ride = '0'
			GROUP BY start_station_name
			) a ON lm.start_station_name = a.start_station_name
			LEFT JOIN
			(
			SELECT start_station_name
				,COUNT(ride_id) AS 'Late_Afternoon'
				,ROW_NUMBER() OVER (ORDER BY COUNT(ride_id) DESC) AS 'Late_Afternoon_rnk'
			FROM trips
			WHERE start_station_name IS NOT NULL 
			AND day_part = '5.Late Afternoon'
			AND no_ride = '0'
			GROUP BY start_station_name
			) la  ON a.start_station_name = la.start_station_name
			LEFT JOIN 
			(
			SELECT start_station_name
				,COUNT(ride_id) AS 'Evening'
				,ROW_NUMBER() OVER (ORDER BY COUNT(ride_id) DESC) AS 'Evening_rnk'
			FROM trips
			WHERE start_station_name IS NOT NULL 
			AND day_part = '6.Evening'
			AND no_ride = '0'
			GROUP BY start_station_name
			) e ON la.start_station_name = e.start_station_name
			LEFT JOIN
			(
			SELECT start_station_name
				,COUNT(ride_id) AS 'Night'
				,ROW_NUMBER() OVER (ORDER BY COUNT(ride_id) DESC) AS 'Night_rnk'
			FROM trips
			WHERE start_station_name IS NOT NULL 
			AND day_part = '7.Night'
			AND no_ride = '0'
			GROUP BY start_station_name
			) n ON e.start_station_name = n.start_station_name
			ORDER BY station DESC	
			
			

![image](https://user-images.githubusercontent.com/73856609/213590112-8d1dbc90-a829-489d-bcc1-0c9359a94790.png)

b. station by part of the day - casual

			SELECT	s.start_station_name
				,s.station_rnk
				,s.station
				,c.casual_rnk
				,c.casual
				,mn.Middle_of_the_night_rnk
				,em.Early_Morning_rnk
				,m.Morning_rnk
				,lm.Late_Morning_rnk
				,a.Afternoon_rnk
				,la.Late_Afternoon_rnk
				,e.Evening_rnk
				,n.Night_rnk
			FROM
			(
			SELECT start_station_name
				,COUNT(ride_id) AS 'casual'
				,ROW_NUMBER() OVER (ORDER BY COUNT(ride_id) DESC) AS 'casual_rnk'
			FROM trips
			WHERE start_station_name IS NOT NULL 
			AND no_ride = '0'
			AND member_casual = 'casual'
			GROUP BY start_station_name
			)c 
			LEFT JOIN 
			(
			SELECT start_station_name
				,COUNT(ride_id) AS 'station'
				,ROW_NUMBER() OVER (ORDER BY COUNT(ride_id) DESC) AS 'station_rnk'
			FROM trips
			WHERE start_station_name IS NOT NULL 
			AND no_ride = '0'
			GROUP BY start_station_name
			) s ON c.start_station_name = s.start_station_name
			LEFT JOIN
			(
			SELECT start_station_name
				,COUNT(ride_id) AS 'Middle_of_the_night'
				,ROW_NUMBER() OVER (ORDER BY COUNT(ride_id) DESC) AS 'Middle_of_the_night_rnk'
			FROM trips
			WHERE start_station_name IS NOT NULL 
			AND day_part = '0.Middle of the night'
			AND no_ride = '0'
			AND member_casual = 'casual'
			GROUP BY start_station_name
			) mn ON s.start_station_name = mn.start_station_name
			LEFT JOIN
			(
			SELECT start_station_name
				,COUNT(ride_id) AS 'Early_Morning'
				,ROW_NUMBER() OVER (ORDER BY COUNT(ride_id) DESC) AS 'Early_Morning_rnk'
			FROM trips
			WHERE start_station_name IS NOT NULL 
			AND day_part = '1.Early Morning'
			AND no_ride = '0'
			AND member_casual = 'casual'
			GROUP BY start_station_name
			) em ON mn.start_station_name = em.start_station_name
			LEFT JOIN
			(
			SELECT start_station_name
				,COUNT(ride_id) AS 'Morning'
				,ROW_NUMBER() OVER (ORDER BY COUNT(ride_id) DESC) AS 'Morning_rnk'
			FROM trips
			WHERE start_station_name IS NOT NULL 
			AND day_part = '2.Morning'
			AND no_ride = '0'
			AND member_casual = 'casual'
			GROUP BY start_station_name
			) m ON em.start_station_name = m.start_station_name
			LEFT JOIN
			(
			SELECT start_station_name
				,COUNT(ride_id) AS 'Late_Morning'
				,ROW_NUMBER() OVER (ORDER BY COUNT(ride_id) DESC) AS 'Late_Morning_rnk'
			FROM trips
			WHERE start_station_name IS NOT NULL 
			AND day_part = '3.Late Morning'
			AND no_ride = '0'
			AND member_casual = 'casual'
			GROUP BY start_station_name
			) lm ON m.start_station_name = lm.start_station_name
			LEFT JOIN
			(
			SELECT start_station_name
				,COUNT(ride_id) AS 'Afternoon'
				,ROW_NUMBER() OVER (ORDER BY COUNT(ride_id) DESC) AS 'Afternoon_rnk'
			FROM trips
			WHERE start_station_name IS NOT NULL 
			AND day_part = '4.Afternoon'
			AND no_ride = '0'
			AND member_casual = 'casual'
			GROUP BY start_station_name
			) a ON lm.start_station_name = a.start_station_name
			LEFT JOIN
			(
			SELECT start_station_name
				,COUNT(ride_id) AS 'Late_Afternoon'
				,ROW_NUMBER() OVER (ORDER BY COUNT(ride_id) DESC) AS 'Late_Afternoon_rnk'
			FROM trips
			WHERE start_station_name IS NOT NULL 
			AND day_part = '5.Late Afternoon'
			AND no_ride = '0'
			AND member_casual = 'casual'
			GROUP BY start_station_name
			) la  ON a.start_station_name = la.start_station_name
			LEFT JOIN 
			(
			SELECT start_station_name
				,COUNT(ride_id) AS 'Evening'
				,ROW_NUMBER() OVER (ORDER BY COUNT(ride_id) DESC) AS 'Evening_rnk'
			FROM trips
			WHERE start_station_name IS NOT NULL 
			AND day_part = '6.Evening'
			AND no_ride = '0'
			AND member_casual = 'casual'
			GROUP BY start_station_name
			) e ON la.start_station_name = e.start_station_name
			LEFT JOIN
			(
			SELECT start_station_name
				,COUNT(ride_id) AS 'Night'
				,ROW_NUMBER() OVER (ORDER BY COUNT(ride_id) DESC) AS 'Night_rnk'
			FROM trips
			WHERE start_station_name IS NOT NULL 
			AND day_part = '7.Night'
			AND no_ride = '0'
			AND member_casual = 'casual'
			GROUP BY start_station_name
			) n ON e.start_station_name = n.start_station_name
			ORDER BY casual DESC
			
			
![image](https://user-images.githubusercontent.com/73856609/213644947-504fee24-b1e8-4a89-95ad-6a8cd11cab2a.png)

	
c. station by part of the day - members


			SELECT	s.start_station_name
				,s.station_rnk
				,s.station
				,c.member_rnk
				,c.member
				,mn.Middle_of_the_night_rnk
				,em.Early_Morning_rnk
				,m.Morning_rnk
				,lm.Late_Morning_rnk
				,a.Afternoon_rnk
				,la.Late_Afternoon_rnk
				,e.Evening_rnk
				,n.Night_rnk
			FROM
			(
			SELECT start_station_name
				,COUNT(ride_id) AS 'member'
				,ROW_NUMBER() OVER (ORDER BY COUNT(ride_id) DESC) AS 'member_rnk'
			FROM trips
			WHERE start_station_name IS NOT NULL 
			AND no_ride = '0'
			AND member_casual = 'member'
			GROUP BY start_station_name
			)c 
			LEFT JOIN 
			(
			SELECT start_station_name
				,COUNT(ride_id) AS 'station'
				,ROW_NUMBER() OVER (ORDER BY COUNT(ride_id) DESC) AS 'station_rnk'
			FROM trips
			WHERE start_station_name IS NOT NULL 
			AND no_ride = '0'
			GROUP BY start_station_name
			) s ON c.start_station_name = s.start_station_name
			LEFT JOIN
			(
			SELECT start_station_name
				,COUNT(ride_id) AS 'Middle_of_the_night'
				,ROW_NUMBER() OVER (ORDER BY COUNT(ride_id) DESC) AS 'Middle_of_the_night_rnk'
			FROM trips
			WHERE start_station_name IS NOT NULL 
			AND day_part = '0.Middle of the night'
			AND no_ride = '0'
			AND member_casual = 'member'
			GROUP BY start_station_name
			) mn ON s.start_station_name = mn.start_station_name
			LEFT JOIN
			(
			SELECT start_station_name
				,COUNT(ride_id) AS 'Early_Morning'
				,ROW_NUMBER() OVER (ORDER BY COUNT(ride_id) DESC) AS 'Early_Morning_rnk'
			FROM trips
			WHERE start_station_name IS NOT NULL 
			AND day_part = '1.Early Morning'
			AND no_ride = '0'
			AND member_casual = 'member'
			GROUP BY start_station_name
			) em ON mn.start_station_name = em.start_station_name
			LEFT JOIN
			(
			SELECT start_station_name
				,COUNT(ride_id) AS 'Morning'
				,ROW_NUMBER() OVER (ORDER BY COUNT(ride_id) DESC) AS 'Morning_rnk'
			FROM trips
			WHERE start_station_name IS NOT NULL 
			AND day_part = '2.Morning'
			AND no_ride = '0'
			AND member_casual = 'member'
			GROUP BY start_station_name
			) m ON em.start_station_name = m.start_station_name
			LEFT JOIN
			(
			SELECT start_station_name
				,COUNT(ride_id) AS 'Late_Morning'
				,ROW_NUMBER() OVER (ORDER BY COUNT(ride_id) DESC) AS 'Late_Morning_rnk'
			FROM trips
			WHERE start_station_name IS NOT NULL 
			AND day_part = '3.Late Morning'
			AND no_ride = '0'
			AND member_casual = 'member'
			GROUP BY start_station_name
			) lm ON m.start_station_name = lm.start_station_name
			LEFT JOIN
			(
			SELECT start_station_name
				,COUNT(ride_id) AS 'Afternoon'
				,ROW_NUMBER() OVER (ORDER BY COUNT(ride_id) DESC) AS 'Afternoon_rnk'
			FROM trips
			WHERE start_station_name IS NOT NULL 
			AND day_part = '4.Afternoon'
			AND no_ride = '0'
			AND member_casual = 'member'
			GROUP BY start_station_name
			) a ON lm.start_station_name = a.start_station_name
			LEFT JOIN
			(
			SELECT start_station_name
				,COUNT(ride_id) AS 'Late_Afternoon'
				,ROW_NUMBER() OVER (ORDER BY COUNT(ride_id) DESC) AS 'Late_Afternoon_rnk'
			FROM trips
			WHERE start_station_name IS NOT NULL 
			AND day_part = '5.Late Afternoon'
			AND no_ride = '0'
			AND member_casual = 'member'
			GROUP BY start_station_name
			) la  ON a.start_station_name = la.start_station_name
			LEFT JOIN 
			(
			SELECT start_station_name
				,COUNT(ride_id) AS 'Evening'
				,ROW_NUMBER() OVER (ORDER BY COUNT(ride_id) DESC) AS 'Evening_rnk'
			FROM trips
			WHERE start_station_name IS NOT NULL 
			AND day_part = '6.Evening'
			AND no_ride = '0'
			AND member_casual = 'member'
			GROUP BY start_station_name
			) e ON la.start_station_name = e.start_station_name
			LEFT JOIN
			(
			SELECT start_station_name
				,COUNT(ride_id) AS 'Night'
				,ROW_NUMBER() OVER (ORDER BY COUNT(ride_id) DESC) AS 'Night_rnk'
			FROM trips
			WHERE start_station_name IS NOT NULL 
			AND day_part = '7.Night'
			AND no_ride = '0'
			AND member_casual = 'member'
			GROUP BY start_station_name
			) n ON e.start_station_name = n.start_station_name
			ORDER BY member DESC
			
			
![image](https://user-images.githubusercontent.com/73856609/214695383-b7505951-2cd6-4d16-9e49-0d74a593e636.png)

shorturl.at/bstFH

![image](https://user-images.githubusercontent.com/73856609/217914399-5ac9ab3f-ed03-4119-bddd-b4747533fe75.png)



## Quick Analysis – stations
In the database there are 713 stations but only 710 count for the analysis with three stations not in the final analysis and are characterized as test rides
- The top start stations differ between the casual and annual riders, the top start stations - Streeter Dr & Grand Ave and Lake Shore Dr , Monroe St & Millennium Park are ranked one, two and three for casual riders but ranked 24th ,41st  and 143rd by annual riders
- In the top 30 start stations, Theater on the Lake and Clark St & Lincoln Ave stations have a similar rank by casual and annual riders (5th - 4th and 14th - 13th )
- Weekdays and weekends are the same top stations for casual riders. For annual rides excluding the top two start stations all other start stations differ between weekdays and weekends with St. Clair St & Erie St station dropping 43 ranks between weekdays and weekends
- Seasons are mostly the same. for annual members, the top start stations are roughly the same amount with a small decrease in the summer (Kingsbury St & Kinzie St dropping from ranked 3rd to 7th , Wells St & Elm St dropping from 4th to 13th ). Casual members ride roughly the same amount from the top start stations all year round. In the second tier of start stations there is an increase in station rank in the fall and winter and it might be explained by the drop in total rides by casual riders during the fall and winter and the rides from those start stations being a higher percent of total casual rides 
- During the day the start station rank changes, for casual members the top stations are popular during the late morning hours through the afternoon and evening (Streeter Dr & Grand Ave , Lake Shore Dr and Monroe St , Millennium Park) but drop significantly during the middle of the night and the morning hours. For annual members with the exception of Clark St & Elm St that is ranked one and two all day and night the top ranked start stations change during the day, Theater on the Lake is ranked an average of two during from the late morning till the evening and drops to an average rank of 100 during the night and early morning hours 


## Concultion (part one)
In this analysis we defined the characteristics of the two types of membership riders, tha annual riders and the casual rider.

- the annual members use the bikes for there regular routines - ride all year round going to work or school and using the bikes for shorter rides, and mostly on weekdays.
- the casual members go on longer rides, ride more on weekends and during non work hours and have the same popular stations durig all different parts of the day, day of the week and seasons.

### Marketing Ideas from the Analysis
#### Creating annual packages for the different types of casual users 

-	3-4 day/Weeklong packages for tourists. 
	This package will give the tourist a weekly pass with a reduced price for the usage by minute. 
	The benefit of this package is a way to explore the city without having to consider the price of a few "one time using" of the bike. 
	This package also will give the visitor a unique experience of the city ride through the streets.  
-	Seasonal packages for the spring and/or summer. This package will give the casual rider the opportunity to enjoy the city during the nice weather in spring and summer without the need to think twice about a nice afternoon at the park or a day of site seeing in the great city.
-	Weekend packages. A reduced prices for a  weekend long pass with the goal of converting the weekend users into weekday users.


#### Casual suburbs targeting

The casual riders not in the prime tourist locations don't have one clear thread, they probably use the bikes for an occasional need (demographic information about the users was not in the data). Focusing on this random group of users "from the suburbs" by advertising the benefits of the annual membership to increase there conversion of membership type




### further analysis will be added as soon as possible 
