# Docker 101 - Linux (Part 2): Understanding the Docker File System and Volumes

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

The following exercises will help to illustrate those concepts in practice. 

Let's start by looking at layers and how files written to a container are managed by something called *copy on write*.

## Layers and Copy on Write

> Note: If you have just completed part 1 of the workshop, please close that session and start a new one. 

1. In PWD click "+Add new instance" and move into that command windows.

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

    So, what does that tell us about the MySQL image? Since each layer is created by a line in the image's *Dockerfile*, we know that the MySQL image is based on the Debian:Jessie base image. We can confirm this by looking at the [Dockerfile on Docker Store](https://github.com/docker-library/mysql/blob/0590e4efd2b31ec794383f084d419dea9bc752c4/5.7/Dockerfile). 

    The first line in the the Dockerfile is: `FROM debian:jessie` This will import that layer into the MySQL image. 

    So layers are created by Dockerfiles and are are shared between images. When you start a container, a writeable layer is added to the base image. 

    Next you will create a file in our container, and see how that's represented on the host file system. 

3. Start a Debian container, shell into it.   

    ```
    $ docker run --tty --interactive --name debian debian:jessie bash
    root@e09203d84deb:/#
    ```

4. Create a file and then list out the directory to make sure it's there:

    ```
    root@e09203d84deb:/# touch test-file
    root@e09203d84deb:/# ls
    bin   dev  home  lib64  mnt  proc  run   srv  test-file  usrboot  etc  lib   media  opt  root  sbin  sys  tmp        var
    ```

    We can see  `test-file` exists in the root of the containers file system. 

    What has happened is that when a new file was written to the disk, the Docker storage driver placed that file in it's own layer. This is called *copy on write* - as soon as a change is detected the change is copied into the writeable layer. That layers is represented by a directory on the host file system. All of this is managed by the Docker storage driver. 

5. Exit the container but leave it running by pressing `ctrl-p` and then `ctrl-q`

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

6. List the contents of the `UpperDir`. 

    ```
    $ cd $(docker inspect -f {{.GraphDriver.Data.UpperDir}} debian)
    
    $ ls
    root       test-file
    ```

    `MergedDir` is going to give us a look at the root filesystem of our container which is a combination of `UpperDir` and `LowerDir`:

7. List the contents of `MergedDir`:

    ```
    $ cd $(docker inspect -f {{.GraphDriver.Data.MergedDir}} debian)

    $ ls
    bin        etc        lib64      opt        run        sys        usr
    boot       home       media      proc       sbin       test-file  var
    dev        lib        mnt        root       srv        tmp
    ```

    Notice that the directory on our host file system has the same contents as the one inside the container. That's because that directory is what we see in the container. 

    > Warning: You should NEVER manipulate your container's file system via the Docker host. This is only being done as an academic exercise. 

8. Write a new file to the host file system in the `UpperDir`, and list the directory to see the contents

    ```
    $ cd $(docker inspect -f {{.GraphDriver.Data.UpperDir}} debian)

    $ touch test-file2

    $ ls
    root        test-file   test-file2
    ```


9. Move back into your Debian container and list the root file system

    ```
    $ docker attach debian

    root@674d7abf10c6:/# ls
    bin   dev  home  lib64  mnt  proc  run   srv  test-file   tmp  var
    boot  etc  lib   media  opt  root  sbin  sys  test-file2  usr
    ```
    
    The file that was created on the local host filesystem (`test-file2`) is now available in the container as well. 

10. Type `exit` to stop your container, which will also stop it

    ```
    root@674d7abf10c6:/# exit
    exit
    ```

11. Ensure that your debian container still exists

    ```
    $ docker container ls --all
    CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS           PORTS               NAMES
    674d7abf10c6        debian:jessie       "bash"              36 minutes ago      Exited (0) 2 minutes ago                       debian
    ```

