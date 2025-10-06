---
title: "Porch DB Repo"
type: docs
weight: 1
description: "A tutorial show how to use Porch DB repos"
---

This tutorial is a guide to using Porch DB repos. It is an extension of the
[Install and use Porch](install-and-using-porch.md) guide.

See also [the Nephio Learning Resource](https://github.com/nephio-project/docs/blob/main/learning.md) page for
background help and information.

## Prerequisites

See the "prerequisites" section in the [Install and use Porch](install-and-using-porch.md) guide.

## Set up a local Porch environment

Follow the steps in [Setting up a local environment](environment-setup.md) guide.

## Install and configure Postgres

Postgres is configured to store its data on an external mount at `/tmp/porch/postgres`. Clear old Postgresql data if it already exists:

```
rm -fr /tmp/porch/postgresql/*
```

From the root of Porch run the command:

```
kubectl apply -f examples/tutorials/database-repo/postgres.yaml
```

Wait for Postgresql to come up in the cluster

```
> kubectl get pods -n porch-db
NAME                         READY   STATUS    RESTARTS   AGE
postgresql-b68bd87b5-hgjkj   1/1     Running   0          11s
liam@saor porch % k get pods -n porch-db
NAME                         READY   STATUS    RESTARTS   AGE
postgresql-b68bd87b5-hgjkj   1/1     Running   0          20s

> kubectl get svc -n porch-db
NAME         TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)           AGE
postgresql   LoadBalancer   10.197.167.117   172.18.255.201   55432:32184/TCP   24s
```

Connect to Postgres and initialize the Porch database (the password is "porch"):
```
psql -h 172.18.255.201 -p 55432 -U porch -d porch < examples/tutorials/database-repo/porch-db.sql 
Password for user porch: 
NOTICE:  table "package_revisions" does not exist, skipping
DROP TABLE
NOTICE:  table "packages" does not exist, skipping
DROP TABLE
NOTICE:  table "repositories" does not exist, skipping
DROP TABLE
NOTICE:  type "package_rev_lifecycle" does not exist, skipping
DROP TYPE
CREATE TABLE
CREATE TABLE
CREATE TYPE
CREATE TABLE
```

Connect to Postgres and look around:
```
psql -h 172.18.255.201 -p 55432 -U porch -d porch                                                
Password for user porch: 
psql (15.8 (Homebrew), server 17.0 (Debian 17.0-1.pgdg120+1))
WARNING: psql major version 15, server major version 17.
         Some psql features might not work.
Type "help" for help.

porch=# \d
             List of relations
 Schema |       Name        | Type  | Owner 
--------+-------------------+-------+-------
 public | package_revisions | table | porch
 public | packages          | table | porch
 public | repositories      | table | porch
(3 rows)

porch=# select * from repositories;
 namespace | repo_name | updated | updatedby | deployment 
-----------+-----------+---------+-----------+------------
(0 rows)

porch=# select * from packages;
 namespace | repo_name | package_name | updated | updatedby 
-----------+-----------+--------------+---------+-----------
(0 rows)

porch=# select * from package_revisions;
 namespace | repo_name | package_name | package_rev | workspace_name | updated | updatedby | lifecycle | resources 
-----------+-----------+--------------+-------------+----------------+---------+-----------+-----------+-----------
(0 rows)

```

## Start Porch and build porchctl

```
make run-in-kind
make porchctl
```

Check the "porchctl" that is in your path is the correct version, the build timestamp should be during the time the "make" command above was running. The prochctl birnary is in `.build/porchctl`

```
porchctl version
Version: development-2024-10-14T11:59:42
Git commit: e8ba860c76cb1f5da79b872d353cac22f8b9bd05 (dirty)
```

## Create a DB repo

```
> porchctl repo get -A

> kubectl apply -f examples/tutorials/database-repo/db_repo.yaml 
namespace/porch-demo created
repository.config.porch.kpt.dev/db-repo created

> porchctl repo get -A                                          
NAME      TYPE   CONTENT   DEPLOYMENT   READY   ADDRESS
db-repo   db     Package   false        True    postgresql://porch:porch@172.18.255.201:55432/porch
```

## Run through a package lifecycle

Go to the "starting-with-porch" tutorial directory
```
cd examples/tutorials/starting-with-porch 
```

Initialize a package:
```
> porchctl -n porch-demo rpkg init network-function --repository=db-repo --workspace=v6
porch-demo.db-repo.network-function.v6.v6 created

> porchctl rpkg get -n porch-demo porch-demo.db-repo.network-function.v6.v6
NAME                                        PACKAGE            WORKSPACENAME   REVISION   LATEST   LIFECYCLE   REPOSITORY
porch-demo.db-repo.network-function.v6.v6   network-function   v6              v6         false    Draft       db-repo

```

Pull the package to a local directory:

```
porchctl -n porch-demo rpkg pull porch-demo.db-repo.network-function.v6.v6 blueprints/initialized/network-function

> ls -a  blueprints/initialized/network-function
.  ..  .KptRevisionMetadata  Kptfile  README.md  package-context.yaml```
```

Update the package locally, adding "deployment.yaml":

```
cp blueprints/local-changes/network-function/* blueprints/initialized/network-function

> ls -a  blueprints/initialized/network-function                                        
.  ..  .KptRevisionMetadata  Kptfile  README.md  deployment.yaml  package-context.yaml
```

Push and check the package and its resources:

```
> porchctl -n porch-demo rpkg push porch-demo.db-repo.network-function.v6.v6 blueprints/initialized/network-function
porch-demo.db-repo.network-function.v6.v6 pushed

> porchctl rpkg get -n porch-demo porch-demo.db-repo.network-function.v6.v6                                         
NAME                                        PACKAGE            WORKSPACENAME   REVISION   LATEST   LIFECYCLE   REPOSITORY
porch-demo.db-repo.network-function.v6.v6   network-function   v6              v6         false    Draft       db-repo

> kubectl get packagerevisionresources.porch.kpt.dev -n porch-demo porch-demo.db-repo.network-function.v6.v6
NAME                                        PACKAGE            WORKSPACENAME   REVISION   REPOSITORY   FILES
porch-demo.db-repo.network-function.v6.v6   network-function   v6              v6         db-repo      4
>  k get packagerevisionresources.porch.kpt.dev -n porch-demo porch-demo.db-repo.network-function.v6.v6 -oyaml
apiVersion: porch.kpt.dev/v1alpha1
kind: PackageRevisionResources
metadata:
  creationTimestamp: "2024-10-14T19:18:18Z"
  name: porch-demo.db-repo.network-function.v6.v6
  namespace: porch-demo
  resourceVersion: porch-demo.db-repo.network-function.v6.v6.1728933498
  uid: db6f4b8c-1c51-59e1-a1a3-4a735561ae4a
spec:
  packageName: network-function
  repository: db-repo
  resources:
    Kptfile: |
      apiVersion: kpt.dev/v1
      kind: Kptfile
      metadata:
        name: network-function
        annotations:
          config.kubernetes.io/local-config: "true"
      info:
        description: network function blueprint
    README.md: |
      # Network Function

      ## Description
      Network Function Blueprint

      ## Usage

      ### Fetch the package
      ```
      kpt pkg get $GIT_HOST/$GIT_USERNAME/$GIT_BLUEPRINTS_REPO[@VERSION] network-function
      ```
      Details: https://kpt.dev/reference/cli/pkg/get/

      ### View package content
      ```
      kpt pkg tree network-function
      ```
      Details: https://kpt.dev/reference/cli/pkg/tree/

      ### Apply the package
      ```
      kpt live init network-function
      kpt live apply network-function --reconcile-timeout=2m --output=table
      ```
      Details: https://kpt.dev/reference/cli/live/
    deployment.yaml: |
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: network-function
      spec:
        replicas: 1
        selector:
          matchLabels:
            app.kubernetes.io/name: network-function
        template:
          metadata:
            labels:
              app.kubernetes.io/name: network-function
          spec:
            containers:
            - name: nginx
              image: nginx:latest
    package-context.yaml: |
      apiVersion: v1
      kind: ConfigMap
      metadata:
        name: kptfile.kpt.dev
        annotations:
          config.kubernetes.io/local-config: "true"
      data:
        name: example
  revision: v6
  workspaceName: v6
status:
  renderStatus:
    error: ""
    result:
      exitCode: 0
      metadata:
        creationTimestamp: null
```

Examine the DB contents:

```
porch=# select * from repositories;
 namespace  | repo_name |           updated            | updatedby | deployment 
------------+-----------+------------------------------+-----------+------------
 porch-demo | db-repo   | 2024-10-14 19:18:18.85215+00 | nonroot   | f
(1 row)

porch=# select * from packages;
 namespace  | repo_name |   package_name   |            updated            | updatedby 
------------+-----------+------------------+-------------------------------+-----------
 porch-demo | db-repo   | network-function | 2024-10-14 19:14:07.831024+00 | nonroot
(1 row)

porch=# select * from package_revisions;
 namespace  | repo_name |   package_name   | package_rev | workspace_name |           updated            | updatedby | lifecycle |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             resources                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              
------------+-----------+------------------+-------------+----------------+------------------------------+-----------+-----------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 porch-demo | db-repo   | network-function | v6          | v6             | 2024-10-14 19:18:18.85215+00 | nonroot   | Draft     | \x4b707466696c653a207c0a202061706956657273696f6e3a206b70742e6465762f76310a20206b696e643a204b707466696c650a20206d657461646174613a0a202020206e616d653a206e6574776f726b2d66756e6374696f6e0a20202020616e6e6f746174696f6e733a0a202020202020636f6e6669672e6b756265726e657465732e696f2f6c6f63616c2d636f6e6669673a202274727565220a2020696e666f3a0a202020206465736372697074696f6e3a206e6574776f726b2066756e6374696f6e20626c75657072696e740a524541444d452e6d643a207c0a202023204e6574776f726b2046756e6374696f6e0a0a20202323204465736372697074696f6e0a20204e6574776f726b2046756e6374696f6e20426c75657072696e740a0a202023232055736167650a0a202023232320466574636820746865207061636b6167650a20206060600a20206b707420706b672067657420244749545f484f53542f244749545f555345524e414d452f244749545f424c55455052494e54535f5245504f5b4056455253494f4e5d206e6574776f726b2d66756e6374696f6e0a20206060600a202044657461696c733a2068747470733a2f2f6b70742e6465762f7265666572656e63652f636c692f706b672f6765742f0a0a20202323232056696577207061636b61676520636f6e74656e740a20206060600a20206b707420706b672074726565206e6574776f726b2d66756e6374696f6e0a20206060600a202044657461696c733a2068747470733a2f2f6b70742e6465762f7265666572656e63652f636c692f706b672f747265652f0a0a2020232323204170706c7920746865207061636b6167650a20206060600a20206b7074206c69766520696e6974206e6574776f726b2d66756e6374696f6e0a20206b7074206c697665206170706c79206e6574776f726b2d66756e6374696f6e202d2d7265636f6e63696c652d74696d656f75743d326d202d2d6f75747075743d7461626c650a20206060600a202044657461696c733a2068747470733a2f2f6b70742e6465762f7265666572656e63652f636c692f6c6976652f0a6465706c6f796d656e742e79616d6c3a207c0a202061706956657273696f6e3a20617070732f76310a20206b696e643a204465706c6f796d656e740a20206d657461646174613a0a202020206e616d653a206e6574776f726b2d66756e6374696f6e0a2020737065633a0a202020207265706c696361733a20310a2020202073656c6563746f723a0a2020202020206d617463684c6162656c733a0a20202020202020206170702e6b756265726e657465732e696f2f6e616d653a206e6574776f726b2d66756e6374696f6e0a2020202074656d706c6174653a0a2020202020206d657461646174613a0a20202020202020206c6162656c733a0a202020202020202020206170702e6b756265726e657465732e696f2f6e616d653a206e6574776f726b2d66756e6374696f6e0a202020202020737065633a0a2020202020202020636f6e7461696e6572733a0a20202020202020202d206e616d653a206e67696e780a20202020202020202020696d6167653a206e67696e783a6c61746573740a7061636b6167652d636f6e746578742e79616d6c3a207c0a202061706956657273696f6e3a2076310a20206b696e643a20436f6e6669674d61700a20206d657461646174613a0a202020206e616d653a206b707466696c652e6b70742e6465760a20202020616e6e6f746174696f6e733a0a202020202020636f6e6669672e6b756265726e657465732e696f2f6c6f63616c2d636f6e6669673a202274727565220a2020646174613a0a202020206e616d653a206578616d706c650a
(1 row)

porch=# 

```

Run through the rest of the package lifecycle:
```
>  porchctl rpkg propose -n porch-demo porch-demo.db-repo.network-function.v6.v6
porch-demo.db-repo.network-function.v6.v6 proposed

>  porchctl rpkg get -n porch-demo porch-demo.db-repo.network-function.v6.v6                           
NAME                                        PACKAGE            WORKSPACENAME   REVISION   LATEST   LIFECYCLE   REPOSITORY
porch-demo.db-repo.network-function.v6.v6   network-function   v6              v6         false    Proposed    db-repo

>  porchctl rpkg approve -n porch-demo porch-demo.db-repo.network-function.v6.v6
porch-demo.db-repo.network-function.v6.v6 approved

>  porchctl rpkg get -n porch-demo porch-demo.db-repo.network-function.
v6.v6    
NAME                                        PACKAGE            WORKSPACENAME   REVISION   LATEST   LIFECYCLE   REPOSITORY
porch-demo.db-repo.network-function.v6.v6   network-function   v6              v6         true     Published   db-repo

>  porchctl rpkg propose-delete -n porch-demo porch-demo.db-repo.network-function.v6.v6
porch-demo.db-repo.network-function.v6.v6 proposed for deletion

>  porchctl rpkg get -n porch-demo porch-demo.db-repo.network-function.v6.v6   
NAME                                        PACKAGE            WORKSPACENAME   REVISION   LATEST   LIFECYCLE          REPOSITORY
porch-demo.db-repo.network-function.v6.v6   network-function   v6              v6         true     DeletionProposed   db-repo

>  porchctl rpkg delete -n porch-demo porch-demo.db-repo.network-function.v6.v6 
porch-demo.db-repo.network-function.v6.v6 deleted

>  porchctl rpkg get -n porch-demo porch-demo.db-repo.network-function.v6.v6   
Error: the server could not find the requested resource (get packagerevisions.porch.kpt.dev porch-demo.db-repo.network-function.v6.v6) 

```

Check the package is gone from the DB:
```
porch=# select * from packages;
 namespace | repo_name | package_name | updated | updatedby 
-----------+-----------+--------------+---------+-----------
(0 rows)

porch=# select * from package_revisions;
 namespace | repo_name | package_name | package_rev | workspace_name | updated | updatedby | lifecycle | resources 
-----------+-----------+--------------+-------------+----------------+---------+-----------+-----------+-----------
(0 rows)

porch=# 
```

## Delete the repository

```
>kubectl delete -f examples/tutorials/database-repo/db_repo.yaml 
namespace "porch-demo" deleted
repository.config.porch.kpt.dev "db-repo" deleted

```

Check the repo is erased in the DB:
```
porch=# select * from repositories;
 namespace | repo_name | updated | updatedby | deployment 
-----------+-----------+---------+-----------+------------
(0 rows)

porch=# 
```
