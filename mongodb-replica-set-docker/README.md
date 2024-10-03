# MongoDB Replica Set with Docker
## Starting the mongo containers

We will set up a replica set with three nodes. For these, we will create three containers with their own separate volumes.

The manual on [Deploying a Self-Managed Replica Set](https://www.mongodb.com/docs/manual/tutorial/deploy-replica-set/) recommends using DNS hostnames instead of IP addresses while configuring the replica set members.

Docker allows you to create user-defined bridge networks which provide automatic hostname resolution, which is very useful in this scenario. We can create containers with predefined hostnames, and configure the replica set using these hostnames.

```
docker network create mongo-network
```

You can find the subnet and other details by running:

```
docker network inspect mongo-network
```

Start the containers.

```
docker run -dit --name mongo1 --network mongo-network --mount source=vol1,target=/data/db mongo:7.0

docker run -dit --name mongo2 --network mongo-network --mount source=vol2,target=/data/db mongo:7.0

docker run -dit --name mongo3 --network mongo-network --mount source=vol3,target=/data/db mongo:7.0
```

## Configuring the Replica Set

Run a shell on the first container, to initiate the replica set. 

```
docker exect -it mongo1 sh
```

Once inside the ```mongo1``` container, run ```mongosh``` and initiate the replica set.

```
test> rs.initiate({
... _id: 'set0',
... members: [
... { _id: 0, host: 'mongo1:27017' },
... { _id: 1, host: 'mongo2:27017' },
... { _id: 2, host: 'mongo3:27017' }
... ]
... })
{ ok: 1 }
set0 [direct: other] test>
```

Running ```rs.status()``` will display the state of the replica set, and which container became the primary. I had initially thought that ```mongo1``` (the instance from which we ran ```rs.initiate()```) becomes the primary, but it need not be the case. Any of the members can become the primary.

To connect to the replica set, we will need to specify the hostnames in /etc/hosts. ```mongosh``` will be able to connect to each member separately using its IP address, but a client like MongoDB Compass will need the hostname to be resolved.

We can find out the IP addresses of the containers by running:

```
docker network inspect mongo-network
```

In my case, the IP addresses were 172.18.0.2, 172.18.0.3 and 172.18.0.4 respectively. So we edit /etc/hosts as follows.

```
172.18.0.2 mongo1
172.18.0.3 mongo2
172.18.0.4 mongo3
```

Now we can connect to the replica set using the connection URI ```mongodb://mongo1:27017```.

## Docker Compose

Since this involves running multiple containers, we can make it easier to repeat by writing a [docker compose file](docker-compose.yml).


