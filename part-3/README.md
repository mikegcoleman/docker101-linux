# Docker 101 - Linux (Part 3): Docker Swarm and Container Networking

So far all of the previous exercises have been based around running a single container on a single host. 

This section will cover how to use multiple hosts to provide fault tolerance as well as incresed performance. As part of that discussion it will also provide an overview of Docker's multi-host networking capabilities. 

#### What is "Orchestration"

If you heard about containers you've probably heard about orchstration as well. Conatiner orchestrators, such as Docker Swarm and Kubernetes, provide a ton of functionality around managing and deploying containerized applications. But the two most fundamental things they do are clustering and scheduling (that's not all they do by a long shot, but they are arguably the two most important fuctions).

Clustering is the concept of taking a group of machines and treating them as a single computing resource. These machines are capable of accepting any workload becaues they all offer the same capabilities. These clustered machines don't have to be running on the same infrastructure - they could be a  mix of bare metal and VMs for instance.  

Scheduling is the process of deciding where a workload should reside. When an admin starts a new instance of website she can decide what region it needs to go on or if it should be on bare metal or in the cloud. The scheduler will make that happen. Schedulers also make sure that the application maintains its desired state. For example, if there were 10 copies of a web site running, and one of them crashed, the scheduler would know this and start up a new instance to take the failed one's place. 


With Docker ther is a built-in orchestrator: Docker Swarm. It provides both clustering and scheduling as well as many other advanced services. 

The next part of the lab will start with the deployment of a 3 node Docker swarm cluster. 

### Build your cluster

1. In the PWD interface click `+ Add new instance` to instantiate a linux node

2. Repeat step 1 to add a second node to the cluster.

3. Click the `Windows containers` slider and then click `+ Add New Instance`

There are now three standalone Docker hosts: 2 linux and 1 Windows. 

4. In the console for `node1` initialize Docker Swarm

