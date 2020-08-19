Table of contents
=================
   * [Docked at home, while RocketChatting with a lot of Mongos](#docked-at-home-while-rocketchatting-with-a-lot-of-mongos)
      * [Objectives and method](#objectives-and-method)
   * [Part 1](#part-1)
      * [Step 1: Creating a Rocket.Chat container](#step-1-creating-a-rocketchat-container)
      * [Step 2: Creating 2 instances of RocketChat](#step-2-creating-2-instances-of-rocketchat)
         * [A note on ports](#a-note-on-ports)
      * [Step 3: Adding a load balancer](#step-3-adding-a-load-balancer)
         * [Is load balancing working?](#is-load-balancing-working)
   * [Part 2](#part-2)
      * [Step 4: Creating a ReplicaSet for our database](#step-4-creating-a-replicaset-for-our-database)
      * [Step 5: Creating 3 instances of RocketChat with 3 ReplicaSets each containing 3 nodes](#step-5-creating-3-instances-of-rocketchat-with-3-replicasets-each-containing-3-nodes)
      * [Step 6: Configuring Prometheus for monitoring the cluster](#step-6-configuring-prometheus-for-monitoring-the-cluster)


# Docked at home, while RocketChatting with a lot of Mongos

Recently a friend requested me to work on a project involving Docker, Rocket.Chat and Mongos. Despite the mentioned projects having online and thorough docs, bits of information for achieving a specific implementation can be scattered and hidden throughout the internet and because of that, I thought I could turn this into a tutorial for further reference.

## Objectives and method
The project is divided into two main parts:
1. Setting up 2 RocketChat containers which share the same Mongo database along with a load balancer
2. Constructing 3 Replica Sets for MongoDB, plus setting up Prometheus as the monitoring tool

We would like to achieve this step by step, and build on top of each accomplishment to make the tutorial more effective. What is more, all the steps will be done utilizing `docker-compose`, a fine tool for creating, dependency management and running a group of containers. All the steps would take place on the same machine as the localhost. The files in this repo have been tested on my production machine which runs Gentoo Linux, but they should work on any other environment that is running Docker and includes `docker-compose`.

Each docker-compose yaml file needs to be run alone and does not depend on a former compose file. When testing a new step, first shutdown the former containers if they are running.
  
# Part 1
## Step 1: Creating a Rocket.Chat container

We want to start with setting up a RocketChat container first. [The docker-compose file](https://github.com/RocketChat/Rocket.Chat/blob/develop/docker-compose.yml) that the project itself has provided gives us a good starting point. [docker-compose-step1.yml](docker-compose-step1.yml) in this repository contains the instructions to pull the latest RocketChat image along with a MongoDB one.

Download the file and run the container set (one MongoDB and one RocketChat instance) by 
```sh
docker-compose -f docker-compose-step1.yml up
```

Inside the yaml file, you can find the definition of 3 services. One `rocketchat` instance, one `mongo` container, and `mongo-init-replica` service which has the duty of configuring the DB. It should be noted that starting in versions 1.0, use of MongoDB ReplicaSets is mandatory, hence the `mongo-init-replica` needs to be run to create a minimum ReplicaSet after the Mongo instance has started.

You could also run the group without having to watch the logs, by running in `detached` mode:
```sh
docker-compose -f docker-compose-step1.yml up -d
```

This will create a RocketChat service at [localhost:3000](http://localhost:3000).

## Step 2: Creating 2 instances of RocketChat
We could just copy the RocketChat service and change the name, ports and the mapped file in the volume section, however, a more elegant solution would be automatic scaling of a service in Docker via `--scale` option. The only change that is needed is to replace the `ports:` section with the following
```yml
ports: 
  - "3000-3001:3000"
```
Or even simpler:
```yml
expose:
  - 3000
```
The file provided in this repository i.e., [docker-compose-step2.yml](docker-compose-step2.yml) takes the first approach so that we know at which port we can access our rocketchats.
Download the file `docker-compose-step2.yml` and run it through issuing
```sh
docker-compose -f docker-compose-step2.yml up --scale rocketchat=2 --remove-orphans
```
When you run the file, it will create two instances accessible at [localhost:3000](http://localhost:3000) and [localhost:3001](http://localhost:3001). Another option is the use of `expose:` instead of `ports`. Upon running the script with the `expose` directive, two instances of RocketChat would emerge, the host ports of which would automatically be taken care of. What happens is that Docker would map an ephemeral port in the host to port 3000 in the app for each of the containers without us fiddling with the compose file.

The option `--remove-orphans` is used to remove containers for services not defined in the compose file (if there is any).

#### A note on ports
The `ports` directive gets its fair share of abuse. Remember that the order is `host : container`. You don't need to worry much about the container port since it won't clash with the others because it is an internal parameter. Furthermore, you cannot easily change the internal port the service runs at, by supplying a new one in this section and expecting it to work. It needs to either be supplied in an `environment` variable or through mapping a config file in `volume`. A common symptom of its misuse is getting a `connection reset` when accessing the service.

## Step 3: Adding a load balancer

We will use NGINX reverse proxy with load balancing for this goal. An Nginx container would be added to the group and to have a simpler Nginx configuration file i.e., `reverse_proxy.conf`, we would utilize the `expose` alternative in our RocketChat definition. This way, we won't need to manually introduce the cluster in the `upstream` section of the config file. 
Download the two files [docker-compose-step3.yml](docker-compose-step3.yml) and [reverse_proxy_step3.conf](reverse_proxy_step3.conf) and put them in the same directory from which you run the commands.
Like before, start the container set by
```sh
docker-compose -f docker-compose-step3.yml up --scale rocketchat=2 --remove-orphans
```
Note the use of `--scale` directive. You can access the RocketChat service by directly going to [localhost](http://localhost) and based on the load balancing method (Round Robin by default) Nginx would direct the requests to one of the two RocketChat services running. Our mini cluster has load balancing now!

### Is load balancing working?
Use Apache benchmark tool `ab` and test it.
![The Load Balancing Test](/LB-Results.png)

# Part 2
## Step 4: Creating a ReplicaSet for our database
A Replica Set simply means a group of Mongo processes which share the same database. This allows redundancy and, of course, better performance under load. One of them needs to be the boss a.k.a Primary and the others would be the employees or Secondary (don't worry, they can change positions). A Replica Set is comprised of usually an odd number of processes because an election takes place to choose the Primary node.

It is relatively easy to get into a vicious loop of problems when creating a replica set. To avoid such complications you need to take the following into account:

- Each container that wants to be a node needs to be started with the same replica set option, like `--replset rs0`
- The nodes must be able to communicate with each other i.e., the IPs bound to the service should be added
- After all of the nodes start, only one of them takes the responsibility to initiate a Replica Set.

At this point, there are several possible ways to achieve this. We could start three instances of MongoDB in the yaml file, then connect to one of the nodes e.g. `mongo1` by `docker exec -it <mongo1 container ID or name> /bin/bash`; fire up a mongo shell and issue the `rs.initiate` directive, or we could do it all through the docker-compose file. Of course, the second option is easier and in the provided file for this step, we have taken the easier approach.

To clearly demonstrate the use of replica sets we have explicitly defined 3 separate mongo services. Also, there are notable changes in our yml file. First, you can see that our `MONGO_OPLOG_URL` and `MONGO_URL` options are set to point to the group of replica set pointing to the 3 mongo containers. Second, all of the Mongos have basically the same options in their `command`, but utilize different ports and DB paths. Third, the `mongo-init-replica` is set to run when all the MongoDBs start (via `depends_on` directive) and sets the members of the replica set in only `mongo1` instance by adding them in an `rs.initiate` directive. In addition, RocketChat is set to wait more between its attempts to connect to the Primary node of the Replica Set (through the use of `&w=majority` which hints that we want to connect to the node with "majority" write concern which mean Primary).

Download [docker-compose-step4.yml](docker-compose-step4.yml) and run it by

```sh
docker-compose -f docker-compose-step4.yml up --remove-orphans
```
We can check if the Replica Set has been set up correctly by logging into one the MongoDB instances:
```sh
docker exec -it <a mongo container name or ID> /bin/bash
```
and fire up a mongo shell by `mongo` command and check the Replica Set:
```sh
rs1:SECONDARY> rs.conf()
{
	"_id" : "rs1",
	"version" : 1,
	"protocolVersion" : NumberLong(1),
	"writeConcernMajorityJournalDefault" : true,
	"members" : [
		{
			"_id" : 0,
			"host" : "mongo11:27017",
			"arbiterOnly" : false,
			"buildIndexes" : true,
			"hidden" : false,
			"priority" : 1,
			"tags" : {
				
			},
			"slaveDelay" : NumberLong(0),
			"votes" : 1
		},
		{
			"_id" : 1,
			"host" : "mongo21:27017",
			"arbiterOnly" : false,
			"buildIndexes" : true,
			"hidden" : false,
			"priority" : 1,
			"tags" : {
				
			},
			"slaveDelay" : NumberLong(0),
			"votes" : 1
		},
		{
			"_id" : 2,
			"host" : "mongo31:27017",
			"arbiterOnly" : false,
			"buildIndexes" : true,
			"hidden" : false,
			"priority" : 1,
			"tags" : {
				
			},
			"slaveDelay" : NumberLong(0),
			"votes" : 1
		}
	],
	"settings" : {
		"chainingAllowed" : true,
		"heartbeatIntervalMillis" : 2000,
		"heartbeatTimeoutSecs" : 10,
		"electionTimeoutMillis" : 10000,
		"catchUpTimeoutMillis" : -1,
		"catchUpTakeoverDelayMillis" : 30000,
		"getLastErrorModes" : {
			
		},
		"getLastErrorDefaults" : {
			"w" : 1,
			"wtimeout" : 0
		},
		"replicaSetId" : ObjectId("5f2afdc2026d0376b1301864")
	}
}
rs1:SECONDARY> 
```

Two points to make here. First, note that by default the MongoDB instances are bound to localhost and we did not need to necessarily use `--bind_ip` command line option, as our setup happens all on localhost. Second, it is important to say that this compose file will only set the Replica Set for the first time, and if you needed to add or remove a node from it, of course, you would be required to open a tty to the running Primary node and issue the changes there.

## Step 5: Creating 3 instances of RocketChat with 3 ReplicaSets each containing 3 nodes 

This step is basically the same as the last one. 3 RocketChat containers will be started that each is linked to a Replica Set containing 3 nodes each. You should only take extreme care of the syntax and directives not to make any mistakes in setting the names, ports, replica sets, etc., or get ready to scrutinize the lengthy logs of the 9 containers that will emerge.

Run [docker-compose-step5.yml](docker-compose-step5.yml) :
```sh
docker-compose -f docker-compose-step5.yml up --remove-orphans
```

This will create three RocketChat services each equipped with a 3-node replica set accessible at [localhost:3000](http://localhost:3000), [localhost:3001](http://localhost:3001) and [localhost:3002](http://localhost:3002).

## Step 6: Configuring Prometheus for monitoring the cluster

Now that we have a cluster of RocketChats and MongoDBs, we want to supervise them. Prometheus is a great tool that can "scrape" data and metrics off services for our monitoring and wrangling needs.

Prometheus needs a config file to operate correctly. In that file we must set which services we want to monitor. However, first, we need to ensure that the services expose their logs for use with Prometheus. For example, for RocketChat as an admin you need to go to Settings>Logs>Prometheus and enable its use. By default, a service might not offer any collectable data for scraping in Prometheus, and MongoDB is an example. That's when we need "exporters" to bind them to the DB, server app, etc. in order to allow extracting data from them.

Therefore, we would need another container service named [`mongodb_expoter`](https://github.com/percona/mongodb_exporter). This service would bind to the replica sets we have created before.

We would add three containers to our list, one for each of the 3 clusters:
```sh
  # one mongodb exporter for each cluster                                                                                                                                                     
  mongodb_exporter1:
    image: bitnami/mongodb-exporter:latest
    command:
      - "--mongodb.uri=mongodb://mongo11:27017,mongo21:27017,mongo31:27017/rocketchat?replicaSet=rs1"
    restart: unless-stopped
    ports:
      - "9001:9216"
    depends_on:
      - mongo-init-replica1


  mongodb_exporter2:
    image: bitnami/mongodb-exporter:latest
    command:
      - "--mongodb.uri=mongodb://mongo12:27017,mongo22:27017,mongo32:27017/rocketchat?replicaSet=rs2"
    restart: unless-stopped
    ports:
      - "9002:9216"
    depends_on:
      - mongo-init-replica2


  mongodb_exporter3:
    image: bitnami/mongodb-exporter:latest
    command:
      - "--mongodb.uri=mongodb://mongo13:27017,mongo23:27017,mongo33:27017/rocketchat?replicaSet=rs3"
    restart: unless-stopped
    ports:
      - "9003:9216"
    depends_on:
      - mongo-init-replica3
```
Now, we would have the MongoDB data at our disposal.

Before adding Prometheus to our docker-compose file, a sane configuration file should be made where we define the services we want to get metrics from. According to the documentation, [prometheus.yml](/prometheus.yml) has been created to do the job. 
In the file, you can see that we have included getting data from the Prometheus itself along with 3 mongo_exporter services.
Now, we can add a Prometheus container to our list in which we bind our newly created config file to it:
```sh
  prometheus:
    image: prom/prometheus:latest
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
    restart: unless-stopped
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      # if chosen to change the default DB path, its permission must be nobody:nogroup                                       
    ports:
      - "9090:9090"
    depends_on:
      - mongodb_exporter1
      - mongodb_exporter2
      - mongodb_exporter3
```

That's it! The beauty of docker-compose is that the dependencies will automatically be resolved according to our definition.
Download [docker-compose-step6.yml](/docker-compose-step6.yml) in addition to [prometheus.yml](/prometheus.yml), and start your swarm by:
```sh
docker-compose -f docker-compose-step6.yml up --remove-orphans
```
This will start all the `mongoXY` (X,Y = 1,2,3) services followed by creating the replica sets in `mongo-init-replicaX` and then `rocketchatX` and `mongo_exporterX` containers would emerge. Finally Prometheus will come to existence.

Enjoy your monitoring tool at [localhost:9090](http://localhost:9090).

![Prometheus Targets Page](/prometheus_targets.png)