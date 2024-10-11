# Gogs how-to


## Introduction
[Gogs](https://github.com/gogs/gogs) can be used to build self-hosted GIT service and it is easy to
be set up. 


## Install

Optionally, specify the [verion](https://github.com/gogs/gogs/pkgs/container/gogs)
```shell
# Pull image from Docker Hub.
$ docker pull gogs/gogs

# Create local directory for volume.
$ mkdir -p /var/gogs

# Use `docker run` for the first time.
$ docker run --name=gogs -p 10022:22 -p 10880:3000 -v /var/gogs:/data gogs/gogs

# Use `docker start` if you have stopped it.
$ docker start gogs
```


After the Gogs is up, set the repos and push your code to Gogs