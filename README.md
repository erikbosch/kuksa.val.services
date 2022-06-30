# Kuksa.VAL.services

- [Kuksa.VAL.services](#kuksavalservices)
  - [Overview](#overview)
  - [Contribution](#contribution)
  - [Build Seat Service Containers](#build-seat-service-containers)
  - [Running Seat Service / Databroker Containers](#running-seat-service--databroker-containers)


## Overview

The Kuksa.VAL.services repository is part of the overall Eclipse Kuksa Vehicle Abstraction Layer (VAL) set of repositories.
The VAL is offering a *Vehicle API*, which is an abstraction of vehicle data and functions to be used by *Vehicle Apps*.
Vehicle data is provided in form of a data model, which is accessible via the Vehicle Data Broker - see [Kuksa.VAL repository](https://github.com/eclipse/kuksa.val).
Vehicle functions are made available by a set of so-called *vehicle services* (short: *vservice*).
This repository contains examples of vservices and their implementations to show, how a Vehicle API and the underlying abstraction layer could be realized.

It currently consists of
* a simple example [HVAC service (Python)](./hvac_service) and
* a more complex example [seat control service (C++)](./seat_service).
  
More elaborate or completely differing implementations are target of "real world grown" projects.


## Contribution

For contribution guidelines see [CONTRIBUTING.md](CONTRIBUTING.md)


## Build Seat Service Containers

From the terminal, make the seat_service as your working directory:

``` bash
cd seat_service
```

When you are inside the seat_service directory, create binaries:

``` bash
./build-release.sh x86_64

#Use following commands for aarch64
./build-release.sh aarch64
```
Build a tar file of all binaries.
``` bash
#Replace x86_64 with aarch64 for arm64 architecture
tar -czvf bin_vservice-seat_x86_64_release.tar.gz \
    target/x86_64/release/install/ \
    target/x86_64/release/licenses/ \
    proto/
```
To build the image execute following commands from root directory as context.
``` bash
docker build -f seat_service/Dockerfile -t seat_service:<tag> .

#Use following command if buildplatform is required
DOCKER_BUILDKIT=1 docker build -f seat_service/Dockerfile -t seat_service:<tag> .
```
The image creation may take around 2 minutes.

## Running Seat Service / Databroker Containers

To directly run the containers following commands can be used:

1. Seat Service container

   By default the container will execute the `./val_start.sh` script, that sets default environment variables for seat service.
   It needs `CAN` environment variable with special value `cansim` (to use simulated socketcan calls) or any valid can device within the container.

    ``` bash
    # executes ./va_start.sh
    docker run --rm -it -p 50051:50051/tcp seat-service
    ```

    To run any specific command in the container, just append you command (e.g. bash) at the end.

    ``` bash
    docker run --rm -it -p 50051:50051/tcp seat-service <command>
    ```


For accessing databroker from seat service container there are two ways of running the containers.

1. The simplest way to run the containers is to sacrifice the isolation a bit and run all the containers in the host's network namespace with <i>docker run --network host</i>

    ``` bash
    #By default the container will execute the ./vehicle-data-broker command as entrypoint.
    docker run --rm -it --network host -e 'RUST_LOG=info,vehicle_data_broker=debug' databroker
    ```

    ``` bash
    #By default the container will execute the ./val_start.sh command as entrypoint
    docker run --rm -it --network host seat-service
    ```

1. There is a more subtle way to share a single network namespace between multiple containers.
   So, we can start a sandbox container that will do nothing but sleep and reusing a network namespace of an this existing container:

    ``` bash
    #Run sandbox container
    docker run -d --rm --name sandbox -p 55555:55555 alpine sleep infinity
    ```

    ``` bash
    #Run databroker container
    docker run --rm -it --network container:sandbox -e HOST=0.0.0.0 -e PORT=55555 databroker
    ```

    ``` bash
    #Run seat-service container
    docker run --rm -it --network container:sandbox -e HOST=0.0.0.0 -e PORT=55555 -e PORT=50051  seat-service
    ```

1. Another option is to use `<container-name>:<port>` and bind to `0.0.0.0` inside containers

