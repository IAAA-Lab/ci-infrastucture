# Continuous Integration Infrastucture

This is a CI infrastructure that offers:

- **Jenkins:** A continuous integration software that builds, tests and deploys artifacts based on it's source code and a given configuration
- **Docker Registry:** A repository of `docker images` where a `docker daemon` can pull from.
- **Portus:** A tool that allows user control over Docker Registry's `images`
- **Sonarqube:** A static code analyzer

## Deploy

In order to deploy this infrastucture, the host machine needs:
- `docker`
- `docker-compose`

### Getting the source code

You can download the `source code` by cloning it o by downloading and copying. The easiest way (and the _less dependent_) is by using a `git docker container` like `alpine/git`.

For the first time, **go to the directory** where you want to clone the project and **execute**:

```bash
GIT_REPO=https://github.com/IAAA-Lab/ci-infrastucture \
docker run -it --rm -u $(id -u):$(id -u) -v $(pwd):/git alpine/git clone $GIT_REPO
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
1. Run a work container using `busybox` image which uses this volume: `docker run --rm -v $2:/vol -v $(pwd):/ext busybox tar xvf /ext/certificates.tar --strip 1 -C /vol;`

### 3... 2... 1... Ignition

Once we have the code, deploying it is as simple as executing de proper `docker-compose` script:

```bash
MACHINE_FQDN=https://example.org \
SECRET_KEY_BASE=randomme \
SECRET_MASTER_PASSWORD=changeme \
docker-compose -f docker-compose.yml up -d
```
**Note1:** If you don't understand certain parts of the script, read the section bellow to learn more about `docker-compose`.

**Note2:** If your host uses `rancher`, your can deploy it with:


## Development

Usually, when using `docker-compose`, we need a way to store host specific values or secrets. One way of doing that (**not the most secure**) is by using `enviroment variables`. The problem is that you need to provide this `env` every time you run the command.  

There is a handy feature in `docker-compose` that helps a lot in a development enviroment, the `.env` file.

Create a `.env` file with the `env variables` that you whant to use in the `docker-compose` command. The used ones in this project are:
```plain
MACHINE_FQDN={your machine FQDN or localhost}
SECRET_KEY_BASE={a random key base}
SECRET_MASTER_PASSWORD={a password to access all services. KEEP IT SECRET}
```
