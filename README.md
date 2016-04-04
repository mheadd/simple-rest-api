# Simple REST API in 3 Easy Steps

A powerful, simple REST API generated from a CSV document in three easy steps.

![Build a REST API from a CSV in 3 easy steps](https://raw.githubusercontent.com/mheadd/simple-rest-api/master/rest-api-3-steps.gif "Build a REST API")

In order to follow these steps, you'll need to have [Docker](https://www.docker.com/) and [csvkit](http://csvkit.readthedocs.org/en/0.9.1/index.html) installed.

## Run Postgres container

Set up a Docker container to run the Postgres database. You can write your own ```Dockerfile``` to do this, or just pull the latest from [Docker Hub](https://hub.docker.com/).

```bash
~$ docker pull postgres
```

Now, run the container and pass in environmental variables to set the user, password and to create a database. (Note, the user created using this command will have superuser privileges but you can access container and set up a read only user if preferred. More info in the [docs](https://hub.docker.com/_/postgres/).)

```bash
~$ docker run --name postres-database \
-p 5432:5432 \
-e POSTGRES_USER=simpleapi \
-e POSTGRES_PASSWORD=simpleapi \
-e POSTGRES_DB=simpleapi \
-d postgres:latest
```

## Run PostgREST and link it to Postgres container

Set up a Docker container to run [PostgREST](http://postgrest.com/) and connect to the Postgres database container. Again, you can write your own ```Dockerfile``` for this (or use [this one](https://github.com/begriffs/postgrest/blob/master/Dockerfile)), or just pull one of the many fine PostgREST images on [Docker Hub](https://hub.docker.com/search/?isAutomated=0&isOfficial=0&page=1&pullCount=0&q=Postgrest&starCount=0).

```bash
~$ docker pull suzel/docker-postgrest
```

Now, run the container and pass in the details of the Postgres container.

```bash
~$ docker run --name postgrest-service \
-p 3000:3000 \
-e POSTGREST_VERSION=0.3.1.1 \
-e POSTGREST_DBHOST={docker-container-ip} \
-e POSTGREST_DBPORT=5432 \
-e POSTGREST_DBNAME=simpleapi \
-e POSTGREST_DBUSER=simpleapi \
-e POSTGREST_DBPASS=simpleapi \
-d suzel/docker-postgrest:latest
```

Now you should be able to see both containers running in Docker:

```bash
~$ docker ps

CONTAINER ID        IMAGE                           COMMAND                  CREATED             STATUS              PORTS                    NAMES
f667f2e68157        suzel/docker-postgrest:latest   "/bin/sh -c 'postgres"   2 minutes ago       Up 2 minutes        0.0.0.0:3000->3000/tcp   postgrest-service
f46707bbd549        postgres:latest                 "/docker-entrypoint.s"   2 minutes ago       Up 2 minutes        0.0.0.0:5432->5432/tcp   postres-database
```

## Insert Data and Test API

Now, lets get some data.

```bash
~$ curl -s "http://data.phl.opendata.arcgis.com/datasets/c6e15e5d253346049892cb19224c742c_0.csv" > complaints.csv
```
Check to make sure the data is [valid CSV](http://csvkit.readthedocs.org/en/0.9.1/scripts/csvclean.html) and then inset it into our Postgres instance from step 1 above using ```csvkit```.

```bash
~$ csvsql --db postgres://simpleapi:simpleapi@{docker-container-ip}:5432/simpleapi --insert complaints.csv
```

Now, you can access the data at: ```http://{docker-container-ip}:3000/```

You can query the data using the [PostgREST query API](http://postgrest.com/api/reading/), like so:

```bash
~$ curl -s "http://{docker-container-ip}:3000/complaints?SEX=eq.Male&RACE=eq.Black&STATUS=eq.Open" -H 'Range-Unit: items' -H 'Range: 0-1' | jq .
```

```json

[
  {
    "GlobalID": "a60c5337-6f29-4fce-8893-dfd6c2524f89",
    "LAT": 39.923172,
    "LONG_": -75.170571,
    "STATUS": "Open",
    "ACTION": "ACCEPT",
    "UNIT": "District 1",
    "﻿X": -75.1705709997512,
    "Y": 39.9231719996889,
    "OBJECTID_1": 19,
    "AGE": 53,
    "RACE": "Black",
    "SEX": "Male",
    "TYPE": "PHYSICAL ABUSE",
    "DATE_": "2010-02-01"
  },
  {
    "GlobalID": "19237a4c-8f82-4f5a-9160-317211f172a1",
    "LAT": 39.924724,
    "LONG_": -75.228861,
    "STATUS": "Open",
    "ACTION": "ACCEPT",
    "UNIT": "District 12",
    "﻿X": -75.2288609997347,
    "Y": 39.9247240002134,
    "OBJECTID_1": 30,
    "AGE": 24,
    "RACE": "Black",
    "SEX": "Male",
    "TYPE": "ABUSE OF AUTHORITY",
    "DATE_": "2009-01-23"
  }
]

```
