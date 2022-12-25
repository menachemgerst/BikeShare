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
	the stations ID's are mostly numbers, but in some month stored also with letters, there for we will alter all stations ID's into NVARCHAR and not into FLOAT
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
