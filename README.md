# README - matrix-synapse

Documentation and notes on how to install the matrix-synapse app

Synapse is a homeserver implementation of the Matrix communcation open standard

## Install

### Docker

Registered on [Docker Hub](https://hub.docker.com/r/matrixdotorg/synapse/)

* Pull: `docker pull matrixdotorg/synapse`

## Run

### With `docker-compose` (Recommended)

It is recommended to use `docker-compose` to run the container and use a Postgres server for backend storage

Synapse provides a sample [`docker-compose.yml`](https://github.com/matrix-org/synapse/blob/master/contrib/docker/docker-compose.yml) and [README](https://github.com/matrix-org/synapse/blob/master/contrib/docker/README.md).

A slightly modified docker-compose.yml file is included.  Use this to get started and run `docker-compose up -d`.

#### (Re)starting Services

Run the following to destroy the containers and rebuild them.  An optional command is used to remove the volumes which resets any storage data.

```
docker-compose down && docker-compose up -d
docker-compose down && docker volumes prune -f && docker-compose up -d
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

#### Edit Config

Included is an edited `homeserver.yaml` file.  Some notes if generated using the command above

* In the config file `my.matrix.host.log.config` check and set `filename: /logs/homeserver.log`
  * Create the `logs` dir (or touch a file) and make sure it is writable by `991`.
  * Update `docker-compose.yaml` appropriately.


#### Generate Self-signed Certificate (Testing only)

Synapse requires a valid TLS certificate, this describes a short lived test certificate, see the Synapse [README](https://github.com/matrix-org/synapse/blob/master/contrib/docker/README.md) for other options.

Create the certificate and move it to a directory to be mounted as a volume in `docker-compose.yaml` (i.e - `./certs:/certs`)

```bash
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 30

# without prompts
openssl req -nodes -new -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 30 \
-subj "/C=CA/ST=Ontario/L=Toronto/O=Company Name/OU=Org/CN=my.matrix.host"

```

----------------------------------------

> __TODO: Investigate and fix this, probably fixed with proper signed certs.__
> 
> The key file generated must be readable from inside the container.  Set the key ownership to `991`.
> 
> ```bash
> mkdir certs
> chown 991 ./key.pem
> mv key.pem cert.pem certs/
> ```

----------------------------------------


#### Logging

`synapse` logs to `/homeserver.log`.  Create this file and ensure that it is writable by the container user (`991`).

```bash
mkdir -p ./logs
chown 991  -R ./logs
```

__TODO__

The `homeserver.log` file must be mounted, currently there seem to be permission errors using a docker storage mount and it must be connected to a 


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
docker logs synapse
```


### Optional `docker-compose` generate config

```
docker-compose run --rm -e SYNAPSE_SERVER_NAME=my.matrix.host synapse generate
```

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

