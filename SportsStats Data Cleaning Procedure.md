# SportsStats Data Cleaning Procedure

**Table of Contents**

1. Introduction
2. Load Raw Data
3. Summary of Issues Detected
4. Issue Resolution
    - Step 1 — Removing Duplicates
    - Step 2 — Handle Missing Values
    - Step 3 — Correct Inaccuracies
    - Step 4 — Standardize Formats
    - Step 5 — Manage Outliers
5. Summary of Cleaning Actions
6. Future Work
7. Export Cleaned Data

---

## I. Introduction

The client I will be working with is SportsStats, a sports analysis firm partnering with local news and elite personal trainers to provide insights to help their partners. They recognize patterns/trends highlighting certain groups, events, countries, etc. for the purpose of developing a news story or discovering key health insights. As someone with a personal history with several sports such as basketball, martial arts, fencing, bicycling, and more, being able to perform analysis to gain insights on sports is something that personally engages me. Additionally, I reside within a culture that is heavily influenced by health and well-being as an ideal to pursue, so I find that this analysis that will be able to provide key health insights will provide value not only to the client I will be working with in this project, but also to the general public as well.

SportsStats has provided two related datasets, and the notebook "SportsStats Data Quality Assessment" has identified the issues with the data. The data consists of 120 years worth of data regarding the Olympics from 1896 to 2016, with a reference dataset consisting of various National Olympic Committee (or NOC) codes along with the regions they represent. This notebook aims to correct the issues discovered, and will do so by using SQLite to alter the database that will host the raw datasets that will eventually become cleaned datasets ready for export and analysis. Note that more understanding from research is necessary to contextualize the discrepancies, and will be available through a separate appendix titled "Olympics Research Appendix".

---

## II. Load Raw Data

First, you may desire to understand, and potentially adjust your working directory. Understanding from which environment you are working from is critical for your project, especially if you intend to replicate the steps provided. The code that will be provided after this step assumes you are working with a relative path to retrieve, load, and use CSV and DB files stored under the same folder as where this notebook would be saved. As such, make sure that your current working directory matches accordingly.

### Working Directory (Optional)


```python
# Import the os library
import os
```


```python
# Then use the following code to find out where your current working directory is
print(os.getcwd())
```

    C:\Users\randy\OneDrive\Documents\Data Analytics\SportsStats Analysis
    


```python
# If the cwd location for your project is not where you would prefer it to be, 
# use something akin to the following to change it:
os.chdir(r"C:\Users\randy\OneDrive\Documents\Data Analytics\SportsStats Analysis")
print(os.getcwd())
```

    C:\Users\randy\OneDrive\Documents\Data Analytics\SportsStats Analysis
    

### Importing Libraries

Before working on importing the data, import the libraries to use for the project. 
- Pandas will be used to import the data from the CSV files provided. 
- SQLite3 will be used to import Jupyter compatible extensions that will enable you to code and query in SQLite.
- re will be used to create a function into SQLite that will enable the usage of Regular Expressions (or Regex).


```python
# Importing the libraries
import pandas as pd
import sqlite3
import re
```

### Importing the data into Jupyter Notebook

Make sure the CSV files `athlete_events.csv` and `noc_regions.csv` are stored within your current working directory first before proceeding.


```python
# Reading the CSV Files as Dataframes
ath_events = pd.read_csv("athlete_events.csv")
noc_regions = pd.read_csv("noc_regions.csv")
```


```python
# Testing the success of reading:
print(ath_events.head(3))
print(noc_regions.head(3))
```

       ID                 Name Sex   Age  Height  Weight     Team  NOC  \
    0   1            A Dijiang   M  24.0   180.0    80.0    China  CHN   
    1   2             A Lamusi   M  23.0   170.0    60.0    China  CHN   
    2   3  Gunnar Nielsen Aaby   M  24.0     NaN     NaN  Denmark  DEN   
    
             Games  Year  Season       City       Sport  \
    0  1992 Summer  1992  Summer  Barcelona  Basketball   
    1  2012 Summer  2012  Summer     London        Judo   
    2  1920 Summer  1920  Summer  Antwerpen    Football   
    
                              Event Medal  
    0   Basketball Men's Basketball   NaN  
    1  Judo Men's Extra-Lightweight   NaN  
    2       Football Men's Football   NaN  
       NOC       region                 notes
    0  AFG  Afghanistan                   NaN
    1  AHO      Curacao  Netherlands Antilles
    2  ALB      Albania                   NaN
    

### Importing the data from Jupyter into SQL Database

We are going to create a connection between the data we have read thus far and a SQLite database. This code can either create a database file if one does not exist under the name you chose or connect the database file that exists with the same name.


```python
## Loading of Data into SQLite
conn = sqlite3.connect('olympics.db')
ath_events.to_sql('ath_events', conn, if_exists='replace', index=False)
noc_regions.to_sql('noc_regions', conn, if_exists='replace', index=False)

# The result of this would be a number. This number corresponds to the number of rows
# existing within the most recent dataset you connected to with your sqlite database.
# We previously set up a database for sqlite called "olympics.db". From there, we
# connected that database to the data we have listed over here.
```




    230



### Loading SQL Extensions and Regex to run SQL Queries

We will be using SQLite code to handle the data from here on out, but first, we need to install jupysql into the notebook itself if not installed yet. From there, we load the SQL extension into your notebook, connect to the olympics.db, configure so that the display limit is turned off, and establish the REGEXP function that will allow for the use of Regex during assessment.


```python
# Install jupysql (Only run code once. If already installed, no need to install again)
#!pip install jupysql
```


```python
# Load the SQL extension
%load_ext sql
```


```python
# Connect to olympics.db database
%sql sqlite:///olympics.db
```


<span style="None">Connecting to &#x27;sqlite:///olympics.db&#x27;</span>



```python
# If not yet configured so that the display limit no longer is active, this code turns it off:
%config SqlMagic.displaylimit = None
```


<span style="None">displaylimit: Value None will be treated as 0 (no limit)</span>



```python
# Define the REGEXP function
def regexp(pattern, value):
    if value is None:
        return False
    return re.search(pattern, value) is not None

# Register the function with SQLite
conn.create_function("REGEXP", 2, regexp)
```

*A note before proceeding with the cleaning procedure, most of the code below will provide outputs for the first three rows simply for presentation purposes. To properly understand the data and its quality, you may need to alter the code to provide more rows.*

---

## III. Summary of Issues Detected

The data was assessed by checking the data both as a whole and by individual columns while considering interdependencies between columns to check for the following: 

- Accuracy
- Completeness
- Validity
- Consistency
- Timeliness
- Uniqueness
- Reliability
                                                                                                                                                 
Here is a chart of the issues we detected

| Column | Data Quality Category | Issue | Records Affected | Priority | Method of Issue Resolution (if applicable) |
| ------ | --------------------- | ----- | ---------------- | -------- | ------------------------------------------ |
| Name | Accuracy | No Notable Issues, but they may appear during manual corrections | NA | Low | If found, correct appropriately |
| Age | Completeness | Many missing values, more during older times rather than recent years | 9,474 | **High** | Cross reference with Olympedia for recent years and fill in wherever you can manually, document issue for any others remaining for future work |
| Height | Completeness | Many missing values, more during older times rather than recent years | 60,171 | **High** | Cross reference with Olympedia for recent years and fill in wherever you can manually, document issue for any others remaining for future work |
| Weight | Completeness | Many missing values, more during older times rather than recent years | 62,875 | **High** | Cross reference with Olympedia for recent years and fill in wherever you can manually, document issue for any others remaining for future work |
| Medal | Completeness | Missing values for what should be documented as instances of no medals being earned | 231,333 | **High** | Impute "No Medal" value on all null values |
| Region | Completeness | Missing values associated with NOCs ROT,TUV, and UNC | 3 | Low | Check notes and impute values appropriately |
| Team | Validity | Instances of two teams combined indicates issues in accuracy | <43 | **High** | Cross reference with Olympedia to correct any inaccuracies |
| Team | Validity | Many instances of teams that don't directly reference a region may harbor potential inaccuracies | NA | Low | Document issue for future work |
| NOC | Consistency | Inconsistency with SIN/SGP for Singapore between ath_events and noc_regions | <290 | *Medium* | Direct update in noc_regions |
| Year | Timeliness | Data extends to 2016, and games from years beyond this could be included | NA | *Medium* | Document for future work |
| ID, Team, Games, Event | Uniqueness | Many instances of duplicate rows | 1455 | **High** | Identify and remove using partition of dataset as pseudo-primary key |

*Note: No issues with Reliability*

## IV. Issue Resolution

### Step 1: Removing Duplicates

First, we note exactly how many duplicates we will be removing to check that the filter will only consider the duplicates and leave the first instances of each row remaining when we delete it. Then, we delete those duplicates using the DELETE function.


```sql
%%sql
SELECT COUNT(*)
FROM ath_events
WHERE ROWID NOT IN (
    SELECT MIN(ROWID)
    FROM ath_events
    GROUP BY ID, Team, Games, Event
);
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>COUNT(*)</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>1455</td>
        </tr>
    </tbody>
</table>




```sql
%%sql
DELETE FROM ath_events
WHERE ROWID NOT IN (
    SELECT MIN(ROWID)
    FROM ath_events
    GROUP BY ID, Team, Games, Event
);
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>



<span style="color: green">1455 rows affected.</span>





<table>
    <thead>
        <tr>
        </tr>
    </thead>
    <tbody>
    </tbody>
</table>



1,455 duplicate rows have been removed, leaving us with only with the original distinct rows.

### Step 2: Handle Missing Values

The following has been noted when it came to missing data:
- There are 9474 null values in Age
- There are 60171 null values in Height
- There are 62875 null values in Weight
- There are 231333 null values in Medal
- There are 2 values listed as UNK for NOC IN ath_events
- There are 3 null values listed in Region from noc_regions

We will further understand the context of the numerous null values in Age, Height, and Weight, checking which conditions the most Nulls appear while seeing if it is possible to find and replace some of the missing values. Additionally, we will be replacing the missing values of the two UNK values and as many of the 3 null values of Region.

#### Age, Height and Weight
    
First, we will begin by looking at the distribution of instances where Age is null through the years. From there, we can see which years have the most missing information, observe any patterns within this that can help later on, and look into years that had little missing values to see if we can find and replace the missing values.


