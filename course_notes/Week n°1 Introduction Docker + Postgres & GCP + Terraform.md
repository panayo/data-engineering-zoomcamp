## Table of content

- <a href ="#1">1. Docker and SQL</a>
	- <a href ="#1.1">1.1 Docker</a>
	- <a href = "#1.2">1.2 Practice with Docker </a>
	- <a href = "#1.3">1.3 Docker and postgres </a>
	- <a href = "#1.4">1.4 Populate database from the jupyter </a>
	- <a href = "#1.5">1.5 Connect PgAdmin and postgres (docker network)</a>
	- <a href = "#1.6">1.6 Dockerizing INgestion Script</a>
	- <a href = "#1.7">1.7 Running Postgres and pgAdmin with Docker-compose</a>
	-  <a href = "#1.8">1.8 SQL Refresher</a>
2.


## <a name = "1">1. Docker and SQL</a>

### <a name = "1.1">1.1 Docker </a>

We wan to dockerize the following data pipeline, for instance we need Python 3.9, Pandas Postgres connection librairy etc.

![data_pipeline](images/data_pipeline.png)

We can also have the Postgres and pgAdmin in another containers.

![containers](images/containers.png)

 - Local experiments
 - Integration tests (CI/CD)
 - Reproducibility
 - Running pipelines on the cloud (AWS batch, Kubernetes jobs)
 - Spark
 - Serverles (AWS Lambda, Google functions)

### <a name = "1.2">1.2 Practice with Docker </a>

- Go to the course folder : `cd ~/data-engineering-zoomcamp/week_1_basics_n_setup/2_docker_sql`
- Start vscode : code .
- An example of docker command : `docker run -it ubuntu bash`  (ubuntu bash image)
- Or : `docker run -it python:3.9`
In the last example we only have acces to python prompt and not in the bash. If we want for instance to have pandas installed we need to add it in the entrypoint parameter bash in order to execute `pip install pandas`:
- `docker run -it --entrypoint=bash python:3.9`
Be careful, is not persistent!
- To exit : CTRl+D
We need to specify all the thigs we want to run in order to create a new image based on a default image:
- Create a file called Dockerfile :
```python
FROM python:3.9.1
 
RUN apt-get install wget
RUN pip install pandas sqlalchemy psycopg2

WORKDIR /app
COPY ingest_data.py ingest_data.py

ENTRYPOINT [ "python", "ingest_data.py" ]
```

- To launch it : `docker build -t test::pandas .`  (test is the name and pandas si the tag and the . is to use the Dockerfile located here). Sometimes is better to use `sudo docker build -t test::pandas` if you have several users accounts.

Let's create a python file `pipeline.py`:
```python
import sys
import pandas as pd

print(sys.argv)
day = sys.argv[1]

# some fancy stuff with pandas

print(f'job finished successfully for day = {day}')
```

And add to the docker file the `WORKDIR` and the `COPY` in order to create a work directory and copy `pipeline.py` inside.

```python 
FROM python:3.9.1

RUN apt-get install wget
RUN pip install pandas 

WORKDIR /app
COPY pipeline.py pipeline.py

ENTRYPOINT [ "bash"]
```

- Build the docker : `sudo docker build -t test:pandas .`
- Now run it : `sudo docker run -it test:pandas`
In the terminal you can see now the `/app` and that the script `pipeline.py` is copied inside.

![docker_test](images/docker_test.png)

Now let's say we want to execute directly the pipeline.py with an argument 2023-01-30, in the Dockerfile you need to change `ENTRYPOINT [ "bash"]` with `ENTRYPOINT [ "python","pipeline"]`. Next:
 - Rebuild the image : `sudo docker build -t test:pandas .`
 - Run with the argument needed for the pipeline script : `sudo docker run -it test:pandas 2022-01-30`
![docker_run_arguments](images/docker_run_arguments.png)


### <a name = "1.3">1.3 Docker and postgres </a>

- Run the official image of postgres and set the environemental variables (-e parameter) and the volume (-v parameter): 
```bash
docker run -it \
	-e POSTGRES_USER="root" \
	-e POSTGRES_PASSWORD="root" \
	-e POSTGRES_DB="ny_taxi" \
	-v /home/panayotis/data-engineering-zoomcamp/week_1_basics_n_setup/2_docker_sql/ny_taxi_postgres_data:/var/lib/postgresql/data \ # ny_taxi_postgres_data is located in the docker and we map with /var/lib/postgresql/data
	-p 5432:5432 \
	postgres:13
```

