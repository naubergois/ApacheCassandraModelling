![](cassandra.gif)



Apache Cassandra is an open-source distributed database management system. An Apache Software Foundation top-level design was created to handle massive amounts of data spread across many commodity servers while providing a highly available service with no single point of failure. It is a NoSQL solution developed by Facebook and powered their Inbox Search feature until late 2010. Jeff Hammerbacher, who led the Facebook Data team, has described Cassandra as a BigTable data model running on an Amazon Dynamo-like infrastructure. Cassandra provides a structured key-value store with tunable consistency. Keys map to multiple values, which are grouped into column families. The column families are fixed when a Cassandra database is created, but columns can be added to a family at any time. Furthermore, columns are added only to specified keys, so different keys can have different numbers of columns in any given family. The values from a column family for each key are stored together. This makes Cassandra a hybrid data management system between a column-oriented DBMS and a row-oriented store. 



This is the second test of Data Engineer Udacity program with one file  Project_1B_ Project_Template.ipynb





# Part I. ETL Pipeline for Pre-Processing the Files

## PLEASE RUN THE FOLLOWING CODE FOR PRE-PROCESSING THE FILES

#### Import Python packages

[1]:

```
# Import Python packages 
import pandas as pd
import cassandra
import re
import os
import glob
import numpy as np
import json
import csv
```

#### Creating list of filepaths to process original event csv data files

[2]:

```
# checking your current working directory
print(os.getcwd())

# Get your current folder and subfolder event data
filepath = os.getcwd() + '/event_data'

# Create a for loop to create a list of files and collect each filepath
for root, dirs, files in os.walk(filepath):
    
# join the file path and roots with the subdirectories using glob
    file_path_list = glob.glob(os.path.join(root,'*'))
    #print(file_path_list
```

#### Processing the files to create the data file csv that will be used for Apache Casssandra tables

# 
full_data_rows_list = [] 

for f in file_path_list:

    with open(f, 'r', encoding = 'utf8', newline='') as csvfile: 
        # creating a csv reader object 
        csvreader = csv.reader(csvfile) 
        next(csvreader)

        for line in csvreader:
            #print(line)
            full_data_rows_list.append(line) 

print(len(full_data_rows_list))

print(full_data_rows_list)

csv.register_dialect('myDialect', quoting=csv.QUOTE_ALL, skipinitialspace=True)

with open('event_datafile_new.csv', 'w', encoding = 'utf8', newline='') as f:
    writer = csv.writer(f, dialect='myDialect')
    writer.writerow(['artist','firstName','gender','itemInSession','lastName','length',\
                'level','location','sessionId','song','userId'])
    for row in full_data_rows_list:
        if (row[0] == ''):
            continue
        writer.writerow((row[0], row[2], row[3], row[4], row[5], row[6], row[7], row[8], row[12], row[13], row[16]))

# Part II. Complete the Apache Cassandra coding portion of your project. 

## Now you are ready to work with the CSV file titled <font color=red>event_datafile_new.csv</font>, located within the Workspace directory.  The event_datafile_new.csv contains the following columns: 
- artist 
- firstName of user
- gender of user
- item number in session
- last name of user
- length of the song
- level (paid or free song)
- location of the user
- sessionId
- song title
- userId

The image below is a screenshot of what the denormalized data should appear like in the <font color=red>**event_datafile_new.csv**</font> after the code above is run:<br>

![](download.jpg)

## egin writing your Apache Cassandra code in the cells below

#### Creating a Cluster

[5]:

```
# This should make a connection to a Cassandra instance your local machine 
# (127.0.0.1)

from cassandra.cluster import Cluster
cluster = Cluster()

# To establish connection and begin executing queries, need a session
session = cluster.connect()
```

#### Create Keyspace

[6]:

```
session.execute("""
    CREATE KEYSPACE IF NOT EXISTS sparkify
    WITH REPLICATION = { 'class' : 'SimpleStrategy', 'replication_factor' : 1 }
    """)
```

[6]:

```
<cassandra.cluster.ResultSet at 0x7f68c583c208>
```

#### Set Keyspace

[7]:

```
session.set_keyspace('sparkify')
```

### Now we need to create tables to run the following queries. Remember, with Apache Cassandra you model the database tables on the queries you want to run.

## Create queries to ask the following three questions of the data

### 1. Give me the artist, song title and song's length in the music app history that was heard during sessionId = 338, and itemInSession = 4

### 2. Give me only the following: name of artist, song (sorted by itemInSession) and user (first and last name) for userid = 10, sessionid = 182

### 3. Give me every user name (first and last) in my music app history who listened to the song 'All Hands Against His Own'