```sql
%%sql
SELECT Year, COUNT(Year)
FROM ath_events
WHERE Age IS NULL
GROUP BY Year
ORDER BY Year
LIMIT 3;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>Year</th>
            <th>COUNT(Year)</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>1896</td>
            <td>163</td>
        </tr>
        <tr>
            <td>1900</td>
            <td>752</td>
        </tr>
        <tr>
            <td>1904</td>
            <td>274</td>
        </tr>
    </tbody>
</table>



From here, we recognize that the amount of missing information varies throughout the years, however, from 1994 onwards, there have been very little missing values. It would be recommeded to observe the rows where Age is missing and the year is 1994 and forward. From there, we retrieve information that we will use to look up records on Olympedia.org to see if there is any available information that can be imputed


```sql
%%sql
SELECT *
FROM ath_events
WHERE Year >= 1994 AND Age IS NULL
LIMIT 3
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>ID</th>
            <th>Name</th>
            <th>Sex</th>
            <th>Age</th>
            <th>Height</th>
            <th>Weight</th>
            <th>Team</th>
            <th>NOC</th>
            <th>Games</th>
            <th>Year</th>
            <th>Season</th>
            <th>City</th>
            <th>Sport</th>
            <th>Event</th>
            <th>Medal</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>14130</td>
            <td>Grayson Hugh Bourne</td>
            <td>M</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
            <td>Great Britain</td>
            <td>GBR</td>
            <td>1996 Summer</td>
            <td>1996</td>
            <td>Summer</td>
            <td>Atlanta</td>
            <td>Canoeing</td>
            <td>Canoeing Men's Kayak Doubles, 1,000 metres</td>
            <td>None</td>
        </tr>
        <tr>
            <td>19504</td>
            <td>Cha Yong-Hwa</td>
            <td>F</td>
            <td>None</td>
            <td>145.0</td>
            <td>39.0</td>
            <td>North Korea</td>
            <td>PRK</td>
            <td>2008 Summer</td>
            <td>2008</td>
            <td>Summer</td>
            <td>Beijing</td>
            <td>Gymnastics</td>
            <td>Gymnastics Women's Individual All-Around</td>
            <td>None</td>
        </tr>
        <tr>
            <td>19504</td>
            <td>Cha Yong-Hwa</td>
            <td>F</td>
            <td>None</td>
            <td>145.0</td>
            <td>39.0</td>
            <td>North Korea</td>
            <td>PRK</td>
            <td>2008 Summer</td>
            <td>2008</td>
            <td>Summer</td>
            <td>Beijing</td>
            <td>Gymnastics</td>
            <td>Gymnastics Women's Uneven Bars</td>
            <td>None</td>
        </tr>
    </tbody>
</table>



Research from Olympedia.org (see Olympics Research Appendix for more information) shows the following information that we can use to fill the missing data:

- Grayson Hugh Bourne:
    - Born May 30, 1959
    - Height 173cm
    - Weight 82kg
- Cha Yong-Hwa
    - Originally recorded to be born Jan 8, 1990, but has been proven incorrect
        - No one knows when her true birthday is, so to impute the data, we will consider original record as an estimate for age
    - Height 145cm
    - Weight 39kg
- Boureima Kimba
    - Born Year 1968
    - No record for Height nor Weight
- Chris Lori
    - Jul 24, 1962
    - Height 178cm
    - Weight 85kg
- Abdou Manzo
    - Born Year 1959
    - No record for Height nor Weight
- Moosaka
    - Name on record in Olympedia.org: Mary Musoke, consider changing
    - Birthday known on Olympedia.org, but removed for privacy at her request
    - Height 165cm
    - Weight 75kg
- Raymond Anthony Papa
    - Born Oct 14, 1976
    - No record for Height nor Weight

Given this information, we will proceed with imputation using the following steps:

- First: Query for all information regarding one person for reference, filtering by ID.
- Second: Update Height and Weight information throughout all observations at once.
- Third: After calculating the age for each games year based on birthday information available, update age information for all observations regarding the same year.
- Fourth: Query for all information regarding the person you updated the information on again to check that the changes were made, then go back to the first step and repeat the cycle for the next person.

**Grayson Hugh Bourne**


```sql
%%sql
-- Dirty Query
SELECT *
FROM ath_events
WHERE ID = 14130
LIMIT 3;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>ID</th>
            <th>Name</th>
            <th>Sex</th>
            <th>Age</th>
            <th>Height</th>
            <th>Weight</th>
            <th>Team</th>
            <th>NOC</th>
            <th>Games</th>
            <th>Year</th>
            <th>Season</th>
            <th>City</th>
            <th>Sport</th>
            <th>Event</th>
            <th>Medal</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>14130</td>
            <td>Grayson Hugh Bourne</td>
            <td>M</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
            <td>Great Britain</td>
            <td>GBR</td>
            <td>1980 Summer</td>
            <td>1980</td>
            <td>Summer</td>
            <td>Moskva</td>
            <td>Canoeing</td>
            <td>Canoeing Men's Kayak Singles, 500 metres</td>
            <td>None</td>
        </tr>
        <tr>
            <td>14130</td>
            <td>Grayson Hugh Bourne</td>
            <td>M</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
            <td>Great Britain</td>
            <td>GBR</td>
            <td>1984 Summer</td>
            <td>1984</td>
            <td>Summer</td>
            <td>Los Angeles</td>
            <td>Canoeing</td>
            <td>Canoeing Men's Kayak Fours, 1,000 metres</td>
            <td>None</td>
        </tr>
        <tr>
            <td>14130</td>
            <td>Grayson Hugh Bourne</td>
            <td>M</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
            <td>Great Britain</td>
            <td>GBR</td>
            <td>1988 Summer</td>
            <td>1988</td>
            <td>Summer</td>
            <td>Seoul</td>
            <td>Canoeing</td>
            <td>Canoeing Men's Kayak Doubles, 500 metres</td>
            <td>None</td>
        </tr>
    </tbody>
</table>




```sql
%%sql
-- Height & Weight
UPDATE ath_events
SET Height = 173, Weight = 82
WHERE ID = 14130;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>



<span style="color: green">5 rows affected.</span>





<table>
    <thead>
        <tr>
        </tr>
    </thead>
    <tbody>
    </tbody>
</table>




```sql
%%sql
-- Age over the years
UPDATE ath_events
SET Age = 21
WHERE ID = 14130 AND Year = 1980;
UPDATE ath_events
SET Age = 25
WHERE ID = 14130 AND Year = 1984;
UPDATE ath_events
SET Age = 29
WHERE ID = 14130 AND Year = 1988;
UPDATE ath_events
SET Age = 33
WHERE ID = 14130 AND Year = 1992;
UPDATE ath_events
SET Age = 37
WHERE ID = 14130 AND Year = 1996;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>



<span style="color: green">1 rows affected.</span>



<span style="color: green">1 rows affected.</span>



<span style="color: green">1 rows affected.</span>



<span style="color: green">1 rows affected.</span>



<span style="color: green">1 rows affected.</span>





<table>
    <thead>
        <tr>
        </tr>
    </thead>
    <tbody>
    </tbody>
</table>




```sql
%%sql
-- Clean Query
SELECT *
FROM ath_events
WHERE ID = 14130
LIMIT 3;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>ID</th>
            <th>Name</th>
            <th>Sex</th>
            <th>Age</th>
            <th>Height</th>
            <th>Weight</th>
            <th>Team</th>
            <th>NOC</th>
            <th>Games</th>
            <th>Year</th>
            <th>Season</th>
            <th>City</th>
            <th>Sport</th>
            <th>Event</th>
            <th>Medal</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>14130</td>
            <td>Grayson Hugh Bourne</td>
            <td>M</td>
            <td>21.0</td>
            <td>173.0</td>
            <td>82.0</td>
            <td>Great Britain</td>
            <td>GBR</td>
            <td>1980 Summer</td>
            <td>1980</td>
            <td>Summer</td>
            <td>Moskva</td>
            <td>Canoeing</td>
            <td>Canoeing Men's Kayak Singles, 500 metres</td>
            <td>None</td>
        </tr>
        <tr>
            <td>14130</td>
            <td>Grayson Hugh Bourne</td>
            <td>M</td>
            <td>25.0</td>
            <td>173.0</td>
            <td>82.0</td>
            <td>Great Britain</td>
            <td>GBR</td>
            <td>1984 Summer</td>
            <td>1984</td>
            <td>Summer</td>
            <td>Los Angeles</td>
            <td>Canoeing</td>
            <td>Canoeing Men's Kayak Fours, 1,000 metres</td>
            <td>None</td>
        </tr>
        <tr>
            <td>14130</td>
            <td>Grayson Hugh Bourne</td>
            <td>M</td>
            <td>29.0</td>
            <td>173.0</td>
            <td>82.0</td>
            <td>Great Britain</td>
            <td>GBR</td>
            <td>1988 Summer</td>
            <td>1988</td>
            <td>Summer</td>
            <td>Seoul</td>
            <td>Canoeing</td>
            <td>Canoeing Men's Kayak Doubles, 500 metres</td>
            <td>None</td>
        </tr>
    </tbody>
</table>



**Cha Yong-Hwa**


```sql
%%sql
-- Dirty Query
SELECT *
FROM ath_events
WHERE ID = 19504;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>ID</th>
            <th>Name</th>
            <th>Sex</th>
            <th>Age</th>
            <th>Height</th>
            <th>Weight</th>
            <th>Team</th>
            <th>NOC</th>
            <th>Games</th>
            <th>Year</th>
            <th>Season</th>
            <th>City</th>
            <th>Sport</th>
            <th>Event</th>
            <th>Medal</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>19504</td>
            <td>Cha Yong-Hwa</td>
            <td>F</td>
            <td>None</td>
            <td>145.0</td>
            <td>39.0</td>
            <td>North Korea</td>
            <td>PRK</td>
            <td>2008 Summer</td>
            <td>2008</td>
            <td>Summer</td>
            <td>Beijing</td>
            <td>Gymnastics</td>
            <td>Gymnastics Women's Individual All-Around</td>
            <td>None</td>
        </tr>
        <tr>
            <td>19504</td>
            <td>Cha Yong-Hwa</td>
            <td>F</td>
            <td>None</td>
            <td>145.0</td>
            <td>39.0</td>
            <td>North Korea</td>
            <td>PRK</td>
            <td>2008 Summer</td>
            <td>2008</td>
            <td>Summer</td>
            <td>Beijing</td>
            <td>Gymnastics</td>
            <td>Gymnastics Women's Uneven Bars</td>
            <td>None</td>
        </tr>
    </tbody>
</table>




```sql
%%sql
-- Age (only participated for one year)
UPDATE ath_events
SET Age = 18
WHERE ID = 19504;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>



<span style="color: green">2 rows affected.</span>





<table>
    <thead>
        <tr>
        </tr>
    </thead>
    <tbody>
    </tbody>
</table>