docker run -it -e POSTGRES_USER="root"  -e POSTGRES_PASSWORD="root"  -e POSTGRES_DB="ny_taxi"  -v /home/panayotis/data-engineering-zoomcamp/week_1_basics_n_setup/2_docker_sql/ny_taxi_postgres_data:/var/lib/postgresql/data -p 5432:5432 postgres:13

![docker_postgres](images/docker_postgres.png)

Now we use pgcli client to acces to the database:
 - Login to the database : `pgcli -h localhost -p 5432 -u root -d ny_taxi`
 - Entre the password here is `root`

### <a name = "1.4">1.4 Populate database from the jupyter </a>

Now we will open **Jupyter** and populate the database. In the terminal inside the VM launch the Jupuyter and port forward the port to open it (see the Setting up the Environment GCP (Cloud VM + SSH access) note).

Follow the codes given to the `upload-data.ipynb` to open the csv/parquet and load to the database. Briefly :

- pd.io.get_schema(df, name= 'yellow_taxi_data') : is the statement on how is the schema in SQL.
- Don't forget to convert to datetime the datetime columns with `df.tpep_pickup_datetime = pd.to_datetime(df.tpep_pickup_datetime)`
- Make the connection with the database with sqlalchemy : `engine = create_engine('postgresql://root:root@localhost:5432/ny_taxi')`
- Create now the schema inside the database : `pd.io.sql.get_schema(df, name='yellow_taxi_data', con=engine)`

We will now **chunk** the file in order to load to the database batch by batch.
 - Create the iterator : `df_iter = pd.read_csv('yellow_tripdata_2019-01.csv', iterator=True, chunksize=100000)`
 - Load now batch by batch use the next(df_iter) :
```python
while True: 
    t_start = time()

    df = next(df_iter)

    df.tpep_pickup_datetime = pd.to_datetime(df.tpep_pickup_datetime)
    df.tpep_dropoff_datetime = pd.to_datetime(df.tpep_dropoff_datetime)
    
    df.to_sql(name='yellow_taxi_data', con=engine, if_exists='append')

    t_end = time()

    print('inserted another chunk, took %.3f second' % (t_end - t_start))
    ```


In the pgcli :
 - See the schema of the database : `\d yellow_taxi_data`
 - Count number of rows : `SELECT count(1) FROM yellow_taxi_data;`


### <a name = "1.5">1.4 Connect PgAdmin and postgres (docker network)</a>

We will connect with pgAdmin in order to have a more convenient way to query the database. Since we use docker we can directly pull the official image of pgAdmin and use it.

Command to run the pgAdmin with docker:
```bash 
docker run -it \
-e PGADMIN_DEFAULT_EMAIL="admin@admin.com" \
-e PGADMIN_DEFAULT_PASSWORD="root" \
-p 8080:80 \
dpage/pgadmin4
```

Now we can go to the localhost:8080 and logging with admin@admin.com and root.

Now we need to **create a new server** :
- Click on 'Add new server' : 
	- Name : Local docker
	- In connection :
		- Host : localhost
		- Username : root
		- Password : root

And when we ave we get an error: 
![pgadmin_error](images/pgadmin_error.png)

The **reason** for that is because we running the pgAdmin inside a container, and it tries to find this postgres inside the specific container (which is only composed of pgAdmin). We need somehow to link the two containers (database and pgadmin). To do this we need to link them under one network.

We need first to stop the containers, and the follow the next steps:

- Create a network with docker : `docker network create pg-network`
- Specify in the two images postgres and pgadmin the `--network=pg-network` argument.

Command to run the pgAdmin and postgres with docker under the `pg-network`:
for the postgres we will also add the `--name pg-database`

```bash
docker run -it \
	-e POSTGRES_USER="root" \
	-e POSTGRES_PASSWORD="root" \
	-e POSTGRES_DB="ny_taxi" \
	-v ny_taxi_postgres_data:/var/lib/postgresql/data \ 
	-p 5432:5432 \
	--network=pg-network \
	--name pg-database \
	postgres:13
```

docker run -it -e POSTGRES_USER="root"  -e POSTGRES_PASSWORD="root"  -e POSTGRES_DB="ny_taxi"  -v /home/panayotis/data-engineering-zoomcamp/week_1_basics_n_setup/2_docker_sql/ny_taxi_postgres_data:/var/lib/postgresql/data -p 5432:5432  --network=pg-network --name pg-database postgres:13

```bash 
docker run -it \
-e PGADMIN_DEFAULT_EMAIL="admin@admin.com" \
-e PGADMIN_DEFAULT_PASSWORD="root" \
-p 8080:80 \
--network=pg-network \
dpage/pgadmin4
```

Now the two docker containes are hosted under the same network (`pg-network`).