12. List out the current directory

    ```
    $ ls
    root        test-file   test-file2
    ```

    Because the container still exists, the files are still available on  your file system. At this point you could `docker start` your container and it would be just as it was before you exited. 

    However, if we remove the container, the directories on the host file system will be removed, and your changes will be gone

13. Remove the container and list the directory contents

    ```
    $ docker container rm debian
    debian

    $ ls
    ```

    The files that were created are now gone. You've actually been left in a sort of "no man's land" as the directory you're in has actually been deleted as well.

14. Copy the directory location from the prompt in the terminal. 

15. CD back to your home directory

    ```
    $ cd
    ```

16. Attempt to list the contents of the old `UpperDir` directory.

    ```
    $ ls /var/lib/docker/overlay2/0dad4d523351851af4872f8c6706fbdf36a6fa60dc7a29fff6eb388bf3d7194e/diff
    ls: /var/lib/docker/overlay2/0dad4d523351851af4872f8c6706fbdf36a6fa60dc7a29fff6eb388bf3d7194e/diff: No such file or directory
    ```

## Understanding Docker Volumes

[Docker volumes](https://docs.docker.com/engine/admin/volumes/volumes/) are directories on the host file system that are not managed by the storage driver. Since they are not managed by the storage drive they offer a couple of important benefits. 

* **Performance**: Because the storage driver has to create the logical filesystem in the container from potentially many directories on the local host, accessing data can be slow. Especially if there is a lot of write activity to that container. In fact you should try and minimize the amount of writes that happen to the container's filesystem, and instead direct those writes to a volume

* **Persistence**: Volumes are not removed when the container is deleted. They exist until explicitly removed. This means data written to a volume can be reused by other containers. 

Volumes can be anonymous or named. Anonymous volumes have no way for the to be explicitly referenced. They are almost exclusively used for performance reasons as you cannot persist data effectively with anonymous volumes. Named volumes can be explicitly referenced so they can be used to persist data and increase performance. 

The next sections will cover both anonymous and named volumes. 

> Special Note: These next sections were adapted from [Arun Gupta's](https://twitter.com/arungupta) excellent [tutorial](http://blog.arungupta.me/docker-mysql-persistence/) on persisting data with MySQL. 

### Anonymous Volumes

If you once again look at the MySQL [Dockerfile](https://github.com/docker-library/mysql/blob/0590e4efd2b31ec794383f084d419dea9bc752c4/5.7/Dockerfile) you will find the following line:

```
VOLUME /var/lib/mysql
```

This line sets up an anonymous volume in order to increase database performance by avoiding sending a bunch of writes through the Docker storage driver.

Note: An anonymous volume is a volume that hasn't been explicitly named. This means that it's extremely difficult to use the volume later with a new container. Named volumes solve that problem, and will be covered later in this section. 


1. Start a MySQL container

    ```
    $ docker run --name mysqldb -e MYSQL_USER=mysql -e MYSQL_PASSWORD=mysql -e MYSQL_DATABASE=sample -e MYSQL_ROOT_PASSWORD=supersecret -d mysql
    acf185dc16e274b2f332266a1bfc6d1df7d7b4f780e6a7ec6716b40cafa5b3c3
    ```

    When we start the container the anonymous volume is created:

2. Use Docker inspect to view the details of the anonymous volume


    ```
    $ docker inspect -f 'in the {{.Name}} container {{(index .Mounts 0).Destination}} is mapped to {{(index .Mounts 0).Source}}' mysqldb
    in the /mysqldb container /var/lib/mysql is mapped to /var/lib/docker/volumes/cd79b3301df29d13a068d624467d6080354b81e34d794b615e6e93dd61f89628/_data
    ```

3. Change into the volume directory on the local host file system and list the contents

    ```
    $ cd $(docker inspect -f '{{(index .Mounts 0).Source}}' mysqldb)

    $ ls
    auto.cnf            ib_buffer_pool      mysql               server-cert.pem
    ca-key.pem          ib_logfile0         performance_schema  server-key.pem
    ca.pem              ib_logfile1         private_key.pem     sys
    client-cert.pem     ibdata1             public_key.pem
    client-key.pem      ibtmp1              sample
    ```

    Notice the the directory name starts with `/var/lib/docker/volumes/` whereas for directories managed by the Overlay2 storage driver it was `/var/lib/docker/overlay2`

    As mentined anonymous volumes will not persist data between containers, they are almost always used to increase performance. 

4. Shell into your running MySQL container and log into MySQL

    ```
    $ docker exec --tty --interactive mysqldb bash

    root@132f4b3ec0dc:/# mysql --user=mysql --password=mysql
    mysql: [Warning] Using a password on the command line interface can be insecure.
    Welcome to the MySQL monitor.  Commands end with ; or \g.
    Your MySQL connection id is 3
    Server version: 5.7.19 MySQL Community Server (GPL)

    Copyright (c) 2000, 2017, Oracle and/or its affiliates. All rights reserved.

    Oracle is a registered trademark of Oracle Corporation and/or its
    affiliates. Other names may be trademarks of their respective
    owners.

    Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
    ```

5. Create a new table

    ```
    mysql> show databases;
    +--------------------+
    | Database           |
    +--------------------+
    | information_schema |
    | sample             |
    +--------------------+
    2 rows in set (0.00 sec)

    mysql> connect sample;
    Connection id:    4
    Current database: sample

    mysql> show tables;
    Empty set (0.00 sec)

    mysql> create table user(name varchar(50));
    Query OK, 0 rows affected (0.01 sec)

    mysql> show tables;
    +------------------+
    | Tables_in_sample |
    +------------------+
    | user             |
    +------------------+
    1 row in set (0.00 sec)
    ```

6. Exit MySQL and the MySQL container. 

    ```
    mysql> exit
    Bye

    root@132f4b3ec0dc:/# exit
    exit
    ```

7. Stop the container and restart it

    ```
    $ docker stop mysqldb
    mysqldb

    $ docker start mysqldb
    mysqldb
    ```

8. Shell back into the running container and log into MySQL

    ```
    $ docker exec --interactive --tty mysqldb bash

    root@132f4b3ec0dc:/# mysql --user=mysql --password=mysql
    mysql: [Warning] Using a password on the command line interface can be insecure.
    Welcome to the MySQL monitor.  Commands end with ; or \g.
    Your MySQL connection id is 3
    Server version: 5.7.19 MySQL Community Server (GPL)

    Copyright (c) 2000, 2017, Oracle and/or its affiliates. All rights reserved.

    Oracle is a registered trademark of Oracle Corporation and/or its
    affiliates. Other names may be trademarks of their respective
    owners.

    Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
    ```

9. Ensure the table created previously table still exists

    ```
    mysql> connect sample;
    Reading table information for completion of table and column names
    You can turn off this feature to get a quicker startup with -A

    Connection id:    4
    Current database: sample

    myslq> show tables;
    +------------------+
    | Tables_in_sample |
    +------------------+
    | user             |
    +------------------+
    1 row in set (0.00 sec)
    ```

10. Exit MySQL and the MySQL container. 

    ```
    mysql> exit
    Bye

    root@132f4b3ec0dc:/# exit
    exit
    ```

    The table persisted across container restarts, which is to be expected. In fact, it would have done this whether or not we had actually used a volume as shown in the previous section. 

11. Let's look at the volume again

    ```
    $ docker inspect -f 'in the {{.Name}} container {{(index .Mounts 0).Destination}} is mapped to {{(index .Mounts 0).Source}}' mysqldb
    in the /mysqldb container /var/lib/mysql is mapped to /var/lib/docker/volumes/cd79b3301df29d13a068d624467d6080354b81e34d794b615e6e93dd61f89628/_data
    ```

    We do see the volume was not affected by the container restart either. 

    Where people often get confused is in expecting that the anonymous volume can be used to persist data BETWEEN containers. 

    To examine that delete the old container, create a new one with the same command, and check to see if the table exists. 

12. Remove the current MySQL container

    ```
    $ docker container rm --force mysqldb
    mysqldb
    ```

13. Start a new container with the same command that was used before

    ```
    $ docker run --name mysqldb -e MYSQL_USER=mysql -e MYSQL_PASSWORD=mysql -e MYSQL_DATABASE=sample -e MYSQL_ROOT_PASSWORD=supersecret -d mysql
    eb15eb4ecd26d7814a8da3bb27cee1a23304fab1961358dd904db37c061d3798
    ```

14. List out the volume details for the new container

    ```
    $ docker inspect -f 'in the {{.Name}} container {{(index .Mounts 0).Destination}} is mapped to {{(index .Mounts 0).Source}}' mysqldb
    in the /mysqldb container /var/lib/mysql is mapped to /var/lib/docker/volumes/e0ffdc6b4e0cfc6e795b83cece06b5b807e6af1b52c9d0b787e38a48e159404a/_data
    ```

    Notice this directory is different than before. 

15. Shell back into the running container and log into MySQL

    ```
    $ docker exec --interactive --tty mysqldb bash

    root@132f4b3ec0dc:/# mysql --user=mysql --password=mysql
    mysql: [Warning] Using a password on the command line interface can be insecure.
    Welcome to the MySQL monitor.  Commands end with ; or \g.
    Your MySQL connection id is 3
    Server version: 5.7.19 MySQL Community Server (GPL)

    Copyright (c) 2000, 2017, Oracle and/or its affiliates. All rights reserved.

    Oracle is a registered trademark of Oracle Corporation and/or its
    affiliates. Other names may be trademarks of their respective
    owners.

    Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
    ```

16. Check to see if table created previously table still exists

    ```
    mysql> connect sample;
    Connection id:    4
    Current database: sample

    mysql> show tables;
    Empty set (0.00 sec)
    ```

17. Exit MySQL and the MySQL container. 

    ```
    mysql> exit
    Bye

    root@132f4b3ec0dc:/# exit
    exit
    ```

18. Remove the container

    ```
    docker container rm --force mysqldb
    mysqldb
    ```

So while a volume was used to store the new table in the original container, because it wasn't a named volume the data could not be persisted between containers. 

To achieve persistence a named volume should be used.

### Named Volumes

A named volume (as the name implies) is a volume that's been explicitly named and can easily be referenced. 

A named volume can be create on the command line, in a docker-compose file, and when you start a new container. They [CANNOT be created as part of the image's dockerfile](https://github.com/moby/moby/issues/30647). 

1. Start a MySQL container with a named volume (`dbdata`)

    ```
    $ docker run --name mysqldb \
    -e MYSQL_USER=mysql \
    -e MYSQL_PASSWORD=mysql \
    -e MYSQL_DATABASE=sample \
    -e MYSQL_ROOT_PASSWORD=supersecret \
    --detach \
    --mount type=volume,source=mydbdata,target=/var/lib/mysql \
    mysql
    ```

    Because the newly created volume is empty, Docker will copy over whatever existed in the container at `/var/lib/mysql` when the container starts. 

    Docker volumes are primatives just like images and containers. As such, they can be listed and removed in the same way. 

2. List the volumes on the Docker host

    ```
    $ docker volume ls
    DRIVER              VOLUME NAME
    local               55c322b9c4a644a5284ccb5e4d7b6b466a0534e26d57c9ef4221637d39cf9a88
    local               cc44059d23e0a914d4390ea860fd35b2acdaa480e83c025fb381da187b652a66
    local               e0ffdc6b4e0cfc6e795b83cece06b5b807e6af1b52c9d0b787e38a48e159404a
    local               mydbdata
    ```

3. Inspect the volume

    ```
    $ docker inspect mydbdata
    [
        {
            "CreatedAt": "2017-10-13T19:55:10Z",
            "Driver": "local",
            "Labels": null,
            "Mountpoint": "/var/lib/docker/volumes/mydbdata/_data",
            "Name": "mydbdata",
            "Options": {},
            "Scope": "local"
        }
    ]
    ```

    Any data written to `/var/lib/mysql` in the container will be rerouted to `/var/lib/docker/volumes/mydbdata/_data` instead. 

4. Shell into your running MySQL container and log into MySQL

    ```
    $ docker exec --tty --interactive mysqldb bash

    root@132f4b3ec0dc:/# mysql --user=mysql --password=mysql
    mysql: [Warning] Using a password on the command line interface can be insecure.
    Welcome to the MySQL monitor.  Commands end with ; or \g.
    Your MySQL connection id is 3
    Server version: 5.7.19 MySQL Community Server (GPL)

    Copyright (c) 2000, 2017, Oracle and/or its affiliates. All rights reserved.

    Oracle is a registered trademark of Oracle Corporation and/or its
    affiliates. Other names may be trademarks of their respective
    owners.

    Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
    ```

5. Create a new table

    ```
    mysql> connect sample;
    Connection id:    4
    Current database: sample

    mysql> show tables;
    Empty set (0.00 sec)

    mysql> create table user(name varchar(50));
    Query OK, 0 rows affected (0.01 sec)

    mysql> show tables;
    +------------------+
    | Tables_in_sample |
    +------------------+
    | user             |
    +------------------+
    1 row in set (0.00 sec)
    ```

6. Exit MySQL and the MySQL container. 

    ```
    mysql> exit
    Bye

    root@132f4b3ec0dc:/# exit
    exit
    ```

7. Remove the MySQL container

    ```
    $ docker container rm --force mysqldb
    ```

    Because the MySQL was writing out to a named volume, we can start a new container with the same data. 

    When the container starts it will not overwrite existing data in a volume. So the data created in the previous steps will be left intact and mounted into the new container. 

8. Start a new MySQL container

    ```
    $ docker run --name new_mysqldb \
    -e MYSQL_USER=mysql \
    -e MYSQL_PASSWORD=mysql \
    -e MYSQL_DATABASE=sample \
    -e MYSQL_ROOT_PASSWORD=supersecret \
    --detach \
    --mount type=volume,source=mydbdata,target=/var/lib/mysql \
    mysql
    ```

9. Shell into your running MySQL container and log into MySQL

    ```
    $ docker exec --tty --interactive new_mysqldb bash

    root@132f4b3ec0dc:/# mysql --user=mysql --password=mysql
    mysql: [Warning] Using a password on the command line interface can be insecure.
    Welcome to the MySQL monitor.  Commands end with ; or \g.
    Your MySQL connection id is 3
    Server version: 5.7.19 MySQL Community Server (GPL)

    Copyright (c) 2000, 2017, Oracle and/or its affiliates. All rights reserved.

    Oracle is a registered trademark of Oracle Corporation and/or its
    affiliates. Other names may be trademarks of their respective
    owners.

    Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
    ```

10. Check to see if the previously created table exists in your new container.

    ```
    mysql> connect sample;
    Reading table information for completion of table and column names
    You can turn off this feature to get a quicker startup with -A

    Connection id:    4
    Current database: sample

    mysql> show tables;
    +------------------+
    | Tables_in_sample |
    +------------------+
    | user             |
    +------------------+
    1 row in set (0.00 sec)
    ```

    The data will exist until the volume is explicitly deleted. 

11. Exit MySQL and the MySQL container. 

    ```
    mysql> exit
    Bye

    root@132f4b3ec0dc:/# exit
    exit
    ```

12. Remove the new MySQL container and volume

    ```
    $ docker container rm --force new_mysqldb
    new_mysqldb

    $ docker volume rm mydbdata
    mydbdata
    ```

    If a new container was started with the previous command, it would create a new empty volume. 
