# Docker 101 - Linux (Part 2)

We had an introduction to volumes by way of bind mounts earlier, but let's take a deeper look at the Docker file system and volumes. 

Let's start by looking at layers and the copy on write file system. 

Each line in a Dockerfile creates a new layer in our image, and layers will be shared across images on the same host. Let's look at this in action

1. Pull down the Debian:Jessie image

```
$ docker image pull debian:jessie
jessie: Pulling from library/debian
85b1f47fba49: Pull complete
Digest: sha256:f51cf81db2de8b5e9585300f655549812cdb27c56f8bfb992b8b706378cd517d
Status: Downloaded newer image for debian:jessie
```

2. Pull down a MySQL image

```
$ docker image pull mysql
Using default tag: latest
latest: Pulling from library/mysql
85b1f47fba49: Already exists
27dc53f13a11: Pull complete
095c8ae4182d: Pull complete
0972f6b9a7de: Pull complete
1b199048e1da: Pull complete
159de3cf101e: Pull complete
963d934c2fcd: Pull complete
f4b66a97a0d0: Pull complete
f34057997f40: Pull complete
ca1db9a06aa4: Pull complete
0f913cb2cc0c: Pull complete
Digest: sha256:bfb22e93ee87c6aab6c1c9a4e7cdc68e9cb9b64920f28fa289f9ffae9fe8e173
Status: Downloaded newer image for mysql:latest
```

What do you notice about those the output from the Docker pull request for MySQL?

The first layer pulled says:

`85b1f47fba49: Already exists`

Notice that the layer id (`85b1f47fba498`) is the same for the first layer of the MySQl image and the only layer in the Debian:Jessie image. And because we already had pulled that layer when we pulled the Debian image, we didn't have to pull it again. 

So, what does that tell us about the MySQL image? Well we know that it's based on the Debian:Jessie base image. We can confirm this by looking at the [Dockerfile on Docker Store](https://github.com/docker-library/mysql/blob/0590e4efd2b31ec794383f084d419dea9bc752c4/5.7/Dockerfile). 

The very first line is: `FROM debian:jessie`

Layers in an image are read only, and all containers started from that image will share them. When we start a container from an image we add a single read-write layer to capture any changes to that container as it's running. When a change to the file system is detected it's automatically copied into that read-write layer - this is referred to as *copy on write*.

1. Start a Debian container, shell into it.   

```
$ docker run --tty --interactive --name debian debian:jessie bash
root@e09203d84deb:/#
```

2. Create a file and then list out the directory to make sure it's there:

```
root@e09203d84deb:/# touch test-fileroot@e09203d84deb:/# ls
bin   dev  home  lib64  mnt  proc  run   srv  test-file  usrboot  etc  lib   media  opt  root  sbin  sys  tmp        var
```

$ docker inspect -f '{{json .GraphDriver.Data}}' debian | jq

$ ls $(docker inspect -f {{.GraphDriver.Data.UpperDir}} debian)

$ ls $(docker inspect -f {{.GraphDriver.Data.MergedDir}} debian)



We can see our `test-file` there in the root of the containers file system. 

3. Exit the container but leave it running by pressing `ctrl-p` and then `ctrl-q`



Let's






```
docker inspect -f 'in the {{.Name}} container we mapped {{(index .Mounts 0).Destination}} to {{(index .Mounts 0).Source}}' mysqldb
```