# Docker 101 (Linux)

We'll start with the basics and get a feel for running Docker on Linux.

## PreRequisites

Before we start you'll need to gain access to your Linux VM, clone a GitHub repo, and make sure you have a DockerID

### Access your Linux VM
1. Visit [Play With Docker](https://hybrid.play-with-docker.com)
2. Click `Start Session` 
2. On the left click `+ Add New Instance`

### Clone the Lab GitHub Repo
All of the exercises will be done in the console window on the right of ths screen. 

1. In the Linux VM clone the demo GitHub repo

```
git clone https://github.com/mikegcoleman/docker101-linux
```

### Make sure you have a DockerID

If you do not have a DockerID (a free login used to access Docker Cloud, Docker Store, and Docker Hub), please visit [Docker Cloud](https://cloud.docker.com) to register for one. 

## Steps

* [1. Run some simple Docker containers](#1)
* [2. Explore the filesystem and users in Windows containers](#2)
* [3. Package and run a custom app using Docker](#3)

## <a name="1"></a>Step 1. Run some simple Windows Docker containers

There are different ways to use containers:

1. In the background for long-running services like websites and databases
2. Interactively for connecting to the container like a remote server
3. To run a single task, which could be a shell script or a custom app

In this section you'll try each of those options and see how Docker manages the workload.

## Run a task in an Alpine Linux container

This is the simplest kind of container to start with. In PowerShell run:

```
docker container run alpine hostname
Unable to find image 'alpine:latest' locally
latest: Pulling from library/alpine
88286f41530e: Pull complete
Digest: sha256:f006ecbb824d87947d0b51ab8488634bf69fe4094959d935c0c103f4820a417d
Status: Downloaded newer image for alpine:latest
888e89a3b36b
```

You'll see the output written from the `hostname` command. 

Docker keeps a container running as long as the process it started inside the container is still running. In this case the `hostname` process completes when the output is written, so the container stops. The Docker platform doesn't delete resources by default, so the container still exists. 

List all containers and you'll see your Alpine Linux container in the `Exited` state:

```
> docker container ls --all
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS            PORTS               NAMES
888e89a3b36b        alpine              "hostname"          50 seconds ago      Exited (0) 49 seconds ago                       awesome_elion
```

> Note that the container ID *is* the hostname that the container displayed. In the example above it's `888e89a3b36b`

Containers which do one task and then exit can be very useful. You could build a Docker image executes a script to configure some component. Anyone can execute that task just by running the container - they don't need the actual scripts or configuration information - they just need to pull the Docker image


## Run an interactive an Ubuntu container

You can run a container that is a different version of Linux than the version running on the the host serving up your containers. For instance we are running Alpine to host these labs, but we're goint to start an Ubuntu container. 

```
docker container run --interactive --tty --rm ubuntu bash
```

When the container starts you'll drop into the bash shell with the default prompt `root@<container id>:/#`. Docker has attached to the shell in the container, relaying input and output between your local session and the shell session in the container.

Run some commands to see how the Windows Server Core image is built:

- `ls /` - lists the contents of the root directory
- `ps aux` - shows all running processes in the container. 
- `cat /etc/issue` - shows which linux distro you are running, in this case Ubuntu 16.04 LTS

Now run `exit` to leave the shell session, which stops the container process. Using the `--rm` flag means Docker now removes that container (if you run `docker container ls --all` again, you won't see the Ubuntu container).

For fun let's check the version of our host VM

```
cat /etc/issue
Welcome to Alpine Linux 3.6
Kernel \r on an \m (\l)
```

Notice that our host VM is Alpine, yet we were able to run an Ubuntu container. With Linux containers the OS of the container does not need to match the OS of the Docker host it is running on. 

Interactive containers are useful when you are putting together your own image. You can run a container and verify all the steps you need to deploy your app, and capture them in a Dockerfile. 

> You *can* [commit](https://docs.docker.com/engine/reference/commandline/commit/) a container to make an image from it - but you should avoid that wherever possible. It's much better to use a repeatable [Dockerfile](https://docs.docker.com/engine/reference/builder/) to build your image. You'll see that shortly.


## Run a background SQL Server container

Background containers are how you'll run most applications. Here's a simple example using MySQL.

Let's run MySQL in the background as a detached container and name the running container `mydb`. We'll also use an environment variable to set the root password (NOTE: This should never be done in production):

```
docker container run \
  --detach \
  --name mydb \
  -e MYSQL_ROOT_PASSWORD=my-secret-pw \
  mysql:latest
Unable to find image 'mysql:latest' locallylatest: Pulling from library/mysql
aa18ad1a0d33: Pull complete
fdb8d83dece3: Pull complete
75b6ce7b50d3: Pull complete
ed1d0a3a64e4: Pull complete
8eb36a82c85b: Pull complete
41be6f1a1c40: Pull complete
0e1b414eac71: Pull complete
914c28654a91: Pull complete
587693eb988c: Pull complete
b183c3585729: Pull complete
315e21657aa4: Pull complete
Digest: sha256:0dc3dacb751ef46a6647234abdec2d47400f0dfbe77ab490b02bffdae57846ed
Status: Downloaded newer image for mysql:latest
41d6157c9f7d1529a6c922acb8167ca66f167119df0fe3d86964db6c0d7ba4e0
```

As long as the MySql process keeps running, Docker will keep the container running in the background.

You can check what's happening in your containers by using a couple of built-in Docker commands: `logs` to see the container logs, and `top` to see the processes running in the container

```
docker container logs mydb
<output truncated>
2017-09-29T16:02:58.605004Z 0 [Note] Executing 'SELECT * FROM INFORMATION_SCHEMA.TABLES;' to get a list of tables using the deprecated partition engine. You may use the startup option '--disable-partition-engine-check' to skip this check.
2017-09-29T16:02:58.605026Z 0 [Note] Beginning of list of non-natively partitioned tables
2017-09-29T16:02:58.616575Z 0 [Note] End of list of non-natively partitioned tables
```
```
docker container top mydb
PID                 USER                TIME                COMMAND
2876                999                 0:00                mysqld
```

The MySQL instance is isolated in the container, because no ports have been made available to the host. Traffic can't get into Docker containers from the host, unless ports are explicitly published. You can't connect an external client. Other containers in the same Docker network can access the SQL Server container, and you can run commands inside the container through Docker.

List the MYSQL version

```
docker exec -it mydb \
  mysql \
  --user=root \
  --password=$MYSQL_ROOT_PASSWORD \
  --version
mysql: [Warning] Using a password on the command line interface can be insecure.
mysql  Ver 14.14 Distrib 5.7.19, for Linux (x86_64) using  EditLine wrapper
```

## <a name="2"></a>Step 2: Package and run a custom app using Docker

Next you'll learn how to package your own apps as Docker images, using a [Dockerfile](https://docs.docker.com/engine/reference/builder/). 

The Dockerfile syntax is straightforward. In this task you'll walk through two Dockerfiles which package websites to run in Docker containers. The first example is very simple, and the second is more involved. By the end of this task you'll have a good understanding of the main Dockerfile instructions.


## Build a simple website image

Have a look at the, which builds a simple website that allows you to send a tweet. 

```
FROM nginx:latest

COPY index.html /usr/share/nginx/html
COPY linux.png /usr/share/nginx/html

EXPOSE 80 443 	

CMD ["nginx", "-g", "daemon off;"]
```

- [FROM](https://docs.docker.com/engine/reference/builder/#from) specifes the image to use as ther starting point for this image. For this example we'll use `nginx` 
- [COPY](https://docs.docker.com/engine/reference/builder/#copy) copies files from the host into the image, at a known location. In our case it copies `index.html` and a graphic that will be used on our webpage.
- [EXPOSE](https://docs.docker.com/engine/reference/builder/#expose) instructs Docker to make certain networking ports available on the container. 
- [CMD](https://docs.docker.com/engine/reference/builder/#cmd) specifies what command to run when our container starts up. Notice that we can specify the command, as well as command line parameters

We will use the Docker build command to create our image. We pass it a name for our image (or a tag) with the `--tag` parameter and then the `.` instructs Docker to use the current directory

```
cd ~/docker101-linux/linux_tweet_app
docker image build --tag <your docker cloud id>/linux_tweet_app .
Sending build context to Docker daemon  32.77kB
Step 1/5 : FROM nginx:latest
latest: Pulling from library/nginx
afeb2bfd31c0: Pull complete
7ff5d10493db: Pull complete
d2562f1ae1d0: Pull complete
Digest: sha256:af32e714a9cc3157157374e68c818b05ebe9e0737aac06b55a09da374209a8f9
Status: Downloaded newer image for nginx:latest
 ---> da5939581ac8
Step 2/5 : COPY index.html /usr/share/nginx/html
 ---> eba2eec2bea9
Step 3/5 : COPY linux.png /usr/share/nginx/html
 ---> 4d080f499b53
Step 4/5 : EXPOSE 80 443
 ---> Running in 47232cb5699f
 ---> 74c968a9165f
Removing intermediate container 47232cb5699f
Step 5/5 : CMD nginx -g daemon off;
 ---> Running in 4623761274ac
 ---> 12045a0df899
Removing intermediate container 4623761274ac
Successfully built 12045a0df899
Successfully tagged <your dockerid>/linux_tweet_app:latest
```

> **NOTE**: Make sure you change the `build` line to include your actual docker id. For example if your Docker ID is `janedoe` the build line would be:
> 
> `docker image build --tag janedoe/linux_tweet_app .`


The output shows Docker executing each instruction in the Dockerfile, and tagging the final image with your Docker ID.

Run your website using a detached container, just like you did with MySQL, but this time publishing the HTTP port so traffic can be passed from the host into the container:

```
docker container run --detach --publish 80:80 --name app <your docker cloud id>/linux_tweet_app
```

Any external traffic coming into the server on port 80 will now be directed into the container. When you're connected to the host, to browse the website you need to fetch the IP address of the container:

Play With Docker should display an `80` at the top of the page, which you can click to access your website.


## Modify a running website

For the next example, the [Dockerfile](part-1/tweet-app/Dockerfile) is a better representation of a real-world script. These are the main features:

- it is based [FROM](https://docs.docker.com/engine/reference/builder/#from) `microsoft/iis:windowsservercore`, so the image will start with a clean Windows Server 2016 deployment, with IIS already installed
- it uses the [SHELL](https://docs.docker.com/engine/reference/builder/#shell) instruction to switch to PowerShell when building the Dockerfile, so the commands to run are all in PowerShell
- it configures IIS to write all log output to a single file, using the `Set-WebConfigurationProperty` cmdlet
- it copies the [start.ps1](tweet-app/start.ps1) startup script and [index.html](tweet-app/index.html) files from the host into the image
- it specifies `start.ps1` as the [ENTRYPOINT](https://docs.docker.com/engine/reference/builder/#entrypoint) to run when containers start. The script starts the IIS Windows Service and relays the log file entries to the console
- it adds a [HEALTHCHECK](https://docs.docker.com/engine/reference/builder/#healthcheck) which makes an HTTP GET request to the site and returns whether it got a 200 response code

To build the Dockerfile, change to the `tweet-app` directory and run the `build` command:

```
cd "$env:workshop\part-1\tweet-app"
docker image build --tag "$env:dockerId/tweet-app" .
```

You'll see output on the screen as Docker runs each instruction in the Dockerfile. Once it's built you'll see a `Successfully built...` message. If you repeat the `docker image build` command again, it will complete in seconds. That's because Docker caches the [image layers](https://docs.docker.com/engine/userguide/storagedriver/imagesandcontainers/) and only runs instructions if the Dockerfile has changed since the cached version.

When the build completes, run the new app in the same way:

```
docker container run --detach --publish 8080:80 --name tweet-app "$env:dockerId/tweet-app"
```

Find the container IP address and browse to it:

```
$ip = docker container inspect --format '{{ .NetworkSettings.Networks.nat.IPAddress }}' tweet-app
firefox "http://$ip"
```

Feel free to hit the Tweet button, sign in and share your workshop progress :)


## Push your images to Docker Hub

Now if you list the images and filter on your Docker ID, you'll see the images you've built today, with the newest at the top:

```
> docker image ls -f reference="$env:dockerId/*"
REPOSITORY              TAG                 IMAGE ID            CREATED             SIZE
sixeyed/tweet-app       latest              0643d4c6116f        2 minutes ago       10.7GB
sixeyed/hostname-app    latest              f2ae4ec014c4        28 minutes ago      11.5GB
```

Those images are only stored in the cache on your Azure VM, and that VM will be deleted after the workshop. Next we'll push the images to a public repository so you can run them from any Windows machine with Docker.

Distribution is built into the Docker platform. You can build images locally and push them to a public or private [registry](https://docs.docker.com/registry/), making them available to other users. Anyone with access can pull that image and run a container from it. The behavior of the app in the container will be the same for everyone, because the image contains the fully-configured app - the only requirements to run it are Windows and Docker.

[Docker Hub](https://hub.docker.com) is the public registry for Docker images. You've already logged in using `docker login`, so now upload your images to the Hub:

```
docker image push $env:dockerId/hostname-app
docker image push $env:dockerId/tweet-app
```

You'll see the upload progress for each layer in the Docker image. The `hostname-app` image uplads quickly as it only adds one small layer on top of Microsoft's ASP.NET image. The `tweet-app` image takes longer to push - there are more layers, and the configured IIS layer runs to 40MB. 

The logical size of those images is over 10GB each, but the bulk of that is in the Windows Server Core base image. Those layers are already stored in Docker Hub, so they don't get uploaded - only the new parts of the image get pushed. And Docker shares layers between images, so every image that builds on Windows Server Core will share the cached layers for that image.

You can browse to *https://hub.docker.com/r/<your docker id>/* and see your newly-pushed Docker images. These are public repositories, so anyone can pull the image - you don't even need a Docker ID to pull public images.


```