```
$ docker swarm init --advertise-addr eth0
Swarm initialized: current node (ujsz5fd1ozr410x1ywuix7sik) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-53ao1ihu684vpcpjuf332bi4et27qiwxajefyqryyj4o0indm7-b2zc5ldudkxcmf6kcncft8t12 192.168.0.13:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

`node1` is now a Swarm manager node. Manager nodes are responsible for ensuring the integrity of the cluster as well as managing running services. 

5. Copy the `docker swarm join` output from `node1`

6. Switch to `node1` and paste in the command. **Be sure to copy the output from your PWD window, not from this lab guide**

```
$ docker swarm join --token SWMTKN-1-53ao1ihu684vpcpjuf332bi4et27qiwxajefyqryyj4o0indm7-b2zc5ldudkxcmf6kcncft8t12 192.168.0.13:2377
This node joined a swarm as a worker.
```

7. Switch to the Windows node and paste the same command at the Powershell prompt

```
PS C:\> docker swarm join --token SWMTKN-1-53ao1ihu684vpcpjuf332bi4et27qiwxajefyqryyj4o0indm7-b2zc5ldudkxcmf6kcncft8t12 192.168.0
.13:2377
This node joined a swarm as a worker.
```

The three nodes have now been clustered into a single Docker swarm. An important thing to note is that clusters can be made up of Linux nodes, Windows nodes, or a combination of both. 

8. Switch back to `node1`

9. List the nodes in the cluster

```
$ docker node ls
$ docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS
ujsz5fd1ozr410x1ywuix7sik *   node1               Ready               Active              Leader60v5h787wkjbtxs6dkoj1kgqn     node2               Ready               Active
t4zejugh4davz37d6h98wheu6     win00003R           Ready               Active
```

Commands against the swarm can only be issued from the manager node. Attempting to run the above command against `node2` or the Windows node would result in an error. 

```
$ docker node ls
Error response from daemon: This node is not a swarm manager. Worker nodes can't be used to view or modify cluster state. Please run this command on a manager node or promote the current node to a manager.
```

With the Swarm cluster built it's time to move to a discussion on networking. 

Docker supports several different networking options, but this lab will cover the two most popular: bridge and overlay.

Bridge networks are only available on the local host, and can be created on hosts in swarm clusters as well as standalone hosts. However, in a swarm cluster, even though the machines are tied together, bridge networks only work on the host on which they were created. 

Overlay networks faciliate the creation of networks that span Docker hosts. While it's possible to network together hosts that are not in a Swarm cluster, it's a very manual task requiring the addition of an external key value store. With Docker swarm creating overlay networks is trivial. 

### Bridge networking overview

1. On `node1` create a bridge network (`mybridge`)

```
$ docker network create my bridge
52fb9de4ad1cbe505e451599df2cb62c53e56893b0c2b8d9b8715b5e76947551
```

2. List the networks on `node1`

```
$ docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
edf9dc771fc4        bridge              bridge              local
e5702f60b7c9        docker_gwbridge     bridge              local
7d6b733ee498        host                host                local
rnyatjul3qhn        ingress             overlay             swarm
52fb9de4ad1c        mybridge            bridge              local
dbd52ffda3ae        none                null                local
```

The newly created `mybridge` network is listed.

> Note: Docker creates several networks by default, however the purpose of those networks is outside the scope of this workshop. 

3. Switch to `node2`

4. List the available networks on `node2`

```
$ docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
3bc2a78be20f        bridge              bridge              local
641bdc72dc8b        docker_gwbridge     bridge              local
a5ef170a2758        host                host                local
rnyatjul3qhn        ingress             overlay             swarm
3dec80db87e4        none                null                local
```

Notice that the same networks names exist on `node2` but their ID's are different. And, `mybridge` does not show up at all. 

5. Move back to `node`

6. Create an Alpine container named `alpine_host` running the `top` process in the `detached` mode and attach it to the `mybridge` network.

```
$ docker container run \
  --detach \
  --network mybridge \
  --name alpine_host \
  alpine top
5cc5eeaf703b8e469b25725ddea4c6e563c03ca0403c64bf87f32c5ec5a890fe
```
> Note: We run the `top` process to keep the container from exiting as soon as it's created. 

7. Start another Alpine container named `alpine client`

```
$ docker container run \
  --detach \
  --name alpine_client \
  alpine top
c81a3a14f43fed93b6ce2eb10338c1749fde0fe7466a672f6d45e11fb3515536
```

8. Attempt to PING `alpine_host` from `alpine_client`

```
docker exec alpine_client ping alpine_host
ping: bad address 'alpine_host'
```

Because the two containers are not on the same network they cannot see each other. 

9. Inspect `alpine_host` and `alpine_client` to see which networks they are attached to.

```
$ docker inspect -f {{.NetworkSettings.Networks}} alpine_host
map[mybridge:0xc420466000]

$ docker inspect -f {{.NetworkSettings.Networks}} alpine_client
map[bridge:0xc4204420c0]
```

`alpine_host` is, as expected, attached to the `mybridge` network. 

`alpine_client` is attached to the default bridge network `bridge`

10. Stop and remove `alpine_client`

```
$ docker container rm --force alpine_client
alpine_client
```

11. Start another container called `alpine_client` but attach it to the `mybridge` network this time. 

```
docker container run \
  --detach \
  --network mybridge \
  --name alpine_client \
  alpine top
```

12. Verify via `inspect` that `alpine_client` is on the `mybridge` network

```
$ docker inspect -f {{.NetworkSettings.Networks}} alpine_client
map[mybridge:0xc42043e0c0]
```

13. PING `alpine_host` from `apline_client`

```
docker exec alpine_client ping -c 5 alpine_host
PING alpine_host (172.20.0.2): 56 data bytes
64 bytes from 172.20.0.2: seq=0 ttl=64 time=0.102 ms
64 bytes from 172.20.0.2: seq=1 ttl=64 time=0.108 ms
64 bytes from 172.20.0.2: seq=2 ttl=64 time=0.088 ms
64 bytes from 172.20.0.2: seq=3 ttl=64 time=0.113 ms
64 bytes from 172.20.0.2: seq=4 ttl=64 time=0.122 ms

