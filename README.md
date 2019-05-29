# README - matrix-synapse

Documentation and notes on how to install the matrix-synapse app

Synapse is a homeserver implementation of the Matrix communcation open standard

__TLDR:__
* Install `docker`, `docker-compose`
* Edit `traefik.toml`, change: 
  * `domain = example.com`
  * `email = "user@example.com"`
* `touch acme.json && chmod 600 acme.json`
* Update DNS for LetsEncrypt to generate certificate (required if using acme `httpChallenge`)


## Requirements

* `docker`: [Get Docker CE for Debian](https://docs.docker.com/install/linux/docker-ce/debian/)
* `docker-compose`: [Install Docker Compose](https://docs.docker.com/compose/install/)

__NOTE: Make sure `docker-compose` version is >=1.24__


## Install

It is easiest to deploy the Synapse homeserver through `docker-compose`.  This document will focus on 
that deployment.

The container is pulled from the [Docker Hub](https://hub.docker.com/r/matrixdotorg/synapse/) and
can be pulled manually: `docker pull matrixdotorg/synapse`.

## Run

### With `docker-compose` (Recommended)

It is recommended to use `docker-compose` to run the container and use a Postgres server for backend storage.

Synapse provides a sample [`docker-compose.yml`](https://github.com/matrix-org/synapse/blob/master/contrib/docker/docker-compose.yml) and [README](https://github.com/matrix-org/synapse/blob/master/contrib/docker/README.md).

The included [docker-compose.yaml](./docker-compose.yaml) will set up the following containers:  

* synapse
* db (postgres) (Optional)
* traefik (Optional)

While the `synapse` container is required, the database server is optional as synapse will store to a local
sqlite database by default.

Traefik is used as a frontend reverse proxy and requires some additional set up to start.

### Setup - Traefik

Traefik requires some additional set to function.  

* Update DNS for Traefik to generate a certificate.  
* Create an empty file called `acme.json` which is used to store certificate information.  Set restrictive permissions on this file.  
  * `touch acme.json && chmod 600 acme.json`.  
* Update `traefik.toml` and change the following sections:  

  ```toml
  [docker]
  domain = "example.com"                # Replace this with your domain.
  #...
  [acme]
  email = "user@example.com"            # Required - change to valid email
  ```

### Setup - Postgres

Postgres is the chosen database server and requires minimal create the database and user on it's initial run.
Postgres configuration can be flushed using `docker volume prune -f` or `docker volume prune <volume name>`.
Once done this the database and initial setup will be re-run on container creation.

### Setup - Synapse

Synapse homeserver can be configured to run using a dynamice or static configuration file (`homeserver.yaml`).
Using a generated configuration is the simplest way to get up and running quickly and is included in this example.
Ensure that the environment variable: `- SYNAPSE_SERVER_NAME=${MATRIX_HOSTNAME}` is set in `docker-compose.yaml`

#### Direct Port Forwarding
Direct port mapping is fine for testing, but it is highly recommended to use a reverse proxy in front of 
the synapse homeserver.  To enable direct port forwarding, make sure to comment out the `ports` section
of the `synapse` container definition in `docker-compose.yaml`.

#### Reverse Proxy
The included `docker-compose.yaml` is set to run a traefik reverse proxy.  This should work with minimal changes.


### (Re)starting Services

Run the following to destroy the containers and rebuild them.  An optional command is used to remove the volumes which resets any storage data.

```
docker-compose down && docker-compose up -d
docker-compose down && docker volumes prune -f && docker-compose up -d
```


## Advanced Setup

Synapse can generate a configuration file that can then be edited and used in subsequent runs.
To generate the config file using `docker-compose`: `docker-compose run --rm -e SYNAPSE_SERVER_NAME=my.matrix.host synapse generate`

### Backing up Docker Volume Files

This generates a config in the docker volume path (`/var/lib/docker/volumes`).  These files can be copied or edited in place manually but this would require root privliege.  To do this a `docker` user it is recommended to build a dummy container, mount the volume and copy the files out.

Create a Dockerfile.

```Dockerfile
FROM scratch
CMD
```

Build the image, mount the volume and copy the data out.

```bash
docker build -t nothing .
docker container create --name dummy -v synapse-config:/data nothing
docker cp  dummy:/data ./data
docker rm dummy
```


### With `docker run`

This has __not__ been tested by the author of this document but the [Docker Hub](https://hub.docker.com/r/matrixdotorg/synapse/) Synapse documentation includes the following run command.

```bash
docker run \
    -d \
    --name synapse \
    --mount type=volume,src=synapse-data,dst=/data \
    -e SYNAPSE_SERVER_NAME=my.matrix.host \
    -e SYNAPSE_REPORT_STATS=yes \
    matrixdotorg/synapse:latest
```

#### Generate a Config File

`docker run` can be used to generate a default `homeserver.yaml` config and signing keys.  Running the command below will generate the following files: 
  * `homeserver.yaml`
  * `my.matrix.host.config`
  * `my.matrix.host.signing.key`  

These files are stored in the docker volume `synapse-data`.  See _Backing up Docker Volume Files_ in the Notes section.

```
docker run -it --rm \
    --mount type=volume,src=synapse-data,dst=/data \
    --mount type=volume,src=synapse-config,dst=/config \
    -e SYNAPSE_CONFIG_PATH=/config/homeserver.yaml \
    -e SYNAPSE_SERVER_NAME=my.matrix.host \
    -e SYNAPSE_REPORT_STATS=yes \
    matrixdotorg/synapse:latest generate
```

#### Generate Self-signed Certificate (Testing only)

Synapse requires a valid TLS certificate, this describes a short lived test certificate, see the Synapse [README](https://github.com/matrix-org/synapse/blob/master/contrib/docker/README.md) for other options.

Create the certificate and move it to a directory to be mounted as a volume in `docker-compose.yaml` (i.e - `./certs:/certs`)

```bash
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 30
```


#### Logging

`synapse` logs to `/homeserver.log`.  If storing this file on the host, create it with the owner uid as `991`.


#### Start container

It should now be possible to start the Synapse and Postgres containers by running `docker-compose up -d`.  

A sample directory tree could look like: 

```bash
user@server:~/matrix-synapse$ tree
.
├── certs
│   ├── cert.pem
│   └── key.pem
├── config
│   ├── homeserver.yaml
│   ├── homeserver.yaml.original
│   ├── my.matrix.host.log.config
│   └── my.matrix.host.signing.key
├── docker-compose.yaml
├── logs
│   └── homeserver.log
└── README.md

```

If the `synapse` container fails to start, investigatee the logs:  
```bash
docker logs -f synapse
```

