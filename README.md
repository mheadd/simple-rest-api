# Simple REST API in 3 Easy Steps

A powerful, simple, RESTful API generated from a CSV or text document in three easy steps.

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
~$ curl -s "http://data.louisvilleky.gov/sites/default/files/temp_import/FoodServiceData.txt" > inspections.csv
```
Check to make sure the data is [valid CSV](http://csvkit.readthedocs.org/en/0.9.1/scripts/csvclean.html) and then inset it into our Postgres instance from step 1 above using ```csvkit```.

```bash
~$ csvsql --db postgres://simpleapi:simpleapi@{docker-container-ip}:5432/simpleapi --insert inspections.csv
```

Now, you can access the data at: ```http://{docker-container-ip}:3000/```

You can query the data using the [PostgREST query API](http://postgrest.com/api/reading/), like so:

```bash
~$ curl -s "http://192.168.99.100:3000/inspections?Grade=eq.A" -H 'Range-Unit: items' -H 'Range: 0-1' | jq .
```

```json
[
  {
    "Intersection": null,
    "State": "KY",
    "City": "LOUISVILLE",
    "Address2": null,
    "Address": "MOBILE FOOD UNIT",
    "PlaceName": null,
    "EstablishmentName": "FUNNEL CAKE #5",
    "InspectionID": 1104831,
    "EstablishmentID": 27867,
    "Zip": 40202,
    "TypeDescription": "SELF-CONTAINED MOBILE FOOD UNITS",
    "Latitude": 38.2526647,
    "Longitude": -85.7584557,
    "InspectionDate": "2015-05-24",
    "Score": 98,
    "Grade": "A",
    "NameSearch": "FUNNELCAKE"
  },
  {
    "Intersection": null,
    "State": "KY",
    "City": "LOUISVILLE",
    "Address2": null,
    "Address": "MOBILE FOOD UNIT",
    "PlaceName": null,
    "EstablishmentName": "FUNNEL CAKE #5",
    "InspectionID": 1132274,
    "EstablishmentID": 27867,
    "Zip": 40202,
    "TypeDescription": "SELF-CONTAINED MOBILE FOOD UNITS",
    "Latitude": 38.2526647,
    "Longitude": -85.7584557,
    "InspectionDate": "2015-08-25",
    "Score": 100,
    "Grade": "A",
    "NameSearch": "FUNNELCAKE"
  }
]

```
