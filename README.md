# Continuous Integration Infrastucture

This is a CI infrastructure that offers:

- **Jenkins:** A continuous integration software that builds, tests and deploys artifacts based on it's source code and a given configuration
- **Docker Registry:** A repository of `docker images` where a `docker daemon` can pull from.
- **Portus:** A tool that allows user control over Docker Registry's `images`
- **Sonarqube:** A static code analyzer

## Contenidos
<!-- TOC START min:1 max:3 link:true update:true -->
- [Continuous Integration Infrastucture](#continuous-integration-infrastucture)
  - [Deploy](#deploy)
    - [Getting the source code](#getting-the-source-code)
    - [Prepare your certs](#prepare-your-certs)
    - [3... 2... 1... Ignition](#3-2-1-ignition)
    - [Deploying in](#deploying-in)
  - [Development](#development)

<!-- TOC END -->

## Deploy

In order to deploy this infrastucture, the host machine needs:
- `docker`
- `docker-compose`

### Getting the source code

You can download the `source code` by cloning it o by downloading and copying. The easiest way (and the _less dependent_) is by using a `git docker container` like `alpine/git`.

For the first time, **go to the directory** where you want to clone the project and **execute**:

```bash
docker run -it --rm -u $(id -u):$(id -u) -v $(pwd):/git alpine/git clone https://github.com/IAAA-Lab/ci-infrastucture
```

If you already have cloned it, you can update all by pulling with the same command

```bash
docker run -it --rm -u $(id -u):$(id -u) -v $(pwd):/git alpine/git pull origin master
```

### Prepare your certs

HTTPS certificates are mandatory for this project. The current decision is to store it in an external volume called `certificates`.

There are different ways of doing that but many steps are common.

#### The `docker cp` way

Assuming you have your certs in an accesible path:

1. Create the external volume: `docker volume create certificates`
1. Run a data container using `busybox` image which uses this volume: `docker run -it --rm -v certificates:/vol --name certs -d busybox`
1. Copy the cert to the volume: `docker cp $CERT_LOCATION certs:vol/domain.crt`
1. Copy the key to the volume: `docker cp $KEY_LOCATION certs:vol/domain.key`
1. Stop the container (it will auto-remove because of the `--rm` opt): `docker stop certs`

#### The `zip` way

Assuming you have your certs in a tar file called `certificates.tar`:
```
- root directory
|- domain.crt
|- domain.key
```

1. Create the external volume: `docker volume create certificates`
1. Run a work container using `busybox` image which uses this volume: `docker run --rm -v certificates:/vol -v $(pwd):/ext busybox tar xvf /ext/certificates.tar --strip 1 -C /vol`

### 3... 2... 1... Ignition

Once we have the code, deploying it is as simple as executing de proper `docker-compose` script:

```bash
MACHINE_FQDN=your.domain.org \
SECRET_KEY_BASE=randomme \
SECRET_MASTER_PASSWORD=changeme \
docker-compose \
-f docker-compose.yml up -d
```
**Note1:** If you don't understand certain parts of the script, read the section bellow to learn more about `docker-compose`.

**Note2:** If your host uses `rancher`, your can deploy it with:

If you don't have docker-compose, you can use a container:
```bash
docker run --rm \
-v $(pwd):/project \
-v /var/run/docker.sock:/var/run/docker.sock \
-w /project \
-e MACHINE_FQDN=pumuky.cps.unizar.es \
-e SECRET_KEY_BASE=qqwef7wadgn2kf87212bf897123r \
-e SECRET_MASTER_PASSWORD=ahabthecaptain \
docker/compose:1.20.0 \
-f docker-compose.yml up -d
```

### Desplegado en Rancher OS

Rancher OS, well known as an operative system that relies on `docker containers`, is an interesting choice when you are deploying clusters. Due to some limitations, it's perfect for designing evolutive infrastuctures.

Para desplegar el sistema, lo mejor es hacerlo desde de un contenedor con las dependencias necesarias (`git`, `docker` y `docker-compose`). La imagen escogida es `docteurklein/compose-ci`, que tiene un propósito similar, pero usa `webhooks`. NO ES BUENA IDEA utilizar esa imagen para su propósito real debido al agujero de seguridad que supone exponer este servicio.

Deploying in Rancher OS may be as simple as connecting to `ssh` and executing one command. That command has two important points: The base image, which is perfect thanks to it's dependencies, and the command itself, which clones a given `git` repo and deploys it with docker-compose.

```bash
GIT_REPO="https://github.com/IAAA-Lab/ci-infrastucture"
ENV_FQDN="MACHINE_FQDN=pumuky.cps.unizar.es"
ENV_KEYBASE="SECRET_KEY_BASE=randme"
ENV_PASSWORD="SECRET_MASTER_PASSWORD=changeme"
GIT_CLONE_COMMAND="git clone ${GIT_REPO} ."
DOCKER_COMPOSE_COMAND="${ENV_FQDN} ${ENV_KEYBASE} ${ENV_PASSWORD} docker-compose -f docker-compose.yml up -d"

docker run --rm \
-w /ci-infrastucture \
-v /var/run/docker.sock:/var/run/docker.sock \
docteurklein/compose-ci \
sh -c "${GIT_CLONE_COMMAND} && ${DOCKER_COMPOSE_COMAND}"
```
Notas:
- `--rm` makes the container auto-removable.
- `-w` defines `workdir`. `docker-compose` uses it as `stack` name if none is provided.
- Specifying the `docker-compose.yml` file ignores the `docker-compose.override.yml` which is used in development.

## Development

Usually, when using `docker-compose`, we need a way to store host specific values or secrets. One way of doing that (**not the most secure**) is by using `enviroment variables`. The problem is that you need to provide this `env` every time you run the command.  

There is a handy feature in `docker-compose` that helps a lot in a development enviroment, the `.env` file.

Create a `.env` file with the `env variables` that you whant to use in the `docker-compose` command. The used ones in this project are:
```plain
MACHINE_FQDN={your machine FQDN or localhost}
SECRET_KEY_BASE={a random key base}
SECRET_MASTER_PASSWORD={a password to access all services. KEEP IT SECRET}
```