[8]:

```
## TO-DO: Query 1:  Give me the artist, song title and song's length in the music app history that was heard during \
## sessionId = 338, and itemInSession = 4


session.execute("""
    CREATE TABLE IF NOT EXISTS session_songs
    (sessionId int, itemInSession int, artist text, song_title text, song_length float,
    PRIMARY KEY(sessionId, itemInSession))
    """)
                    
```

[8]:

```
<cassandra.cluster.ResultSet at 0x7f68c58ba6a0>
```

[10]:

```
# We have provided part of the code to set up the CSV file. Please complete the Apache Cassandra code below#
file = 'event_datafile_new.csv'

with open(file, encoding = 'utf8') as f:
    csvreader = csv.reader(f)
    next(csvreader) # skip header
    for line in csvreader:
## TO-DO: Assign the INSERT statements into the `query` variable
        query = "INSERT INTO session_songs (sessionId, itemInSession, artist, song_title, song_length)"
        query = query + " VALUES (%s, %s, %s, %s, %s)"
        artist_name, user_name, gender, itemInSession, user_last_name, length, level, location, sessionId, song, userId = line
        session.execute(query, (int(sessionId), int(itemInSession), artist_name, song, float(length)))
```

#### Do a SELECT to verify that the data have been inserted into each table

[11]:

```
## TO-DO: Add in the SELECT statement to verify the data was entered into the table
rows = session.execute("""SELECT artist, song_title, song_length FROM session_songs WHERE sessionId = 338 AND itemInSession = 4""")

for row in rows:
    print(row.artist, row.song_title, row.song_length)
```

### COPY AND REPEAT THE ABOVE THREE CELLS FOR EACH OF THE THREE QUESTIONS

[14]:

```
## TO-DO: Query 2: Give me only the following: name of artist, song (sorted by itemInSession) and user (first and last name)\
## for userid = 10, sessionid = 182

session.execute("""
    CREATE TABLE IF NOT EXISTS user_songs
    (userId int, sessionId int, artist text, song text, firstName text, lastName text, itemInSession int,
    PRIMARY KEY((userId, sessionId), itemInSession))
    """)
                
file = 'event_datafile_new.csv'

with open(file, encoding = 'utf8') as f:
    csvreader = csv.reader(f)
    next(csvreader) # skip header
    for line in csvreader:
        query = "INSERT INTO user_songs (userId, sessionId, artist, song, firstName, lastName, itemInSession)"
        query = query + " VALUES (%s, %s, %s, %s, %s, %s, %s)"
        artist, firstName, gender, itemInSession, lastName, length, level, location, sessionId, song, userId = line
        session.execute(query, (int(userId), int(sessionId), artist, song, firstName, lastName, int(itemInSession)))
        
rows = session.execute("""SELECT itemInSession, artist, song, firstName, lastName FROM user_songs WHERE userId = 10 AND sessionId = 182""")

for row in rows:
    print(row.iteminsession, row.artist, row.song, row.firstname, row.lastname)
0 Down To The Bone Keep On Keepin' On Sylvie Cruz
1 Three Drives Greece 2000 Sylvie Cruz
2 Sebastien Tellier Kilometer Sylvie Cruz
3 Lonnie Gordon Catch You Baby (Steve Pitron & Max Sanna Radio Edit) Sylvie Cruz
```

[ ]:

```
## TO-DO: Query 3: Give me every user name (first and last) in my music app history who listened to the song 'All Hands Against His Own'

session.execute("""
    CREATE TABLE IF NOT EXISTS app_history
    (song text, firstName text, lastName text, userId int,
    PRIMARY KEY(song, userId))
    """)

file = 'event_datafile_new.csv'

with open(file, encoding = 'utf8') as f:
    csvreader = csv.reader(f)
    next(csvreader) # skip header
    for line in csvreader:
        query = "INSERT INTO app_history (song, firstName, lastName, userId)"
        query = query + " VALUES (%s, %s, %s, %s)"
        artist, firstName, gender, itemInSession, lastName, length, level, location, sessionId, song, userId = line
        session.execute(query, (song, firstName, lastName, int(userId)))
        
rows = session.execute("""SELECT firstName, lastName FROM app_history WHERE song = 'All Hands Against His Own'""")

for row in rows:
    print(row.firstname, row.lastname)

                    
```

[

### Drop the tables before closing out the sessions

[4]:

```

session.execute("""DROP TABLE app_history""")

session.execute("""DROP TABLE user_songs""")

session.execute("""DROP TABLE session_songs""")
```



### Close the session and cluster connection¶

[ ]:

```
session.shutdown()
cluster.shutdown()
```