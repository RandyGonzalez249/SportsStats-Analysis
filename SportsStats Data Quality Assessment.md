# SportsStats Data Quality Assessment

**Table of Contents**

1. Introduction
2. Load Raw Data
3. Schema Overview and First Look
4. Data Quality Assessment
    - Dimension 1 — Accuracy
    - Dimension 2 — Completeness
    - Dimension 3 — Validity
    - Dimension 4 — Consistency
    - Dimension 5 — Timeliness
    - Dimension 6 — Uniqueness
    - Dimension 7 — Reliability
5. Summary of Issues Detected

---

## I. Introduction

The client I will be working with is SportsStats, a sports analysis firm partnering with local news and elite personal trainers to provide insights to help their partners. They recognize patterns/trends highlighting certain groups, events, countries, etc. for the purpose of developing a news story or discovering key health insights. As someone with a personal history with several sports such as basketball, martial arts, fencing, bicycling, and more, being able to perform analysis to gain insights on sports is something that personally engages me. Additionally, I reside within a culture that is heavily influenced by health and well-being as an ideal to pursue, so I find that this analysis that will be able to provide key health insights will provide value not only to the client I will be working with in this project, but also to the general public as well.

SportsStats has provided two related datasets, and the aim of this notebook will be to assess the quality of the data to better understand what needs to be cleaned before we can procede with analysis. The data consists of 120 years worth of data regarding the Olympics from 1896 to 2016, with a reference dataset consisting of various National Olympic Committee (or NOC) codes along with the regions they represent. The notebook here will not only provide what has been checked, but also how SQLite is used to discover some of the discrepancies. Note that more understanding from research is necessary to contextualize the discrepancies, and will be available through a separate appendix titled "Olympics Research Appendix".

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

*A note before proceeding with the DQA, most of the code below will provide outputs for the first three rows simply for presentation purposes. To properly understand the data and its quality, you may need to alter the code to provide more rows.*

---

## III. Schema Overview and First Look

From here, we will observe the table structure of both the `ath_events` and `noc_regions` tables as well as a the first few rows of each table to gain an understanding of the schema and note any observations that may be useful for context as we proceed with the DQA.


```python
%sql PRAGMA table_info(ath_events)
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>cid</th>
            <th>name</th>
            <th>type</th>
            <th>notnull</th>
            <th>dflt_value</th>
            <th>pk</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>0</td>
            <td>ID</td>
            <td>INTEGER</td>
            <td>0</td>
            <td>None</td>
            <td>0</td>
        </tr>
        <tr>
            <td>1</td>
            <td>Name</td>
            <td>TEXT</td>
            <td>0</td>
            <td>None</td>
            <td>0</td>
        </tr>
        <tr>
            <td>2</td>
            <td>Sex</td>
            <td>TEXT</td>
            <td>0</td>
            <td>None</td>
            <td>0</td>
        </tr>
        <tr>
            <td>3</td>
            <td>Age</td>
            <td>REAL</td>
            <td>0</td>
            <td>None</td>
            <td>0</td>
        </tr>
        <tr>
            <td>4</td>
            <td>Height</td>
            <td>REAL</td>
            <td>0</td>
            <td>None</td>
            <td>0</td>
        </tr>
        <tr>
            <td>5</td>
            <td>Weight</td>
            <td>REAL</td>
            <td>0</td>
            <td>None</td>
            <td>0</td>
        </tr>
        <tr>
            <td>6</td>
            <td>Team</td>
            <td>TEXT</td>
            <td>0</td>
            <td>None</td>
            <td>0</td>
        </tr>
        <tr>
            <td>7</td>
            <td>NOC</td>
            <td>TEXT</td>
            <td>0</td>
            <td>None</td>
            <td>0</td>
        </tr>
        <tr>
            <td>8</td>
            <td>Games</td>
            <td>TEXT</td>
            <td>0</td>
            <td>None</td>
            <td>0</td>
        </tr>
        <tr>
            <td>9</td>
            <td>Year</td>
            <td>INTEGER</td>
            <td>0</td>
            <td>None</td>
            <td>0</td>
        </tr>
        <tr>
            <td>10</td>
            <td>Season</td>
            <td>TEXT</td>
            <td>0</td>
            <td>None</td>
            <td>0</td>
        </tr>
        <tr>
            <td>11</td>
            <td>City</td>
            <td>TEXT</td>
            <td>0</td>
            <td>None</td>
            <td>0</td>
        </tr>
        <tr>
            <td>12</td>
            <td>Sport</td>
            <td>TEXT</td>
            <td>0</td>
            <td>None</td>
            <td>0</td>
        </tr>
        <tr>
            <td>13</td>
            <td>Event</td>
            <td>TEXT</td>
            <td>0</td>
            <td>None</td>
            <td>0</td>
        </tr>
        <tr>
            <td>14</td>
            <td>Medal</td>
            <td>TEXT</td>
            <td>0</td>
            <td>None</td>
            <td>0</td>
        </tr>
    </tbody>
</table>




```python
%sql PRAGMA table_info(noc_regions)
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>cid</th>
            <th>name</th>
            <th>type</th>
            <th>notnull</th>
            <th>dflt_value</th>
            <th>pk</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>0</td>
            <td>NOC</td>
            <td>TEXT</td>
            <td>0</td>
            <td>None</td>
            <td>0</td>
        </tr>
        <tr>
            <td>1</td>
            <td>region</td>
            <td>TEXT</td>
            <td>0</td>
            <td>None</td>
            <td>0</td>
        </tr>
        <tr>
            <td>2</td>
            <td>notes</td>
            <td>TEXT</td>
            <td>0</td>
            <td>None</td>
            <td>0</td>
        </tr>
    </tbody>
</table>




```python
%sql SELECT * FROM ath_events LIMIT 3
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
    </tbody>
</table>




```python
%sql SELECT * FROM noc_regions LIMIT 3
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
            <td>AFG</td>
            <td>Afghanistan</td>
            <td>None</td>
        </tr>
        <tr>
            <td>AHO</td>
            <td>Curacao</td>
            <td>Netherlands Antilles</td>
        </tr>
        <tr>
            <td>ALB</td>
            <td>Albania</td>
            <td>None</td>
        </tr>
    </tbody>
</table>



### Notes on Table Schema:

We have two tables, ath_events with 15 columns, and noc_regions with 3 columns

**Data Types:**

- Columns from ath_events with TEXT data: Name, Sex, Team, NOC, Games, Season, City, Sport, Event, Medal
- Columns from ath_events with INTEGER data: ID, Year
- Columns from ath_events with REAL data: Age, Height, Weight
    - Note that the REAL data is representable wih decimal values, but the information only (if not mainly) consists of integer values. We will keep as is instead of potentially changing to integer to enable future calculations during analysis.
- All columns from noc_regions are TEXT variables.

### Inferences from First Look:

#### ath_events
- We can partition the data uniquely with `ID`, `Games`, `Team`, and `Event`. The combination of those four help identify each observation.
- Multiple copies of the same name can be found in the `Name` and `ID` columns, but this reflects upon different events and/or different years of the games. In other words, the structure of the data takes a long structure, where there would be less columns but more rows to represent the data.
- The medal section appears to contain more Null values than other values such as Gold or Silver. That should mean that we have data on most, if not all of the participants of the games, not just those who won medals during those games, and if they did not receive a medal, their value is relegated to a Null value.
- There are some occurances of Null values for height and weight, which may reflect in missing values due to lack of data as opposed to missing values by design of the data structure. We may be interested in replacing the Null values in the `Medals` column with another value representing not having a medal.
- Upon inspection of the first ten observations, there seems to be a lack of data for `height` and `weight` only during the earlier years of the olympics, such as 1900 and 1920, when the dates of the games which occurred closer to 2000 contains `height` and `weight` data. It could be hypothesized that such data simply was not recorded at the time, and that records start to appear sometime in between the 1900s and the 2000s. Further exploration of the data will need to be made to confirm or challenge this hypothesis.

