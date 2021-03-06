# Archivematica on Docker Compose

- [Audience](#audience)
- [Requirements](#requirements)
- [Installation](#installation)
- [Web UIs](#web-uis)
- [Source code auto-reloading](#source-code-auto-reloading)
- [Logs](#logs)
- [Scaling](#scaling)
- [Ports](#ports)
- [Tests](#tests)
- [Cleaning up](#cleaning-up)
- [Troubleshooting](#troubleshooting)
  - [Nginx returns 502 Bad Gateway](#nginx-returns-502-bad-gateway)
  - [MCPClient osdeps cannot be updated](#mcpclient-osdeps-cannot-be-updated)
  - [Error while mounting volume](#error-while-mounting-volume)
  - [Tests are too slow](#tests-are-too-slow)

## Audience

This Archivematica environment is based on Docker Compose and it is specifically
**designed for developers**. Compose can be used in a production environment but that is
beyond the scope of this recipe.

Artefactual developers use Docker Compose heavily so it's important that you're familiar with it.
Please read the [documentation](https://docs.docker.com/compose/reference/overview/).

## Requirements

Docker, Docker Compose, git and make.

It is beyond the scope of this document to explain how these dependencies are
installed in your computer. If you're using Ubuntu 16.04 the following commands
may work:

    $ sudo apt update
    $ sudo apt install -y build-essential python-dev git
    $ sudo pip install -U docker-compose

And install Docker CE following [these instructions](https://docs.docker.com/engine/installation/linux/docker-ce/ubuntu/).

### Docker and Linux

Docker will provide instructions on how to use it as a non-root user. This may not be desirable for all. 

	If you would like to use Docker as a non-root user, you should now consider
	adding your user to the "docker" group with something like:

	  sudo usermod -aG docker <user>

	Remember that you will have to log out and back in for this to take effect!

	WARNING: Adding a user to the "docker" group will grant the ability to run
			 containers which can be used to obtain root privileges on the
			 docker host.
			 Refer to https://docs.docker.com/engine/security/security/#docker-daemon-attack-surface
			 for more information.

The impact to those following this recipe is that any of the commands below which call Docker will need to be
run as a root user using 'sudo'.

## Installation
If you haven't already, create a directory to store this repository using git clone: 
    
    $ git clone https://github.com/artefactual-labs/am.git
    (or in my case: https://github.com/joel-simpson/am.git )

Run the installation (and all Docker Compose) commands from within the compose directory: 

    $ cd ./am/compose

These are the commands you need to run when starting from scratch:

    $ git pull --rebase
    $ git submodule update --init --recursive
    $ make create-volumes
    $ docker-compose up -d --build
    $ make bootstrap
    $ make restart-am-services

`make create-volumes` creates two external volumes. They're heavily used in our
containers but they are provided in the host machine:

- `/media/joel/Data/archivematicaData/am-pipeline-data` - the shared directory.
- `/media/joel/Data/archivematicaData/ss-location-data` - the transfer source location.

### GNU make

Make commands above, and any subsequent calls to it below can be reviewed using the following command
from the compose directory:

    $ make help
    
## Upgrading to the latest version of Archivematica

The installation instructions above will install the submodules defined in https://github.com/artefactual-labs/am/tree/master/src which are from the qa/1.x branches of Archivematica and the Storage Service

To upgrade your installation to include the most recent changes in the qa/1.x branches, use the following commands: 

    $ git pull --rebase
    $ git submodule update --init --recursive
    $ docker-compose build
    $ docker-compose up -d
    $ make bootstrap
    $ make restart-am-services

## Web UIs

- Archivematica Dashboard: http://127.0.0.1:62080/
- Archivematica Storage Service: http://127.0.0.1:62081/

## Source code auto-reloading

Dashboard and Storage Service are both served by Gunicorn. We set up Gunicorn
with the [reload](http://docs.gunicorn.org/en/stable/settings.html#reload)
setting enabled meaning that the Gunicorn workers will be restarted as soon as
code changes.

Other components in the stack like the `MCPServer` don't offer this option and
they need to be restarted manually, e.g.:

    $ docker-compose up -d --force-recreate --no-deps archivematica-mcp-server

If you've added new dependencies or changes the `Dockerfile` you should also
add the `--build` argument to the previous command in order to ensure that the
container is using the newest image, e.g.:

    $ docker-compose up -d --force-recreate --build --no-deps archivematica-mcp-server

## Logs

In recent versions of Archivematica we've changed the logging configuration so
the log events are sent to the standard streams. This is a common practice
because it makes much easier to aggregate the logs generated by all the
replicas that we may be deploying of our services across the cluster.

Docker Compose aggregates the logs for us so you can see everything from one
place. Some examples:

- `docker-compose logs -f`
- `docker-compose logs -f archivematica-storage-service`
- `docker-compose logs -f nginx archivematica-dashboard`

Docker keeps the logs in files using the [JSON File logging driver][logs-0].
If you want to clear them, we provide a simple script that can do it for us
quickly but it needs root privileges, e.g.:

    $ sudo make flush-logs

[logs-0]: https://docs.docker.com/config/containers/logging/json-file/

## Scaling

With Docker Compose we can run as many containers as we want for a service,
e.g. by default we only provision a single replica of the
`archivematica-mcp-client` service but nothing stops you from running more:

    $ docker-compose up -d --scale archivematica-mcp-client=3

We still have one service but three containers. Let's verify that the workers
are connected to Gearman:

    $ echo workers | socat - tcp:127.0.0.1:62004,shut-none | grep "_v0.0" | awk '{print $2}' - | sort -u
    172.19.0.15
    172.19.0.16
    172.19.0.17

## Ports

| Service                                 | Container port | Host port   |
| --------------------------------------- | -------------- | ----------- |
| mysql                                   | `tcp/3306`     | `tcp/62001` |
| elasticsearch                           | `tcp/9200`     | `tcp/62002` |
| redis                                   | `tcp/6379`     | `tcp/62003` |
| gearman                                 | `tcp/4730`     | `tcp/62004` |
| fits                                    | `tcp/2113`     | `tcp/62005` |
| clamavd                                 | `tcp/3310`     | `tcp/62006` |
| nginx » archivematica-dashboard         | `tcp/80`       | `tcp/62080` |
| nginx » archivematica-storage-service   | `tcp/8000`     | `tcp/62081` |
| selenium-hub                            | `tcp/4444`     | `tcp/62100` |

## Tests

The `Makefile` includes many useful targets for testing. List them all with:

    $ make 2>&1 | grep test-

The sources of the [acceptance tests](../src/archivematica-acceptance-tests)
have been made available inside Docker using volumes so you can edit them and
the changes will apply immediately.

## Resetting the environment

In many cases, as a tester or a developer, you want to restart all the
containers at once and make sure the latest version of the images are built.
But also, you don't want to lose your data like the search index or the
database. If this is case, run the following command:

    $ docker-compose up -d --force-recreate --build

Additionally you may want to delete all the deta including the stuff in the
external volumes:

    $ make flush

Both snippets can be combined or used separately.

You may need to update the codebase, and for that you can run this command:

    $ git submodule update --init --recursive

## Cleaning up

The most effective way is:

    $ docker-compose down --volumes

It doesn't delete the external volumes described in the
[Installation](#installation) section of this document. You have to delete the
volumes manually with:

    $ docker volume rm am-pipeline-data
    $ docker volume rm ss-location-data

Optionally you may also want to delete the directories:

    $ rm -rf $HOME/.am/am-pipeline-data $HOME/.am/ss-location-data

## Troubleshooting

##### Nginx returns 502 Bad Gateway

We're using Nginx as a proxy. Likely the underlying issue is that either the
Dashboard or the Storage Service died. Run `docker-compose ps` to confirm it:

                     Name                    State
    -------------------------------------------------
    compose_archivematica-storage-service_1  Exit 3

You want to see what's in the logs, e.g.:

    $ docker-compose logs -f archivematica-storage-service

    ImportError: No module named storage_service.wsgi
    [2017-10-26 19:28:24 +0000] [11] [INFO] Worker exiting (pid: 11)
    [2017-10-26 19:28:24 +0000] [7] [INFO] Shutting down: Master
    [2017-10-26 19:28:24 +0000] [7] [INFO] Reason: Worker failed to boot.

Now we know why -  I had deleted the `wsgi` module. The worker crashed and
Gunicorn gave up. This could happen for example when we're rebasing a branch
and git is not atomically moving things around. But it's fixed now and you want
to give it another shot so we run `docker-compose up -d` to ensure that all the
services are up again. Next run `docker-compose ps` to verify that it's all up.

##### MCPClient osdeps cannot be updated

The MCPClient Docker image bundles a number of tools that are used by
Archivematica. They're listed in the [osdeps files][0] but the
[MCPClient.Dockerfile][1] is not making use of it yet. If you need to
introduce dependency changes in MCPClient you will need to update both.

In [#931][2] we started publishing a MCPClient base Docker image to speed up
development workflows. This means that the dependencies listed in the osdeps
files are now included in [MCPClient-base.Dockerfile][3]. Once the file is
updated you would publish the corresponding image as follows:

```
$ docker build -f MCPClient-base.Dockerfile -t artefactual/archivematica-mcp-client-base:20180219.01.52dc9959 .
$ docker push artefactual/archivematica-mcp-client-base:20180219.01.52dc9959
```

Where the tag `20180219.01.52dc9959` is a combination of the date, build number
that day and the commit that updates the Dockerfile.

As a developer you may want to build the image and test it before publishing it,
e.g.:

1. Edit `MCPClient-base.Dockerfile`.
2. Build image with new tag.
3. Update the `FROM` instruciton in `MCPClient.Dockerfile` to use it.
4. Build the image of the `archivematica-mcp-client` service.
5. Test it.
6. Publish image.

[0]: https://github.com/artefactual/archivematica/tree/qa/1.x/src/MCPClient/osdeps
[1]: https://github.com/artefactual/archivematica/blob/qa/1.x/src/MCPClient.Dockerfile
[2]: https://github.com/artefactual/archivematica/pull/931
[3]: https://github.com/artefactual/archivematica/blob/qa/1.x/src/MCPClient-base.Dockerfile

##### Error while mounting volume

Our Docker named volumes are stored under `/tmp` which means that it is
possible that they will be recycled at some point by the operative system. This
frequently happens when you restart your machine.

Under this scenario, if you try to bring up the services again you will likely
see one or more errors like the following:

    ERROR: for compose_archivematica-mcp-server_1  Cannot create container for service archivematica-mcp-server: error while mounting volume with options: type='none' device='/home/user/.am/am-pipeline-data' o='bind': no such file or directory

The solution is simple. You need to create the volumes again:

    $ make create-volumes

And now you're ready to continue as usual:

    $ docker-compose up -d --build

Optionally, you can define new persistent locations for the external volumes.
The defaults are defined in the `Makefile`:

    # Paths for Docker named volumes
    AM_PIPELINE_DATA ?= $(HOME)/.am/am-pipeline-data
    SS_LOCATION_DATA ?= $(HOME)/.am/ss-location-data

##### Tests are too slow

Running tests with `make test-mcp-client` and such can be very slow because the database is re-created on each attempt. When the tests are done the database is removed unless you use `--reuse-db`, e.g.: you can use the following command to run the MCPClient tests.

    docker-compose run --no-deps --user=root --workdir /src/MCPClient --rm --entrypoint=py.test archivematica-mcp-client --reuse-db --exitfirst

The difference is noticeable.
