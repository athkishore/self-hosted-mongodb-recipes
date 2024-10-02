# Setting up a MongoDB server with Docker

## Creating the MongoDB container

We will first set up a single MongoDB container using the official ```mongo``` image.

```
docker run -dit --name mongo1 mongo:7.0
```

We did not publish any ports - we will connect to it using the IP on the docker bridge network. To find the docker network, run:

```
docker network ls
```

You'll see some output like the following. The ```bridge``` network is the default docker network.

```
NETWORK ID     NAME            DRIVER    SCOPE
83b1149b285e   bridge          bridge    local
7dce3b440fa6   host            host      local
b893af0de349   none            null      local
```

To find the IP address of the MongoDB container we have created, we will ```inspect``` the ```bridge``` network.

```
docker network inspect bridge
```

It will output a lot of details, but we're looking only for the IP address of the ```mongo1``` container. You should find something like:

```
"Containers": {
    "e7924ecbd6ea6b39617117f623ba217948daf1a003b42b2f682ed36f91c6aacc": {
        "Name": "mongo1",
        "EndpointID": "8daa769cec38a79523942d878716d2db47e6b472bcc5b0267c0b3dcca47c1d28",
        "MacAddress": "02:42:ac:11:00:02",
        "IPv4Address": "172.17.0.2/16",
        "IPv6Address": ""
    }
},
```

Our container ```mongo1``` has been assigned an IP address of ```172.17.0.2```. Let's try connecting to it using ```mongosh```.

```
mongosh --host 172.17.0.2
```

The MongoDB client should successfully connect to the server running on the ```mongo1``` container.

```
Current Mongosh Log ID:	66fd3f2d617bb9c2f0964032
Connecting to:		mongodb://172.17.0.2:27017/?directConnection=true&appName=mongosh+2.3.1
Using MongoDB:		7.0.14
Using Mongosh:		2.3.1
---
test>
```

## Setting up persistent storage

The container's storage is ephemeral. When we remove a container all the data stored during runtime is deleted by Docker. We can test this.

Let's create a new database called ```awesomeapp``` and insert a user document into the ```users``` collection.

```
test> use awesomeapp
switched to db awesomeapp
awesomeapp> db.users.insertOne({ username: 'user1', email: 'user1@example.org' })
{
  acknowledged: true,
  insertedId: ObjectId('66fd3fb7617bb9c2f0964033')
}
awesomeapp> db.users.find()
[
  {
    _id: ObjectId('66fd3fb7617bb9c2f0964033'),
    username: 'user1',
    email: 'user1@example.org'
  }
]
awesomeapp>
```

We'll now stop and delete the ```mongo1``` container. When we run the container again, we would have lost the data.

```
docker stop mongo1
docker rm mongo1
docker run -dit --name mongo1 mongo:7.0

mongosh --host 172.17.0.2

test> show dbs
admin    8.00 KiB
config  12.00 KiB
local    8.00 KiB
test>
```

To make the database persistent, we will create a ```volume``` and attach it when we create the container.

```
docker run -dit --name mongo1 --mount source=vol1,target=/data/db mongo:7.0
```

Let's connect to the new container using ```mongosh``` and create the users collection and insert a user document as before. 

```
test> use awesomeapp
switched to db awesomeapp
awesomeapp> db.users.insertOne({ username: 'user1', email: 'user1@example.org' })
```

We'll then stop and remove the container, and then create a new container with the same volume.

```
$ docker stop mongo1
$ docker rm mongo1
$ docker run -dit --name mongo1 --mount source=vol1,target=/data/db mongo:7.0
```

This time, when we connect using ```mongosh```, we'll still have the database and collection we had created. It became persistent because we mounted a volume for the container to write to.

```
test> use awesomeapp
switched to db awesomeapp
awesomeapp> db.users.find()
[
  {
    _id: ObjectId('66fd43226775222b3e964033'),
    username: 'user1',
    email: 'user1@example.org'
  }
]
awesomeapp>
```

