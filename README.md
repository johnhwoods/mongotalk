# MongoTalk [![Build Status](https://travis-ci.org/pharo-nosql/mongotalk.png)](http://travis-ci.org/pharo-nosql/mongotalk) [![Test Status](https://api.bob-bench.org/v1/badgeByUrl?branch=master&hosting=github&ci=travis-ci&repo=pharo-nosql%2Fmongotalk)](https://bob-bench.org/r/gh/pharo-nosql/mongotalk)

A Pharo driver for [MongoDB](https://www.mongodb.com/). 

## Getting Started

We can open a connection to a mongodb server and perform a write operation as follows:
~~~Smalltalk
mongo := Mongo host: 'localhost' port: 27017.
mongo open.
((mongo 
	databaseNamed: 'test')
	getCollection: 'pilots')
	add: { 'name' -> 'Fangio' } asDictionary.
~~~

## Install Mongo driver

Evaluate the following script in Pharo:
```Smalltalk
Metacello new 
	repository: 'github://pharo-nosql/mongotalk/mc';
	baseline: 'MongoTalk';
	load
```

---
# Client for Replica Sets

## Introduction

The driver described above is enough in a [MongoDB standalone server](https://docs.mongodb.com/manual/reference/glossary/#term-standalone) configuration where there only one server can execute the operations.
This job can get much more complex when the configuration is a [MongoDB Replica Set](https://docs.mongodb.com/manual/reference/glossary/#term-replica-set).
In this case, a group of servers maintain the same data set providing redundancy and high availability access.

The following figure shows a Replica Set configuration composed by 3 servers (a.k.a. members).
The [Primary server](https://docs.mongodb.com/manual/core/replica-set-primary/) is the only member in the replica set that receives **write** operations.
However, all members of the replica set can accept **read** operations (see [Read Preference](https://docs.mongodb.com/v4.0/core/read-preference/)).

![ReplicaSetFigure](https://docs.mongodb.com/manual/_images/replica-set-read-write-operations-primary.bakedsvg.svg)

The replica set can have at most one primary. If the current primary becomes unavailable, an election determines the new primary. 

## MongoClient

To help in this kind of scenarios, the `MongoClient` monitors the Replica Set status to provide the instance of `Mongo` that your application requires to perform an operation.

You can create a client and make it start monitoring as follows:
~~~Smalltalk
client := MongoClient withUrls: urlsOfSomeReplicaSetMembers.
client start.
~~~

After some milliseconds, it should be ready to, for example, receive write operations such as:
~~~Smalltalk
client primaryMongoDo: [ :mongo |
	((mongo 
		databaseNamed: 'test')
		getCollection: 'pilots')
		add: { 'name' -> 'Fangio' } asDictionary ].
~~~

Until more documentation is available, you have these options to learn about this client:

* **Example.** Evaluate and browse this code: `MongoClientExample openInWindows`.

* **Test suites.**
Browse the class hierarchy of `MongoClientTest` where you can see diverse tests, setUps, and tearDowns.

* **Visual Monitor.**
You can check [this repository](https://github.com/tinchodias/pharo-mongo-client-monitor) which watches the events announced by a `MongoClient` to better understand them via visualizations.

## Install MongoClient

```Smalltalk
Metacello new 
	repository: 'github://pharo-nosql/mongotalk/mc';
	baseline: 'MongoTalk';
	load: #(SDAM)
```

---

# The MongoDB specification

The MongoDB core team proposes a [specification](https://github.com/mongodb/specifications) with suggested behavior for drivers (clients).
The next subsections describe a key part of such specification and is useful to understand how our Pharo client works.

## Server Discovery And Monitoring

You can have an introduction to the [Server Discovery And Monitoring Specification](http://emptysqua.re/server-discovery-and-monitoring.html) (SDAM) in a [blog post](https://www.mongodb.com/blog/post/server-discovery-and-monitoring-next-generation-mongodb-drivers) and in [this talk](https://www.mongodb.com/presentations/mongodb-drivers-and-high-availability-deep-dive).

***Discovery.*** The `MongoClient` receives a *seed list* of URLs when instantiated, which is the initial list of server addresses. 
After `#start`, the client starts to *ping* the seed addresses to discover replica set data.
This *ping* consists on a [ismaster](https://docs.mongodb.com/v4.0/reference/command/isMaster/) command, which is very light for servers.

***Monitors.*** The spec considers 3 kinds of implementations for monitoring: the single-threaded (Perl), multi-threaded (Python, Java, Ruby, C#), and hybrid (C). The hybrid mixes single and multi.
Our `MongoClient` has a **multi-threaded approach** to monitor servers. 
More concretely, it uses [TaskIt](https://github.com/pharo-contributions/taskit) services which relies in `Process` (i.e. [green threads](https://en.wikipedia.org/wiki/Green_threads)).

***States.*** The `MongoClient>>isMonitoringSteadyState` let's you know the internal state of the client, which is one of the following:

* **Crisis state**: It is the initial state: ping each seed address to get the first topology, then move to Steady state. Enqueue incoming commands and ping every 0.5 seconds (not customizable) all known servers until a primary is found. Then, change to Steady state.

* **Steady state**: Ping servers every 10 seconds (by default) to update data, and keep track of latency. When there is a failure, the exception is raised to the application, and it will move to Crisis state.

***Error handling.*** Applications may retry once after a connection failure, and only notify user if this first retry failed too. 
The explanation is that first failure moves the client to Crisis State, and the retry will have a long Server Selection and hopefully retry after a new primary is elected. 
It is less likely that a second retry will succeed.


## Server Selection

When a client receives a read command in *steady state* with *primaryPreferred* as read preference and the primary is not available, it might have several possible servers to execute the command.
The [Server Selection specification](https://docs.mongodb.com/manual/core/read-preference-mechanics/) proposes the algorithm for server selection that has the goals of being predictable, resilient, and low-latency.
[This blog post](https://www.mongodb.com/blog/post/server-selection-next-generation-mongodb-drivers) describes this algorithm in detail:

> Users will be able to control how long server selection is allowed to take with the [serverSelectionTimeoutMS](https://docs.mongodb.com/master/reference/connection-string/) configuration variable and control the size of the acceptable latency window with the localThresholdMS configuration variable.

Our `PharoClient` implements this algotrihm in the `MongoServerSelection` class.

## What's not implemented?

This is only a partial implementation of the whole MongoDB specification.

For example, our `MongoClient` doesn't provide any direct read or write operation (as the specification requires). 
Instead, such operations are supported by first obtaining an instance of `Mongo` (the connection to a particular server) and then obtaining the db/collection to perform the operations.