```sql
%%sql
-- Clean query
SELECT *
FROM ath_events
WHERE ID= 19504;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>ID</th>
            <th>Name</th>
            <th>Sex</th>
            <th>Age</th>
            <th>Height</th>
            <th>Weight</th>
            <th>Team</th>
            <th>NOC</th>
            <th>Games</th>
            <th>Year</th>
            <th>Season</th>
            <th>City</th>
            <th>Sport</th>
            <th>Event</th>
            <th>Medal</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>19504</td>
            <td>Cha Yong-Hwa</td>
            <td>F</td>
            <td>18.0</td>
            <td>145.0</td>
            <td>39.0</td>
            <td>North Korea</td>
            <td>PRK</td>
            <td>2008 Summer</td>
            <td>2008</td>
            <td>Summer</td>
            <td>Beijing</td>
            <td>Gymnastics</td>
            <td>Gymnastics Women's Individual All-Around</td>
            <td>None</td>
        </tr>
        <tr>
            <td>19504</td>
            <td>Cha Yong-Hwa</td>
            <td>F</td>
            <td>18.0</td>
            <td>145.0</td>
            <td>39.0</td>
            <td>North Korea</td>
            <td>PRK</td>
            <td>2008 Summer</td>
            <td>2008</td>
            <td>Summer</td>
            <td>Beijing</td>
            <td>Gymnastics</td>
            <td>Gymnastics Women's Uneven Bars</td>
            <td>None</td>
        </tr>
    </tbody>
</table>



**Boureima Kimba**


```sql
%%sql
-- Dirty Query
SELECT *
FROM ath_events
WHERE ID = 60395;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>ID</th>
            <th>Name</th>
            <th>Sex</th>
            <th>Age</th>
            <th>Height</th>
            <th>Weight</th>
            <th>Team</th>
            <th>NOC</th>
            <th>Games</th>
            <th>Year</th>
            <th>Season</th>
            <th>City</th>
            <th>Sport</th>
            <th>Event</th>
            <th>Medal</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>60395</td>
            <td>Boureima Kimba</td>
            <td>M</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
            <td>Niger</td>
            <td>NIG</td>
            <td>1992 Summer</td>
            <td>1992</td>
            <td>Summer</td>
            <td>Barcelona</td>
            <td>Athletics</td>
            <td>Athletics Men's 200 metres</td>
            <td>None</td>
        </tr>
        <tr>
            <td>60395</td>
            <td>Boureima Kimba</td>
            <td>M</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
            <td>Niger</td>
            <td>NIG</td>
            <td>1996 Summer</td>
            <td>1996</td>
            <td>Summer</td>
            <td>Atlanta</td>
            <td>Athletics</td>
            <td>Athletics Men's 100 metres</td>
            <td>None</td>
        </tr>
    </tbody>
</table>




```sql
%%sql
-- Age
UPDATE ath_events
SET Age = 24
WHERE ID = 60395 AND Year = 1992;
UPDATE ath_events
SET Age = 28
WHERE ID = 60395 AND Year = 1996;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>



<span style="color: green">1 rows affected.</span>



<span style="color: green">1 rows affected.</span>





<table>
    <thead>
        <tr>
        </tr>
    </thead>
    <tbody>
    </tbody>
</table>




```sql
%%sql
-- Clean Query
SELECT *
FROM ath_events
WHERE ID = 60395;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>ID</th>
            <th>Name</th>
            <th>Sex</th>
            <th>Age</th>
            <th>Height</th>
            <th>Weight</th>
            <th>Team</th>
            <th>NOC</th>
            <th>Games</th>
            <th>Year</th>
            <th>Season</th>
            <th>City</th>
            <th>Sport</th>
            <th>Event</th>
            <th>Medal</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>60395</td>
            <td>Boureima Kimba</td>
            <td>M</td>
            <td>24.0</td>
            <td>None</td>
            <td>None</td>
            <td>Niger</td>
            <td>NIG</td>
            <td>1992 Summer</td>
            <td>1992</td>
            <td>Summer</td>
            <td>Barcelona</td>
            <td>Athletics</td>
            <td>Athletics Men's 200 metres</td>
            <td>None</td>
        </tr>
        <tr>
            <td>60395</td>
            <td>Boureima Kimba</td>
            <td>M</td>
            <td>28.0</td>
            <td>None</td>
            <td>None</td>
            <td>Niger</td>
            <td>NIG</td>
            <td>1996 Summer</td>
            <td>1996</td>
            <td>Summer</td>
            <td>Atlanta</td>
            <td>Athletics</td>
            <td>Athletics Men's 100 metres</td>
            <td>None</td>
        </tr>
    </tbody>
</table>



**"Chris" Lori**


```sql
%%sql
-- Dirty Query
SELECT *
FROM ath_events
WHERE ID = 71557
LIMIT 3;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>ID</th>
            <th>Name</th>
            <th>Sex</th>
            <th>Age</th>
            <th>Height</th>
            <th>Weight</th>
            <th>Team</th>
            <th>NOC</th>
            <th>Games</th>
            <th>Year</th>
            <th>Season</th>
            <th>City</th>
            <th>Sport</th>
            <th>Event</th>
            <th>Medal</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>71557</td>
            <td>Christopher Paul "Chris" Lori</td>
            <td>M</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
            <td>Canada-1</td>
            <td>CAN</td>
            <td>1988 Winter</td>
            <td>1988</td>
            <td>Winter</td>
            <td>Calgary</td>
            <td>Bobsleigh</td>
            <td>Bobsleigh Men's Four</td>
            <td>None</td>
        </tr>
        <tr>
            <td>71557</td>
            <td>Christopher Paul "Chris" Lori</td>
            <td>M</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
            <td>Canada-1</td>
            <td>CAN</td>
            <td>1992 Winter</td>
            <td>1992</td>
            <td>Winter</td>
            <td>Albertville</td>
            <td>Bobsleigh</td>
            <td>Bobsleigh Men's Four</td>
            <td>None</td>
        </tr>
        <tr>
            <td>71557</td>
            <td>Christopher Paul "Chris" Lori</td>
            <td>M</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
            <td>Canada-2</td>
            <td>CAN</td>
            <td>1994 Winter</td>
            <td>1994</td>
            <td>Winter</td>
            <td>Lillehammer</td>
            <td>Bobsleigh</td>
            <td>Bobsleigh Men's Two</td>
            <td>None</td>
        </tr>
    </tbody>
</table>




```sql
%%sql
-- Height & Weight
UPDATE ath_events
SET Height = 178, Weight = 85
WHERE ID = 71557;
-- Age over the years
UPDATE ath_events
SET Age = 25
WHERE ID = 71557 AND Year = 1988;
UPDATE ath_events
SET Age = 29
WHERE ID = 71557 AND Year = 1992;
UPDATE ath_events
SET Age = 31
WHERE ID = 71557 AND Year = 1994;
UPDATE ath_events
SET Age = 35
WHERE ID = 71557 AND Year = 1998;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>



<span style="color: green">6 rows affected.</span>



<span style="color: green">1 rows affected.</span>



<span style="color: green">1 rows affected.</span>



<span style="color: green">2 rows affected.</span>



<span style="color: green">2 rows affected.</span>





<table>
    <thead>
        <tr>
        </tr>
    </thead>
    <tbody>
    </tbody>
</table>




```sql
%%sql
-- Clean query
SELECT *
FROM ath_events
WHERE ID = 71557;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>ID</th>
            <th>Name</th>
            <th>Sex</th>
            <th>Age</th>
            <th>Height</th>
            <th>Weight</th>
            <th>Team</th>
            <th>NOC</th>
            <th>Games</th>
            <th>Year</th>
            <th>Season</th>
            <th>City</th>
            <th>Sport</th>
            <th>Event</th>
            <th>Medal</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>71557</td>
            <td>Christopher Paul "Chris" Lori</td>
            <td>M</td>
            <td>25.0</td>
            <td>178.0</td>
            <td>85.0</td>
            <td>Canada-1</td>
            <td>CAN</td>
            <td>1988 Winter</td>
            <td>1988</td>
            <td>Winter</td>
            <td>Calgary</td>
            <td>Bobsleigh</td>
            <td>Bobsleigh Men's Four</td>
            <td>None</td>
        </tr>
        <tr>
            <td>71557</td>
            <td>Christopher Paul "Chris" Lori</td>
            <td>M</td>
            <td>29.0</td>
            <td>178.0</td>
            <td>85.0</td>
            <td>Canada-1</td>
            <td>CAN</td>
            <td>1992 Winter</td>
            <td>1992</td>
            <td>Winter</td>
            <td>Albertville</td>
            <td>Bobsleigh</td>
            <td>Bobsleigh Men's Four</td>
            <td>None</td>
        </tr>
        <tr>
            <td>71557</td>
            <td>Christopher Paul "Chris" Lori</td>
            <td>M</td>
            <td>31.0</td>
            <td>178.0</td>
            <td>85.0</td>
            <td>Canada-2</td>
            <td>CAN</td>
            <td>1994 Winter</td>
            <td>1994</td>
            <td>Winter</td>
            <td>Lillehammer</td>
            <td>Bobsleigh</td>
            <td>Bobsleigh Men's Two</td>
            <td>None</td>
        </tr>
        <tr>
            <td>71557</td>
            <td>Christopher Paul "Chris" Lori</td>
            <td>M</td>
            <td>31.0</td>
            <td>178.0</td>
            <td>85.0</td>
            <td>Canada-2</td>
            <td>CAN</td>
            <td>1994 Winter</td>
            <td>1994</td>
            <td>Winter</td>
            <td>Lillehammer</td>
            <td>Bobsleigh</td>
            <td>Bobsleigh Men's Four</td>
            <td>None</td>
        </tr>
        <tr>
            <td>71557</td>
            <td>Christopher Paul "Chris" Lori</td>
            <td>M</td>
            <td>35.0</td>
            <td>178.0</td>
            <td>85.0</td>
            <td>Canada-2</td>
            <td>CAN</td>
            <td>1998 Winter</td>
            <td>1998</td>
            <td>Winter</td>
            <td>Nagano</td>
            <td>Bobsleigh</td>
            <td>Bobsleigh Men's Two</td>
            <td>None</td>
        </tr>
        <tr>
            <td>71557</td>
            <td>Christopher Paul "Chris" Lori</td>
            <td>M</td>
            <td>35.0</td>
            <td>178.0</td>
            <td>85.0</td>
            <td>Canada-2</td>
            <td>CAN</td>
            <td>1998 Winter</td>
            <td>1998</td>
            <td>Winter</td>
            <td>Nagano</td>
            <td>Bobsleigh</td>
            <td>Bobsleigh Men's Four</td>
            <td>None</td>
        </tr>
    </tbody>
</table>



**Abdou Manzo**


```sql
%%sql
-- Dirty query
SELECT *
FROM ath_events
WHERE ID = 74668;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>ID</th>
            <th>Name</th>
            <th>Sex</th>
            <th>Age</th>
            <th>Height</th>
            <th>Weight</th>
            <th>Team</th>
            <th>NOC</th>
            <th>Games</th>
            <th>Year</th>
            <th>Season</th>
            <th>City</th>
            <th>Sport</th>
            <th>Event</th>
            <th>Medal</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>74668</td>
            <td>Abdou Manzo</td>
            <td>M</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
            <td>Niger</td>
            <td>NIG</td>
            <td>1988 Summer</td>
            <td>1988</td>
            <td>Summer</td>
            <td>Seoul</td>
            <td>Athletics</td>
            <td>Athletics Men's Marathon</td>
            <td>None</td>
        </tr>
        <tr>
            <td>74668</td>
            <td>Abdou Manzo</td>
            <td>M</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
            <td>Niger</td>
            <td>NIG</td>
            <td>1992 Summer</td>
            <td>1992</td>
            <td>Summer</td>
            <td>Barcelona</td>
            <td>Athletics</td>
            <td>Athletics Men's Marathon</td>
            <td>None</td>
        </tr>
        <tr>
            <td>74668</td>
            <td>Abdou Manzo</td>
            <td>M</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
            <td>Niger</td>
            <td>NIG</td>
            <td>1996 Summer</td>
            <td>1996</td>
            <td>Summer</td>
            <td>Atlanta</td>
            <td>Athletics</td>
            <td>Athletics Men's Marathon</td>
            <td>None</td>
        </tr>
    </tbody>
