# docker-compose

<div align="center">
  <img src=".artwork/banner.jpg" align="center" alt="Cargo ship with containers" />
</div>

## Description
A collection of [**Docker Compose**](https://docs.docker.com/compose/) files to configure various application services using Docker [containers](https://www.docker.com/products/container-runtime).

## Disclaimer
All the compose files included in this repository is the author's personal preference to deploy application services on a single Docker host system. This includes resource reservations and security settings specific for the Docker host. It does not suite all use cases and is subject to debate and changes in the future.

## Usage
Deploy a specific container using the `docker-compose -f docker-compose.yml --compatibility up -d` command.

Deploy with the `docker-compose -f docker-compose.yml --compatibility up -d; docker-compose logs -tf --tail="50" <container name>` command to stream the log output directly to the command-line of a container.

## Networking
The correct type of host networking must be configured prior to application service deployment. Many containers included in this reposirory are using the [**macvlan**](https://docs.docker.com/network/macvlan/) network driver. This is to enable network connectivity between host and the running container. Hence a **macvlan interface** must be created on the host system.

### Create macvlan Interface
Use the `create-macvlan.sh` [script](https://github.com/pwyde/create-macvlan) on the Docker host system and create a macvlan interface named `docker_subnet`.

```
# sh create-macvlan.sh --macvlan docker_subnet --link <parent interface name> --ip-address <host ipv4 address> --network <ipv4 network block>
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

## Variables
Variable are used in many of the Docker compose files and must be set using [environment variables](https://wiki.archlinux.org/index.php/Environment_variables) or a [`.env`](https://docs.docker.com/compose/env-file/) file, which is sourced by the `docker-compose` command during execution.

See the individual compose files for which variables are used and configure accordingly.

## Secrets
[**Docker secrets**](https://docs.docker.com/engine/swarm/secrets/#use-secrets-in-compose) is used during container deployment without including sensitive data in the `docker-compose.yml` or [`.env`](https://docs.docker.com/compose/env-file/) file.

### Usage
Create the Docker secrets directory and change permissions.

```
# mkdir ${DOCKER_DIR}/secrets
# chmod 700 ${DOCKER_DIR}/secrets
```

Configure directory permissions for [**user namespaces**](https://docs.docker.com/engine/security/userns-remap/) if enabled in Docker.

```
# chown root:${CONTAINER_GID} ${DOCKER_DIR}/secrets
# chmod 770 ${DOCKER_DIR}/secrets
# chmod g+s ${DOCKER_DIR}/secrets
# setfacl -dm g::r-- ${DOCKER_DIR}/secrets
```

**Note:** `${CONTAINER_GID}` is the **GID** used for user namespace remapping with `userns-remap` setting in the the `/etc/docker/daemon.json` configuration file. For more information, see [this](https://www.jujens.eu/posts/en/2017/Jul/02/docker-userns-remap/) blog article.

## Traefik Reverse Proxy
[**Traefik**](https://docs.traefik.io/) is used as a [reverse proxy](https://en.wikipedia.org/wiki/Reverse_proxy) for all [HTTP](https://en.wikipedia.org/wiki/Hypertext_Transfer_Protocol)/[HTTPS](https://en.wikipedia.org/wiki/HTTPS)-based application services. See the official [documentation](https://doc.traefik.io/traefik/v2.2/) for more information.

### Traefik v1 vs. Traefik v2
As of Traefik v2 a number of internal pieces and components were rewritten and reorganized. As such, the combination of core notions such as frontends and backends in v1 has been replaced with the combination of [routers](https://doc.traefik.io/traefik/v2.2/routing/routers/), [services](https://doc.traefik.io/traefik/v2.2/routing/services/), and [middlewares](https://doc.traefik.io/traefik/v2.2/middlewares/overview/).

Traefik v2 has also a different syntax and format for its configuration which means that the compose files and [`traefik.toml`](traefik1/traefik.toml) configuration file used for Traefik v1 cannot be used with the latest version of Traefik.

Traefik v1 is no longer being developed, but will still receive any necessary security updates by using the Docker tag **maroilles**.

Traefik v2 is for the reason stated above the preferred version to use. It is also pinned to the v2.2.x version using the Docker tag **chevrotin** in the [`docker-compose-t2.yml`](traefik2/docker-compose-t2.yml) file. 

The repository contains two different variants of compose files:
  * `docker-compose-t1.yml` uses Traefik v1.7.x (maroilles).
  * `docker-compose-t2.yml` uses Traefik v2.2.x (chevrotin).

The `docker-compose-t1.yml` files are no longer actively maintained and should not be used for deployment.

### Configure Persistent Storage
Create a [persistent storage](https://docs.docker.com/storage/) location for the Traefik v2 container.

```
# mkdir -p ${DOCKER_DIR}/traefik/config
# mkdir -p ${DOCKER_DIR}/traefik/certs/acme
# mkdir -p ${DOCKER_DIR}/traefik/certs/docker
# mkdir -p ${DOCKER_DIR}/traefik/secret
```

Configure owner and permissions with commands below.

```
# chown -R ${CONTAINER_UID}:${CONTAINER_GID} ${DOCKER_DIR}/traefik
# chmod -R 770 ${DOCKER_DIR}/traefik
```

**Note:** `${CONTAINER_UID}` and `${CONTAINER_GID}` is the **UID** and **GID** of the unprivileged user and group on the host. It is used for user namespace remapping with `userns-remap` setting in the the `/etc/docker/daemon.json` configuration file. For more information, see [this](https://www.jujens.eu/posts/en/2017/Jul/02/docker-userns-remap/) blog article.

### Enable Docker Over TLS
By default Traefik uses the Docker daemon **socket** which is not considered best practice. It should therefor use a protected network connection over **TLS** instead.

Traefik must be able to read the Docker client [certificate](https://docs.docker.com/engine/security/https/) and **key**. Copy the Docker client certificate files to `${DOCKER_DIR}/traefik/certs/docker`:

  * `ca.pem`
  * `cert.pem`
  * `key.pem`

### Let's Encrypt (ACME)
Traefik is configured to use [wildcard](https://en.wikipedia.org/wiki/Wildcard_certificate) SSL certificates from [**Let's Encrypt**](https://letsencrypt.org/) (ACME).

ACME verification ([Automated Certificate Management Environment](https://en.wikipedia.org/wiki/Automated_Certificate_Management_Environment)) is done through [DNS challenge](https://letsencrypt.org/docs/challenge-types/#dns-01-challenge). A key requirement is that the DNS is managed by a supported DNS provider, for example CloudFlare. DNS Challenge requires that a TXT record is added using the [CloudFlare](https://dash.cloudflare.com/login) API.

Let's Encrypt certificate information is stored in the `${DOCKER_DIR}/traefik/certs/acme/acme.json` file and must be created prior to Traefik container deployment.

```
# touch ${DOCKER_DIR}/traefik/certs/acme/acme.json
# chmod 600 ${DOCKER_DIR}/traefik/certs/acme/acme.json
```

### Traefik API/Dashboard with Basic Authentication
Traefik comes with an **API** and **dashboard** which can be accessed on port `8080` from a web browser. By default it is accessible to anyone and should be secured with at least basic authentication.

This can be accomplished using a `.htpasswd` file, which is created with command below.

```
# printf "admin:$(openssl passwd -apr1)" > ${DOCKER_DIR}/traefik/secret/.htpasswd
```

## Application Services

### DokuWiki
Create a [persistent storage](https://docs.docker.com/storage/) location for the container.

 ```
 # mkdir -p ${DOCKER_DIR}/dokuwiki/config
 ```

Configure owner and permissions with commands below.

```
# chown -R ${UID}:${GID} ${DOCKER_DIR}/dokuwiki
# chmod -R 770 ${DOCKER_DIR}/dokuwiki
```

### GitLab CE
Create a [persistent storage](https://docs.docker.com/storage/) location for the container.

```
# mkdir -p ${DOCKER_DIR}/gitlab/config
# mkdir -p ${DOCKER_DIR}/gitlab/data
# mkdir -p ${DOCKER_DIR}/gitlab/logs
```

Configure owner and permissions with commands below.

```
# chown -R ${UID}:${GID} ${DOCKER_DIR}/gitlab
# chmod -R 770 ${DOCKER_DIR}/gitlab
```

### Nextcloud
Create a [persistent storage](https://docs.docker.com/storage/) location for the container.

```
# mkdir -p ${DOCKER_DIR}/nextcloud/config
# mkdir -p ${DOCKER_DIR}/nextcloud/data
# mkdir -p ${DOCKER_DIR}/nextcloud/db
# mkdir -p ${DOCKER_DIR}/nextcloud/redis
```

Configure owner and permissions with commands below.

```
# chown -R ${UID}:${GID} ${DOCKER_DIR}/nextcloud
# chmod -R 770 ${DOCKER_DIR}/nextcloud
# chown -R ${CONTAINER_UID}:${CONTAINER_GID} ${DOCKER_DIR}/nextcloud/redis
```

**Note:** `${CONTAINER_UID}` and `${CONTAINER_GID}` is the **UID** and **GID** of the unprivileged user and group on the host. It is used for user namespace remapping with `userns-remap` setting in the the `/etc/docker/daemon.json` configuration file. For more information, see [this](https://www.jujens.eu/posts/en/2017/Jul/02/docker-userns-remap/) blog article.

### Plex
Create a [persistent storage](https://docs.docker.com/storage/) location for the container.

```
# mkdir -p ${DOCKER_DIR}/plex/config
# mkdir -p ${DOCKER_DIR}/plex/transcode
# mkdir -p ${MEDIA_DIR}
```

Configure owner and permissions with commands below.

```
# chown -R ${UID}:${GID} ${DOCKER_DIR}/plex
# chmod -R 770 ${DOCKER_DIR}/plex
# chown -R ${UID}:${MEDIA_GID} ${MEDIA_DIR}
# chmod g+s ${MEDIA_DIR}
# setfacl -dm g::r-- ${MEDIA_DIR}
```

**Note:** `${MEDIA_GID}` is the **GID** of a dedicated group for accessing the media directory, which can be utilized in the future for sharing using [**Samba**](https://www.samba.org/).

### Portainer
Create a [persistent storage](https://docs.docker.com/storage/) location for the container.

```
# mkdir -p ${DOCKER_DIR}/portainer/data
```

Configure owner and permissions with commands below. 

```
# chown -R ${CONTAINER_UID}:${CONTAINER_GID} ${DOCKER_DIR}/portainer
# chmod -R 770 ${DOCKER_DIR}/portainer
```

**Note:** `${CONTAINER_UID}` and `${CONTAINER_GID}` is the **UID** and **GID** of the unprivileged user and group on the host. It is used for user namespace remapping with `userns-remap` setting in the the `/etc/docker/daemon.json` configuration file. For more information, see [this](https://www.jujens.eu/posts/en/2017/Jul/02/docker-userns-remap/) blog article.

### ruTorrent & Flood
Create a [persistent storage](https://docs.docker.com/storage/) location for the container.

```
# mkdir -p ${DOCKER_DIR}/rutorrent-flood/config
# mkdir -p ${DOWNLOADS_DIR}
```

Configure owner and permissions with commands below. 

```
# chown ${UID}:${DOWNLOADS_GID} ${DOCKER_DIR}/rutorrent-flood
# chmod -R 770 ${DOCKER_DIR}/rutorrent-flood
# chmod g+s ${DOWNLOADS_DIR}
# setfacl -dm g::r-- ${DOWNLOADS_DIR}
```

**Note:** `${DOWNLOADS_GID}` is the **GID** of a dedicated group for accessing the downloads directory, which can be utilized in the future for sharing using [**Samba**](https://www.samba.org/).

### UniFi Controller
Create a [persistent storage](https://docs.docker.com/storage/) location for the container.

```
# mkdir -p ${DOCKER_DIR}/unifi-controller/config
```

Configure owner and permissions with commands below.

```
# chown -R ${UID}:${GID} ${DOCKER_DIR}/unifi-controller
# chmod -R 770 ${DOCKER_DIR}/unifi-controller
```

## Non-Docker or External Application Services
Traefik v2 can also be used to proxy traffic for non-Docker application services or external devices. This is perfoermed by adding a dynamic router rule to the [`${DOCKER_DIR}/traefik2/rules`](./traefik2/rules/) directory using the Traefik [**file provider**](https://doc.traefik.io/traefik/v2.0/providers/file/).

This repository contains rules for [Pi-hole](https://pi-hole.net/) and [UniFi Cloud Key/UniFi Network Controller](https://unifi-protect.ui.com/cloud-key-gen2). Please note that environment variables are not applied to `.toml` files and must be editied before implementation.

## License
All files containing in this repository are licensed under [**The Unlicense**](https://unlicense.org/) license.

A license with no conditions whatsoever which dedicates works to the public domain. Unlicensed works, modifications, and larger works may be distributed under different terms and without source code. See the [LICENSE](LICENSE) file for more information.