#### noc_regions 
- The values of `NOC` in this table seem to correspond with the `NOC` values of the `ath_events` data. Additionally, some values of `Region` in this table seem to correspond with some of the `Team` values of the `ath_events` data.

---

## IV. Data Quality Assessment

Professional Data Practices dictate that thorough diagnosis precedes remediation. As such, all columns will be checked systematically before implementing changes. This holistic assessment identifies interdependencies (e.g., Team names must align with NOC codes) and prevents redundant work. The following will be observed to verify data quality:

1. Accuracy
2. Completeness
3. Validity
4. Consistency
5. Timeliness
6. Uniqueness
7. Reliability

*Note: A heavy emphasis on Validity will be placed, as we are verifying the validity of each column, not just observing the data as a whole.

### Dimension 1: Accuracy

The dataset that has been granted to us needs to correctly reflect real-world values. Checking whether or not each and every value that is listed in the database to be accurate would be a grueling task that is outside of the scope of work for this project. However, research on the origin of the dataset shows that it has been scraped from the website "www.sports-reference.com" in May 2018, according to this [Kaggle webpage](https://www.kaggle.com/code/chadalee/olympics-data-cleaning-exploration-prediction) which states that the data was scraped and wrangled through R. From there, further research in the reviews of the company shows that the data they hold is accurate, and later exploration for data quality assessment will show that the data mostly aligns with data from other websites such as "www.olympedia.org".

### Dimension 2: Completeness

We must verify that all required data is present and usable. While we do have much data in the dataset that can be considered present and usable, the dataset also has MANY null values. Below is the SQL showcase of how many null values exist within each column of the tables:


```sql
%%sql
-- The Count function counts all the non null values of a column. 
-- Taking the difference between the number of rows and the number of non-null values 
-- will reveal the number of null values in each column.
SELECT 
    COUNT(*) as "Number of Observations in ath_events",
    COUNT(*) - COUNT(ID) as "Number of Nulls in ID",
    COUNT(*) - COUNT(Name) as "Number of Nulls in Name",
    COUNT(*) - COUNT(Sex) as "Number of Nulls in Sex",
    COUNT(*) - COUNT(Age) as "Number of Nulls in Age",
    COUNT(*) - COUNT(Height) as "Number of Nulls in Height",
    COUNT(*) - COUNT(Weight) as "Number of Nulls in Weight",
    COUNT(*) - COUNT(Team) as "Number of Nulls in Team",
    COUNT(*) - COUNT(NOC) as "Number of Nulls in NOC",
    COUNT(*) - COUNT(Games) as "Number of Nulls in Games",
    COUNT(*) - COUNT(Year) as "Number of Nulls in Year",
    COUNT(*) - COUNT(Season) as "Number of Nulls in Season",
    COUNT(*) - COUNT(City) as "Number of Nulls in City",
    COUNT(*) - COUNT(Sport) as "Number of Nulls in Sport",
    COUNT(*) - COUNT(Event) as "Number of Nulls in Event",
    COUNT(*) - COUNT(Medal) as "Number of Nulls in Medal"
FROM
    ath_events;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>Number of Observations in ath_events</th>
            <th>Number of Nulls in ID</th>
            <th>Number of Nulls in Name</th>
            <th>Number of Nulls in Sex</th>
            <th>Number of Nulls in Age</th>
            <th>Number of Nulls in Height</th>
            <th>Number of Nulls in Weight</th>
            <th>Number of Nulls in Team</th>
            <th>Number of Nulls in NOC</th>
            <th>Number of Nulls in Games</th>
            <th>Number of Nulls in Year</th>
            <th>Number of Nulls in Season</th>
            <th>Number of Nulls in City</th>
            <th>Number of Nulls in Sport</th>
            <th>Number of Nulls in Event</th>
            <th>Number of Nulls in Medal</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>271116</td>
            <td>0</td>
            <td>0</td>
            <td>0</td>
            <td>9474</td>
            <td>60171</td>
            <td>62875</td>
            <td>0</td>
            <td>0</td>
            <td>0</td>
            <td>0</td>
            <td>0</td>
            <td>0</td>
            <td>0</td>
            <td>0</td>
            <td>231333</td>
        </tr>
    </tbody>
</table>




```sql
%%sql
SELECT
    COUNT(*) as "Number of Observations in noc_regions",
    COUNT(*) - COUNT(NOC) as "Number of Nulls in NOC",
    COUNT(*) - COUNT(Region) as "Number of Nulls in Region",
    COUNT(*) - COUNT(notes) as "Number of Nulls in notes"
FROM
    noc_regions;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>Number of Observations in noc_regions</th>
            <th>Number of Nulls in NOC</th>
            <th>Number of Nulls in Region</th>
            <th>Number of Nulls in notes</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>230</td>
            <td>0</td>
            <td>3</td>
            <td>209</td>
        </tr>
    </tbody>
</table>



The following queries tell us this much:

About ath_events:
- There are no null values except in the columns of `Age`, `Height`, `Weight`, and `Medal`.
- For the lack of some data for `Age`, `Height`, and `Weight`, considering that the data spans from before 1900 and ranges 120 years, there may have been a period of time where such information was never considered significant enough to warrant holding records for them until a certain time period when that sentiment changed. This is a hypothesis that will need further investigation to prove.
- For the lack of data for `Medal`, considering that the amount of nulls covers almost all the observations, I am lead to believe that the reason null values exist is because in those circumstances, the person on record did not receive a medal for the event they participated in, and as such, was left empty for simplicity's sake. If this happens to be the case, it may be better to assign a value to represent that no medal was given rather than leaving it blank.

About noc_regions:
- There are no null values in the `NOC` column, but there are for `Region` and `notes`.
- For the lack of data in `notes`, I surmise that notes were taken only when special circumstances should appear within the assignment between `NOC` and `Region` that warrant a needed explanation for clarification, and as such, the null values exist because in those moments, no notes were needed, similar to how the `Medal` column in `ath_events` has many null values that may suggest that no medals were given for those observations
- There are very few Nulls in `Region`, so we may wish to investigate and see how we can replace the missing data wherever possible.

### Dimension 3: Validity

We must check that the data falls within expected ranges and is consistent with real-life conditions. We will check each column to see if the data falls within expected ranges.

- ID
    - If we work with the understanding that the ID number is assigned sequentially, prove this is the case.
- Name
    - Check that each ID is assigned to only one name.
- Sex
    - Check how many different categories there are, and if there may be some discrepancies.
- Age
    - Current understanding: In real-world scenarios, people who are too young or too old would not be able to qualify for the olympics since they would not normally possess the requisite strength or skillset necessary to compete in labor-intensive sports.
        - Examine the age range to see that this assumption holds.
        - If it doesn't, then understand where the discrepancy may lie.
- Height
    - Check that the range for values is within expectations.
- Weight
    - Check that the range for values is within expectations.
- Team
    - Check how many distinct teams exist and see if there may be some discrepancies.
- NOC
    - Check that the number of distinct values aligns with the number of distinct values for NOC in noc_regions.
- Games
    - Check that they match accordingly with the year and the season it is assigned to.
- Year
    - Check that the actual range we are working with matches how it was originally described.
- Season
    - Check how many different categories there are and if there are some discrepancies.
- City
    - Check how many different cities there are and if that aligns with how many distinct games there were.
- Sport
    - Cross reference with another list of Olympic sports from a reputable source to check for inconsistencies.
- Event
    - Check how many events exist and if they operate within expectations relative to Sex
- Medal
    - Considering each event (in theory) should award a gold, silver, and bronze medal, check to see if those are the only available values, and check to see that the numbers align with the number of distinct events from each year.
- Region from noc_regions
    - Check if all the regions align with the NOC codes correctly, also noting the existence of nulls

#### i. ID

In looking at ID, if the ID integer values follow a sequence, we want to see that the sequence is valid and falls within expectations. The expectation is that there would not exist a break in the sequence. As such, let's see the range of values and how many distinct values exist within the range:


```sql
%%sql
-- SQL Query to check ID sequentiality
SELECT MIN(ID) as "Lowest ID", MAX(ID) as "Highest ID", COUNT(DISTINCT ID) as "Number of distinct IDs"
FROM ath_events;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>Lowest ID</th>
            <th>Highest ID</th>
            <th>Number of distinct IDs</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>1</td>
            <td>135571</td>
            <td>135571</td>
        </tr>
    </tbody>
</table>



**Key Takeaways**

With the range of integer ID values between 1 and 135571, where there are 135571 unique values, we can see that there is indeed a sequentiality with the values of ID that is never broken throughout the dataset itself. This not only provides evidence for the validity of the dataset, but also it's completeness.

***ID Check: Pass***

#### ii. Name

Now that we have confirmed that there are 135571 unique id's, we will want to see that the number of unique names does not exceed the number of unique ids. If it does, it may signify invalid names that demand correction.


```sql
%%sql
-- SQL Query to check Name
SELECT COUNT(DISTINCT Name) as "Number of distinct names"
FROM ath_events;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>Number of distinct names</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>134732</td>
        </tr>
    </tbody>
</table>



We see that there are less distinct names than there are distinct IDs. Considering that there are no null values for Name, we may be handling a case where multiple different people share the same names. Let's confirm if any ID's hold different names. If the number of distinct ID-Name pairs exceed the number of distinct IDs, it's evidence of one ID holding multiple names. If the number matches the number of distinct IDs, then we can understand that each ID is paired to only one name, even if each name is not paired to only one ID.


```sql
%%sql
-- Checking to see if number of distinct ID-Name pairs exceed the number of distinct IDs. If it does, it's evidence of one ID holding multiple names
SELECT COUNT(*)
FROM (
    -- Subquery of distinct ID-Name pairs that exist in ath_events
    SELECT DISTINCT Name, ID
    FROM ath_events
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
            <td>135571</td>
        </tr>
    </tbody>
</table>



**Key Takeaways**

We see that the number of distinct pairs matches the number of distinct IDs (135571). As such, we understand that each ID is paired to only one name, which checks out with how an ID is supposed to function in terms of assignment toward another individual.

***Name Check: Pass***

#### iii. Sex

Now, let's proceed to check how many different values exist for the Sex column. An expectation would be to have two categories, male and female, with the possibility of there existing different answers that don't correspond to male or female.


```sql
%%sql
SELECT DISTINCT Sex
FROM ath_events;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>Sex</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>M</td>
        </tr>
        <tr>
            <td>F</td>
        </tr>
    </tbody>
</table>



**Key Takeaways**

We see that there are exactly two categories, M (male) and F (female). This falls in line with expectations.

***Sex Check: Pass***

#### iv. Age

We can first check the age by looking at the largest and smallest ages on record and see if they are within realistic expectations.


```sql
%%sql
SELECT MIN(Age), MAX(Age)
FROM ath_events;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>MIN(Age)</th>
            <th>MAX(Age)</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>10.0</td>
            <td>97.0</td>
        </tr>
    </tbody>
</table>



Considering these initially do not fall within expectations, we will observe the data from the youngest contestants and the oldest contestants to understand potential context with what is seen from this age range.


```sql
%%sql
-- Observing the youngest ages.
-- Display shows first 3 numbers, 
-- but insight is gained by oberving 10+ rows
SELECT *
FROM ath_events
WHERE Age <= 13
ORDER BY Age ASC
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
            <td>71691</td>
            <td>Dimitrios Loundras</td>
            <td>M</td>
            <td>10.0</td>
            <td>None</td>
            <td>None</td>
            <td>Ethnikos Gymnastikos Syllogos</td>
            <td>GRE</td>
            <td>1896 Summer</td>
            <td>1896</td>
            <td>Summer</td>
            <td>Athina</td>
            <td>Gymnastics</td>
            <td>Gymnastics Men's Parallel Bars, Teams</td>
            <td>Bronze</td>
        </tr>
        <tr>
            <td>22411</td>
            <td>Magdalena Cecilia Colledge</td>
            <td>F</td>
            <td>11.0</td>
            <td>152.0</td>
            <td>None</td>
            <td>Great Britain</td>
            <td>GBR</td>
            <td>1932 Winter</td>
            <td>1932</td>
            <td>Winter</td>
            <td>Lake Placid</td>
            <td>Figure Skating</td>
            <td>Figure Skating Women's Singles</td>
            <td>None</td>
        </tr>
        <tr>
            <td>37333</td>
            <td>Carlos Bienvenido Front Barrera</td>
            <td>M</td>
            <td>11.0</td>
            <td>None</td>
            <td>None</td>
            <td>Spain</td>
            <td>ESP</td>
            <td>1992 Summer</td>
            <td>1992</td>
            <td>Summer</td>
            <td>Barcelona</td>
            <td>Rowing</td>
            <td>Rowing Men's Coxed Eights</td>
            <td>None</td>
        </tr>
    </tbody>
</table>




```sql
%%sql
-- Observing the oldest ages.
-- Display shows first 3 numbers, 
-- but insight is gained by oberving 10+ rows
SELECT *
FROM ath_events
WHERE Age >= 65
ORDER BY Age DESC
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
            <td>128719</td>
            <td>John Quincy Adams Ward</td>
            <td>M</td>
            <td>97.0</td>
            <td>None</td>
            <td>None</td>
            <td>United States</td>
            <td>USA</td>
            <td>1928 Summer</td>
            <td>1928</td>
            <td>Summer</td>
            <td>Amsterdam</td>
            <td>Art Competitions</td>
            <td>Art Competitions Mixed Sculpturing, Statues</td>
            <td>None</td>
        </tr>
        <tr>
            <td>49663</td>
            <td>Winslow Homer</td>
            <td>M</td>
            <td>96.0</td>
            <td>None</td>
            <td>None</td>
            <td>United States</td>
            <td>USA</td>
            <td>1932 Summer</td>
            <td>1932</td>
            <td>Summer</td>
            <td>Los Angeles</td>
            <td>Art Competitions</td>
            <td>Art Competitions Mixed Painting, Unknown Event</td>
            <td>None</td>
        </tr>
        <tr>
            <td>31173</td>
            <td>Thomas Cowperthwait Eakins</td>
            <td>M</td>
            <td>88.0</td>
            <td>None</td>
            <td>None</td>
            <td>United States</td>
            <td>USA</td>
            <td>1932 Summer</td>
            <td>1932</td>
            <td>Summer</td>
            <td>Los Angeles</td>
            <td>Art Competitions</td>
            <td>Art Competitions Mixed Painting, Unknown Event</td>
            <td>None</td>
        </tr>
    </tbody>
</table>



While Age 10 is the lowest age we have seen and appears for only one instance, there are many instances where ages close to 10 such as 11, 12, and 13 appear on records for the Olympics. Age 10 would not be considered an outlier outright. As for Age 97, we found that there is only one other instance of a person competing who is older than age 90, and that would be a man age 96. From there, the rest of the ages are 88 and below with no noticeable gaps past the age of 81 upon initial inspection. After inspecting the data, it is understood that older contestants competed in art competitions held by the Olympics, and as such, it starts to make sense that the events that require more skill and mental ability and less physical demand would allow for contestants of a far more varied age to participate. Once that gets taken into consideration, Age becomes a variable that falls within expectations, and we may wish to consider the correlation between age and sports/events in further analysis. Outlier analysis will be performed in another notebook describing data cleaning procedures.

**Key Takeaways**

While it was surprising to learn the maximum and minimum ages in the data, further inspection of the data showed precedence for the matter, and as such, the validity of the data is no longer called into question for the Age column.

***Age Check: Pass***

#### v. Height

Checking for Height, we begin by seeing the minimum and maximum values.


```sql
%%sql
-- Checking Height
SELECT MIN(Height), MAX(Height)
FROM ath_events
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>MIN(Height)</th>
            <th>MAX(Height)</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>127.0</td>
            <td>226.0</td>
        </tr>
    </tbody>
</table>



**Key Takeaways**

We can see from the numbers that height has been recorded using the international metric system rather than the US system, and that the height is valued in centimeters. For those that rely on the US system, the minimum height comes to around 4 ft 2 in and the maximum comes to around 7 ft 5 in. Considering that the tallest man ever recorded was reported to be 8 ft 11 in (272 cm), we can say that the range presented to us falls within realistic expectations.

***Height Check: Pass***

#### vi. Weight

After looking into the data values and verifying with Olympedia.org, it is understood that the weight is going to be listed in kgs, not lbs. This is something to take into consideration when looking into the range of values.


```sql
%%sql
-- Checking range of weights
SELECT MIN(Weight), MAX(Weight)
FROM ath_events;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>MIN(Weight)</th>
            <th>MAX(Weight)</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>25.0</td>
            <td>214.0</td>
        </tr>
    </tbody>
</table>



**Key Takeaways**

While these minimum and maximum weights may appear to provide an extensive range for the weight variable, it is important to consider the circumstances of these values recorded. With additional context, we see that the person who had the minimum weight recorded had a BMI classified as severe thinness, and the person who had the maximum weight recorded had a BMI classified as Obese Class III. It did not seem realistic at first that these people who would be classified as in unhealthy ranges would be able to compete in the Olympics; however, after perfomring some research on "Olympedia.org", I found that these people did exist with the measurements presented in the data. More information on these contestants will be provided in the Olympics Research Appendix.
                                                                                                        
***Weight Check: Pass***
                                                                                                        
#### vii. Team

For Team, we now need to understand how many teams there are and compare that to how many NOCs there are on record. From there, we should see how many distnict pairs can be formed between team and NOC.


```sql
%%sql
SELECT COUNT(DISTINCT Team), COUNT(DISTINCT NOC)
FROM ath_events;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>COUNT(DISTINCT Team)</th>
            <th>COUNT(DISTINCT NOC)</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>1184</td>
            <td>230</td>
        </tr>
    </tbody>
</table>




```sql
%%sql
SELECT COUNT(*)
FROM (
    -- Subquery of distinct ID-Name pairs that exist in ath_events
    SELECT DISTINCT Team, NOC
    FROM ath_events
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
            <td>1231</td>
        </tr>
    </tbody>
</table>



We see that, unlike ID and Name, Team and NOC have more unique pairs than they do unique teams and unique noc's. As such, we can conlude that there are instances where a team can represent multiple NOC's, and NOC's can be represented by multiple teams. Let's examine this further to see if we ought to notice any discrepancies. First, we will start with the teams that are connected to more than one NOC.


```sql
%%sql
-- Table that lists all the teams with more than one NOC related to them
WITH team_noc (team, noc) AS (
    SELECT DISTINCT Team, NOC
    FROM ath_events
)
SELECT team, COUNT(noc)
FROM team_noc
GROUP BY team
HAVING COUNT(noc) > 1
ORDER BY COUNT(noc) DESC
LIMIT 3;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>team</th>
            <th>COUNT(noc)</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>Union des Socits Franais de Sports Athletiques</td>
            <td>4</td>
        </tr>
        <tr>
            <td>Univ. of Brussels</td>
            <td>3</td>
        </tr>
        <tr>
            <td>BLO Polo Club, Rugby</td>
            <td>3</td>
        </tr>
    </tbody>
</table>




```sql
%%sql
-- Count of rows from table above.
WITH team_noc (team, noc) AS (
    SELECT DISTINCT Team, NOC
    FROM ath_events
)
SELECT COUNT(*)
FROM (
    SELECT team, COUNT(noc)
    FROM team_noc
    GROUP BY team
    HAVING COUNT(noc) > 1
    ORDER BY COUNT(noc) DESC
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
            <td>43</td>
        </tr>
    </tbody>
</table>



We see that there are 43 teams that represent multiple NOCs, and that the highest number of NOCs any team has represented is 4. We now take a look at how many teams each NOC had represent them.


```sql
%%sql
WITH team_noc (team, noc) AS (
    SELECT DISTINCT Team, NOC
    FROM ath_events
)
SELECT noc, COUNT(team)
FROM team_noc
GROUP BY noc
HAVING COUNT(team) > 1
ORDER BY COUNT(team) DESC
LIMIT 3;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>noc</th>
            <th>COUNT(team)</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>FRA</td>
            <td>160</td>
        </tr>
        <tr>
            <td>USA</td>
            <td>97</td>
        </tr>
        <tr>
            <td>GBR</td>
            <td>96</td>
        </tr>
    </tbody>
</table>



While we are able to see that there are, in fact, several NOCs with multiple teams representing them, we are surprised to see the country France holding the maximum number of teams related to them with a total count of 160 according to the data. We may wish to dig deeper into this to see if this is the result of a discrepancy.


```sql
%%sql
SELECT Team, COUNT(Event)
FROM ath_events
WHERE NOC = "FRA"
GROUP BY Team
ORDER BY Count(Event) DESC
LIMIT 3
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>Team</th>
            <th>COUNT(Event)</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>France</td>
            <td>11988</td>
        </tr>
        <tr>
            <td>France-1</td>
            <td>135</td>
        </tr>
        <tr>
            <td>France-2</td>
            <td>121</td>
        </tr>
    </tbody>
</table>



From here, we notice two things. First, the top three teams, France, France-1, and France-2 are the only teams with over 100 records attributed to them. The rest of the temas have less than 20 attributed. We may wish to consider the circumstances of each of the teams and their relation to the NOC based in France. That being said, there are a few instances recorded where a supposed team referencing two different countries are on record for one specific NOC for France. I believe these specific team names referencing multiple countries may need to be explored further. When checking for the dual team, the format appears to always have a "/" in it, which means that you can use Regex that matches the whole string only if it contains "/".


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




```sql
%%sql
WITH dual_team AS (
SELECT Team, NOC, COUNT(Team) AS team_count
FROM ath_events
WHERE Team REGEXP '^.*\/.*$'
GROUP BY Team, NOC
ORDER BY Team, NOC
)
SELECT SUM(team_count)
FROM dual_team;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>SUM(team_count)</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>43</td>
        </tr>
    </tbody>
</table>



**Key Takeaway**

There are at least 43 instances of dual teams that need to be addressed to improve the validity of the dataset. There may be some other things we need to look into to validate the information, but that is something that requires extensive research on and is outside the scope of work for this project, so we will proceed simply by correcting the dual teams and documenting that further research to verify the validity of the dataset may be needed.

***Team Check: Fail (Partial)***

#### viii. NOC

Checking now for NOCs specifically, we will refer to the `noc_regions` dataset as a reference to check for possible discrepancies in the `NOC` column in `ath_events`.


```sql
%%sql
SELECT COUNT(*) AS "Count of Rows in noc_regions", COUNT(DISTINCT NOC) AS "Count of Distinct NOCs in noc_regions"
FROM noc_regions;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>Count of Rows in noc_regions</th>
            <th>Count of Distinct NOCs in noc_regions</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>230</td>
            <td>230</td>
        </tr>
    </tbody>
</table>



We see here that each NOC in noc_regions is unique and distinct, with a count of 230 rows in noc_regions setting the expectation that there should only be 230 different NOCs in the ath_events category, and that they should all match the NOCs of the noc_regions category. We will perform a full outer join to merge the two tables and see if we can catch any discrepancies.


```sql
%%sql
-- The Full Outer Join of both tables, highlighting specifically the unique pairs between ath_events NOCs and noc_Regions NOCs
SELECT DISTINCT ae.NOC AS "ath_events_noc", nr.NOC AS "noc_regions_noc"
FROM ath_events ae
FULL JOIN noc_regions nr
ON ae.NOC = nr.NOC
ORDER BY ae.NOC
LIMIT 3;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>ath_events_noc</th>
            <th>noc_regions_noc</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>None</td>
            <td>SIN</td>
        </tr>
        <tr>
            <td>AFG</td>
            <td>AFG</td>
        </tr>
        <tr>
            <td>AHO</td>
            <td>AHO</td>
        </tr>
    </tbody>
</table>




```sql
%%sql
-- The count of the above table for reference.
SELECT COUNT(*) AS "Count of Full Outer Join between ath_events and noc_regions"
FROM (
    SELECT DISTINCT ae.NOC AS "ath_events_noc", nr.NOC AS "noc_regions_noc"
    FROM ath_events ae
    FULL JOIN noc_regions nr
    ON ae.NOC = nr.NOC
);
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>Count of Full Outer Join between ath_events and noc_regions</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>231</td>
        </tr>
    </tbody>
</table>



We see that the number of distinct pairs between ath_events NOCs and noc_regions NOCs exceed the number of distinct NOCs in noc_regions, which highlights a discrepancy within the data. Normally, we would perform both a search for NOCs that are paired with multiple NOCs and a search for NOCs that are paired with a null value, but after seeing the top observation show a pair with a null value, we can assess that the only discrepancy that brings the pair count from what should be 230 to 231 is the matching of a null category. If there was a circumstance where multiple NOCs were paired with the same one, either there would be more distinct pairs to notice, or there wouldn't be a pairing with a null value, and neither of these appear to be the case.


```sql
%%sql
SELECT DISTINCT ae.NOC AS "ath_events_noc", nr.NOC AS "noc_regions_noc"
FROM ath_events ae
FULL JOIN noc_regions nr
ON ae.NOC = nr.NOC
WHERE ae.NOC IS NULL or nr.NOC IS NULL;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>ath_events_noc</th>
            <th>noc_regions_noc</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>SGP</td>
            <td>None</td>
        </tr>
        <tr>
            <td>None</td>
            <td>SIN</td>
        </tr>
    </tbody>
</table>



**Key Takeaway**

  The ath events NOC is written as SGP while the noc regions NOC is written as SIN. In researching the names of these specific NOCs, we find that both of them represent the country of Singapore, however for a period of time, one code was used over the other. This may explain the discrepancy within the data, and this is simply a circumstance where the code needs to be normalized. I may suggest changing SIN to SGP simply because there will be many obserevations holding SGP in ath_events data while the noc_regions data holds one record of SIN. It would be more resource-effective to change the one value in the smaller dataset rather than changing many over a large dataset. In terms of the validity of the dataset, it's accurate in terms of what the values represent according to real-world circumstances, but not exactly precisely accurate with which values are listed where.

***NOC Check: Pass (Partial)***

#### ix. Games

We notice upon initial inspection of the observations that the values noted in Games are essentially a combination of the Year and Season that the games take place in. As such, to check the validity of the column, we must check to see that each distinct games value all match accordingly with the year and the season it is assigned to. If there is a circumstance where there are multiple distinct years and/or seasons for the same value of games, then that is the discrepancy we must search for.


```sql
%%sql
WITH ath_gys AS (
    SELECT DISTINCT Games, Year, Season
    FROM ath_events
)    
SELECT COUNT(DISTINCT Games) AS "Number of Distinct Games", COUNT(*) AS "Number of Distinct Game-Year-Season Triplets"
FROM ath_gys;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>Number of Distinct Games</th>
            <th>Number of Distinct Game-Year-Season Triplets</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>51</td>
            <td>51</td>
        </tr>
    </tbody>
</table>



Initial search of counts for distinct games and distinct game-year-season triples match, which means that each value of games should have only one matching with the year and season of the games. Now, we need to verify that the value of Games is consistent with the year and season it represents. First, we will recognize that, if consistent, the Games text can be separated into two substrings where the first substring will always match the Year column (given that the Year is casted from Integer to Text) and the second substring will always match the Season column. We will test code that shows us circumstances when the first substring matches with Year and the second matches with Season. Then, we will filter to find any circumstances where the opposite is true, meaning that either the first subtring does not match with Year or the second does not match with Season. If an empty table appears when we apply the filter for the opposite, that confirms that the logic behind the Games values stays consistent.


```sql
%%sql
SELECT Games, 
       SUBSTR(Games, 1, INSTR(Games, ' ') - 1) AS first_word_of_games,
       CAST(ath_events.Year AS TEXT) AS YearTxt,
       SUBSTR(Games, 
              INSTR(Games, ' ') + 1) 
             AS last_word_of_games,
       Season
FROM ath_events
WHERE first_word_of_games == YearTxt AND last_word_of_games == Season
LIMIT 3;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>Games</th>
            <th>first_word_of_games</th>
            <th>YearTxt</th>
            <th>last_word_of_games</th>
            <th>Season</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>1992 Summer</td>
            <td>1992</td>
            <td>1992</td>
            <td>Summer</td>
            <td>Summer</td>
        </tr>
        <tr>
            <td>2012 Summer</td>
            <td>2012</td>
            <td>2012</td>
            <td>Summer</td>
            <td>Summer</td>
        </tr>
        <tr>
            <td>1920 Summer</td>
            <td>1920</td>
            <td>1920</td>
            <td>Summer</td>
            <td>Summer</td>
        </tr>
    </tbody>
</table>




```sql
%%sql
SELECT Games, 
       SUBSTR(Games, 1, INSTR(Games, ' ') - 1) AS first_word_of_games,
       CAST(ath_events.Year AS TEXT) AS YearTxt,
       SUBSTR(Games, 
              INSTR(Games, ' ') + 1) 
             AS last_word_of_games,
       Season
FROM ath_events
WHERE first_word_of_games != YearTxt OR last_word_of_games != Season;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>Games</th>
            <th>first_word_of_games</th>
            <th>YearTxt</th>
            <th>last_word_of_games</th>
            <th>Season</th>
        </tr>
    </thead>
    <tbody>
    </tbody>
</table>



**Key Takeaway**
The code above confirms what we suspected, which is that Games matches with Year and Season the way we anticipated to, with no inconsistencies. The data of the Games column is valid.

***Games Check: Pass***

#### x. Year

According to the website where we derive the dataset, the data is supposed to range over 120 years of games from 1896 to 2016. We will check the range of year and the number of distinct values to see if there are any discrepancies. Something that was noted from the source of the data was that before 1992, both the summer and winter games were held within the same year, and it wasn't until after when we notice a stagger in years between the summer and winter games.


```sql
%%sql
SELECT MIN(Year), MAX(Year)
FROM ath_events;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>MIN(Year)</th>
            <th>MAX(Year)</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>1896</td>
            <td>2016</td>
        </tr>
    </tbody>
</table>




```sql
%%sql
-- May wish to observe all years for context
SELECT DISTINCT Year
FROM ath_events
ORDER BY Year
LIMIT 3;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>Year</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>1896</td>
        </tr>
        <tr>
            <td>1900</td>
        </tr>
        <tr>
            <td>1904</td>
        </tr>
    </tbody>
</table>



**Key Takeaways**

The range is as expected, where the minimum value is 1896 and the maximum value is 2016. We also see that 1992 and onwards, the games occur within intervals of two while before, they occurred in intervals of four for the most part. We do notice some gaps in between certain years before 1992, however. The first is in between 1936 and 1948, and the other is a missing year of 1916. Additionally, we note the existence of games in 1906 when there should only be games in the four year sequence of years 1900, 1904, 1908, 1912, and so on. Research shows that the discrepancies we see here are in fact valid pieces of history that should stay on record. See Olympics Research Appendix for more information

***Year Check: Pass***

#### xi. Season

Now, let's proceed to check how many different values exist for the Season column. An expectation would be to have two categories, summer and winter, with the possibility of there existing different answers that don't correspond to summer or winter.


```sql
%%sql
SELECT DISTINCT Season
FROM ath_events
GROUP BY Season;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>Season</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>Summer</td>
        </tr>
        <tr>
            <td>Winter</td>
        </tr>
    </tbody>
</table>



**Key Takeaways**

We see that there are exactly two categories, Summer and Winter. This falls in line with expectations.

***Season Check: Pass***

#### xii. City

Considering that there are only 51 Games, the number of cities where these games are held should not exceed this number. First, check the list of cities that hosted the games to verify that nothing was misspelled, then check how many cities there are and compare with the number of games.


```sql
%%sql
SELECT DISTINCT City
FROM ath_events
ORDER BY City
LIMIT 3;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>City</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>Albertville</td>
        </tr>
        <tr>
            <td>Amsterdam</td>
        </tr>
        <tr>
            <td>Antwerpen</td>
        </tr>
    </tbody>
</table>




```sql
%%sql
SELECT COUNT(DISTINCT City)
FROM ath_events;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>COUNT(DISTINCT City)</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>42</td>
        </tr>
    </tbody>
</table>



There are no misspellings of the cities that duplicate the number of distinct cities in the data and the number of distinct cities does not exceed the number of games. There are differences in the way the cities are spelled overall compared to spellings in English, but I believe that is more so due to translation and how the spelling reflects how the city would be shown in their native dialects instead. We do see, however, that the number of cities is lower than the number of games. There may be cities where the games were held multiple times. We may also need to consider the possibility that the games may have been hosted in different cities


```sql
%%sql
-- Obtaining subset of data that only includes distinct pairs between each city and the games they hosted
WITH city_games AS (
    SELECT DISTINCT City, Games
    FROM ath_events
)
-- The main query will count how many times a city has hosted the Olympics, only considering those that did so more than once
SELECT City, COUNT(Games) AS "Number of Games"
FROM city_games
GROUP BY City
HAVING COUNT(Games) > 1
ORDER BY COUNT(Games) DESC, City
LIMIT 3;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>City</th>
            <th>Number of Games</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>Athina</td>
            <td>3</td>
        </tr>
        <tr>
            <td>London</td>
            <td>3</td>
        </tr>
        <tr>
            <td>Innsbruck</td>
            <td>2</td>
        </tr>
    </tbody>
</table>




```sql
%%sql
-- This CTE filters out the distinct pairs between City and Games
WITH city_games AS (
    SELECT DISTINCT City, Games
    FROM ath_events
),
-- This CTE takes the above CTE and filters further by 
-- recognizing only the games that are hosted by more than one city
multi_city_games AS (
SELECT Games
FROM city_games
GROUP BY Games
HAVING COUNT(City) > 1
)
-- This effectively joins the list of games that host more than one city 
-- with the list of distinct cities and games to see which cities hosted the same games
SELECT mcg.Games, cg.City
FROM multi_city_games mcg
INNER JOIN city_games cg
ON mcg.Games = cg.Games;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>Games</th>
            <th>City</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>1956 Summer</td>
            <td>Melbourne</td>
        </tr>
        <tr>
            <td>1956 Summer</td>
            <td>Stockholm</td>
        </tr>
    </tbody>
</table>



**Key Takeaways**

While there does exist multiple instances of the same city running multiple games and one instance of one game being held by multiple cities, research has shown that these instances in the data do reflect itself in Olympics history. See Olympics Research Appendix for more information. The City column is valid.

***City Check: Pass***

#### xiii. Sport

Something that gets noted is that when you look at the list of all the sports in the olympics from olympics.com, you find that there are a significant amount of mismatches between that list and the query for distinct sports in our dataset. For reference, I compiled the list from this webpage into another .csv file to incorporate this into our existing dataset and directly compare from there. The file was named "sports_odc.csv" where odc stands for olympics-dot-com.


```python
# Read the sports_odc file
sports_odc = pd.read_csv("sports_odc.csv")
```


```python
# Testing the success of read
sports_odc.head(3)
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Sport</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>Acrobatic Gymnastics</td>
    </tr>
    <tr>
      <th>1</th>
      <td>Alpine Skiing</td>
    </tr>
    <tr>
      <th>2</th>
      <td>Archery</td>
    </tr>
  </tbody>
</table>
</div>




```python
# Connect the new dataframe to the sql database
sports_odc.to_sql('sports_odc', conn, if_exists='replace', index=False)
```




    74




```sql
%%sql
-- CTE for a table with distinct list of sports
WITH sports_ae AS (
    SELECT DISTINCT Sport
    FROM ath_events
    ORDER BY Sport
)
-- Full Outer Join compares sports from ath_events to Olympics.com
SELECT ae.Sport AS "Sport from Dataset", odc.Sport AS "Sport from Olympics.com"
FROM sports_ae ae
FULL JOIN sports_odc odc
ON ae.Sport = odc.Sport
ORDER BY ae.Sport
LIMIT 3;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>Sport from Dataset</th>
            <th>Sport from Olympics.com</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>None</td>
            <td>Acrobatic Gymnastics</td>
        </tr>
        <tr>
            <td>None</td>
            <td>Artistic Gymnastics</td>
        </tr>
        <tr>
            <td>None</td>
            <td>Artistic Swimming</td>
        </tr>
    </tbody>
</table>



From the full join, we notice quite a few things:

First, there are instances where there may be different naming schemes to describe the same sport, such as the case with "Syncronized Swimming" and "Artistic Swimming".
Second, there are multiple categories of what should be the same sport in the Olympics.com list compared to the original dataset, such as "Baseball 5", "Basketball 3x3", "Cycling BMX Freestyle", "Cycling BMX Racing", etc.
Third, there are certain instances where certain sports in the original dataset simply do not exist in the list from Olympics.com, such as "Art Cmopetitions" and "Tug-Of-War". I suspect that the true reason why this is the case is due to these sports existing in the olympics within earlier years, but no longer exist in recent times. There may be a way to check this:


```sql
%%sql
SELECT ae.Sport, MIN(ae.Year), MAX(ae.Year)
FROM ath_events ae
LEFT JOIN sports_odc odc
ON ae.Sport = odc.Sport
WHERE ae.Sport NOT IN sports_odc
GROUP BY ae.Sport
ORDER BY ae.Sport
LIMIT 3;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>Sport</th>
            <th>MIN(ae.Year)</th>
            <th>MAX(ae.Year)</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>Aeronautics</td>
            <td>1936</td>
            <td>1936</td>
        </tr>
        <tr>
            <td>Alpinism</td>
            <td>1924</td>
            <td>1936</td>
        </tr>
        <tr>
            <td>Art Competitions</td>
            <td>1912</td>
            <td>1948</td>
        </tr>
    </tbody>
</table>



From what I could tell looking at the query of sports not listed in the Olympics.com list, any games where the most recent year was before 1950 can be considered an outdated game that potentially justifies why it is not being taken into consideration in Olympics.com. That being said, there are a multitude of sports listed in the dataset where they have been held within recent years yet not listed in Olympics.com. The reasoning behind this can be attributed to differences in naming schematics when it comes to categorizing sports in both the dataset and the list on Olympics.com. Considering the time constraints on the project itself, verifying whether this is valid or not is something that should be considered for future work.

**Key Takeaways**

The Sports column will show data that is appropriate for analysis. It will not contain information that is inaccurate nor invalid. That being said, cross referencing the list of distinct sports and normalizing to accomodate with the list from Olympics.com is something that can be done for future work, and is not a priority that will heavily contribute to our analysis as the project continues.

***Sport Check: Pass(partial)***

#### xiv. Events

The expectation before viewing this information was that all sports would have at least two events, one for men and one for women. That being said, that isn't to say that men and women will always stay separated. There may exist co-ed events that brings a sport to host an odd number of events rather than an even one.


```sql
%%sql
SELECT COUNT(DISTINCT Event)
FROM ath_events;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>COUNT(DISTINCT Event)</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>765</td>
        </tr>
    </tbody>
</table>




```sql
%%sql
SELECT Sport, COUNT(DISTINCT EVENT)
FROM ath_events
GROUP BY Sport
ORDER BY Sport
LIMIT 3;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>Sport</th>
            <th>COUNT(DISTINCT EVENT)</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>Aeronautics</td>
            <td>1</td>
        </tr>
        <tr>
            <td>Alpine Skiing</td>
            <td>10</td>
        </tr>
        <tr>
            <td>Alpinism</td>
            <td>1</td>
        </tr>
    </tbody>
</table>




```sql
%%sql
-- CTE Filters for instances where a sport holds only one distinct event
WITH only_event AS (
    SELECT Sport
    FROM ath_events
    GROUP BY Sport
    HAVING COUNT(DISTINCT EVENT) = 1
    ORDER BY Sport
)
-- Queries will show which sexes have appeared in the events filtered through the CTE
SELECT DISTINCT Sex, Sport
FROM ath_events
WHERE Sport IN only_event
ORDER BY Sport
LIMIT 3;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>Sex</th>
            <th>Sport</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>M</td>
            <td>Aeronautics</td>
        </tr>
        <tr>
            <td>M</td>
            <td>Alpinism</td>
        </tr>
        <tr>
            <td>F</td>
            <td>Alpinism</td>
        </tr>
    </tbody>
</table>



Looking at the data and how a significant amount of sports have only one event attributed to them brings an important fact into consideration, especially for sports that were held only during the earlier years of the Olympics. Historically speaking, women were not allowed to compete in certain sports in the Olympics. It wasn't until later years down the line where women were allowed to compete in their own events, with few, if any instances of both men and women competing in a co-ed events. The output above shows that only males competed in those sports, with an exception for Alpinism that operated as a co-ed event only and Softball played only by women.

**Key Takeaway**

The data can be interpreted to be valid, even when strayed away from initial expectations.

***Event Check: Pass***

#### xv. Medal

Considering each event should award a gold, silver, and bronze medal, we wll check to see if those are the only available values and check to see that the numbers align with the number of distinct events from each year. First, we will confirm the number of events that will inevitably provide a medal, calculated by taking the number of distinct pairs of events and games provided in the dataset and comparing them to the number of medals that have been given. If the number is so drastic to the point where it questions the validity of the dataset, it will be noted.


```sql
%%sql
SELECT COUNT(*)
FROM (
    SELECT DISTINCT Games, Event
    FROM ath_events
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
            <td>6192</td>
        </tr>
    </tbody>
</table>




```sql
%%sql
SELECT Medal, COUNT(Medal)
FROM ath_events
GROUP BY Medal
ORDER BY Medal;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>Medal</th>
            <th>COUNT(Medal)</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>None</td>
            <td>0</td>
        </tr>
        <tr>
            <td>Bronze</td>
            <td>13295</td>
        </tr>
        <tr>
            <td>Gold</td>
            <td>13372</td>
        </tr>
        <tr>
            <td>Silver</td>
            <td>13116</td>
        </tr>
    </tbody>
</table>



Notes:

- Having the number of events be 6192 while the number of gold medals is 13372, a little over double the number of events, is not 100% surprising considering team events are a common occurrence within the Olympics, and as such, the data takes into consideration the individual players that won the gold medals, even if they won it by participating as part of a team.
- Having Gold be higher than both Silver and Bronze is not unsurprising considering there should always be a circumstance of first place within the competition
- Having more Bronze medals exist over Silver medals is a noteworthy observation, however. The expectation of a competition should be that there are people who win silver medals before others win bronze medals, and yet the data seems to suggest otherwise. There are some explanations for the matter:
    - The data is incomplete
    - There are instances where two parties tie for bronze simply by the nature of a tournament bracket.
    - There are some competitions that exist out there that function with a point-ranking system to award different colored medals so long as they are within a range of points rather than a relative-ranking system where awards are given to those who performed the best, the next best award goes to the next best performer, and so on and so forth.

**Key Takeaways**

We cannot confirm that the data is completely valid as there is evidence to suggest otherwise. Doing a full research project on those who lack a record for medals won in competitions would prove to be an arduous task that is outside of the scope of work for this project. As such, it will be noted as potential work to perform in the future, but for the meantime, for the purposes of analysis, we should treat the data as valid.

***Medal Check: Inconclusive***

#### xvi. Region on noc_regions
                                                                                                                                                       
We were able to observe the validity of the NOC from noc_regions when also observing the validity of the NOC from ath_events. We also understand that the notes column effectively serves us as optional additional context considering the immense presence of null values in the column. As such, we will check for the validity of Region to conclude our validity checks.


```sql
%%sql
SELECT COUNT(DISTINCT Region)
FROM noc_regions;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>COUNT(DISTINCT Region)</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>206</td>
        </tr>
    </tbody>
</table>



We understand from here that there are some regions that take up multiple NOC associations, so we will take a look specifically at the regions that do so with their notes right by them so we have a better understanding of the context.


```sql
%%sql
WITH multi_region AS (
    SELECT Region
    FROM noc_regions
    GROUP BY Region
    HAVING COUNT(NOC) > 1
)
SELECT Region, NOC, notes
FROM noc_regions
WHERE Region IN multi_region
ORDER BY Region
LIMIT 3;
```


<span style="None">Running query in &#x27;sqlite:///olympics.db&#x27;</span>





<table>
    <thead>
        <tr>
            <th>region</th>
            <th>NOC</th>
            <th>notes</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>Australia</td>
            <td>ANZ</td>
            <td>Australasia</td>
        </tr>
        <tr>
            <td>Australia</td>
            <td>AUS</td>
            <td>None</td>
        </tr>
        <tr>
            <td>Canada</td>
            <td>CAN</td>
            <td>None</td>
        </tr>
    </tbody>
</table>



There are several circumstances where the code is in reference to a specific part of the region as opposed to the region as a a whole such as CHN representing all of China with HKG representing Hong Kong of China. There exist a few circumstances of a code referring to a larger region that encompasses the main region in question, but is listed as the region because of how prominent it is compared to the whole, such as AUS representing Australia specifically with ANZ representing Australasia (Australia + New Zealand). Lastly, there are circumstances where the code used to represent the region simply changed over time, most likely due to changes within the region, such as URS representing not necessarily Russia but the Soviet Union (USSR), and RUS representing Russia as it is right now. This showcases enough precedent to hypothesize that the data is valid, with further exploration being something to consider for future work, but before we leave, earlier exploration showed null values for specific regions, so let's take a closer look:


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



Looking at the only three NOCs that do not have a region, there are some things to note. First, ROT standing for Refugee Olympic Team makes sense as to why there is no specific region attributed to them. This effectively functions as a team that does not represent any particular region. UNK refering to Unknown feels like a placeholder for what is effectively a Null Value. We will want to see how many instances of UNK appearing within the original dataset, and if it is a low enough dataset, we can make it work for us. Last but not least, TUV is the NOC code that represents Tuvalu, from what I understand when looking at the adjacent notes. It doesn't seem consistent to have the region be stated in the notes but not in the region column. This will require ammendment in the cleaning stage.


```sql
%%sql
SELECT COUNT(*)
FROM ath_events
WHERE NOC = "UNK";
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
            <td>2</td>
        </tr>
    </tbody>
</table>



**Key Takeaways**

Most of the information in the noc_regions table is valid, and the information that may appear to not be valid can be corrected rather easily. Additionally, the further exploration of NOC values shows that there are very little unidentified NOCs, and as such, this warrants the ability to change what is effectively missing data to further complete the dataset. Last but not least, it was discovered by further observing the noc_regions dataset that the same region can be referred to by different names for different times, which is something we need to take into consideration when applying corrections to Singapore (SGP/SIN).

***Region Check: Pass (Partial)***

#### Summary of Validity Checks

List of Columns that pass with no issues:
- ID
- Name
- Sex
- Age
- Height
- Weight
- Games
- Year
- Season
- City
- Event

List of Columns that showed issues with validity:
- Team (Much evidence of team names warranting correction or further inspection, and will only be able to correct a few records)
- NOC (Almost all data is valid, but there is the discrepancy with SGP/SIN to address)
- Sport (Most of the data appears to be valid, but cross-referencing with other sources is problematic and an ineffective method of checking)
- Medal (With most of the data consisting of null values, it's hard to tell which null is the result of missing information and which null represent no medals being earned. Changing to "No Medal" values will show no medals being earned, and if assumptions are correct, almost all of these changes will correct understandings, but we may miss on a few circumstances of data actually missing.)
- Region (Almost all data is valid, but corrections need to be made for values related to ROT, TUV, and UNK)

### Dimension 4: Consistency

When we look at consistency, what we are really looking at is that there is a standardization of formats and naming conventions across the datasets. Most of the columns have this quality, with a fixed data type ensuring that formats stay consistent, and no misspellings nor renamings of the same value that would cause confusion and demand standardization. That being said, there are a few things to note about consistency:

- NOC has shown to be inconsistent with the various values for a particular region AND the misalignment with Singapore's NOC value. This is something that will need to be addressed when resolving the issues found here.
- Team has also shown to be inconsistent with the names of the teams, especially with the case of the dual-region team names when they actually represent a specific region instead of two or more.
- Any inconsistencies with the Year sequence has been addressed from research on the subject. The same applies for City. See Olympics Research Appendix.
- In terms of consistency within the database, Sport has proven to stay consistent. That being said, when considering other data sources, many inconsistencies arise. It should be noted that while these exist, they do not have too great of an impact on the analysis of the dataset at hand.
- Events column has too many distinct values to check whether the data is consistent or not. Will simply work with the assumption that the Events column is consistent, and leave the verification of this hypothesis for future work.
- Medal is consistent in terms of distinct values and formatting, but not in terms of how many medals are distributed within any given event.

### Dimension 5: Timeliness

Timeliness simply refers to whether the data is up-to-date and relevant for its intended use. The data spans 120 years, but this span ends at 2016. It is late 2025 at the time of writing this, and as such, there are 2018 and 2022 Winter Games as well as 2020 and 2024 Summer Games we could incorporate into this dataset. That being said, the dataset was given to me for analysis purposes, and as such, I do not have the code nor the coding ability to rescrape the original website that hosted the data from the dataset to provide the updates necessary for analysis on more recent data. That being said, I believe 120 years worth of historical data will serve effectively for providing insights that would prove valuable to trainers.

### Dimension 6: Uniqueness

Uniqueness refers to making sure that duplicates are avoided within the dataset that could skew analysis. We will analyze for any duplicates by observing the distinct rows of the dataset. Through earlier analysis, I was able to ascertain that noc_regions has no issues with uniqueness. Let's consider ath_events.


```sql
%%sql
SELECT COUNT(*)
FROM ath_events;
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
            <td>271116</td>
        </tr>
    </tbody>
</table>




```sql
%%sql
-- To select the distinct rows, we will take note of the columns that can serve almost as a primary key for identifying each row when grouped. 
-- If there is a mismatch in the count of these distinct rows with the number of rows in the dataset specifically,
-- That will serve as evidence for duplicates in the dataset.
SELECT COUNT(*)
FROM (
    SELECT DISTINCT ID, Team, Games, Event
    FROM ath_events
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
            <td>269661</td>
        </tr>
    </tbody>
</table>



We notice less distinct rows than actual rows, specifically 1455 cases of rows that could be duplicates. We will need to look into correcting this in the issue resolution section later.

### Dimension 7: Reliability

Reliability refers to the ability to access the data's source and the methods used to collect and store it. The source was verified earlier to be a reputable source of Olympics information, "www.sportsreference.com", and the code to obtain this information can be found on GitHub where the creator, Rgriffin [scrapes](https://github.com/rgriff23/Olympic_history/blob/master/R/olympics%20scrape.R) and [wrangles](https://github.com/rgriff23/Olympic_history/blob/master/R/olympics%20wrangle.R) the data.

---

## V. Summary of Issues Detected

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

We will be sure to correct as many of the issues we have detected here in the notebook "SportsStats Data Cleaning Procedure"


```python

```