</table>




```sql
%%sql
-- Age over the years
UPDATE ath_events
SET Age = 29
WHERE ID = 74668 AND Year = 1988;
UPDATE ath_events
SET Age = 33
WHERE ID = 74668 AND Year = 1992;
UPDATE ath_events
SET Age = 37
WHERE ID = 74668 AND Year = 1996;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>



<span style="color: green">1 rows affected.</span>



<span style="color: green">1 rows affected.</span>



<span style="color: green">1 rows affected.</span>





<table>
    <thead>
        <tr>
        </tr>
    </thead>
    <tbody>
    </tbody>
</table>




```sql
%%sql
-- Clean query
SELECT *
FROM ath_events
WHERE ID = 74668;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>ID</th>
            <th>Name</th>
            <th>Sex</th>
            <th>Age</th>
            <th>Height</th>
            <th>Weight</th>
            <th>Team</th>
            <th>NOC</th>
            <th>Games</th>
            <th>Year</th>
            <th>Season</th>
            <th>City</th>
            <th>Sport</th>
            <th>Event</th>
            <th>Medal</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>74668</td>
            <td>Abdou Manzo</td>
            <td>M</td>
            <td>29.0</td>
            <td>None</td>
            <td>None</td>
            <td>Niger</td>
            <td>NIG</td>
            <td>1988 Summer</td>
            <td>1988</td>
            <td>Summer</td>
            <td>Seoul</td>
            <td>Athletics</td>
            <td>Athletics Men's Marathon</td>
            <td>None</td>
        </tr>
        <tr>
            <td>74668</td>
            <td>Abdou Manzo</td>
            <td>M</td>
            <td>33.0</td>
            <td>None</td>
            <td>None</td>
            <td>Niger</td>
            <td>NIG</td>
            <td>1992 Summer</td>
            <td>1992</td>
            <td>Summer</td>
            <td>Barcelona</td>
            <td>Athletics</td>
            <td>Athletics Men's Marathon</td>
            <td>None</td>
        </tr>
        <tr>
            <td>74668</td>
            <td>Abdou Manzo</td>
            <td>M</td>
            <td>37.0</td>
            <td>None</td>
            <td>None</td>
            <td>Niger</td>
            <td>NIG</td>
            <td>1996 Summer</td>
            <td>1996</td>
            <td>Summer</td>
            <td>Atlanta</td>
            <td>Athletics</td>
            <td>Athletics Men's Marathon</td>
            <td>None</td>
        </tr>
    </tbody>
</table>



**Moosaka/Mary Musoke**


```sql
%%sql
-- Dirty Query
SELECT *
FROM ath_events
WHERE ID = 81706;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>ID</th>
            <th>Name</th>
            <th>Sex</th>
            <th>Age</th>
            <th>Height</th>
            <th>Weight</th>
            <th>Team</th>
            <th>NOC</th>
            <th>Games</th>
            <th>Year</th>
            <th>Season</th>
            <th>City</th>
            <th>Sport</th>
            <th>Event</th>
            <th>Medal</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>81706</td>
            <td>Moosaka</td>
            <td>F</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
            <td>Uganda</td>
            <td>UGA</td>
            <td>1992 Summer</td>
            <td>1992</td>
            <td>Summer</td>
            <td>Barcelona</td>
            <td>Table Tennis</td>
            <td>Table Tennis Women's Singles</td>
            <td>None</td>
        </tr>
        <tr>
            <td>81706</td>
            <td>Moosaka</td>
            <td>F</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
            <td>Uganda</td>
            <td>UGA</td>
            <td>1996 Summer</td>
            <td>1996</td>
            <td>Summer</td>
            <td>Atlanta</td>
            <td>Table Tennis</td>
            <td>Table Tennis Women's Singles</td>
            <td>None</td>
        </tr>
        <tr>
            <td>81706</td>
            <td>Moosaka</td>
            <td>F</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
            <td>Uganda</td>
            <td>UGA</td>
            <td>1996 Summer</td>
            <td>1996</td>
            <td>Summer</td>
            <td>Atlanta</td>
            <td>Table Tennis</td>
            <td>Table Tennis Women's Doubles</td>
            <td>None</td>
        </tr>
        <tr>
            <td>81706</td>
            <td>Moosaka</td>
            <td>F</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
            <td>Uganda</td>
            <td>UGA</td>
            <td>2000 Summer</td>
            <td>2000</td>
            <td>Summer</td>
            <td>Sydney</td>
            <td>Table Tennis</td>
            <td>Table Tennis Women's Singles</td>
            <td>None</td>
        </tr>
    </tbody>
</table>




```sql
%%sql
-- Name, Height, & Weight
UPDATE ath_events
SET Name = "Mary Musoke", Height = 165, Weight = 75
WHERE ID = 81706;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>



<span style="color: green">4 rows affected.</span>





<table>
    <thead>
        <tr>
        </tr>
    </thead>
    <tbody>
    </tbody>
</table>




```sql
%%sql
-- Clean query
SELECT *
FROM ath_events
WHERE ID = 81706;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>ID</th>
            <th>Name</th>
            <th>Sex</th>
            <th>Age</th>
            <th>Height</th>
            <th>Weight</th>
            <th>Team</th>
            <th>NOC</th>
            <th>Games</th>
            <th>Year</th>
            <th>Season</th>
            <th>City</th>
            <th>Sport</th>
            <th>Event</th>
            <th>Medal</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>81706</td>
            <td>Mary Musoke</td>
            <td>F</td>
            <td>None</td>
            <td>165.0</td>
            <td>75.0</td>
            <td>Uganda</td>
            <td>UGA</td>
            <td>1992 Summer</td>
            <td>1992</td>
            <td>Summer</td>
            <td>Barcelona</td>
            <td>Table Tennis</td>
            <td>Table Tennis Women's Singles</td>
            <td>None</td>
        </tr>
        <tr>
            <td>81706</td>
            <td>Mary Musoke</td>
            <td>F</td>
            <td>None</td>
            <td>165.0</td>
            <td>75.0</td>
            <td>Uganda</td>
            <td>UGA</td>
            <td>1996 Summer</td>
            <td>1996</td>
            <td>Summer</td>
            <td>Atlanta</td>
            <td>Table Tennis</td>
            <td>Table Tennis Women's Singles</td>
            <td>None</td>
        </tr>
        <tr>
            <td>81706</td>
            <td>Mary Musoke</td>
            <td>F</td>
            <td>None</td>
            <td>165.0</td>
            <td>75.0</td>
            <td>Uganda</td>
            <td>UGA</td>
            <td>1996 Summer</td>
            <td>1996</td>
            <td>Summer</td>
            <td>Atlanta</td>
            <td>Table Tennis</td>
            <td>Table Tennis Women's Doubles</td>
            <td>None</td>
        </tr>
        <tr>
            <td>81706</td>
            <td>Mary Musoke</td>
            <td>F</td>
            <td>None</td>
            <td>165.0</td>
            <td>75.0</td>
            <td>Uganda</td>
            <td>UGA</td>
            <td>2000 Summer</td>
            <td>2000</td>
            <td>Summer</td>
            <td>Sydney</td>
            <td>Table Tennis</td>
            <td>Table Tennis Women's Singles</td>
            <td>None</td>
        </tr>
    </tbody>
</table>



**Raymond Anthony Papa**


```sql
%%sql
-- Dirty query
SELECT *
FROM ath_events
WHERE ID = 91182
LIMIT 3;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>ID</th>
            <th>Name</th>
            <th>Sex</th>
            <th>Age</th>
            <th>Height</th>
            <th>Weight</th>
            <th>Team</th>
            <th>NOC</th>
            <th>Games</th>
            <th>Year</th>
            <th>Season</th>
            <th>City</th>
            <th>Sport</th>
            <th>Event</th>
            <th>Medal</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>91182</td>
            <td>Raymond Anthony Papa</td>
            <td>M</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
            <td>Philippines</td>
            <td>PHI</td>
            <td>1992 Summer</td>
            <td>1992</td>
            <td>Summer</td>
            <td>Barcelona</td>
            <td>Swimming</td>
            <td>Swimming Men's 100 metres Backstroke</td>
            <td>None</td>
        </tr>
        <tr>
            <td>91182</td>
            <td>Raymond Anthony Papa</td>
            <td>M</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
            <td>Philippines</td>
            <td>PHI</td>
            <td>1992 Summer</td>
            <td>1992</td>
            <td>Summer</td>
            <td>Barcelona</td>
            <td>Swimming</td>
            <td>Swimming Men's 200 metres Backstroke</td>
            <td>None</td>
        </tr>
        <tr>
            <td>91182</td>
            <td>Raymond Anthony Papa</td>
            <td>M</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
            <td>Philippines</td>
            <td>PHI</td>
            <td>1992 Summer</td>
            <td>1992</td>
            <td>Summer</td>
            <td>Barcelona</td>
            <td>Swimming</td>
            <td>Swimming Men's 4 x 100 metres Medley Relay</td>
            <td>None</td>
        </tr>
    </tbody>
</table>




