# docker-compose

## Description
A collection of [**Docker Compose**](https://docs.docker.com/compose/) files to configure various application services using Docker [containers](https://www.docker.com/products/container-runtime).

## Disclaimer
All the compose files included in this repository is the author's personal preference to deploy application services on a Docker host system. This includes resource reservations and security settings specific for the Docker host. It does not suite all use cases and is subject to debate and changes in the future.

## Usage
Deploy a specific application service using the `docker-compose -f docker-compose.yml --compatibility up -d` command.

To also stream the log output directly to the command-line of a starting service, deploy with the `docker-compose -f docker-compose.yml --compatibility up -d; docker-compose logs -tf --tail="50" <container name>` command.

## Networking
Prior to application service deployment, the correct type of host networking must be configured. Many containers included in this reposirory are using the [**macvlan**](https://docs.docker.com/network/macvlan/) network driver. This is to enable network connectivity between host and the running container. Hence a **macvlan interface** must be created on the host system.

### Create macvlan Interface
Use the `create-macvlan.sh` [script](https://github.com/pwyde/create-macvlan) on the Docker host system and create a macvlan interface named `docker_subnet` with command below.

```
# bash create-macvlan.sh --macvlan docker_subnet --link <parent interface name> --ip-address <host ipv4 address> --network <ipv4 network block>
```

See the script [README](https://github.com/pwyde/create-macvlan/blob/master/README.md) file for more information.

Also use the provided [systemd service unit](https://github.com/pwyde/create-macvlan#execute-as-a-systemd-service-unit) to make the macvlan interface persistent between reboots.

### Create Docker Network
Edit the provided compose [file](docker_subnet/docker-compose.yml) which will deploy a Docker network with the [macvlan](https://docs.docker.com/v17.09/engine/userguide/networking/get-started-macvlan/) network driver.

```yaml
version: "2.2"
networks:
  docker_subnet:
    name: docker_subnet
    driver: macvlan
    driver_opts:
      parent: <parent interface name>
    ipam:
      config:
        - subnet: <ipv4 network>/24
          ip_range: <ipv4 network block>
          gateway: <gateway ipv4 address>
          aux_addresses:
            host: <host ipv4 address>
services:
  scratch:
    image: scratch
    networks:
      - docker_subnet
```

Deploy using the `docker-compose -f docker-compose.yml up --no-start` command.

## Reverse Proxy
[**Traefik**](https://docs.traefik.io/) is used as a [reverse proxy](https://en.wikipedia.org/wiki/Reverse_proxy) for all [HTTP](https://en.wikipedia.org/wiki/Hypertext_Transfer_Protocol)/[HTTPS](https://en.wikipedia.org/wiki/HTTPS)-based application services. See the official [documentation](https://docs.traefik.io/v1.7/) for more information.

## Variables
Variable are used in many of the Docker compose files and must be set using [environment variables](https://wiki.archlinux.org/index.php/Environment_variables) or a [`.env`](https://docs.docker.com/compose/env-file/) file, which is sourced by the `docker-compose` command during execution.

See the individual compose files for which variables are used and configure accordingly.

## License
All files containing in this repository are licensed under [**The Unlicense**](https://unlicense.org/) license.

A license with no conditions whatsoever which dedicates works to the public domain. Unlicensed works, modifications, and larger works may be distributed under different terms and without source code. See the [LICENSE](LICENSE) file for more information.