--- alpine_host ping statistics ---
5 packets transmitted, 5 packets received, 0% packet loss
round-trip min/avg/max = 0.088/0.106/0.122 ms
```

Something to notice is that it was not necessary to specify an IP address.  Docker has a built in DNS that resolved `alpine_host` to the correct address. 

Being able to network containers on a single host is not extremely useful. It might be fine for a simple test envrionment, but production environments require the ability provide the scalability and fault tolerance that comes from having multiple interconnected hosts. 

This is where overlay networking comes in

### Overlay Neworking Overview

1. Remove the existing Alpine containers

```
$ docker container rm --force $(docker ps --all --quiet)
e65629beeb57
5cc5eeaf703b
```

2. Create a new overlay network (`-d` specifies the networking driver to use, if it's omitted `bridge` is the default).

```
$ docker network create --attachable -d overlay myoverlay
z16nhzxwbeukjnz3e6nk2159p
```

> Note: We have to use the `--attachable` flag because by default you cannot use `docker run` on overlay networks that are part of a swarm. The preferred method is to use a Docker *service* which is covered later in the workshop.

3. List the networks on the host to verify that the `myoverlay` network was created. 

```
docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
edf9dc771fc4        bridge              bridge              local
e5702f60b7c9        docker_gwbridge     bridge              local
7d6b733ee498        host                host                local
rnyatjul3qhn        ingress             overlay             swarm
52fb9de4ad1c        mybridge            bridge              local
z16nhzxwbeuk        myoverlay           overlay             swarm
dbd52ffda3ae        none                null                local
```

3. Create an Alpine container and attach it to the `myoverlay` network. 

```
$ docker container run \
  --detach \
  --network myoverlay \
  --name alpine_host \
  alpine top
a604aa48660835aeec75f3239964d35c334bcdf33d1b5574c319aaf344c2119a
```

4. Move to `node2`

5. List the available networks

```
$ docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
3bc2a78be20f        bridge              bridge              local
641bdc72dc8b        docker_gwbridge     bridge              local
a5ef170a2758        host                host                local
rnyatjul3qhn        ingress             overlay             swarm
3dec80db87e4        none                null                local
```

Notice anything out of the ordinary? Where's the `myoverlay` network?

Docker won't extend the network to hosts where it's not needed. In this case, there are no containers attached to `myoverlay` on `node2` so the network has not been extended to the host. 

6. Start an alpine container and attach it to `myoverlay`

```
$ docker container run \
  --detach \
  --network myoverlay \
  --name alpine_client \
  alpine top
Unable to find image 'alpine:latest' locally
latest: Pulling from library/alpine
88286f41530e: Pull complete
Digest: sha256:f006ecbb824d87947d0b51ab8488634bf69fe4094959d935c0c103f4820a417d
Status: Downloaded newer image for alpine:latest
5d67e360d8e42c618dc8ea40ecd745280a8002652c7bcdc7982cb5c6cdd4fd13
```

7. List the available networks on `node2`

```
$ docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
3bc2a78be20f        bridge              bridge              local
641bdc72dc8b        docker_gwbridge     bridge              local
a5ef170a2758        host                host                local
rnyatjul3qhn        ingress             overlay             swarm
z2fh5l7g1b4k        myoverlay           overlay             swarm
3dec80db87e4        none                null                local
```

The `myoverlay` network is now available on `node2`

8. Ping `apine_host`

```
$ docker exec alpine_client ping -c 5 alpine_host
PING alpine_host (10.0.0.2): 56 data bytes
64 bytes from 10.0.0.2: seq=0 ttl=64 time=0.244 ms
64 bytes from 10.0.0.2: seq=1 ttl=64 time=0.122 ms
64 bytes from 10.0.0.2: seq=2 ttl=64 time=0.166 ms
64 bytes from 10.0.0.2: seq=3 ttl=64 time=0.201 ms
64 bytes from 10.0.0.2: seq=4 ttl=64 time=0.137 ms
```
Networking also works betwen Linux and Windows nodes

9. Ping `alpine_host` from the Windows node

```
PS C:\> docker container run `
 --network myoverlay `
 --name windows_client `
 --rm `
 microsoft/windowsservercore powershell ping alpine_host

 Pinging alpine_host [10.0.0.2] with 32 bytes of data:
Reply from 10.0.0.2: bytes=32 time<1ms TTL=64
Reply from 10.0.0.2: bytes=32 time<1ms TTL=64
Reply from 10.0.0.2: bytes=32 time<1ms TTL=64
Reply from 10.0.0.2: bytes=32 time<1ms TTL=64
```

> Note: In some cases it may take a few seconds for the Windows client to find the alpine host resutling in PING timeouts. 

## Deploying Swarm services

While we have been using `docker run` to instantiate docker containers on our Swarm cluster, the preferred way to actually run applications is via a  [*service*](https://docs.docker.com/engine/swarm/how-swarm-mode-works/services/).

Services are an abstraction in Docker that represent an application or component of an application. For instance a web front end service connecting to a database backend service. You can deploy an application made up of a single service. In fact, this is quite common when [modernizing traditional applications](https://goto.docker.com/rs/929-FJL-178/images/SB_MTA_04.14.2017.pdf). 

The service construct provides a host of useful features including:

* Load balancing
* Layer 4 and layer 7 routing meshes
* Desired state reconciliation
* Service discovery
* Helthchecks
* Upgrades and rollback
* Scaling

This workshop cannot possibly cover all these topics, but will address several key points. 

This lab will deploy a two service application.  The application features a Java-based web front end running on Linux, and a Microsoft SQL server running on Windows. 

### Deploying a Multi-OS Application with Docker Swarm

1. Move to `node`

2. Create an overlay network for the application

```
$ docker network create -d overlay atseafoqztzic1x95kiuq9cuqwuldi
```

3. Deploy the database service

```
$ docker service create \
  --name database \
  --endpoint-mode dnsrr \
  --network atsea \
  --publish mode=host,target=1433 \
  --detach=true \
 dockersamples/atsea-db:mssql
ywlkfxw2oim67fuf9tue7ndyi
```
The service is created with the following parameters:

* `--name`: Gives the service an easily remembered name
* `--endpoint-mode`: Today all services running on Windows need to be started in DNS round robin mode.
* `--network`: Attaches the containers from our service to the `atsea` network
* `--publish`: Exposes port 1433 but only on the Windows host. 
* `--detach`: Runs the service in the background
* Our service is based off the image `dockersamples/atsea-db:mssql`

4. Check the status of your service

```
$ docker service ps database
docker service ps database
ID                  NAME                IMAGE                          NODE                DESIRED STATE       CURRENT STATE      ERROR               PORTS
rgwtocu21j0f        database.1          dockersamples/atsea-db:mssql   win00003R           Running             Running 3 minutesago
```

5. Check the logs of the running service

```
$ docker service logs database
<output snipped>
database.1.rgwtocu21j0f@win00003R    | VERBOSE: Inserted Product seed data
database.1.rgwtocu21j0f@win00003R    | VERBOSE: Update complete.
database.1.rgwtocu21j0f@win00003R    | VERBOSE: Started SQL Server.
```

6. Start the web front-end service

```
$ docker service create \
 --publish 8080:8080 \
 --network atsea \
 --name appserver \
 --detach=true \
 dockersamples/atsea-appserver:1.0