```sql
%%sql
-- Age
UPDATE ath_events
SET Age = 15
WHERE ID = 91182 AND Year = 1992;
UPDATE ath_events
SET Age = 19
WHERE ID = 91182 AND Year = 1996;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>



<span style="color: green">3 rows affected.</span>



<span style="color: green">3 rows affected.</span>





<table>
    <thead>
        <tr>
        </tr>
    </thead>
    <tbody>
    </tbody>
</table>




```sql
%%sql
-- Clean query
SELECT *
FROM ath_events
WHERE ID = 91182
LIMIT 3;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>ID</th>
            <th>Name</th>
            <th>Sex</th>
            <th>Age</th>
            <th>Height</th>
            <th>Weight</th>
            <th>Team</th>
            <th>NOC</th>
            <th>Games</th>
            <th>Year</th>
            <th>Season</th>
            <th>City</th>
            <th>Sport</th>
            <th>Event</th>
            <th>Medal</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>91182</td>
            <td>Raymond Anthony Papa</td>
            <td>M</td>
            <td>15.0</td>
            <td>None</td>
            <td>None</td>
            <td>Philippines</td>
            <td>PHI</td>
            <td>1992 Summer</td>
            <td>1992</td>
            <td>Summer</td>
            <td>Barcelona</td>
            <td>Swimming</td>
            <td>Swimming Men's 100 metres Backstroke</td>
            <td>None</td>
        </tr>
        <tr>
            <td>91182</td>
            <td>Raymond Anthony Papa</td>
            <td>M</td>
            <td>15.0</td>
            <td>None</td>
            <td>None</td>
            <td>Philippines</td>
            <td>PHI</td>
            <td>1992 Summer</td>
            <td>1992</td>
            <td>Summer</td>
            <td>Barcelona</td>
            <td>Swimming</td>
            <td>Swimming Men's 200 metres Backstroke</td>
            <td>None</td>
        </tr>
        <tr>
            <td>91182</td>
            <td>Raymond Anthony Papa</td>
            <td>M</td>
            <td>15.0</td>
            <td>None</td>
            <td>None</td>
            <td>Philippines</td>
            <td>PHI</td>
            <td>1992 Summer</td>
            <td>1992</td>
            <td>Summer</td>
            <td>Barcelona</td>
            <td>Swimming</td>
            <td>Swimming Men's 4 x 100 metres Medley Relay</td>
            <td>None</td>
        </tr>
    </tbody>
</table>



**Results of Cleaning for Age, Height, and Weight:**

- Age: 24 missing values have been replaced
- Height: 15 missing values have been replaced
- Weight: 15 missing values have been replaced

We will now check how many missing values remain:


```sql
%%sql
SELECT COUNT(*) - COUNT(Age) AS "Nulls of Age",
       COUNT(*) - COUNT(Height) AS "Nulls of Height",
       COUNT(*) - COUNT(Weight) AS "Nulls of Weight"
FROM ath_events;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>Nulls of Age</th>
            <th>Nulls of Height</th>
            <th>Nulls of Weight</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>9272</td>
            <td>58730</td>
            <td>61443</td>
        </tr>
    </tbody>
</table>



**Remaining missing values for Age, Height, and Weight**

- Age: 9,272
- Height: 58,730
- Weight: 61,443

#### Medal

We will now proceed with the Medal column. It is hard to say whether a null value is a null value because the person never received a medal, or if it is a null value because the information does not exist. That being said, it is reasonable enough to make the assumption that most people in these records who have a null value in the Medal column simply did not receive a medal, so we will work with this assumption and change all the null values and impute it so that it holds the phrase, "No Medal".


```sql
%%sql
SELECT *
FROM ath_events
WHERE Medal IS NULL
LIMIT 5;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>ID</th>
            <th>Name</th>
            <th>Sex</th>
            <th>Age</th>
            <th>Height</th>
            <th>Weight</th>
            <th>Team</th>
            <th>NOC</th>
            <th>Games</th>
            <th>Year</th>
            <th>Season</th>
            <th>City</th>
            <th>Sport</th>
            <th>Event</th>
            <th>Medal</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>1</td>
            <td>A Dijiang</td>
            <td>M</td>
            <td>24.0</td>
            <td>180.0</td>
            <td>80.0</td>
            <td>China</td>
            <td>CHN</td>
            <td>1992 Summer</td>
            <td>1992</td>
            <td>Summer</td>
            <td>Barcelona</td>
            <td>Basketball</td>
            <td>Basketball Men's Basketball</td>
            <td>None</td>
        </tr>
        <tr>
            <td>2</td>
            <td>A Lamusi</td>
            <td>M</td>
            <td>23.0</td>
            <td>170.0</td>
            <td>60.0</td>
            <td>China</td>
            <td>CHN</td>
            <td>2012 Summer</td>
            <td>2012</td>
            <td>Summer</td>
            <td>London</td>
            <td>Judo</td>
            <td>Judo Men's Extra-Lightweight</td>
            <td>None</td>
        </tr>
        <tr>
            <td>3</td>
            <td>Gunnar Nielsen Aaby</td>
            <td>M</td>
            <td>24.0</td>
            <td>None</td>
            <td>None</td>
            <td>Denmark</td>
            <td>DEN</td>
            <td>1920 Summer</td>
            <td>1920</td>
            <td>Summer</td>
            <td>Antwerpen</td>
            <td>Football</td>
            <td>Football Men's Football</td>
            <td>None</td>
        </tr>
        <tr>
            <td>5</td>
            <td>Christine Jacoba Aaftink</td>
            <td>F</td>
            <td>21.0</td>
            <td>185.0</td>
            <td>82.0</td>
            <td>Netherlands</td>
            <td>NED</td>
            <td>1988 Winter</td>
            <td>1988</td>
            <td>Winter</td>
            <td>Calgary</td>
            <td>Speed Skating</td>
            <td>Speed Skating Women's 500 metres</td>
            <td>None</td>
        </tr>
        <tr>
            <td>5</td>
            <td>Christine Jacoba Aaftink</td>
            <td>F</td>
            <td>21.0</td>
            <td>185.0</td>
            <td>82.0</td>
            <td>Netherlands</td>
            <td>NED</td>
            <td>1988 Winter</td>
            <td>1988</td>
            <td>Winter</td>
            <td>Calgary</td>
            <td>Speed Skating</td>
            <td>Speed Skating Women's 1,000 metres</td>
            <td>None</td>
        </tr>
    </tbody>
</table>




```sql
%%sql
UPDATE ath_events
SET Medal = "No Medal"
WHERE Medal IS NULL;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>



<span style="color: green">229906 rows affected.</span>





<table>
    <thead>
        <tr>
        </tr>
    </thead>
    <tbody>
    </tbody>
</table>




```sql
%%sql
SELECT *
FROM ath_events
WHERE Medal = "No Medal"
LIMIT 5
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>ID</th>
            <th>Name</th>
            <th>Sex</th>
            <th>Age</th>
            <th>Height</th>
            <th>Weight</th>
            <th>Team</th>
            <th>NOC</th>
            <th>Games</th>
            <th>Year</th>
            <th>Season</th>
            <th>City</th>
            <th>Sport</th>
            <th>Event</th>
            <th>Medal</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>1</td>
            <td>A Dijiang</td>
            <td>M</td>
            <td>24.0</td>
            <td>180.0</td>
            <td>80.0</td>
            <td>China</td>
            <td>CHN</td>
            <td>1992 Summer</td>
            <td>1992</td>
            <td>Summer</td>
            <td>Barcelona</td>
            <td>Basketball</td>
            <td>Basketball Men's Basketball</td>
            <td>No Medal</td>
        </tr>
        <tr>
            <td>2</td>
            <td>A Lamusi</td>
            <td>M</td>
            <td>23.0</td>
            <td>170.0</td>
            <td>60.0</td>
            <td>China</td>
            <td>CHN</td>
            <td>2012 Summer</td>
            <td>2012</td>
            <td>Summer</td>
            <td>London</td>
            <td>Judo</td>
            <td>Judo Men's Extra-Lightweight</td>
            <td>No Medal</td>
        </tr>
        <tr>
            <td>3</td>
            <td>Gunnar Nielsen Aaby</td>
            <td>M</td>
            <td>24.0</td>
            <td>None</td>
            <td>None</td>
            <td>Denmark</td>
            <td>DEN</td>
            <td>1920 Summer</td>
            <td>1920</td>
            <td>Summer</td>
            <td>Antwerpen</td>
            <td>Football</td>
            <td>Football Men's Football</td>
            <td>No Medal</td>
        </tr>
        <tr>
            <td>5</td>
            <td>Christine Jacoba Aaftink</td>
            <td>F</td>
            <td>21.0</td>
            <td>185.0</td>
            <td>82.0</td>
            <td>Netherlands</td>
            <td>NED</td>
            <td>1988 Winter</td>
            <td>1988</td>
            <td>Winter</td>
            <td>Calgary</td>
            <td>Speed Skating</td>
            <td>Speed Skating Women's 500 metres</td>
            <td>No Medal</td>
        </tr>
        <tr>
            <td>5</td>
            <td>Christine Jacoba Aaftink</td>
            <td>F</td>
            <td>21.0</td>
            <td>185.0</td>
            <td>82.0</td>
            <td>Netherlands</td>
            <td>NED</td>
            <td>1988 Winter</td>
            <td>1988</td>
            <td>Winter</td>
            <td>Calgary</td>
            <td>Speed Skating</td>
            <td>Speed Skating Women's 1,000 metres</td>
            <td>No Medal</td>
        </tr>
    </tbody>
</table>



229906 null values have been changed to "No Medal" in the Medal column. Something to note is that we originally anticipated 231333 null values, so there would be 1427 values unaccounted for. What needs to be taken into consideration is that the 1455 duplicate rows were removed first before we changed the values. From there, we understand that amongst the 1455 duplicates, 1427 of them had null values for the medals column.

#### NOC in ath_events

We will now proceed with the unknown values of NOC in ath_events. First, we must find which observations show for unknown values so we can research them and work from there.


