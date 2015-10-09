# dockerize-lc
How to dockerize a Linked Connections server

## Overview

LC (`Linked Connections`) are a type of Linked Data Fragments and are used for `clientside` routeplanning. This way a [LC server](https://github.com/linkedconnections/server.js) is just a dataprovider of Linked Connections documents. The [client](https://github.com/linkedconnections/client.js) does all the weight lifting by requesting connections and calculating the Minimum Spanning Tree to get from point A to B.

To research intermodality, we need to setup different endpoints. By using [Docker](https://www.docker.com/), we can put every transit company into it's own container. More specifically, we'll make two containers: one for the Node Express web application and one that contains the data in MongoDB.

In this tutorial I'll explain how to setup one endpoint. In this case, I'll make a LC server for the Dutch NS transit company: http://www.gtfs-data-exchange.com/agency/ns-hispeed/

## Docker

Make sure to have [Docker](https://docs.docker.com/installation/) installed and running.

### First things first

We'll need to generate Linked Connections from the GTFS. Follow the instructions in [this](https://github.com/brechtvdv/gtfs2connections) project. 
Feel free to make an issue if you have a question or issue.

### Setup database container

0) Make a seperate directory for the project and put the generated connections.jsonldstream inside it.
In this case, I'll name the directory `ns`.

1) Make a new file with your favorite texteditor
```vim Dockerfile``` 
and put following content in it:
```
# Dockerizing MongoDB: Dockerfile for building MongoDB images
# Based on ubuntu:latest, installs MongoDB following the instructions from:
# http://docs.mongodb.org/manual/tutorial/install-mongodb-on-ubuntu/
# Format: FROM    repository[:version]
FROM       ubuntu:latest
# Format: MAINTAINER Name <email@addr.ess>
MAINTAINER brechtvdv <brecht@irail.be>
# Installation:
# Import MongoDB public GPG key AND create a MongoDB list file
RUN apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 7F0CEB10
RUN echo 'deb http://downloads-distro.mongodb.org/repo/ubuntu-upstart dist 10gen' | tee /etc/apt/sources.list.d/10gen.list

# Update apt-get sources AND install MongoDB
RUN apt-get update
RUN apt-get install -y mongodb-org

# Create the MongoDB data directory
RUN mkdir -p /data/db
# Expose port 27017 from the container to the host
EXPOSE 27017
# Set usr/bin/mongod as the dockerized entry-point application
ENTRYPOINT usr/bin/mongod
```
Press `:wq` to save and quite.

2) Now we can build the Docker image with the following command:
`docker build --tag [your-accountname]/[name-image] .`

In my case: `docker build --tag brechtvdv/mongons .`

Check if you can see your image in the Docker list: `docker images`

3) Run following command to run the image inside a container:
`docker run -p [port]:27017 --name [name-db-container] [your-accountname]/[name-image]`

In my case: `docker run -p 28001:27017 --name mongonscontainer brechtvdv/mongons`

With `-p [port]:27017`, we open port 27017 of the Mongo container to port `[port]` of the host. This way, we can connect to the container and load the connections.

If you did something wrong, use these commands:
`docker rm -f CONTAINER` to remove the container,
`docker rmi -f IMAGE` to remove the image.

4) To import connections into the MongoDB container, we'll need to know on which host Docker is running.
For MAC OS X and Windows users, Docker is running inside it's own virtual machine. You can see it's IP adress when you start the `Docker Quick start terminal` or when you type `ifconfig`/`ipconfig`.
For the happy Linux users, Docker runs directly on your localhost so you can just use `localhost` or `0.0.0.0`

Run following command to import the data and convert datetimes to ISO8601:
```bash
mongoimport --db lc --collection connections --file [connections-filename].jsonldstream --host [docker-hostname] --port [port]
```

In my case: 
```bash
mongoimport --db lc --collection connections --file connections-100-october.jsonldstream --host 192.168.99.100 --port 28001
```

```bash
mongo lc --host [docker-hostname] --port [port] --eval 'db.connections.find({"@context": { "$exists": false}}).forEach(function(conn){conn["arrivalTime"] = new ISODate(conn["arrivalTime"]);conn["departureTime"] = new ISODate(conn["departureTime"]);db.connections.save(conn)});'
```

In my case: 
```bash
mongo lc --host 192.168.99.100 --port 28001 --eval 'db.connections.find({"@context": { "$exists": false}}).forEach(function(conn){conn["arrivalTime"] = new ISODate(conn["arrivalTime"]);conn["departureTime"] = new ISODate(conn["departureTime"]);db.connections.save(conn)});'
```

### Setup Node container

1) Clone server.js into the directory:
```bash
git clone https://github.com/linkedconnections/server.js
```

2) Go into the server.js directory and make a new config.json file: 

```bash
cp config-example.json config.json
```

Change the configuration in following form:
```json
{
  "source" : {
    "type" : "mongodb",
    "string" : "mongodb://[name-database-container-step-3]:27017/lc",
    "collections" : {
      "connections" : "connections",
      "stops" : "stations"
    }
  },
  "port" : 8080,
  "proxy" : {
    "port" : 80,
    "hostname" : "[custom-hostname-for-development]:[custom-port]",
    "protocol" : "http"
  }
}
```

In my case: 
```json
{
  "source" : {
    "type" : "mongodb",
    "string" : "mongodb://mongonscontainer:27017/lc",
    "collections" : {
      "connections" : "connections",
      "stops" : "stations"
    }
  },
  "port" : 8080,
  "proxy" : {
    "port" : 80,
    "hostname" : "ns.linkedconnections.dev:32777",
    "protocol" : "http"
  }
}
```

3) Make another `Dockerfile` with the following content: 
```
FROM node:0.10-onbuild

EXPOSE 8080
```

4) Build the image:
```bash
docker build --tag [your-accountname]/[name-image] .
```

In my case: 
```bash
docker build --tag brechtvdv/nodens .
```

5) Run the Node image inside a container. To make sure that the web application can get data from our previous database container, we need to link them with `--link`:

```bash
docker run -p [random-port]:8080 [-d] --name [name-container-server] --link [name-db-container] [your-accountname]/[name-image]
```

In my case:
```bash
docker run -p 32777:8080 [-d] --name nodenscontainer --link mongonscontainer brechtvdv/nodens
```

`-p` links listening port '8080' of the container with port '32777' of the Docker host.
Add `-d` to run the container in background. For developement, don't use `-d` to see if everything works as expected.

6) Add host to `/etc/hosts` on your local machine:
```bash
sudo vim /etc/hosts
```

```
[docker-host-ip-adress] [custom-hostname-for-development]
```

In my case:
```
192.168.99.100 ns.linkedconnections.dev
```

7) Done! You can now retrieve Linked Connections within your browser, e.g.:
```
http://ns.linkedconnections.dev:32777/connections/?departureTime=2015-10-01T14:20
```

## Next step

Now that you have setup your own Linked Connection server, you can start experimenting with the Linked Connection [client](https://github.com/linkedconnections/client.js) by adding your endpoint to the configuration file.












