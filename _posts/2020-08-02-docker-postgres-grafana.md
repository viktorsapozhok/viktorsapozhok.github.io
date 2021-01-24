---
layout: post
title: "Multi-container Docker app with Postgres, InfluxDB and Grafana. 
    Building animated maps with GeoLoop Panel plugin."
slug: docker-postgres-grafana
description: How to build docker with Postgres, InfluxDB and Grafana, create map overlays with Worldmap Panel, build animated maps using GeoLoop Panel. 
keywords: postgres postgresql docker influxdb grafana worldmap geoloop geojson
---

This tutorial provides a quick guide of how to install a dashboard environment
from Grafana, PostgreSQL and InfluxDB with docker-compose, create map overlays with Worldmap Panel plugin and
build animated maps using GeoLoop Panel plugin.

To illustrate the process of building the animated maps with GeoLoop,
we will use time series data tracking the number of people affected by COVID-19 worldwide,
including confirmed cases of Coronavirus infection, the number of people died while
sick with Coronavirus, and the number of people recovered from it.

The data is borrowed from "covid-19" [dataset][1] and stored as csv files in "data" directory.

<img src="https://github.com/viktorsapozhok/docker-postgres-influxdb-grafana/blob/master/docs/source/images/dashboard.gif?raw=true">

## Quick Start