```sql
%%sql
SELECT *
FROM ath_events
WHERE NOC = "UNK";
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>ID</th>
            <th>Name</th>
            <th>Sex</th>
            <th>Age</th>
            <th>Height</th>
            <th>Weight</th>
            <th>Team</th>
            <th>NOC</th>
            <th>Games</th>
            <th>Year</th>
            <th>Season</th>
            <th>City</th>
            <th>Sport</th>
            <th>Event</th>
            <th>Medal</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>31292</td>
            <td>Fritz Eccard</td>
            <td>M</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
            <td>Unknown</td>
            <td>UNK</td>
            <td>1912 Summer</td>
            <td>1912</td>
            <td>Summer</td>
            <td>Stockholm</td>
            <td>Art Competitions</td>
            <td>Art Competitions Mixed Architecture</td>
            <td>No Medal</td>
        </tr>
        <tr>
            <td>65813</td>
            <td>A. Laffen</td>
            <td>M</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
            <td>Unknown</td>
            <td>UNK</td>
            <td>1912 Summer</td>
            <td>1912</td>
            <td>Summer</td>
            <td>Stockholm</td>
            <td>Art Competitions</td>
            <td>Art Competitions Mixed Architecture</td>
            <td>No Medal</td>
        </tr>
    </tbody>
</table>



When we search for these two individuals on Olympedia, they show records for their participation also by the code UNK to show their unknown origins. It appears that Olympedia does not know which country they are from, and if I had to make a hypothesis, here's something to take into consideration:

There are some countries/nations/empires that existed in 1912 that no longer exist today. Some examples include the Ottoman Empire, the Ethiopian Empire, and Czechslovakia. Based on the naming scheme of the participants, it is more than likely that their background lies within Europe rather than Africa or near Russia. Pair that with the understanding that the Ottoman Empire dissolved in 1923, which means it is less likely to maintain records compared to the other nations/empires which dissolved later in the 20th century, and there is a decent chance that these participants hail from the Ottoman Empire. However, seeing as there doesn't exist a NOC code for them and that there isn't any feasible way to confirm this information, we can leave this as is.

#### Region in noc_regions


```sql
%%sql
SELECT *
FROM noc_regions
WHERE Region IS NULL;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>NOC</th>
            <th>region</th>
            <th>notes</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>ROT</td>
            <td>None</td>
            <td>Refugee Olympic Team</td>
        </tr>
        <tr>
            <td>TUV</td>
            <td>None</td>
            <td>Tuvalu</td>
        </tr>
        <tr>
            <td>UNK</td>
            <td>None</td>
            <td>Unknown</td>
        </tr>
    </tbody>
</table>




```sql
%%sql
SELECT *
FROM noc_regions
WHERE NOC = "IOA";
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>NOC</th>
            <th>region</th>
            <th>notes</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>IOA</td>
            <td>Individual Olympic Athletes</td>
            <td>Individual Olympic Athletes</td>
        </tr>
    </tbody>
</table>



We must note here that while the Refugee Olympic Team is not given its own region, the Individual Olympic Athletes is given their own region titled as Individual Olympic Athletes. As such, to normalize the data and fill the missing value for ROT, we should have the region for ROT be recognized as Refugee Olympic Team as well. The same will go for Tuvalu and Unknown, especially since there exists a Team value in ath_events called "Unknown".


```sql
%%sql
UPDATE noc_regions
SET region = "Refugee Olympic Team"
WHERE NOC = "ROT";
UPDATE noc_regions
SET region = "Tuvalu"
WHERE NOC = "TUV";
UPDATE noc_regions
SET region = "Unknown"
WHERE NOC = "UNK";
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>



<span style="color: green">1 rows affected.</span>



<span style="color: green">1 rows affected.</span>



<span style="color: green">1 rows affected.</span>





<table>
    <thead>
        <tr>
        </tr>
    </thead>
    <tbody>
    </tbody>
</table>




```sql
%%sql
SELECT *
FROM noc_regions
WHERE NOC IN ("ROT", "TUV", "UNK");
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>NOC</th>
            <th>region</th>
            <th>notes</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>ROT</td>
            <td>Refugee Olympic Team</td>
            <td>Refugee Olympic Team</td>
        </tr>
        <tr>
            <td>TUV</td>
            <td>Tuvalu</td>
            <td>Tuvalu</td>
        </tr>
        <tr>
            <td>UNK</td>
            <td>Unknown</td>
            <td>Unknown</td>
        </tr>
    </tbody>
</table>



The 3 null values from Region have been imputed.

### Step 3: Correct Inaccuracies

We have already corrected an inaccuracy with the name of Mary Musoke in the previous section. There may be more inaccuracies when it comes to names, but this is something we should focus on in future work. For now, the biggest priority is handling the inaccuracies of the dual region teams in the Team column. To start, we will work on each pairing one by one and understand where discrepancies may lie by cross referencing with Olympedia.org.


```sql
%%sql
SELECT Team, NOC, COUNT(*)
FROM ath_events
WHERE Team REGEXP '^.*\/.*$'
GROUP BY Team, NOC
ORDER BY Team, NOC
LIMIT 3
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>Team</th>
            <th>NOC</th>
            <th>COUNT(*)</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>Australia/Great Britain</td>
            <td>AUS</td>
            <td>1</td>
        </tr>
        <tr>
            <td>Australia/Great Britain</td>
            <td>GBR</td>
            <td>1</td>
        </tr>
        <tr>
            <td>Barion/Bari-2</td>
            <td>ITA</td>
            <td>3</td>
        </tr>
    </tbody>
</table>



After seeing how many teams there are listed here in this query, researching through Olympedia (see Olympics Research Appendix) gave us the following insights for data cleaning for each dual team (Full team names are abbreviated for simplicity):

- AUS/GBR: Team name should only represent Great Britain since Australia was under British rule until 1901, and the joint team was established in the 1896 games.
- Barion/Bari-2: Team name should be changed to Bari to represent the only city on record for representation from this team.
- BOH/GBR: Inaccuracy with Warden. Did not represent Great Britain but rather France. Was born in GBR, which explains discrepancy.
- DEN/SWE: Records are accurate. No change.
- GBR/FRA: Records are accurate. No change.
- GER/USA: Records are accurate. No change.
- GBR/GER: Records are accurate. No change.
- Pannonia RC/National RC: Records are accurate. Change can be made, but is not suggested.
- Pistoja/Firenze: Records are accurate. Change can be made, but is not suggested.
- USA/FRA: Records are accurate. No change.
- USA/GBR: Records are accurate for Tennis Mixed Doubles but not for Tennis Men's Doubles. Each participant in the "USA/GBR" team for that event actually represented two separate French tennis clubs, and then joined as one team both representing France. Team name and NOC must be changed.

To summarize, AUS/GBR, Barion/Bari-2, BOH/GBR, and USA/GBR require cleaning.

**Australia/Great Britain**                             


```sql
%%sql
-- Observing Australia/Great Britain
SELECT *
FROM ath_events
WHERE Team = "Australia/Great Britain"
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>ID</th>
            <th>Name</th>
            <th>Sex</th>
            <th>Age</th>
            <th>Height</th>
            <th>Weight</th>
            <th>Team</th>
            <th>NOC</th>
            <th>Games</th>
            <th>Year</th>
            <th>Season</th>
            <th>City</th>
            <th>Sport</th>
            <th>Event</th>
            <th>Medal</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>35698</td>
            <td>Edwin Harold "Teddy" Flack</td>
            <td>M</td>
            <td>22.0</td>
            <td>None</td>
            <td>None</td>
            <td>Australia/Great Britain</td>
            <td>AUS</td>
            <td>1896 Summer</td>
            <td>1896</td>
            <td>Summer</td>
            <td>Athina</td>
            <td>Tennis</td>
            <td>Tennis Men's Doubles</td>
            <td>Bronze</td>
        </tr>
        <tr>
            <td>101352</td>
            <td>George Stuart Robertson</td>
            <td>M</td>
            <td>23.0</td>
            <td>None</td>
            <td>None</td>
            <td>Australia/Great Britain</td>
            <td>GBR</td>
            <td>1896 Summer</td>
            <td>1896</td>
            <td>Summer</td>
            <td>Athina</td>
            <td>Tennis</td>
            <td>Tennis Men's Doubles</td>
            <td>Bronze</td>
        </tr>
    </tbody>
</table>




```sql
%%sql
-- Observing "Teddy" Flack
SELECT *
FROM ath_events
WHERE ID = 35698
LIMIT 3;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>ID</th>
            <th>Name</th>
            <th>Sex</th>
            <th>Age</th>
            <th>Height</th>
            <th>Weight</th>
            <th>Team</th>
            <th>NOC</th>
            <th>Games</th>
            <th>Year</th>
            <th>Season</th>
            <th>City</th>
            <th>Sport</th>
            <th>Event</th>
            <th>Medal</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>35698</td>
            <td>Edwin Harold "Teddy" Flack</td>
            <td>M</td>
            <td>22.0</td>
            <td>None</td>
            <td>None</td>
            <td>Australia</td>
            <td>AUS</td>
            <td>1896 Summer</td>
            <td>1896</td>
            <td>Summer</td>
            <td>Athina</td>
            <td>Tennis</td>
            <td>Tennis Men's Singles</td>
            <td>No Medal</td>
        </tr>
        <tr>
            <td>35698</td>
            <td>Edwin Harold "Teddy" Flack</td>
            <td>M</td>
            <td>22.0</td>
            <td>None</td>
            <td>None</td>
            <td>Australia/Great Britain</td>
            <td>AUS</td>
            <td>1896 Summer</td>
            <td>1896</td>
            <td>Summer</td>
            <td>Athina</td>
            <td>Tennis</td>
            <td>Tennis Men's Doubles</td>
            <td>Bronze</td>
        </tr>
        <tr>
            <td>35698</td>
            <td>Edwin Harold "Teddy" Flack</td>
            <td>M</td>
            <td>22.0</td>
            <td>None</td>
            <td>None</td>
            <td>Australia</td>
            <td>AUS</td>
            <td>1896 Summer</td>
            <td>1896</td>
            <td>Summer</td>
            <td>Athina</td>
            <td>Athletics</td>
            <td>Athletics Men's 800 metres</td>
            <td>Gold</td>
        </tr>
    </tbody>
</table>




```sql
%%sql
-- Observing George Robinson
SELECT *
FROM ath_events
WHERE ID = 101352;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>ID</th>
            <th>Name</th>
            <th>Sex</th>
            <th>Age</th>
            <th>Height</th>
            <th>Weight</th>
            <th>Team</th>
            <th>NOC</th>
            <th>Games</th>
            <th>Year</th>
            <th>Season</th>
            <th>City</th>
            <th>Sport</th>
            <th>Event</th>
            <th>Medal</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>101352</td>
            <td>George Stuart Robertson</td>
            <td>M</td>
            <td>23.0</td>
            <td>None</td>
            <td>None</td>
            <td>Great Britain</td>
            <td>GBR</td>
            <td>1896 Summer</td>
            <td>1896</td>
            <td>Summer</td>
            <td>Athina</td>
            <td>Tennis</td>
            <td>Tennis Men's Singles</td>
            <td>No Medal</td>
        </tr>
        <tr>
            <td>101352</td>
            <td>George Stuart Robertson</td>
            <td>M</td>
            <td>23.0</td>
            <td>None</td>
            <td>None</td>
            <td>Australia/Great Britain</td>
            <td>GBR</td>
            <td>1896 Summer</td>
            <td>1896</td>
            <td>Summer</td>
            <td>Athina</td>
            <td>Tennis</td>
            <td>Tennis Men's Doubles</td>
            <td>Bronze</td>
        </tr>
        <tr>
            <td>101352</td>
            <td>George Stuart Robertson</td>
            <td>M</td>
            <td>23.0</td>
            <td>None</td>
            <td>None</td>
            <td>Great Britain</td>
            <td>GBR</td>
            <td>1896 Summer</td>
            <td>1896</td>
            <td>Summer</td>
            <td>Athina</td>
            <td>Athletics</td>
            <td>Athletics Men's Discus Throw</td>
            <td>No Medal</td>
        </tr>
    </tbody>
</table>




```sql
%%sql
-- Correcting Inaccuracies
UPDATE ath_events
SET Team = "Great Britain", NOC = "GBR"
WHERE Year IN (1896, 1900) AND (Team = "Australia/Great Britain" OR NOC = "AUS")
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>



