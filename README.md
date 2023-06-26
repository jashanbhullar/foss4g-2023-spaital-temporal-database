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

Let's get the admin access

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