Now we need to **create a new server** on pdAdmin:
- Click on 'Add new server' : 
	- Name : Docker localhost
	- In connection :
		- Host :  pg-database (from --name pg-database)
		- Username : root
		- Password : root

And now we can explore our database and query with the pgAdmin GUI.

### <a name = "1.6">1.6 Dockerizing INgestion Script</a>

We will turn the notebook `upload-data.ipynb` into a script and dockerize it in the `pipeline.py`. to populate the database. We can directly convert the note into a script :

 - `nbconvert --to=script upload-data.ipynb`

We will use the `argparse` package to pass the parameters of the script, see the end of the following script (called `ingest_data.py`).

After cleaning and refactoring :

```python
#!/usr/bin/env python
# coding: utf-8
import os
import argparse
from time import time
import pandas as pd
from sqlalchemy import create_engine


def main(params):
	user = params.user
	password = params.password
	host = params.host
	port = params.port
	db = params.db
	table_name = params.table_name
	url = params.url
	# the backup files are gzipped, and it's important to keep the correct extension
	
	# for pandas to be able to open the file
	if url.endswith('.csv.gz'):
		csv_name = 'output.csv.gz'
	else:
		csv_name = 'output.csv'
	os.system(f"wget {url} -O {csv_name}")

	engine = create_engine(f'postgresql://{user}:{password}@{host}:{port}/{db}')
	df_iter = pd.read_csv(csv_name, iterator=True, chunksize=100000)
	df = next(df_iter)
	df.tpep_pickup_datetime = pd.to_datetime(df.tpep_pickup_datetime)
	df.tpep_dropoff_datetime = pd.to_datetime(df.tpep_dropoff_datetime)
	df.head(n=0).to_sql(name=table_name, con=engine, if_exists='replace')
	df.to_sql(name=table_name, con=engine, if_exists='append')

while True:
	try:
		t_start = time()
		df = next(df_iter)
		
		df.tpep_pickup_datetime = pd.to_datetime(df.tpep_pickup_datetime)
		df.tpep_dropoff_datetime = pd.to_datetime(df.tpep_dropoff_datetime)
		df.to_sql(name=table_name, con=engine, if_exists='append')
		t_end = time()
		
		print('inserted another chunk, took %.3f second' % (t_end - t_start))

	except StopIteration:
		print("Finished ingesting data into the postgres database")
		break

  

if __name__ == '__main__':

	parser = argparse.ArgumentParser(description='Ingest CSV data to Postgres')
	parser.add_argument('--user', required=True, help='user name for postgres')
	parser.add_argument('--password', required=True, help='password for postgres')
	parser.add_argument('--host', required=True, help='host for postgres')
	parser.add_argument('--port', required=True, help='port for postgres')
	parser.add_argument('--db', required=True, help='database name for postgres')
	parser.add_argument('--table_name', required=True, help='name of the table where we will write the results to')
	parser.add_argument('--url', required=True, help='url of the csv file')
	args = parser.parse_args()
	main(args)
```

To test if the script is working we will drop the table :
 - In the query tool of pgAdmin : `DROP TABLE yellow_taxi_data;`

Let's run the script manually :
```bash
URL="https://github.com/DataTalksClub/nyc-tlc-data/releases/download/yellow/yellow_tripdata_2021-01.csv.gz"
```
And now the script:
```bash
python ingest_data.py \
	--user=root \
	--password=root \
	--host=localhost \
	--port=5432 \
	--db=ny_taxi \
	--table_name=yellow_taxi_trips \
	--url=${URL}
```

![ingest_data_scipt](images/ingest_data_scipt.png)


When the script is finished we can now refresh the database in the pgAdmin ans see the populated database.

Let's dockerize this script, here the Docerfile :

```python

RUN apt-get install wget
RUN pip install pandas sqlalchemy psycopg2

WORKDIR /app

COPY ingest_data.py ingest_data.py


ENTRYPOINT [ "python", "ingest_data.py" ]
```

Now build the container :
- `sudo docker build -t taxi_ingest:v001 .`
Now run it, in docker run don't forget to change `--host=` from localhost  to `pg-database`:


```bash

URL="https://github.com/DataTalksClub/nyc-tlc-data/releases/download/yellow/yellow_tripdata_2021-01.csv.gz"

docker run -it \
--network=pg-network \
taxi_ingest:v001 \
--user=root \
--password=root \
--host=pg-database \
--port=5432 \
--db=ny_taxi \
--table_name=yellow_taxi_trips \
--url=${URL}
```

### <a name = "1.7">1.7 Running Postgres and pgAdmin with Docker-compose</a>

One can specify one yaml file to compose all the cotainers with their details. Docker-compose allows to handles complex configuration of setup.

For our project we have two services pgAdmin and postgres the `docker-compose.yaml`:

