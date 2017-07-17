backpack-coordinator
====

Coordination service for a Backpack cluster.

### Installation

```
npm install backpack-coordinator
```

### Usage

Run the coordinator as follows:

```
backpack-coordinator <zk_servers> </zk/root> <listen_host> <listen_port>
```

* zk_servers: a list of Zookeeper servers to connect to, separated with commas.
* /zk/root: the root node in the Zookeeper for Backpack's data.
* listen_host: local host name or IP to listen on.
* listen_port: port to listen on.

### Example

This section will show an example where we create a Backpack cluster
from six nodes on two servers with 2x data replication for failover.

The servers have hostnames `one` and `two`.

Final setup will look like this:

|  one |    two  |             |
|------|---------|-------------|
| 001  |  004    | ← shard #1 |
| 002  |  005    | ← shard #2 |
| 003  |  006    | ← shard #3 |


#### Installing Backpack instances

Please go to [Backpack project page](https://github.com/Topface/backpack)
to see how to install Backpack instances. Run six of them:

* http://one:10001/ (node 001)
* http://one:10002/ (node 002)
* http://one:10003/ (node 003)
* http://two:10004/ (node 004)
* http://two:10005/ (node 005)
* http://two:10005/ (node 006)

#### Setting up coordination services

Now we create one coordination service on each physical server:

* http://one:12001/ — coordinator #1
* http://two:12002/ — coordinator #2

Let's assume the Zookeeper service is running on one.local on port 2181.
(In the real world, you'll need 3 or 5 (2n+1 rule) Zookeeper instances to eliminate
the single point of failure in your cluster.)

You also need some redis servers to store the replication queue. You may have
as many servers as you like, but more servers require more time to process).
Remember that if you have one redis then you make it the single point if failure,
so make some more. Suppose we have redis instances on one:13001 and two:13002.

Initialize coordinator settings. `backpack-coordinator-init` should be called like this: `backpack-coordinator-init <zk_servers> </zk/root> <queue_key> <redis_host1:redis_port2,...>`

In this example, the coordinator package is installed in /opt/backpack-coordinator:

```
$ /opt/backpack-coordinator/bin/backpack-coordinator-init one.local:2181 /backpack backpack-queue one:13001,two:13002
Servers map initialized
Shards map initialized
Queue initialized
```

Now you may run coordinator services on `one` and `two`:

```
[one] $ /opt/backpack-coordinator/bin/backpack-coordinator one.local:2181 /backpack one 12001
```

```
[two] $ /opt/backpack-coordinator/bin/backpack-coordinator one.local:2181 /backpack two 12002
```

#### Adding capacity

Coordinator nodes automatically update their configuration (just as `backpack-replicator`
nodes do), so we may add more Backpack nodes on the fly. Let's create three shards,
two nodes each.

We need to register the servers first.

The general syntax is: `backpack-coordinator-add-server <zk_servers> </zk/root> <id> <url>`.

```
$ /opt/backpack-coordinator/bin/backpack-coordinator-add-server one.local:2181 /backpack 1 http://one:10001
$ /opt/backpack-coordinator/bin/backpack-coordinator-add-server one.local:2181 /backpack 2 http://one:10002
$ /opt/backpack-coordinator/bin/backpack-coordinator-add-server one.local:2181 /backpack 3 http://one:10003
$ /opt/backpack-coordinator/bin/backpack-coordinator-add-server one.local:2181 /backpack 4 http://two:10004
$ /opt/backpack-coordinator/bin/backpack-coordinator-add-server one.local:2181 /backpack 5 http://two:10005
$ /opt/backpack-coordinator/bin/backpack-coordinator-add-server one.local:2181 /backpack 6 http://two:10006
```

Now register the shards. Let's make them 100 GB each.

```
$ /opt/backpack-coordinator/bin/backpack-coordinator-add-shard one.local:2181 /backpack 1 1,4 100gb
$ /opt/backpack-coordinator/bin/backpack-coordinator-add-shard one.local:2181 /backpack 2 2,5 100gb
$ /opt/backpack-coordinator/bin/backpack-coordinator-add-shard one.local:2181 /backpack 1 3,6 100gb
```

Shards are added as read-only by default, you'll need to enable writing manually:

```
$ /opt/backpack-coordinator/bin/backpack-coordinator-enable-shard one.local:2181 /backpack 1
```

Good! We're done setting up the coordinators. Set up replicators and you're ready to go!

#### Setting up replication service

The coordinator only uploads to one Backpack node and creates a task to replicate
data to the rest of them. You should set up [backpack-replicator](http://github.com/Topface/backpack-replicator)
to make this work.

Just run as many replicators as your load requires. Arguments are Zookeeper servers
and Zookeeper root from backpack-coordinator. Let's spawn one replicator per physical
server.

```
[one] $ /opt/backpack-replicator/bin/backpack-replicator one.local:2181 /backpack
```

```
[two] $ /opt/backpack-replicator/bin/backpack-replicator one.local:2181 /backpack
```

### Uploading files

Make a PUT request to any of the coordinator nodes and receive the id of the shard
in which the file is stored.

```bash
$ echo 'hello, backpack!' > hello.txt
$ curl -X PUT -T hello.txt http://two:12002/hi.txt
{"shard_id":"1"}
$ curl http://one:10001/hi.txt
hello, backpack!
```

If a GET request to the first node fails with the 404 status, you should try
the next node in the shard. Eventually replicators will copy the new file
to every node in the shard.

### Nginx recipe

In a real-world application, you might have Nginx in front of Backpack nodes.

Configuration for our case looks like this (on the host `one`):

```
upstream backpack-shard-1 {
    server one:10001 max_fails=3 fail_timeout=5s;
    server one:10004 max_fails=3 fail_timeout=5s;
}

upstream backpack-shard-2 {
    server one:10002 max_fails=3 fail_timeout=5s;
    server one:10005 max_fails=3 fail_timeout=5s;
}

upstream backpack-shard-3 {
    server one:10003 max_fails=3 fail_timeout=5s;
    server one:10006 max_fails=3 fail_timeout=5s;
}

server {
    listen one:80;
    server_name one;

    # some reasonable values
    proxy_connect_timeout 5s;
    proxy_send_timeout 5s;
    proxy_read_timeout 10s;

    # retry on the next node if a request fails or returns the 404 status
    proxy_next_upstream error timeout http_404 http_502 http_504;

    # this is important
    # don't let anyone upload files via the frontend
    if ($request_method !~ ^(GET|HEAD)$ ) {
        return 403;
    }

    # extract the shard number and the file name from the url
    location ~ ^/(.*):(.*)$ {
        set $shard $1;
        set $file  $2;

        proxy_pass http://backpack-shard-$shard/$file;
    }
}
```

With this config, you'll be able to download a stored file at this url:
[http://one/1:hi.txt](http://one/1:hi.txt). Run nginx on more than
one physical server to eliminate the single point of failure.

### Todo

* [docs] Make docs better.
* [feature] Having files counter (successful uploads and not) would be nice.

### Authors

* [Ian Babrou](https://github.com/bobrik)
