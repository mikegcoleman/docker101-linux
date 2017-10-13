# Docker 101 - Linux (Part 2)

We had an introduction to volumes by way of bind mounts earlier, but let's take a deeper look at the Docker file system and volumes. 

The [Docker documentation](https://docs.docker.com/engine/userguide/storagedriver/imagesandcontainers/_) gives a great explanation on how storage works with Docker images and containers, but here's the high points. 

* Images are comprised of layers
* These layers are added by each line in a Dockerfile
* Images on the same host or registry will share layers if possible
* When container is started it gets a unique writeable layer of its own to capture changes that occur while it's running
* Layers exist on the host file system in some form (usually a directory, but not always) and are managed by a [storage driver](https://docs.docker.com/engine/userguide/storagedriver/selectadriver/) to present a logical filesystem in the running container. 
* When a container is removed the unique writeable layer (and everything in it) is removed as well
* To persist data (and improve performance) Volumes are used. 
* Volumes (and the directories they are built on) are not managed by the storage driver, and will live on if a container is removed.  

The following exercises will help to illustrate those concepts in practicc. 

Let's start by looking at layers and how files written to a container are managed by something called *copy on write*.

## Layers and Copy on Write

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

The very first line is the the Dockerfile is: `FROM debian:jessie`

Next we'll create a file in our container, and see how that's represented on the host file system. 

1. Start a Debian container, shell into it.   

```
$ docker run --tty --interactive --name debian debian:jessie bash
root@e09203d84deb:/#
```

2. Create a file and then list out the directory to make sure it's there:

```
root@e09203d84deb:/# touch test-file
root@e09203d84deb:/# ls
bin   dev  home  lib64  mnt  proc  run   srv  test-file  usrboot  etc  lib   media  opt  root  sbin  sys  tmp        var
```


We can see  `test-file` exists in the root of the containers file system. 

What has happened is that when a new file was written to the disk, the Docker storage driver placed that file in it's own layer. This is called *copy on write* - as soon as a change is detected the change is copied into a new layer. That layers is represented by a directory on the host file system. All of this is managed by the Docker storage driver. 

3. Exit the container but leave it running by pressing `ctrl-p` and then `ctrl-q`

The Docker hosts for the labs today use OverlayFS with the [overlay2](https://docs.docker.com/engine/userguide/storagedriver/overlayfs-driver/#how-the-overlay2-driver-works) storage driver. 

OverlayFS layers two directories on a single Linux host and presents them as a single directory. These directories are called layers and the unification process is referred to as a union mount. OverlayFS refers to the lower directory as lowerdir and the upper directory a upperdir. "Upper" and "Lower" refer to when the layer was added to the image. In our example the writeable layer is the most "upper" layer.  The unified view is exposed through its own directory called merged. 

We can use Docker's *inspect* command to look at where these directories live on our Docker host's file system. 

> Note: The *inspect* command uses Go templates to allow us to extract out specific information from its output. For more information on how these templates work with *inspect* read this [excellent tutorial](http://container-solutions.com/docker-inspect-template-magic/). 

```
$ docker inspect -f '{{json .GraphDriver.Data}}' debian | jq
{
  "LowerDir": "/var/lib/docker/overlay2/0dad4d523351851af4872f8c6706fbdf36a6fa60dc7a29fff6eb388bf3d7194e-init/diff:/var/lib/docker/overlay2/c2e2db4221ad5dca9f35a92e04d17c79b861ddee30015fa3ddc77c66ae1bf758/diff",
  "MergedDir": "/var/lib/docker/overlay2/0dad4d523351851af4872f8c6706fbdf36a6fa60dc7a29fff6eb388bf3d7194e/merged",
  "UpperDir": "/var/lib/docker/overlay2/0dad4d523351851af4872f8c6706fbdf36a6fa60dc7a29fff6eb388bf3d7194e/diff",
  "WorkDir": "/var/lib/docker/overlay2/0dad4d523351851af4872f8c6706fbdf36a6fa60dc7a29fff6eb388bf3d7194e/work"
}
```
> Note: `WorkDir` is a working directory for the Overlay2 driver

Since the change we made is the newest modification to the Debian container's file system, it's going to be stored in `UpperDir`. 

3. List the contents of the `UpperDir`. 

```
$ cd $(docker inspect -f {{.GraphDriver.Data.UpperDir}} debian)

$ ls
root       test-file
```

`MergedDir` is going to give us a look at the root filesystem of our container which is a combination of `UpperDir` and `LowerDir`:

4. List the contents of `MergedDir`:

```
$ cd $(docker inspect -f {{.GraphDriver.Data.MergedDir}} debian)

$ ls
bin        etc        lib64      opt        run        sys        usr
boot       home       media      proc       sbin       test-file  var
dev        lib        mnt        root       srv        tmp
```

Notice that the directory on our host file system has the same contents as the one inside the container. That's because that directory is what we see in the container. 

> Warning: You should NEVER manipulate your container's file system via the Docker host. This is only being done as an academic exercise. 

5. Write a new file to the host file system in the `UpperDir`, and list the directory to see the contents

```
$ cd $(docker inspect -f {{.GraphDriver.Data.UpperDir}} debian)

$ touch test-file2

$ ls
root        test-file   test-file2
```


```
docker inspect -f 'in the {{.Name}} container we mapped {{(index .Mounts 0).Destination}} to {{(index .Mounts 0).Source}}' mysqldb
```

6. Move back into your Debian container and list the root file system

```
$ docker attach debian

root@674d7abf10c6:/# ls
bin   dev  home  lib64  mnt  proc  run   srv  test-file   tmp  var
boot  etc  lib   media  opt  root  sbin  sys  test-file2  usr
```
The file that was created on the local host filesystem (`test-file2`) is now available in the container as well. 

7. Type `exit` to stop your container, which will also stop it

```
root@674d7abf10c6:/# exit
exit
```

8. Ensure that your debian container still exists

```
$ docker container ls --all
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS           PORTS               NAMES
674d7abf10c6        debian:jessie       "bash"              36 minutes ago      Exited (0) 2 minutes ago                       debian
```

9. List out the current directory

```
$ ls
root        test-file   test-file2
```

Because the container still exists, the files are still available on  your file system. At this point you could `docker start` your container and it would be just as it was before you exited. 

However, if we remove the container, the directories on the host file system will be removed, and your changes will be gone

10. Remove the container and list the directory contents

```
$ docker container rm debian
debian

$ ls
```

The files that were created are now gone. You've actually been left in a sort of "no man's land" as the directory you're in has actually been deleted as well.

11. Copy the directory location from the prompt in the terminal. 

12. CD back to your home directory

```
$ cd
```

13. Attempt to list the contents of the old `UpperDir` directory.

```
$ ls /var/lib/docker/overlay2/0dad4d523351851af4872f8c6706fbdf36a6fa60dc7a29fff6eb388bf3d7194e/diff
ls: /var/lib/docker/overlay2/0dad4d523351851af4872f8c6706fbdf36a6fa60dc7a29fff6eb388bf3d7194e/diff: No such file or directory
```