```bash
services:
	pgdatabase:
		image: postgres:13
		environment:
			- POSTGRES_USER=root
			- POSTGRES_PASSWORD=root
			- POSTGRES_DB=ny_taxi
		volumes:
			- "./ny_taxi_postgres_data:/var/lib/postgresql/data:rw"
		ports:
			- "5432:5432"
	pgadmin:
		image: dpage/pgadmin4
		environment:
			- PGADMIN_DEFAULT_EMAIL=admin@admin.com
			- PGADMIN_DEFAULT_PASSWORD=root
		ports:
			- "8080:80"
```
Run it (don't forget to kill running containers):
```bash
docker-compose up
```
The problem with `pgAdmin`is that the confiuration is not persistent so we need to create again the server:
![pgadmin_recreate](images/pgadmin_recreate.png)

Run in detached mode:
```bash
docker-compose up -d
```

Shutting it down:
```bash
docker-compose down
```

 ### <a name = "1.8">1.8 SQL Refresher</a>

We have two tables in the database `yellow_taxi_trips` and `zones`.

**INNER JOIN**
Join the `yellow_taxi_trips` with `zones` to add the pickup and drop off locations information (stored in zones).

```sql
SELECT 
	*
FROM 
	yellow_taxi_trips t,
	zones zpu,
	zones zdo
WHERE 
	t."PULocationID" = zpu."LocationID" AND
	t."DOLocationID" = zdo."LocationID"
```

Query only the desired columns from the previous query :

```sql
SELECT
	tpep_pickup_datetime,
	tpep_dropoff_datetime,
	total_amount,
	CONCAT(zpu."Borough",' / ', zpu."Zone") AS "pick_up_loc",
	CONCAT(zdo."Borough",'/',zdo."Zone") AS "dropoff_loc"
FROM 
	yellow_taxi_trips t,
	zones zpu,
	zones zdo
WHERE 
	t."PULocationID" = zpu."LocationID" AND
	t."DOLocationID" = zdo."LocationID"
```

**ANOTHER WAY with JOIN:**

```sql 
SELECT
	tpep_pickup_datetime,
	tpep_dropoff_datetime,
	total_amount,
	CONCAT(zpu."Borough",' / ', zpu."Zone") AS "pick_up_loc",
	CONCAT(zdo."Borough",'/',zdo."Zone") AS "dropoff_loc"
FROM 
	yellow_taxi_trips t JOIN zones zpu 
		ON t."PULocationID" = zpu."LocationID" 
		JOIN zones zdo ON t."DOLocationID" = zdo."LocationID"
```

Checking for Location IDs is null  :

```sql
SELECT
	tpep_pickup_datetime,
	tpep_dropoff_datetime,
	total_amount
FROM
	yellow_taxi_trips t
WHERE 
	"DOLocationID" is NULL
```
Checking for Location IDs is in the zones tables:
```sql
SELECT
	tpep_pickup_datetime,
	tpep_dropoff_datetime,
	total_amount
FROM
	yellow_taxi_trips t
WHERE 
	"DOLocationID" NOT IN (
	SELECT "LocationID" FROM zones)
```

**LEFT/RIGHT JOIN :**

```sql
SELECT
	tpep_pickup_datetime,
	tpep_dropoff_datetime,
	total_amount,
	CONCAT(zpu."Borough",' / ', zpu."Zone") AS "pick_up_loc",
	CONCAT(zdo."Borough",'/',zdo."Zone") AS "dropoff_loc"
FROM 
	yellow_taxi_trips t LEFT JOIN zones zpu 
		ON t."PULocationID" = zpu."LocationID" 
		LEFT JOIN zones zdo ON t."DOLocationID" = zdo."LocationID"
```

The **GROUPBY**:
```sql
SELECT
	CAST(tpep_dropoff_datetime AS DATE) as "day",
	COUNT(1)
FROM
	yellow_taxi_trips t
GROUP BY
	"day"
ORDER BY "day" ASC;
```


```sql
SELECT
	CAST(tpep_dropoff_datetime AS DATE) as "day",
	COUNT(1),
	MAX(total_amount),
	MAX(passenger_count)
FROM
	yellow_taxi_trips t
GROUP BY
	"day"
ORDER BY "count" DESC;
```

**GROUP BY multiple fields**:

```sql
SELECT
	CAST(tpep_dropoff_datetime AS DATE) as "day",
	"DOLocationID",
	COUNT(1) as "count",
	MAX(total_amount),
	MAX(passenger_count)
FROM
	yellow_taxi_trips t
GROUP BY
	1,2
ORDER BY 
	"day" ASC,
	"DOLocationID" ASC;
	```
