# bi-playground <!-- omit in toc -->

explore different data viz solution for Business intelligence (BI)

- [Redash](https://redash.io): acquired by Databricks in 2020

In this repo, we are using the [Kubernetes](https://kubernetes.io/) to deploy all components.

- [prerequisites](#prerequisites)
- [setup](#setup)
  - [namespace](#namespace)
  - [postgresql](#postgresql)
    - [load data from sqlite to psql](#load-data-from-sqlite-to-psql)
  - [redash](#redash)
  - [superset](#superset)
- [explore](#explore)
  - [redash](#redash-1)
    - [parameterized query](#parameterized-query)
    - [dashboard based on queries](#dashboard-based-on-queries)
  - [superset](#superset-1)
    - [create the connection from web ui](#create-the-connection-from-web-ui)
- [cleanup](#cleanup)
- [references](#references)

## prerequisites

- [Rancher Desktop](https://github.com/rancher-sandbox/rancher-desktop): `1.4.1`
- Kubernetes: `v1.24.3`
- kubectl `v1.23.3`
- Helm: `v3.9.0`

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

follow the [official helm chart](https://superset.apache.org/docs/installation/running-on-kubernetes/) to deploy Redash

```sh
helm repo add superset https://apache.github.io/superset
```

```sh
helm upgrade --install bi-superset superset/superset -f superset/values.yaml -n bi
```

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

```sh
kubectl port-forward svc/bi-superset -n bi 8088:8088
```

- FIXME: lost connection
  - `E1225 18:33:45.746629   82460 portforward.go:406] an error occurred forwarding 8088 -> 8088: error forwarding port 8088 to pod 78d8fb8629c8ab4dc2baa54d14207d413094f45c96de2378811cf54862124671, uid : failed to execute portforward in network namespace "/var/run/netns/cni-3e90f492-496d-9f08-9e0f-6b0a5d1e1008": readfrom tcp4 127.0.0.1:59662->127.0.0.1:8088: write tcp4 127.0.0.1:59662->127.0.0.1:8088: write: broken pipe`

login with the default confidential

```sh
user: admin
password: admin
```

#### create the connection from web ui

![superset connection](assets/superset_connection.png)

## cleanup

tl;dr: `./scripts/down.sh`

```sh
helm uninstall bi-redash -n bi
helm uninstall bi-superset -n bi
helm uninstall bi-postgresql -n bi
kubectl delete pvc --all -n bi
kubectl delete namespace bi
```

## references
