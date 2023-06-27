## Step 0

### Setting up the cluster

Install virtualbox (needed for minikube)
https://www.virtualbox.org/wiki/Downloads

Install minikube
https://minikube.sigs.k8s.io/docs/start/

```bash
minikube start --kubernetes-version=v1.27.0
minikube addons enable metrics-server
minikube dashboard

# If need to start over you can
minikube delete
# and try the above commands again
```

Install kubectl cli
https://kubernetes.io/docs/tasks/tools/

```bash
kubectl version --short
```

Install psql client
https://www.postgresql.org/download/

```bash
psql --version
```

Install QGIS
https://www.qgis.org/en/site/forusers/download.html

Install pgAdmin
https://www.pgadmin.org/download/

### Setting up PGO

PGO Quick Start
https://access.crunchydata.com/documentation/postgres-operator/latest/quickstart/

```bash
cd postgres-operator-examples
kubectl apply -k kustomize/install/namespace
kubectl apply --server-side -k kustomize/install/default
kubectl -n postgres-operator get pods
```

Change the default namespace

```bash
kubectl config set-context --current --namespace=postgres-operator
```

Connect to the database

```bash
# Expose the 6432 port
PG_CLUSTER_PRIMARY_POD=$(kubectl get pod -n postgres-operator -o name \
  -l postgres-operator.crunchydata.com/cluster=hippo,postgres-operator.crunchydata.com/role=master)
kubectl -n postgres-operator port-forward "${PG_CLUSTER_PRIMARY_POD}" 6432:5432

# Get the connection string
kubectl -n postgres-operator get secrets hippo-pguser-hippo -o go-template='{{.data.uri | base64decode}}'

# Connect using psql. Change the host to `localhost` and port to `6432`
psql postgresql://hippo:ClW0%7B3%28%3Ff.%7DUuT4%28GXnJW0cz@localhost:6432/hippo
```

Check some SQL commands

```SQL
-- look at all relations
\d
-- look at all available extensions
\dx

```

## Step 1

Let's get the admin access
Update the manifest

```yaml
spec:
  users:
    - name: postgres
```

```bash
# Update the cluster before the following commands
# Get the postgres uer password
kubectl get secrets hippo-pguser-postgres -o go-template='{{.data.password | base64decode}}'

# Connect using postgres user. Change the user and password
psql 'postgresql://postgres:;>=*h7>Y)=iS]sI=dw+s1rVn@localhost:6432/hippo'
```

Let's add some data

```SQL

CREATE TABLE foss4g (
    year INTEGER,
    city TEXT
);

INSERT INTO foss4g (year, city) VALUES (2023, 'Prizren');
INSERT INTO foss4g (year, city) VALUES (2022, 'Firenze');
INSERT INTO foss4g (year, city) VALUES (2021, 'Buenos Aires');
INSERT INTO foss4g (year, city) VALUES (2020, 'Canada');
INSERT INTO foss4g (year, city) VALUES (2019, 'Bucharest');
INSERT INTO foss4g (year, city) VALUES (2017, 'Boston');
```

Now let's delete the pod

```bash
kubectl delete pod hippo-instance[SOMETHING]
```

it'll automatically recover along with the data

## Step 2

Let's add some GIS data
Before that we add data, we to update the container image from here, select the gis containers:

https://www.crunchydata.com/developers/download-postgres/containers

Update the database manifest with:

```yaml
spec:
  image: registry.developers.crunchydata.com/crunchydata/crunchy-postgres-gis:ubi8-15.3-3.3-0
```

```SQL
-- create the postgis extension before adding any geometry
Create extension postgis;

-- Check if it's added and the version
\dx
```

Optional Step

- Go to - https://raw-data-api0.hotosm.org/v1/docs#/default
- GET /countries - Entry country name e.g. Kosovo. This'll give you the shapefile for the country
- POST /snapshot - Use the shape geojson from above and bulid the request body with filename as .sql. Check the sample request body in data/kosovo-request.sh. This'll give you a task ID.
  -GET /task/status/{task_id} - Check the status and when it's done, download the file and move onto the next steps

```bash
CONN_DB='postgresql://postgres:;>=*h7>Y)=iS]sI=dw+s1rVn@localhost:6432/hippo'
cd ../data/
psql $CONN_DB -f data/kosovo.sql
psql $CONN_DB -f data/albania.sql
psql $CONN_DB
```