<span style="color: green">12 rows affected.</span>





<table>
    <thead>
        <tr>
        </tr>
    </thead>
    <tbody>
    </tbody>
</table>




```sql
%%sql
-- Review Changes
SELECT *
FROM ath_events
WHERE ID = 35698 OR ID = 101352
LIMIT 3;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>ID</th>
            <th>Name</th>
            <th>Sex</th>
            <th>Age</th>
            <th>Height</th>
            <th>Weight</th>
            <th>Team</th>
            <th>NOC</th>
            <th>Games</th>
            <th>Year</th>
            <th>Season</th>
            <th>City</th>
            <th>Sport</th>
            <th>Event</th>
            <th>Medal</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>35698</td>
            <td>Edwin Harold "Teddy" Flack</td>
            <td>M</td>
            <td>22.0</td>
            <td>None</td>
            <td>None</td>
            <td>Great Britain</td>
            <td>GBR</td>
            <td>1896 Summer</td>
            <td>1896</td>
            <td>Summer</td>
            <td>Athina</td>
            <td>Tennis</td>
            <td>Tennis Men's Singles</td>
            <td>No Medal</td>
        </tr>
        <tr>
            <td>35698</td>
            <td>Edwin Harold "Teddy" Flack</td>
            <td>M</td>
            <td>22.0</td>
            <td>None</td>
            <td>None</td>
            <td>Great Britain</td>
            <td>GBR</td>
            <td>1896 Summer</td>
            <td>1896</td>
            <td>Summer</td>
            <td>Athina</td>
            <td>Tennis</td>
            <td>Tennis Men's Doubles</td>
            <td>Bronze</td>
        </tr>
        <tr>
            <td>35698</td>
            <td>Edwin Harold "Teddy" Flack</td>
            <td>M</td>
            <td>22.0</td>
            <td>None</td>
            <td>None</td>
            <td>Great Britain</td>
            <td>GBR</td>
            <td>1896 Summer</td>
            <td>1896</td>
            <td>Summer</td>
            <td>Athina</td>
            <td>Athletics</td>
            <td>Athletics Men's 800 metres</td>
            <td>Gold</td>
        </tr>
    </tbody>
</table>



The changes accounted for 12 rows

**Barion/Bari-2**


```sql
%%sql
SELECT *
FROM ath_events
WHERE Team = "Barion/Bari-2";
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>ID</th>
            <th>Name</th>
            <th>Sex</th>
            <th>Age</th>
            <th>Height</th>
            <th>Weight</th>
            <th>Team</th>
            <th>NOC</th>
            <th>Games</th>
            <th>Year</th>
            <th>Season</th>
            <th>City</th>
            <th>Sport</th>
            <th>Event</th>
            <th>Medal</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>19459</td>
            <td>Emilio Cesarana</td>
            <td>M</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
            <td>Barion/Bari-2</td>
            <td>ITA</td>
            <td>1906 Summer</td>
            <td>1906</td>
            <td>Summer</td>
            <td>Athina</td>
            <td>Rowing</td>
            <td>Rowing Men's Coxed Pairs (1 kilometres)</td>
            <td>Silver</td>
        </tr>
        <tr>
            <td>21717</td>
            <td>Francesco Civera</td>
            <td>M</td>
            <td>23.0</td>
            <td>None</td>
            <td>None</td>
            <td>Barion/Bari-2</td>
            <td>ITA</td>
            <td>1906 Summer</td>
            <td>1906</td>
            <td>Summer</td>
            <td>Athina</td>
            <td>Rowing</td>
            <td>Rowing Men's Coxed Pairs (1 kilometres)</td>
            <td>Silver</td>
        </tr>
        <tr>
            <td>28174</td>
            <td>Luigi Diana</td>
            <td>M</td>
            <td>40.0</td>
            <td>None</td>
            <td>None</td>
            <td>Barion/Bari-2</td>
            <td>ITA</td>
            <td>1906 Summer</td>
            <td>1906</td>
            <td>Summer</td>
            <td>Athina</td>
            <td>Rowing</td>
            <td>Rowing Men's Coxed Pairs (1 kilometres)</td>
            <td>Silver</td>
        </tr>
    </tbody>
</table>




```sql
%%sql
UPDATE ath_events
SET Team = "Bari"
WHERE Team = "Barion/Bari-2";
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>



<span style="color: green">3 rows affected.</span>





<table>
    <thead>
        <tr>
        </tr>
    </thead>
    <tbody>
    </tbody>
</table>




```sql
%%sql
SELECT *
FROM ath_events
WHERE Team = "Bari";
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>ID</th>
            <th>Name</th>
            <th>Sex</th>
            <th>Age</th>
            <th>Height</th>
            <th>Weight</th>
            <th>Team</th>
            <th>NOC</th>
            <th>Games</th>
            <th>Year</th>
            <th>Season</th>
            <th>City</th>
            <th>Sport</th>
            <th>Event</th>
            <th>Medal</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>19459</td>
            <td>Emilio Cesarana</td>
            <td>M</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
            <td>Bari</td>
            <td>ITA</td>
            <td>1906 Summer</td>
            <td>1906</td>
            <td>Summer</td>
            <td>Athina</td>
            <td>Rowing</td>
            <td>Rowing Men's Coxed Pairs (1 kilometres)</td>
            <td>Silver</td>
        </tr>
        <tr>
            <td>21717</td>
            <td>Francesco Civera</td>
            <td>M</td>
            <td>23.0</td>
            <td>None</td>
            <td>None</td>
            <td>Bari</td>
            <td>ITA</td>
            <td>1906 Summer</td>
            <td>1906</td>
            <td>Summer</td>
            <td>Athina</td>
            <td>Rowing</td>
            <td>Rowing Men's Coxed Pairs (1 kilometres)</td>
            <td>Silver</td>
        </tr>
        <tr>
            <td>28174</td>
            <td>Luigi Diana</td>
            <td>M</td>
            <td>40.0</td>
            <td>None</td>
            <td>None</td>
            <td>Bari</td>
            <td>ITA</td>
            <td>1906 Summer</td>
            <td>1906</td>
            <td>Summer</td>
            <td>Athina</td>
            <td>Rowing</td>
            <td>Rowing Men's Coxed Pairs (1 kilometres)</td>
            <td>Silver</td>
        </tr>
    </tbody>
</table>



The changes accounted for 3 rows

**Bohemia/Great Britain**


```sql
%%sql
SELECT *
FROM ath_events
WHERE Team = "Bohemia/Great Britain";
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>ID</th>
            <th>Name</th>
            <th>Sex</th>
            <th>Age</th>
            <th>Height</th>
            <th>Weight</th>
            <th>Team</th>
            <th>NOC</th>
            <th>Games</th>
            <th>Year</th>
            <th>Season</th>
            <th>City</th>
            <th>Sport</th>
            <th>Event</th>
            <th>Medal</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>102589</td>
            <td>Hedwiga Rosenbaumov (Austerlitz-, -Raabe)</td>
            <td>F</td>
            <td>35.0</td>
            <td>None</td>
            <td>None</td>
            <td>Bohemia/Great Britain</td>
            <td>BOH</td>
            <td>1900 Summer</td>
            <td>1900</td>
            <td>Summer</td>
            <td>Paris</td>
            <td>Tennis</td>
            <td>Tennis Mixed Doubles</td>
            <td>Bronze</td>
        </tr>
        <tr>
            <td>128739</td>
            <td>Archibald Adam Warden</td>
            <td>M</td>
            <td>31.0</td>
            <td>None</td>
            <td>None</td>
            <td>Bohemia/Great Britain</td>
            <td>GBR</td>
            <td>1900 Summer</td>
            <td>1900</td>
            <td>Summer</td>
            <td>Paris</td>
            <td>Tennis</td>
            <td>Tennis Mixed Doubles</td>
            <td>Bronze</td>
        </tr>
    </tbody>
</table>




```sql
%%sql
UPDATE ath_events
SET Team = "Bohemia/France"
WHERE Team = "Bohemia/Great Britain";
UPDATE ath_events
SET NOC = "FRA"
WHERE Team = "Bohemia/France" AND NOC = "GBR";
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>



<span style="color: green">2 rows affected.</span>



<span style="color: green">1 rows affected.</span>





<table>
    <thead>
        <tr>
        </tr>
    </thead>
    <tbody>
    </tbody>
</table>




