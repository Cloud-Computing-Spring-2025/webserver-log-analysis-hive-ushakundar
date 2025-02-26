# Web-Server-Log-Analysis
## Project Overview
This project implements a Hive-based analytical process for analyzing web server logs. The objective is to process a dataset of web server logs stored in CSV format and extract meaningful insights about website traffic patterns. The analysis involves counting web requests, analyzing HTTP status codes, identifying the most visited pages, detecting suspicious activity, and analyzing traffic trends. Partitioning is used to optimize query performance.

## Implementation Approach

The analysis was conducted using Apache Hive with the following steps:

i) Setup Hive Environment: Created a Hive database and an external table for web server logs.

ii) Data Loading: Loaded the CSV file into HDFS and then into the Hive table.

iii) Query Execution:

a) Count total web requests.

b) Analyze the frequency of HTTP status codes.

c) Extract the top 3 most visited URLs.

d) Identify the most common user agents (browsers).

e) Detect IP addresses with more than 3 failed requests (status 404 or 500).

f) Compute traffic trends by calculating requests per minute.

g) Partitioning Implementation: Created a partitioned table to optimize query performance.

## Execution Steps

### 1. Start the hadoop cluster
Run the following command to start the Hadoop cluster:
```sh
docker compose up -d
```

### 2.Move Dataset to Resourcemanager

Copy the dataset to the Hadoop ResourceManager container:

```bash
docker cp web_server_logs.csv resourcemanager:/
```

### 3. Move the File to HDFS

```bash
docker exec -it resourcemanager bash
```

```bash
hdfs dfs -mkdir -p /user/hive/warehouse/web_logs/
hdfs dfs -put /web_server_logs.csv /user/hive/warehouse/web_logs/
hdfs dfs -ls /user/hive/warehouse/web_logs/
```

### 4.Setup Hive Environment

Create a Hive database:
```sql
CREATE DATABASE web_logs;
USE web_logs;
```

### 5.Define an external table for web server logs

```sql
CREATE EXTERNAL TABLE web_logs (
    ip STRING,
    tstamp STRING,
    url STRING,
    status INT,
    user_agent STRING
)
ROW FORMAT DELIMITED 
FIELDS TERMINATED BY ',' 
STORED AS TEXTFILE 
LOCATION '/user/hive/warehouse/web_logs/';
```

### 6.Load Data

```sql
LOAD DATA INPATH '/user/hive/warehouse/web_logs/web_server_logs.csv' INTO TABLE web_logs;
```

### 7.Execute Queries
#### a. Count Total Web Requests

```sql
SELECT COUNT(*) AS total_requests FROM web_logs;
```

#### b. Analyze Status Codes
```sql
SELECT status, COUNT(*) AS count FROM web_logs GROUP BY status;
```

#### c.Identify Most Visited Pages
```sql
SELECT url, COUNT(*) AS visits 
FROM web_logs 
GROUP BY url 
ORDER BY visits DESC 
LIMIT 3;
```

#### d.Traffic Source Analysis
```sql
SELECT user_agent, COUNT(*) AS count 
FROM web_logs 
GROUP BY user_agent 
ORDER BY count DESC;
```

#### e.Detect Suspicious Activity (IPs with >3 failed requests)
```sql
SELECT ip, COUNT(*) AS failed_requests
FROM web_logs
WHERE status IN (404, 500)
GROUP BY ip
HAVING COUNT(*) > 3;
```
#### f.Analyze Traffic Trends (Requests Per Minute)
```sql
SELECT substr(timestamp, 0, 16) AS minute, COUNT(*) AS requests 
FROM web_logs 
GROUP BY substr(timestamp, 0, 16) 
ORDER BY minute;
```
### 8.Implement Partitioning
#### a.Create a partitioned table:
```sql
CREATE TABLE web_logs_partitioned (
    ip STRING,
    timestamp STRING,
    url STRING,
    user_agent STRING
)
PARTITIONED BY (status INT)
ROW FORMAT DELIMITED 
FIELDS TERMINATED BY ',' 
STORED AS TEXTFILE;
```

#### b.Load data into partitions:
```sql
INSERT INTO web_logs_partitioned PARTITION(status)
SELECT ip, timestamp, url, user_agent, status FROM web_logs;
```

### 9. Generate Output
```sql
INSERT OVERWRITE DIRECTORY '/user/hive/output/total_requests' 
SELECT COUNT(*) FROM web_logs;
```

## Challenges Faced

1. Dynamic Partitioning Issue: Hive's strict mode required at least one static partition column. This was resolved by setting hive.exec.dynamic.partition.mode = nonstrict.

2. Table Not Found Error: Hive was not recognizing the web_logs table. The issue was resolved by explicitly using the database (USE web_logs;) and verifying the table’s existence with SHOW TABLES;

## Sample Input and Output

### Sample Input (web_server_logs.csv)

```bash
ip,timestamp,url,status,user_agent
192.168.1.1,2024-02-01 10:15:00,/home,200,Mozilla/5.0
192.168.1.2,2024-02-01 10:16:00,/products,200,Chrome/90.0
192.168.1.3,2024-02-01 10:17:00,/checkout,404,Safari/13.1
192.168.1.10,2024-02-01 10:18:00,/home,500,Mozilla/5.0
192.168.1.15,2024-02-01 10:19:00,/products,404,Chrome/90.0
```

### Expected Output
#### 1. Total Web Requests:
```bash
Total Requests: 100
```
#### 2.Status Code Analysis:
```bash
200: 80
404: 10
500: 10
```

#### 3.Most Visited Pages:
```bash
/home: 50
/products: 30
/checkout: 20
```

#### 4.Traffic Source Analysis:
```bash
Mozilla/5.0: 60
Chrome/90.0: 30
Safari/13.1: 10
```

#### 5.Suspicious IP Addresses:
```bash
192.168.1.10: 5 failed requests
192.168.1.15: 4 failed requests
```

#### 6.Traffic Trend Over Time:
```bash
2024-02-01 10:15: 5 requests
2024-02-01 10:16: 7 requests
```




