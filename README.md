# Docker 101 (Linux)

In this lab take a look at some basic Docker commands and a simple build-ship-run workflow. 

## PreRequisites

Before we start you'll need to gain access to your Linux VM, clone a GitHub repo, and make sure you have a DockerID

### Access your Linux VM
1. Visit [Play With Docker](https://labs.play-with-docker.com)
2. Click `Start Session` 
2. On the left click `+ Add New Instance`

### Clone the Lab GitHub Repo
All of the exercises will be done in the console window on the right of ths screen. 

1. In the Linux console clone the demo GitHub repo

```
	git clone https://github.com/mikegcoleman/docker101-linux
```

### Make sure you have a DockerID

If you do not have a DockerID (a free login used to access Docker Cloud, Docker Store, and Docker Hub), please visit [Docker Cloud](https://cloud.docker.com) to register for one. 

## Table of Contents

* [Task 1: Run some simple Docker containers](#1)
* [Task 2: Package and run a custom app using Docker](#2)
* [Task 3: Modify a Running Website](#3)

## <a name="1"></a>Step 1. Run some simple Docker containers

There are different ways to use containers:

1. In the background for long-running services like websites and databases
2. Interactively for connecting to the container like a remote server
3. To run a single task, which could be a shell script or a custom app

In this section you'll try each of those options and see how Docker manages the workload.

### Run a task in an Alpine Linux container

This is the simplest kind of container to start with. In your Linux console run:

```
docker container run alpine hostname
```

You will see some output that a local Docker image can't be found locally, so one is pulled. And after the image is pulled the hostname will be displayed (`888e89a3b36b` in our example below)

```
Unable to find image 'alpine:latest' locally
latest: Pulling from library/alpine
88286f41530e: Pull complete
Digest: sha256:f006ecbb824d87947d0b51ab8488634bf69fe4094959d935c0c103f4820a417d
Status: Downloaded newer image for alpine:latest
888e89a3b36b
```

Docker keeps a container running as long as the process it started inside the container is still running. In this case the `hostname` process completes when the output is written, so the container stops. The Docker platform doesn't delete resources by default, so the container still exists. 

List all containers and you'll see your Alpine Linux container in the `Exited` state:

```
docker container ls --all
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS            PORTS               NAMES
888e89a3b36b        alpine              "hostname"          50 seconds ago      Exited (0) 49 seconds ago                       awesome_elion
```

> Note that the container ID *is* the hostname that the container displayed. In the example above it's `888e89a3b36b`

Containers which do one task and then exit can be very useful. You could build a Docker image executes a script to configure some component. Anyone can execute that task just by running the container - they don't need the actual scripts or configuration information - they just need to pull and run the Docker image


### Run an interactive an Ubuntu container

You can run a container that is a different version of Linux than the version running on the the host serving up your containers. For instance we are running Alpine to host these labs, but we're going to start an Ubuntu container. 

```
docker container run --interactive --tty --rm ubuntu bash
```

When the container starts you'll drop into the bash shell with the default prompt `root@<container id>:/#`. Docker has attached to the shell in the container, relaying input and output between your local session and the shell session in the container.

Run some commands:

- `ls /` - lists the contents of the root directory
- `ps aux` - shows all running processes in the container. 
- `cat /etc/issue` - shows which linux distro you are running, in this case Ubuntu 16.04 LTS

Now run `exit` to leave the shell session, which stops the container process. Using the `--rm` flag when we started up the container means Docker now removes that container (if you run `docker container ls --all` again, you won't see the Ubuntu container).

For fun let's check the version of our host VM

```
cat /etc/issue
Welcome to Alpine Linux 3.6
Kernel \r on an \m (\l)
```

Notice that our host VM is Alpine, yet we were able to run an Ubuntu container. As previously mentioned, with Linux containers the OS of the container does not need to match the OS of the Docker host it is running on. 

Interactive containers are useful when you are putting together your own image. You can run a container and verify all the steps you need to deploy your app, and capture them in a Dockerfile. 

> You *can* [commit](https://docs.docker.com/engine/reference/commandline/commit/) a container to make an image from it - but you should avoid that wherever possible. It's much better to use a repeatable [Dockerfile](https://docs.docker.com/engine/reference/builder/) to build your image. You'll see that shortly.


### Run a background MySQL container

Background containers are how you'll run most applications. Here's a simple example using MySQL.

Let's run MySQL in the background as a detached container (`--detach`) and name the running container `mydb`. We'll also use an environment variable to set the root password (NOTE: This should never be done in production):

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

The MySQL instance is isolated in the container, because no ports have been made available to the host. Traffic can't get into Docker containers from the host, unless ports are explicitly published. You can't connect an external client. Other containers in the same Docker network can access the MySQL container, and you can run commands inside the container through Docker. 

List the MYSQL version using `docker exec`:

```
docker exec -it mydb \
  mysql \
  --user=root \
  --password=$MYSQL_ROOT_PASSWORD \
  --version
mysql: [Warning] Using a password on the command line interface can be insecure.
mysql  Ver 14.14 Distrib 5.7.19, for Linux (x86_64) using  EditLine wrapper
```

## <a name="2"></a>Task 2: Package and run a custom app using Docker

Next you'll learn how to package your own apps as Docker images, using a [Dockerfile](https://docs.docker.com/engine/reference/builder/). 

The Dockerfile syntax is straightforward. In this task we're going to create an NGINX website from a Dockerfile.

### Build a simple website image

Let's have a look at the folowing Dockerfile, which builds a simple website that allows you to send a tweet. 

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

We will use the Docker build command to create our image. We pass it a name for our image (or a tag) with the `--tag` parameter (which includes a version number `1.0`) and then the `.` instructs Docker to use the current directory

Make sure you're in the right directory:

```
cd ~/docker101-linux/linux_tweet_app
```

Then issue the `docker image build` commmand to create a new Docker image. 

```
docker image build --tag <your docker cloud id>/linux_tweet_app:1.0 .

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
> `docker image build --tag janedoe/linux_tweet_app:1.0 .`


The output shows Docker executing each instruction in the Dockerfile, and tagging the final image with your Docker ID.

Run your website using a detached container, just like you did with MySQL, but this time publishing the HTTP port so traffic can be passed from the host into the container:

```
docker container run \
--detach \
--publish 80:80 \
--name linux_tweet_app \
<your docker cloud id>/linux_tweet_app:1.0
```

Any external traffic coming into the server on port 80 will now be directed into the container. When you're connected to the host, to browse the website you need to fetch the IP address of the container:

### Access the running website
Play With Docker should display an `80` at the top of the page, which you can click to access your website.

Once you've accessed your website, let's shut it remove it. 

`docker container rm --force linux_tweet_app`

## <a name="3"></a>Task 3: Modify a running website

If you're actively working on an application it would be inconvenient to have to stop the container, rebuild the image, and run a new version every time you make a change to your source code. 

One way to streamline this process is to mount the source code directory on the local machine into the running container. This will allow any changes made on the local files to be immediately reflected in the container. 

We do this using something called a [bind mount](https://docs.docker.com/engine/admin/volumes/bind-mounts/). When you use a bind mount, a file or directory on the host machine is mounted into a container.

### Start our web app with a bind mount 

Let's start the our web app and mount the current directory into the container. 

> Make sure you're still in the `linux_tweet_app` directory

```
docker container run \
--detach \
--publish 80:80 \
--name linux_tweet_app \
--mount type=bind,source="$(pwd)",target=/usr/share/nginx/html \
<your docker cloud id>/linux_tweet_app:1.0
```

> Remember from our Dockerfile `usr/share/nginx/html` is where are html files are stored for our web app

Click the `80` the top of the screen to verify your website is running.

### Modify the running website

Because we did a bind mount, any changes made to the local file system are immediately reflected in the running container. 

In the real world you would use an IDE to modify the source code, in the case of our lab we're just going to copy over a new `index.html` file. 

`cp index-new.html index.html`

Now go to your running website and refresh the page. Notice that the site has changed. 

> If you are comfortable with `vi` you can use it to load the `index.html` file and make additional changes. Those too would be reflected when you reload the webpage. 

Even though we've modified the local file system and that was reflected in the running container, we've not actually changed the original Docker image, all we've changed is the local copy of the `index.html` file.

To show this, let's stop the current container and re-run the `1.0` image without a bind mount. 

1. Stop and remove the currrently running container

	`docker rm --force linux_tweet_app`

2. Rerun the current version without a bind mount. 

	```
	docker container run \
	--detach \
	--publish 80:80 \
	--name linux_tweet_app \
	<your docker cloud id>/linux_tweet_app:1.0
	```

	Click the `80` in the PWD interface to view the website. Notice it's back to the original version. 

### Save our Changes

To save our changes we need to build a new version of our Docker image that includes the updated `index.html`

1.  Stop and remove the current container

	`docker rm --force linux_tweet_app`


2. Let's build a new version of our docker image to capture our changes. This build will copy in the updated `index.html` file. By using `2.0` in our image tag, we'll create a new version

	```
	docker image build --tag <your docker cloud id>/linux_tweet_app:2.0 .
	Sending build context to Docker daemon  32.77kB
	Step 1/5 : FROM nginx:latest
	 ---> da5939581ac8
	Step 2/5 : COPY index.html /usr/share/nginx/html
	 ---> 930df73e7f34
	Step 3/5 : COPY linux.png /usr/share/nginx/html
	 ---> 61135e5c6147
	Step 4/5 : EXPOSE 80 443
	 ---> Running in bc9208e41196
	 ---> db5be93530c2
	Removing intermediate container bc9208e41196
	Step 5/5 : CMD nginx -g daemon off;
	 ---> Running in 1799d496012e
	 ---> 36f86c329a7d
	Removing intermediate container 1799d496012e
	Successfully built 36f86c329a7d
	Successfully tagged <your dockerid>/linux_tweet_app:2.0
   	```
	
	> Notice how fast that built? It's because Docker only modified the portion of the image that changed vs. rebuilding the whole image. 

2. Let's look at the images on our system

	```
	docker image ls
	REPOSITORY                       	TAG                 IMAGE ID            CREATED             SIZE
	<your dockerid>/linux_tweet_app   	2.0                 36f86c329a7d        32 seconds ago      108MB
	<your dockerid>/linux_tweet_app   	1.0                 d11c8f88a335        12 minutes ago      108MB
	nginx                          		latest              da5939581ac8        2 weeks ago         108MB
	```
	
We now have two versions of our web app. 

### Test the new version	

1. Let's run our new version
	
	```
	docker container run \
	--detach \
	--publish 80:80 \
	--name linux_tweet_app \
	<your docker cloud id>/linux_tweet_app:2.0
	```

2. Click on the `80` to view the updated version

	We can run both versions side by side. The only thing we need to be aware of is that a port can only be exposed once per host. Since we're already using port 80 for the new version, we will tell all requests coming into port 8080 to go to the old version. Additionally, we need to give our container a unique name (`old_linux_tweet_app`)
	
3. Run the old version
	
	```
	docker container run \
	--detach \
	--publish 8080:80 \
	--name old_linux_tweet_app \
	<your docker cloud id>/linux_tweet_app:1.0
	```
4. PWD will add an `8080` at the top of the screen click that to view the old website.

### Push your images to Docker Hub

Now if you list the images and filter on your Docker ID, you'll see the images you've built today, with the newest at the top:

```
> docker image ls -f reference="<your dockerid>/*"
REPOSITORY                        TAG                 IMAGE ID            CREATED             SIZE
<your dockerid>/linux_tweet_app   2.0                 b73b8f06499d        4 seconds ago       108MB
<your dockerid>/linux_tweet_app   1.0                 27659868865b        14 minutes ago      108MB
```

Those images are only stored in the cache on your linux VM, and that VM will be deleted after the workshop. Next we'll push the images to a public repository so you can run them from any Linux machine with Docker.

Distribution is built into the Docker platform. You can build images locally and push them to a public or private [registry](https://docs.docker.com/registry/), making them available to other users. Anyone with access can pull that image and run a container from it. The behavior of the app in the container will be the same for everyone, because the image contains the fully-configured app - the only requirements to run it are Linux and Docker.

[Docker Hub](https://hub.docker.com) is the public registry for Docker images. 

1. Before you can push your images, you will need to log into Docker Hub

	`docker login`

2. Push each version of your web app using `docker push`

	```
	docker image push <your dockerid>/linux_tweet_app:1.0
	The push refers to a repository [docker.io/mikegcoleman/linux_tweet_app]
	910e84bcef7a: Pushed
	1dee161c8ba4: Pushed
	110566462efa: Pushed
	305e2b6ef454: Pushed
	24e065a5f328: Pushed
	1.0: digest: sha256:51e937ec18c7757879722f15fa1044cbfbf2f6b7eaeeb578c7c352baba9aa6dc size: 1363
	```
	
	```
	docker image push <your dockerid>/linux_tweet_app:2.0
	The push refers to a repository [docker.io/mikegcoleman/linux_tweet_app]
	0b171f8fbe22: Pushed
	70d38c767c00: Pushed
	110566462efa: Layer already exists
	305e2b6ef454: Layer already exists
	24e065a5f328: Layer already exists
	2.0: digest: sha256:7c51f77f90b81e5a598a13f129c95543172bae8f5850537225eae0c78e4f3add size: 1363
	```
 
You can browse to *https://hub.docker.com/r/<your docker id>/* and see your newly-pushed Docker images. These are public repositories, so anyone can pull the image - you don't even need a Docker ID to pull public images.