```sql
%%sql
SELECT *
FROM ath_events
WHERE Team = "Bohemia/France";
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>ID</th>
            <th>Name</th>
            <th>Sex</th>
            <th>Age</th>
            <th>Height</th>
            <th>Weight</th>
            <th>Team</th>
            <th>NOC</th>
            <th>Games</th>
            <th>Year</th>
            <th>Season</th>
            <th>City</th>
            <th>Sport</th>
            <th>Event</th>
            <th>Medal</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>102589</td>
            <td>Hedwiga Rosenbaumov (Austerlitz-, -Raabe)</td>
            <td>F</td>
            <td>35.0</td>
            <td>None</td>
            <td>None</td>
            <td>Bohemia/France</td>
            <td>BOH</td>
            <td>1900 Summer</td>
            <td>1900</td>
            <td>Summer</td>
            <td>Paris</td>
            <td>Tennis</td>
            <td>Tennis Mixed Doubles</td>
            <td>Bronze</td>
        </tr>
        <tr>
            <td>128739</td>
            <td>Archibald Adam Warden</td>
            <td>M</td>
            <td>31.0</td>
            <td>None</td>
            <td>None</td>
            <td>Bohemia/France</td>
            <td>FRA</td>
            <td>1900 Summer</td>
            <td>1900</td>
            <td>Summer</td>
            <td>Paris</td>
            <td>Tennis</td>
            <td>Tennis Mixed Doubles</td>
            <td>Bronze</td>
        </tr>
    </tbody>
</table>



The changes accounted for 2 rows. 2 team names and 1 NOC has been changed.

**United States/Great Britain**


```sql
%%sql
SELECT *
FROM ath_events
WHERE Team = "United States/Great Britain"
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>ID</th>
            <th>Name</th>
            <th>Sex</th>
            <th>Age</th>
            <th>Height</th>
            <th>Weight</th>
            <th>Team</th>
            <th>NOC</th>
            <th>Games</th>
            <th>Year</th>
            <th>Season</th>
            <th>City</th>
            <th>Sport</th>
            <th>Event</th>
            <th>Medal</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>29107</td>
            <td>Hugh Lawrence "Laurie" Doherty</td>
            <td>M</td>
            <td>24.0</td>
            <td>178.0</td>
            <td>None</td>
            <td>United States/Great Britain</td>
            <td>GBR</td>
            <td>1900 Summer</td>
            <td>1900</td>
            <td>Summer</td>
            <td>Paris</td>
            <td>Tennis</td>
            <td>Tennis Mixed Doubles</td>
            <td>Bronze</td>
        </tr>
        <tr>
            <td>55702</td>
            <td>Marion Jones (-Farquhar)</td>
            <td>F</td>
            <td>20.0</td>
            <td>None</td>
            <td>None</td>
            <td>United States/Great Britain</td>
            <td>USA</td>
            <td>1900 Summer</td>
            <td>1900</td>
            <td>Summer</td>
            <td>Paris</td>
            <td>Tennis</td>
            <td>Tennis Mixed Doubles</td>
            <td>Bronze</td>
        </tr>
        <tr>
            <td>105390</td>
            <td>Charles Edward Sands</td>
            <td>M</td>
            <td>34.0</td>
            <td>181.0</td>
            <td>None</td>
            <td>United States/Great Britain</td>
            <td>USA</td>
            <td>1900 Summer</td>
            <td>1900</td>
            <td>Summer</td>
            <td>Paris</td>
            <td>Tennis</td>
            <td>Tennis Men's Doubles</td>
            <td>No Medal</td>
        </tr>
        <tr>
            <td>128739</td>
            <td>Archibald Adam Warden</td>
            <td>M</td>
            <td>31.0</td>
            <td>None</td>
            <td>None</td>
            <td>United States/Great Britain</td>
            <td>GBR</td>
            <td>1900 Summer</td>
            <td>1900</td>
            <td>Summer</td>
            <td>Paris</td>
            <td>Tennis</td>
            <td>Tennis Men's Doubles</td>
            <td>No Medal</td>
        </tr>
    </tbody>
</table>




```sql
%%sql
UPDATE ath_events
SET Team = "TC de Puteaux/Island TC", NOC = "FRA"
WHERE (ID = 105390 OR ID = 128739) AND Team = "United States/Great Britain";
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>



<span style="color: green">2 rows affected.</span>





<table>
    <thead>
        <tr>
        </tr>
    </thead>
    <tbody>
    </tbody>
</table>




```sql
%%sql
-- Observe that Mixed Doubles remains unchanged
SELECT *
FROM ath_events
WHERE Team = "United States/Great Britain"
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>ID</th>
            <th>Name</th>
            <th>Sex</th>
            <th>Age</th>
            <th>Height</th>
            <th>Weight</th>
            <th>Team</th>
            <th>NOC</th>
            <th>Games</th>
            <th>Year</th>
            <th>Season</th>
            <th>City</th>
            <th>Sport</th>
            <th>Event</th>
            <th>Medal</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>29107</td>
            <td>Hugh Lawrence "Laurie" Doherty</td>
            <td>M</td>
            <td>24.0</td>
            <td>178.0</td>
            <td>None</td>
            <td>United States/Great Britain</td>
            <td>GBR</td>
            <td>1900 Summer</td>
            <td>1900</td>
            <td>Summer</td>
            <td>Paris</td>
            <td>Tennis</td>
            <td>Tennis Mixed Doubles</td>
            <td>Bronze</td>
        </tr>
        <tr>
            <td>55702</td>
            <td>Marion Jones (-Farquhar)</td>
            <td>F</td>
            <td>20.0</td>
            <td>None</td>
            <td>None</td>
            <td>United States/Great Britain</td>
            <td>USA</td>
            <td>1900 Summer</td>
            <td>1900</td>
            <td>Summer</td>
            <td>Paris</td>
            <td>Tennis</td>
            <td>Tennis Mixed Doubles</td>
            <td>Bronze</td>
        </tr>
    </tbody>
</table>




```sql
%%sql
-- Observe that Men's Doubles have changed as desired
SELECT *
FROM ath_events
WHERE Team = "TC de Puteaux/Island TC"
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>ID</th>
            <th>Name</th>
            <th>Sex</th>
            <th>Age</th>
            <th>Height</th>
            <th>Weight</th>
            <th>Team</th>
            <th>NOC</th>
            <th>Games</th>
            <th>Year</th>
            <th>Season</th>
            <th>City</th>
            <th>Sport</th>
            <th>Event</th>
            <th>Medal</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>105390</td>
            <td>Charles Edward Sands</td>
            <td>M</td>
            <td>34.0</td>
            <td>181.0</td>
            <td>None</td>
            <td>TC de Puteaux/Island TC</td>
            <td>FRA</td>
            <td>1900 Summer</td>
            <td>1900</td>
            <td>Summer</td>
            <td>Paris</td>
            <td>Tennis</td>
            <td>Tennis Men's Doubles</td>
            <td>No Medal</td>
        </tr>
        <tr>
            <td>128739</td>
            <td>Archibald Adam Warden</td>
            <td>M</td>
            <td>31.0</td>
            <td>None</td>
            <td>None</td>
            <td>TC de Puteaux/Island TC</td>
            <td>FRA</td>
            <td>1900 Summer</td>
            <td>1900</td>
            <td>Summer</td>
            <td>Paris</td>
            <td>Tennis</td>
            <td>Tennis Men's Doubles</td>
            <td>No Medal</td>
        </tr>
    </tbody>
</table>



The changes accounted for 2 rows.

**A total of 19 rows have had their inaccuracies corrected.**

### Step 4: Standardize Formats

The biggest correction that needs to be made in terms of standardizations is the NOC columns when dealing with Singapore. Both SIN and SGP can be used as symbols to represent Singapore and are technically correct, but while research revealed that some years used SIN while others used SGP, ath_events shows that only one is used over the other while noc_regions uses another. If we wish to maintain historical accuracy, we would make the changes to account for both NOC codes. However, for the sake of future analysis, which may include geographical analysis based on NOC codes, it is more practical to standardize the formatting to make sure that only one code refers to the region in question. As such, we will change the code from noc_region since it is the most efficient option.



```sql
%%sql
SELECT *
FROM noc_regions
WHERE NOC IN ("SGP","SIN");
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>NOC</th>
            <th>region</th>
            <th>notes</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>SIN</td>
            <td>Singapore</td>
            <td>None</td>
        </tr>
    </tbody>
</table>




```sql
%%sql
UPDATE noc_regions
SET NOC = "SGP"
WHERE NOC = "SIN"
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>



<span style="color: green">1 rows affected.</span>





<table>
    <thead>
        <tr>
        </tr>
    </thead>
    <tbody>
    </tbody>
</table>




```sql
%%sql
SELECT *
FROM noc_regions
WHERE NOC IN ("SGP","SIN");
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>NOC</th>
            <th>region</th>
            <th>notes</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>SGP</td>
            <td>Singapore</td>
            <td>None</td>
        </tr>
    </tbody>
</table>



### Step 5: Manage Outliers

In the context of this dataset, while the only outliers we could note are the Age, Height, and Weight of Olympics participants as well as the number of teams paired with the NOC value "FRA", there is no context within the data that suggests that the datapoints need to be removed from consideration. All available data has been either validated, cross-referenced, or considered in terms of their potential discrepancies. The "outliers" have their place in future analysis, and therefore should not be removed.

## V. Summary of Cleaning Actions

| Column | Data Quality Category | Issue | Records Affected | Priority | Method of Issue Resolution (if applicable) | Records Cleaned | Remaining Affected |
| ------ | --------------------- | ----- | ---------------- | -------- | ------------------------------------------ | --------------- | ---------|
| Name | Accuracy | No Notable Issues, but they may appear during manual corrections | NA | Low | If found, correct appropriately | 1 | NA |
| Age | Completeness | Many missing values, more during older times rather than recent years | 9,474 | **High** | Cross reference with Olympedia for recent years and fill in wherever you can manually, document issue for any others remaining for future work | 24 | 9,272 |
| Height | Completeness | Many missing values, more during older times rather than recent years | 60,171 | **High** | Cross reference with Olympedia for recent years and fill in wherever you can manually, document issue for any others remaining for future work | 15 | 58,730 |
| Weight | Completeness | Many missing values, more during older times rather than recent years | 62,875 | **High** | Cross reference with Olympedia for recent years and fill in wherever you can manually, document issue for any others remaining for future work | 15 | 61,443 |
| Medal | Completeness | Missing values for what should be documented as instances of no medals being earned | 231,333 | **High** | Impute "No Medal" value on all null values | 229,906 | 0
| Region | Completeness | Missing values associated with NOCs ROT,TUV, and UNC | 3 | Low | Check notes and impute values appropriately | 3 | 0
| Team | Validity | Instances of two teams combined indicates issues in accuracy | <43 | **High** | Cross reference with Olympedia to correct any inaccuracies | 19 | 0 |
| Team | Validity | Many instances of teams that don't directly reference a region may harbor potential inaccuracies | NA | Low | Document issue for future work | 0 | NA |
| NOC | Consistency | Inconsistency with SIN/SGP for Singapore between ath_events and noc_regions | <290 | *Medium* | Direct update in noc_regions | 1 | 0 |
| Year | Timeliness | Data extends to 2016, and games from years beyond this could be included | NA | *Medium* | Document for future work | 0 | NA |
| ID, Team, Games, Event | Uniqueness | Many instances of duplicate rows | 1,455 | **High** | Identify and remove using partition of dataset as pseudo-primary key | 1,455 | 0 |

## VI. Future Work

Here are a list of potential tasks that would further the cleaning of this dataset:

- Complete Cross-Reference with Olympedia to fill in as many gaps as possible for Age, Height, Weight, and confirm whether or not people received medals or if the information is not available.
    - Suggestion: Apply data scraping and wrangling practices to Olympedia.org and compare the datasets to the datasets of www.sports-reference.com
- Update the data to consist of the four olympic games of 2018, 2020, 2022, and 2024.

## VII. Export Cleaned Data

Once the entire code has ran through to update the databases to work with cleaner data, you will wish to export the data for reference in case any transformations need to happen in the future. Here is the code that will allow you to obtain the cleaned datasets in CSV files:


```python
# Export cleaned data for future use
ath_events_cleaned = pd.read_sql("SELECT * FROM ath_events", conn)
noc_regions_cleaned = pd.read_sql("SELECT * FROM noc_regions", conn)

# Save to CSV
ath_events_cleaned.to_csv("athlete_events_cleaned.csv", index=False)
noc_regions_cleaned.to_csv("noc_regions_cleaned.csv", index=False)
```


```python

```
