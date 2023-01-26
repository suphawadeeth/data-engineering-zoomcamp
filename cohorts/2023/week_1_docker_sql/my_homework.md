## Week 1 Homework | Preparing the environment | Practice with Docker and SQL

# What I've learned on Week 1.1 | Data Engineering Zoomcamp

Note: 
1) I refer my learning process to the homework since it best represents the entire lessons in week 1
2) I'm working on macOS

# 1. Learn Docker tags

### What is Docker? 
(Learn in detail here https://www.youtube.com/watch?v=pTFZFxd4hOI)
- Docker is a **platform** where you can **build** your application on it, then **run** it and also you can **ship** your application/code/program/data pipeline to other machines and it will consistently run it on the other machines that run Docker. It will function the same as it is on your machine. 
- Plus point is that it won't interfere with any other system in the machine.
- Your application will work on other machines even if it runs on a different software version, different configuration settings. Meaning if it works on your development machine, it will work on your test machine as well as your production machine
- Magic line to run application on the machine is
```docker-compose up -d``` (need to setup your environment in **docker-compose.yml** file to run it ---> example below)

- Now when you don't want that one application anymore. You tell Docker to remove it ```docker-compose down --rmi all``` And it will remove the application without interfering other environment
(reference: https://www.youtube.com/watch?v=pTFZFxd4hOI)

## Install docker 
Download here ---> https://docs.docker.com/get-docker/

Download & install depends on your operating system 

Once installed, you **need to start docker engine** before working on building your application 
- by click on a Docker app
- when you see the Docker icon appear on the status bar, that means you're ready to go

Now on your terminal run ```docker --version``` to check your docker version. 

Or you can also check if you're connected to Docker hub by typing ```docker run hello-world``` and it will execute this hello-world image on your computer.
If your terminal shows something else, you're probably not connected to Docker hub. 

!! First, before you go over the install process again, 
**make sure  you launch the Docker app and wait for the icon to show up on your status bar**

## Docker Tags
```docker --help```

```docker build --help```

# 2. Docker Run
Run docker with the python:3.9 image in an interactive mode and the entrypoint of bash.

```docker run -it --entrypoint=bash python:3.9```


#### Some other/useful tags:

To stop docker container 
```docker stop [container ID]```

Check running containers
```docker ps```

Check all containers (stopped & running)
```docker ps -a```

To nuke all the containers
```docker rm -f $(docker ps -aq)```

To check docker network
```docker network ls```

To prune/remove network
```docker network prune```

# Prepare Postgres
## Run Postgres and Load data
In your working directory, On Terminal prompt run:

```
docker run -it \
  -e POSTGRES_USER="root" \
  -e POSTGRES_PASSWORD="root" \
  -e POSTGRES_DB="green_taxi" \
  -v $(pwd)/green_taxi_postgres_data:/var/lib/postgresql/data \
  -p 5431:5432 \
postgres:13
```


## Access database with pgcli

```pgcli -h localhost -p 5431 -U root -d green_taxi```

## download file
On Terminal prompt, run this code to download data

```wget https://github.com/DataTalksClub/nyc-tlc-data/releases/download/green/green_tripdata_2019-01.csv.gz```

and 

```wget https://s3.amazonaws.com/nyc-tlc/misc/taxi+_zone_lookup.csv```

# Data Ingestion
Once you're on Postgress & pgcli

Run Jupyter Notebook:

```
import pandas as pd

df_green = pd.read_csv("green_tripdata_2019-01.csv", nrows=100)

df_green.lpep_pickup_datetime = pd.to_datetime(df_green.lpep_pickup_datetime)
df_green.lpep_dropoff_datetime = pd.to_datetime(df_green.lpep_dropoff_datetime)

#### Next, create connection to Postgress
from sqlalchemy import create_engine
engine = create_engine("postgresql://root:root@localhost:5431/green_taxi")

#### check connection
engine.connect()

print(pd.io.sql.get_schema(df_green, name="green_taxi_data", con=engine))

df_green_iter = pd.read_csv("green_tripdata_2019-01.csv", iterator=True, chunksize=100000)

df = next(df_green_iter)

df_green.lpep_pickup_datetime = pd.to_datetime(df_green.lpep_pickup_datetime)
df_green.lpep_dropoff_datetime = pd.to_datetime(df_green.lpep_dropoff_datetime)

#### create an empty table
df_green.head(n=0).to_sql(name="green_taxi_data", con=engine, if_exists="replace")

### Fill data to table
#### to see how much time does it take to complete filling a chunk in the table
from time import time

#### running loop to fill chunks of data in the table until complete
while True:
    t_start = time()
    df_green = next(df_green_iter)
    
    df_green.lpep_pickup_datetime = pd.to_datetime(df_green.lpep_pickup_datetime)
    df_green.lpep_dropoff_datetime = pd.to_datetime(df_green.lpep_dropoff_datetime)
    
    df_green.to_sql(name="green_taxi_data", con=engine, if_exists="append")
    
    t_end = time()
    print("insert another chunk, took %.1f second" % (t_end - t_start))


#### check if all data is in the table
len(df_green)

#### Do the same with Taxi_Zone

df_zone = pd.read_csv("taxi+_zone_lookup.csv")

df_zone.head()

df_zone.to_sql(name="taxi_zone", con=engine, if_exists="replace")
```

# Work with data on Postgresql 

## First, run docker-compose & setup network
**Create docker-compose.yml**:

```
services:
  postgres:
    image: postgres:13
    environment:
      - POSTGRES_USER=root
      - POSTGRES_PASSWORD=root
      - POSTGRES_DB=green_taxi
    volumes:
      - "./green_taxi_postgres_data:/var/lib/postgresql/data:rw"
    ports:
      - "5431:5432"
    networks:
      - pg-network

  pgadmin:
    image: dpage/pgadmin4
    environment:
      - PGADMIN_DEFAULT_EMAIL=admin@admin.com
      - PGADMIN_DEFAULT_PASSWORD=root
      - PGADMIN_CONFIG_SERVER_MODE=False
    volumes:
      - ./pgadmin:/var/lib/pgadmin
    ports:
      - "8080:80"
    networks:
      - pg-network

networks:
  pg-network:
    name: pg-network
```


On Terminal prompt run:

```docker-compose up -d```

Once connected, go to http://localhost:8080
- Setup server 
- Select dataset to work on (---> green_taxi_data)

# Use query tool
# 3. Count Records

```
SELECT
	CAST(lpep_pickup_datetime AS date) as "date",
	COUNT(1)
FROM 
	green_taxi_data
GROUP BY
	CAST(lpep_pickup_datetime AS date)
ORDER BY "date" ASC;
```


# 4. Largest trip for each day 
```
SELECT
	CAST(lpep_pickup_datetime AS date) as "date",
	COUNT(1) as "count",
	MAX(trip_distance) as "distance"
FROM 
	green_taxi_data
GROUP BY
	CAST(lpep_pickup_datetime AS date)
ORDER BY "distance" DESC;
```

# 5. The number of passengers
```
SELECT
    passenger_count,
    count(*)
FROM
   green_taxi_data
WHERE
   passenger_count IN (2, 3) AND
   CAST(lpep_pickup_datetime AS date) = '2019-01-01'
GROUP BY 
    passenger_count;
```

# 6. Largest tip
```
SELECT 
	MAX(tip_amount),
  	CONCAT (zpu."Borough", ' / ', zpu."Zone") AS "pickup_location",
  	CONCAT (zdo."Borough", ' / ', zdo."Zone") AS "dropoff_location"
FROM 
  	green_taxi_data t 
  	JOIN taxi_zone zpu ON t."PULocationID" = zpu."LocationID"
  	JOIN taxi_zone zdo ON t."DOLocationID" = zdo."LocationID"
GROUP BY
	dropoff_location,
	pickup_location
ORDER BY
	MAX(tip_amount) DESC;
 ```
 



## Submitting the solutions

* Form for submitting: [form](https://forms.gle/EjphSkR1b3nsdojv7)
* You can submit your homework multiple times. In this case, only the last submission will be used. 

