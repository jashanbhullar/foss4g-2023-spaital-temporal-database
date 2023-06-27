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
