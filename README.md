# bi-playground <!-- omit in toc -->

explore different data viz solution for Business intelligence (BI)

- [Redash](https://redash.io): acquired by Databricks in 2020
- [Apache Superset](https://superset.apache.org)
- [Metabase](https://www.metabase.com)

In this repo, we are using the [Kubernetes](https://kubernetes.io/) to deploy all components.

- [prerequisites](#prerequisites)
- [setup](#setup)
  - [namespace](#namespace)
  - [postgresql](#postgresql)
    - [load data from sqlite to psql](#load-data-from-sqlite-to-psql)
  - [redash](#redash)
  - [superset](#superset)
  - [metabase](#metabase)
- [explore](#explore)
  - [redash](#redash-1)
    - [parameterized query](#parameterized-query)
    - [dashboard based on queries](#dashboard-based-on-queries)
  - [superset](#superset-1)
    - [create the connection from web ui](#create-the-connection-from-web-ui)
    - [dashboard with filters](#dashboard-with-filters)
  - [metabase](#metabase-1)
    - [create the connection from web ui](#create-the-connection-from-web-ui-1)
    - [parameterized query](#parameterized-query-1)
    - [dashboard with filters](#dashboard-with-filters-1)
- [cleanup](#cleanup)
- [references](#references)

## prerequisites

- [Rancher Desktop](https://github.com/rancher-sandbox/rancher-desktop): `1.7.0`
- Kubernetes: `v1.25.4`
- kubectl `v1.26.0`
- Helm: `v3.10.2`

## setup

tl;dr: `./scripts/up.sh`

### namespace

```sh
kubectl create namespace bi --dry-run=client -o yaml | kubectl apply -f -
```

### postgresql

follow the [bitnami postgresql chart](https://github.com/bitnami/charts/tree/master/bitnami/postgresql) to install postgresql

```sh
helm repo add bitnami https://charts.bitnami.com/bitnami
```

```sh
helm upgrade --install bi-postgresql bitnami/postgresql -n bi -f postgresql/values.yaml
```

verify the installation

```sh
kubectl run bi-postgresql-client --rm --tty -i --restart='Never' --namespace bi --image docker.io/bitnami/postgresql:15.0.0-debian-11-r3 --env="PGPASSWORD=bi_password" -- psql --host bi-postgresql -U bi_user -d bi -p 5432 -c "\l"
```

```sh
                                                 List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    | ICU Locale | Locale Provider |   Access privileges
-----------+----------+----------+-------------+-------------+------------+-----------------+-----------------------
 bi        | bi_user  | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            | =Tc/bi_user          +
           |          |          |             |             |            |                 | bi_user=CTc/bi_user
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            |
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            | =c/postgres          +
           |          |          |             |             |            |                 | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            | =c/postgres          +
           |          |          |             |             |            |                 | postgres=CTc/postgres
(4 rows)
```

#### load data from sqlite to psql

```sh
kubectl run bi-pgloader -n bi -ti --rm --restart=Never --image=ghcr.io/dimitri/pgloader --overrides='
{
  "spec": {
    "containers":[{
      "name": "main",
      "image": "ghcr.io/dimitri/pgloader",
      "imagePullPolicy":"IfNotPresent",
      "command": ["pgloader", "/var/lib/pgloader/sqlite/stats_can.db", "postgresql://bi_user:bi_password@bi-postgresql/bi"],
      "stdin": true,
      "tty": true,
      "volumeMounts": [{"mountPath": "/var/lib/pgloader/sqlite","name": "store"}]
   }],
   "volumes": [{"name":"store","hostPath":{"path":"'$PWD/sqlite'","type":"Directory"}}]
  }
}'
```

### redash

follow the [community maintained helm chart](https://github.com/getredash/contrib-helm-chart) to deploy Redash

```sh
helm repo add redash https://getredash.github.io/contrib-helm-chart/
```

```sh
helm upgrade --install bi-redash redash/redash -f redash/values.yaml -n bi
```

### superset

use the docker-compose to deploy superset due to the technical issue with the helm chart of Superset

```sh
git clone --depth=1 https://github.com/apache/superset.git
cd superset
nerdctl compose -f docker-compose-non-dev.yml up -d
```

FIXME: follow the [official helm chart](https://superset.apache.org/docs/installation/running-on-kubernetes/) to deploy Superset

- lost connection after port-forwarding
  - `E1225 18:33:45.746629   82460 portforward.go:406] an error occurred forwarding 8088 -> 8088: error forwarding port 8088 to pod 78d8fb8629c8ab4dc2baa54d14207d413094f45c96de2378811cf54862124671, uid : failed to execute portforward in network namespace "/var/run/netns/cni-3e90f492-496d-9f08-9e0f-6b0a5d1e1008": readfrom tcp4 127.0.0.1:59662->127.0.0.1:8088: write tcp4 127.0.0.1:59662->127.0.0.1:8088: write: broken pipe`

### metabase

use the docker-compose to deploy metabase because [the official helm chart is pending](https://github.com/metabase/metabase/pull/16603)

```sh
nerdctl compose -f metabase/docker-compose.yml up -d
```

TODO: there is [no official arm64 image](https://github.com/metabase/metabase/issues/13119) of metabase

## explore

### redash

```sh
kubectl port-forward svc/bi-redash -n bi 8080:80
```

create the admin user and organization

- create from the web ui
- FIXME: create the user using `manage.py`
  - `./manage.py users create_root admin@example.com admin --password=admin_password --org default`
  - `sqlalchemy.exc.OperationalError: (psycopg2.OperationalError) could not connect to server: No such file or directory`

#### parameterized query

```sql
SELECT
  REF_DATE,
  GEO as "geo::multi-filter",
  INDEX_TYPE as "index_type::filter",
  VALUE
FROM new_housing_price_index
WHERE TRUE
AND REF_DATE BETWEEN '{{ REF_DATE.start }}' AND '{{ REF_DATE.end}}'
```

![redash query](assets/redash_query.png)

#### dashboard based on queries

![redash dashboard](assets/redash_dashboard.png)

### superset

visit http://localhost:8088 for the superset web ui

login with the default confidential

```sh
user: admin
password: admin
```

#### create the connection from web ui

in order to connect with the psql instance on kubernetes

```sh
kubectl port-forward svc/bi-postgresql -n bi 5432:5432
```

`192.168.5.2` is the equivalent `host.docker.internal` based on [this discussion](https://github.com/containerd/nerdctl/issues/747)

![superset connection](assets/superset_connection.png)

#### dashboard with filters

![superset dashboard](./assets/superset_dashboard.png)

### metabase

visit http://localhost:3000 for the metabase web ui

create the admin user for the first time

#### create the connection from web ui

in order to connect with the psql instance on kubernetes

```sh
kubectl port-forward svc/bi-postgresql -n bi 5432:5432
```

`192.168.5.2` is the equivalent `host.docker.internal` based on [this discussion](https://github.com/containerd/nerdctl/issues/747)

![metabase connection](./assets/metabase_connection.png)

#### parameterized query

```sql
SELECT
  REF_DATE,
  GEO,
  INDEX_TYPE,
  MAX(VALUE) AS VALUE -- dummy aggregate func
FROM new_housing_price_index
wHERE TRUE
AND REF_DATE >= {{ref_date_start}} -- variable type date
AND REF_DATE <= {{ref_date_end}} -- variable type date
AND {{index_type}} -- variable type field filter
AND {{geos}} -- variable type field filter
GROUP BY 1, 2, 3
```

**caveats**:

- hard-coded row limit: [2,000 unaggregated, 10,000 aggregated](https://discourse.metabase.com/t/how-can-i-change-default-row-limits-currently-its-2000/10707/2)
  - that's why we use the dummy aggregate func in the above sql
  - this is not configurable, you have to [build your own binary](https://github.com/CDCgov/prime-reportstream/issues/4308) in order to tweak this

![metabase query](./assets/metabase_query.png)

#### dashboard with filters

![metabase dashboard](./assets/metabase_dashboard.png)

## cleanup

tl;dr: `./scripts/down.sh`

```sh
helm uninstall bi-redash -n bi
helm uninstall bi-postgresql -n bi
kubectl delete pvc --all -n bi
kubectl delete namespace bi
```

## references
