---
layout: blog
title: Meteor Cluster - Introduction & how it works
category: blog
summery: '<p>Meteor Cluster is a <a href="https://atmosphere.meteor.com/package/cluster">Smart Package</a> for meteor which allows you to run cluster of meteor nodes, which is <a href="http://stackoverflow.com/a/13716069/457224">not possible</a> with the default meteor setup.</p>'
---

> TL;DR: Meteor Cluster is a [Smart Package](https://atmosphere.meteor.com/package/cluster) for meteor which allows you to run cluster of meteor nodes, which is [not possible](http://stackoverflow.com/a/13716069/457224) with the default meteor setup.

Meteor Core teams' current focus is to make meteor more developer friendly, which is really good. It has some plans to support multi-node meteor apps in the future, but we are not sure about the exact timeline. People want to build multi-node production apps with meteor right now. So we need to find a way to do that until there is an official way of doing it.

When we are trying to run multi-node meteor app, we need to solve 2 major issues.

1. We need to load balance, WebSockets/SockJS connections to our meteor nodes correctly
2. We need to sync mongodb write operations across all the meteor nodes

Here we are only focusing on syncing mongodb write operations and load balancing can be done later or simply we can just rely on someone else. (we'll discuss more about this later)

## Introducing Meteor Cluster

Meteor Cluster is a [smart package](https://atmosphere.meteor.com/package/cluster) which sync mongodb write operations across all the meteor nodes. It uses [Redis Pub/Sub](http://redis.io/topics/pubsub) to communicate between meteor nodes. Let's try Meteor Cluster.

1. Download and install [Redis](http://redis.io)
2. Run `redis-server` in your local machine
3. Install `cluster` [smart package](https://atmosphere.meteor.com/package/cluster)
5. Install and Run [mongodb](http://www.mongodb.org/) instance
4. Add following code to your app

`Posts, Notifications, Comments` are Meteor Collections which needs to be synced with the cluster. Replace them accordingly.

    Meteor.startup(function() {
        Meteor.Cluster.init(); //assumes you are running a redis-server locally
        Meteor.Cluster.sync(Posts, Notifications, Comments); //replace with your collections
    });

Now run 2 meteor nodes with following command.

1. `MONGO_URL=mongodb://localhost/meteor mrt --port 8090`
2. `MONGO_URL=mongodb://localhost/meteor mrt --port 8091`

Now you've a Meteor Cluster of 2 nodes. Visit [http://localhost:8091](http://localhost:8091) and [http://localhost:8092](http://localhost:8091) in separate browsers and see how it works.

### [See this demo video on YouTube](http://www.youtube.com/watch?v=12NkUJEdFCw&feature=youtu.be)

## How it works

Although Meteor Cluster solves a huge problem, it's code base is very small. You check it out on [Github](https://github.com/arunoda/meteor-cluster) and it is releases under MIT License.

<iframe src="http://ghbtns.com/github-btn.html?user=arunoda&repo=meteor-cluster&type=watch&count=true&size=large" allowtransparency="true" frameborder="0" scrolling="0" width="125px" height="30px">
</iframe>
<iframe src="http://ghbtns.com/github-btn.html?user=arunoda&repo=meteor-cluster&type=fork&count=true&size=large" allowtransparency="true" frameborder="0" scrolling="0" width="152px" height="30px">
</iframe>

Let's discuss how meteor cluster works. Imagine our meteor app is a Group Chat.

* We have 2 meteor nodes - (A and B)
* They are connected to Redis via Meteor Cluster and listening for write operations
* We have 2 users (Tom and Jerry)
* Tom is connected to A and Jerry is connected to B
* When Tom sends a message (insert operation), `cluster` detects that and send it to Redis
* Then Redis forward it to other nodes, in our case B
* B applies that write operation and it detects there is a new message 
* Then B simply send that message to Jerry
* This process happens under few milli-seconds

### Does this duplicate my write operations?

It seems like we are applying the insert operation in two places into a single mongodb database. And there should be some collisions in the database.

**No. its not like that. There won't be any collisions and this is completely safe. Here is what is exactly happens when applying the write operation.**

* For `insert` and `update` operations cluster update the db with a [empty object](http://goo.gl/hS9cx) which has no effect to the db
* But meteor, detects it and synced with the database
* For `remove` operation we simply apply it

## Deploy an app with Meteor Cluster 

> [This Meteor App](http://meteor-cluster.jit.su) runs with 3 node Meteor Cluster. App is hosted on [Nodejitsu](nodejitsu.com) and using [Redis Cloud](http://redis-cloud.com/) for a hosted redis server. Nodejitsu can do the load balancing for our multi-node meteor app.

Let's try to deploy an multi-node meteor app to a PAAS provider. Yes you can deploy it to your own hardware too.

* First you need to make your meteor app into a standard nodejs app
* You can use [Demeteorizer](https://github.com/OnModulus/demeteorizer) to do that
* Now this is standard nodejs app, which can be deployed very easily 
* Make sure your PAAS provider support WebSockets and Sticky Sessions
* Provision a cloud mongodb database (try MongoHQ or MongoLab)
* Provision a cloud redis server (try Redis Cloud or Redistogo)

make sure following **Environment Variables** are correctly set

    MONGO_URL="mongodb://user:pass@host:port"
    REDIS_URL="redis://redis:pass@host:port" 
    ROOT_URL="http://yourdomainname.com"
    PORT=80 #check with your PAAS provider

Now simply deploy and add few nodes to your app, and you have a full functioning Meteor Cluster.

## Production Usage

One of my contractor uses Meteor Cluster for a production app. We have done a stress test with 8 node meteor cluster with self hosted mongodb and redis. It performs really well under the load. Unfortunately I cannot disclose exact numbers. 

If you have any queries about Meteor Cluster and wanna know about more. Just ping me via [@arunoda](http://twitter.com/arunoda)