```SQL
-- check the schema for the table
\d sql_statement;
-- check the total rows
Select count(*) from sql_statement;
-- Convert the json columns to jsonb (for peformance)
ALter table sql_statement ALTER column tags type jsonb using tags::jsonb;
-- Filter on tags
SELECT ogc_fid FROM sql_statement WHERE tags->>'internet_access' = 'yes';
```

Let's see this in QGIS

- Open QGIS
- Select postgres connection
- Add port, user and password configuration. Make sure SSL is allowed
- Test connection
- Load the data on the map by clicking on the table
- Optionally load OSM tiles as base layer
- You can also see the attributes in the attributes table

Similarly you can connect pgAdmin4 or any client with the database. Later on we'll see better ways to connect to the database using pgBouncer.

## Step 3

Let's extend the capabilities of our database.

- timescaleDB for the temporal component
- uber h3 index for h3 grids in the database, used for faster spatial queries

Let's update the manifest

```yaml
spec:
  patroni:
    dynamicConfiguration:
      postgresql:
        parameters:
          shared_preload_libraries: timescaledb
```

Let's create the extension

```SQL
-- create the extension
create extension timescaleDB;
-- verify the extension
\dx
```

Not let's add some data

```SQL
CREATE TABLE countries (
    name VARCHAR(255),
    geom GEOMETRY
);
```

```bash
psql $CONN_DB -f data/countries.sql
```

Let's create some spatial-temporal data

```SQL
-- let's create sample weather data
CREATE TABLE weather_data (
    id SERIAL PRIMARY KEY,
    timestamp TIMESTAMP,
    temperature FLOAT,
    humidity FLOAT,
    precipitation FLOAT,
	location GEOMETRY(Point, 4326)
);

-- Optional: These will take some time to complete, use the .sql file below. it has relatively smaller set of data
INSERT INTO weather_data (timestamp,
 temperature, humidity, precipitation, location)
SELECT
    timestamp,
    random() * 50, -- generate random temperature between 0 and 50
    random() * 100, -- generate random humidity between 0 and 100
    random() * 10, -- generate random precipitation between 0 and 10
	(ST_DumpPoints(ST_GeneratePoints((Select geom from countries where name = 'Kosovo'), 1))).geom
FROM generate_series(
    '2021-01-01'::timestamp, -- start date
    '2023-01-01'::timestamp, -- end date
    '30 sec' -- interval of 1 minute
) AS timestamp;

INSERT INTO weather_data (timestamp,
 temperature, humidity, precipitation, location)
SELECT
    timestamp,
    random() * 50, -- generate random temperature between 0 and 50
    random() * 100, -- generate random humidity between 0 and 100
    random() * 10, -- generate random precipitation between 0 and 10
	(ST_DumpPoints(ST_GeneratePoints((Select geom from countries where name = 'Albania'), 1))).geom
FROM generate_series(
    '2021-01-01'::timestamp, -- start date
    '2023-01-01'::timestamp, -- end date
    '30 sec' -- interval of 1 minute
) AS timestamp;

-- this will take some time
psql $CONN_DB -f data/weather-data.sql
```

Let's do some temporal stuff

```SQL
-- let's find avg temperature for Kosovo and Albania (Obviously not correct)
SELECT AVG(temperature) AS avg_temperature
FROM weather_data
where ST_Within(location, (Select geom from countries where name ='Albania'));

SELECT AVG(temperature) AS avg_temperature
FROM weather_data
where ST_Within(location, (Select geom from countries where name ='Kosovo'));

-- let's create a hypetable
create table weather_data_hypertable as select * from weather_data;

SELECT create_hypertable('weather_data_hypertable','timestamp', chunk_time_interval => INTERVAL '1 day', migrate_data => true);

\d+ weather_data_hypertable;

-- avg temperature by day
SELECT date_trunc('day', timestamp) AS day,
       AVG(temperature) AS average_temperature
FROM weather_data
GROUP BY day
ORDER BY day;

-- avg temperature by day in hypertable
SELECT time_bucket(INTERVAL '1 day', timestamp::TIMESTAMP)
  AS day, avg(temperature)
FROM weather_data_hypertable
GROUP BY day
ORDER BY day;

-- https://docs.timescale.com/api/latest/hyperfunctions/gapfilling/time_bucket_gapfill/
-- gapfill can fill values where the data is missing

```