To start the application, install [docker-compose][2] on the host, clone this repo 
and run docker-compose from the [docker](https://github.com/viktorsapozhok/docker-postgres-influxdb-grafana/tree/master/docker)
directory.

```
$ cd docker && docker-compose up -d

Starting influx   ... done
Starting postgres ... done
Starting grafana  ... done

$ docker ps

CONTAINER ID        IMAGE                       PORTS                    NAMES
e00c0bd0d4a5        grafana/grafana:latest      0.0.0.0:3000->3000/tcp   grafana
2b33999d97b3        influxdb:latest             0.0.0.0:8086->8086/tcp   influx
c4453cac69eb        postgres:latest             0.0.0.0:5433->5432/tcp   postgres
```

Three containers have been created and started. For the app services we expose the following ports:
3000 (grafana), 8086 (influx), 5432 (postgres). Note that we use `HOST:CONTAINER` mapping when specifying
postgres ports in docker-compose file. Inside a docker container, postgres is running on port `5432`,
whereas the publicly exposed port outside the container is `5433`.

```yaml
version: '3'

services:
  postgres:
    ports:
      - "5433:5432"
```

Before we login to Grafana UI, we need to create PostgreSQL database. Note that we have already
created InfluxDB database specifying `INFLUXDB_DB` environment variable in docker-compose file.

To create postgres database we use [psql][3], postgres terminal, inside the docker container. 
See [Makefile](https://github.com/viktorsapozhok/docker-postgres-influxdb-grafana/tree/master/docker/Makefile)
for more details.

```
$ make postgres-create-db
$ make postgres-init-schema
```

Now we can login to the Grafana web UI in browser (http://localhost:3000/grafana/) with the login `admin` and
password `password` and initialize data sources.

<a href="https://github.com/viktorsapozhok/docker-postgres-influxdb-grafana/blob/master/docs/source/images/grafana_login.png?raw=true">
    <img 
        src="https://github.com/viktorsapozhok/docker-postgres-influxdb-grafana/blob/master/docs/source/images/grafana_login.png?raw=true" 
        alt="Grafana login page"
    >
</a>

## Data Sources

Before we create a dashboard, we need to add InfluxDB and PostgreSQL data sources. Follow
[this guide][4] to find out how to do this.

Here is the configuration parameters we use to add InfluxDB data source.

<a href="https://github.com/viktorsapozhok/docker-postgres-influxdb-grafana/blob/master/docs/source/images/influx.png?raw=true">
    <img 
        src="https://github.com/viktorsapozhok/docker-postgres-influxdb-grafana/blob/master/docs/source/images/influx.png?raw=true" 
        alt="InfluxDB datasource configuration"
    >
</a>

And this is the configuration parameters we use to add PostgreSQL data source.

<a href="https://github.com/viktorsapozhok/docker-postgres-influxdb-grafana/blob/master/docs/source/images/postgres.png?raw=true">
    <img 
        src="https://github.com/viktorsapozhok/docker-postgres-influxdb-grafana/blob/master/docs/source/images/postgres.png?raw=true" 
        alt="PostgreSQL data source"
    >
</a>

The valid password for both data sources is `password`. You can change the credentials in
[docker/.env](https://github.com/viktorsapozhok/docker-postgres-influxdb-grafana/tree/master/docker/.env)
file before starting the service via `docker-compose up`.

## Populating the Database

To illustrate the process of building the animated map with GeoLoop Panel plugin, we will use time series data
tracking the number of people affected by COVID-19 worldwide, including confirmed cases of Coronavirus infection,
the number of people died while sick with Coronavirus, the number of people recovered from it.

We borrowed data from "covid-19" [dataset][1] and store it as csv files in 
[data](https://github.com/viktorsapozhok/docker-postgres-influxdb-grafana/tree/master/data) directory:

* [countries-aggregated.csv](https://github.com/viktorsapozhok/docker-postgres-influxdb-grafana/tree/master/data/countries-aggregated.csv) - cumulative cases (confirmed, recovered, deaths) across the globe.
* [us_confirmed.csv](https://github.com/viktorsapozhok/docker-postgres-influxdb-grafana/tree/master/data/us_confirmed.csv) - cumulative confirmed cases for US regions.
* [reference.csv](https://github.com/viktorsapozhok/docker-postgres-influxdb-grafana/tree/master/data/reference.csv) - regions metadata, location, names.

For writing this data to postgres tables we use `COPY FROM` command available via postgres console.
See [Makefile](https://github.com/viktorsapozhok/docker-postgres-influxdb-grafana/tree/master/docker/Makefile)
for more details.

```    
$ make postgres-copy-data
```

After we have written data to the tables, we can login to the terminal and view schema contents.

```
$ make postgres-console

psql (12.3 (Debian 12.3-1.pgdg100+1))
Type "help" for help.

grafana=# \dt+ covid.*
                            List of relations
 Schema |         Name         | Type  |  Owner   |  Size   | Description
--------+----------------------+-------+----------+---------+-------------
 covid  | countries_aggregated | table | postgres | 1936 kB |
 covid  | countries_ref        | table | postgres | 496 kB  |
 covid  | us_confirmed         | table | postgres | 74 MB   |
(3 rows)
```

Now we calculate logarithm of the number of active cases and write it to InfluxDB database (measurement "covid").
We can also login to influx database from console and view the database contents.

```
$ make influx-console

Connected to http://localhost:8086 version 1.8.1
InfluxDB shell version: 1.8.1

> SHOW MEASUREMENTS
name: measurements
name
----
covid

> SHOW SERIES FROM covid LIMIT 5
key
---
covid,Country=Afghanistan
covid,Country=Albania
covid,Country=Algeria
covid,Country=Andorra
covid,Country=Angola
```

## Worldmap Panel

Let's visualize the number of confirmed cases across the US regions using Worldmap panel.
This panel is a tile map that can be overlaid with circles representing data points from a query.
It needs two sources of data: a location (latitude and longitude) and data that has link to a location.

The screenshot below shows query and configuration settings we used.

<a href="https://github.com/viktorsapozhok/docker-postgres-influxdb-grafana/blob/master/docs/source/images/worldmap.png?raw=true">
    <img 
        src="https://github.com/viktorsapozhok/docker-postgres-influxdb-grafana/blob/master/docs/source/images/worldmap.png?raw=true" 
        alt="Configuring Worldmap Panel"
    >
</a>

And as the result we obtain the following map.

<a href="https://github.com/viktorsapozhok/docker-postgres-influxdb-grafana/blob/master/docs/source/images/us.png?raw=true">
    <img 
        src="https://github.com/viktorsapozhok/docker-postgres-influxdb-grafana/blob/master/docs/source/images/us.png?raw=true" 
        alt="Worldmap Panel example"
    >
</a>

See Worldmap Panel plugin [documentation](https://grafana.com/grafana/plugins/grafana-worldmap-panel)
for more details.

## GeoLoop Panel

Now everything is ready to configure the GeoLoop panel and visualize Covid-19 growth rates.
Following [this tutorial](https://github.com/CitiLogics/citilogics-geoloop-panel/blob/master/README.md),
we create a [GeoJSON](https://github.com/viktorsapozhok/docker-postgres-influxdb-grafana/tree/master/data/countries.geojson)
with countries coordinates and wrap it up in a callback:

```
geo({ "type": "FeatureCollection", ... });
```

To access geojson from grafana, we need to put it on a server somewhere. In this tutorial,
we will confine ourselves to serving the local directory where geojson is stored
(however, this approach is not recommended for production).

```
$ make data-server
```

The GeoJSON URL: `http://0.0.0.0:8000/countries.geojson`

A further step is to obtain a free [MapBox API Key](https://www.mapbox.com/developers/),
the only thing is you need to create a mapbox account.

Here is the panel configuration settings.

<a href="https://github.com/viktorsapozhok/docker-postgres-influxdb-grafana/blob/master/docs/source/images/geoloop.png?raw=true">
    <img 
        src="https://github.com/viktorsapozhok/docker-postgres-influxdb-grafana/blob/master/docs/source/images/geoloop.png?raw=true" 
        alt="Configuring GeoLoop Panel"
    >
</a>

And that's how the panel looks like.

![GeoLoop Panel](https://github.com/viktorsapozhok/docker-postgres-influxdb-grafana/blob/master/docs/source/images/preview.gif?raw=true)

## Repository

All data and source codes can be found in [this][5] repository.

[https://github.com/viktorsapozhok/docker-postgres-influxdb-grafana](https://github.com/viktorsapozhok/docker-postgres-influxdb-grafana)

[1]: https://github.com/datasets/covid-19 "COVID-19 dataset"
[2]: https://docs.docker.com/compose/install
[3]: http://postgresguide.com/utilities/psql.html
[4]: https://grafana.com/docs/grafana/latest/datasources/add-a-data-source/ "Add a data source in Grafana"
[5]: https://github.com/viktorsapozhok/docker-postgres-influxdb-grafana "Docker app with postgres, influxdb and grafana"
