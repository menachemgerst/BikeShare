# BikeShare

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
Answering these three questions can help build a new stratgy fo convert casual to annual riders

### The Data
The data contains all the rides recorded in 12 month. the data of each use of a bike is recorderd and stored in the DWH. 
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


the query shows us that not all the columns in the db have the same data type and they need to be altered inorder to ba able to inserte all the tables into one main table.

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
	
sinch the quey returned zero rows we can understand that stations ID's with letters or distinctandcan't be replaced with existing FLOAT ID's)
	
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
- anual members are 56% of total rides
- anual riders ride an average of 15 minutes to an average of 40 minutes by casual riders
- riders (both members and casual riders) prefer the regualr bike (75%) over the electric bikes (25%)
- casual riders ride more on weekends  - 57% of casual rides are on Fri-Sun. 
- anual riders ride a little bit more on weekends but not significantly
- when it's cold the number of rides drop - 73% of the rides or in the spring and summer
- the number of casual riders drops in the winter, only 25% of the rides in the winter are by casual riders
- between 10:00 and 21:00 the casual and anual riders use the bikes the same. members ride more in the earler morning hours (05:00 - 10:00, 70% + of the rides), and casual riders like the late hours more (21:00-05:00, 60% +/- of the rides)


## Further Analysis - Stations

in the next step we will analyze the characteristics of the riders by stations - top stations by membership, days of the week, seasons etc.

the number of stations is:

		SELECT	 COUNT(DISTINCT start_station_name) AS 'Total_start_stations'
			,COUNT(DISTINCT end_station_name) AS 'Total_end_stations'
		FROM 	 trips



	
![image](https://user-images.githubusercontent.com/73856609/211014914-dd206105-a99e-43d4-81f5-4d65336e8380.png)

a. top stations
		
		SELECT	 start_station_name 
			,COUNT(*) AS 'rides_per _station'
			,FORMAT(COUNT(*)*1.0 / SUM(COUNT(*)) OVER (), 'P') AS pct
		FROM trips
		WHERE start_station_name IS NOT NULL
		GROUP BY start_station_name
		ORDER BY rides_per_station DESC
	
	

	
![image](https://user-images.githubusercontent.com/73856609/211015115-0e3eaa16-7959-41eb-8d89-3e7d039cdd50.png)
	
b. stations by membership

		SELECT	s.start_station_name
			,s.statioins AS 'station_rides'
			,ROW_NUMBER() OVER (ORDER BY s.statioins DESC) AS 'top_stations'
			,c.statioins AS 'casual_rides'
			,ROW_NUMBER() OVER (ORDER BY c.statioins DESC) AS 'casual_top_stations'
			,c.statioins AS 'member_rides'
			,ROW_NUMBER() OVER (ORDER BY m.statioins DESC) AS 'member_top_stations'
		FROM
		(
		SELECT	 start_station_name
			,COUNT(*) AS 'statioins'
		FROM trips
		WHERE start_station_name IS NOT NULL 
		GROUP BY start_station_name
		)s 
		JOIN
		(
		SELECT	 start_station_name
			,COUNT(*) AS 'statioins'
		FROM trips
		WHERE start_station_name IS NOT NULL 
		      AND member_casual = 'casual'
		GROUP BY start_station_name
		) c ON s.start_station_name = c.start_station_name
		JOIN
		(
		SELECT	 start_station_name
				,COUNT(*) AS 'statioins'
		FROM trips
		WHERE start_station_name IS NOT NULL 
		      AND member_casual = 'member'
		GROUP BY start_station_name
		) m
		ON c.start_station_name = m.start_station_name
		ORDER BY ROW_NUMBER() OVER (ORDER BY s.statioins DESC)


![image](https://user-images.githubusercontent.com/73856609/211173033-8eb82de0-b6eb-4f97-bad4-fbbda06dbbaf.png)



c. stations by weekday 
d. stations by weekday membership

## Near the Stations

## Ride Disstance

## Concultion 
