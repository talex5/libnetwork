# Overlay Driver

### Design

XXX: This is a strawman design, written by someone unfamiliar with libnetwork.
It exists only so that libnetwork developers can point out what is wrong with it.

There are three databases used with overlay networking:

1. A cluster-wide persistent database that records networks and IP allocations. This could be a Consul service, for example. When used with SwarmKit, it is the SwarmKit managers' raft store.

2. A peer-to-peer gossip network that records which worker nodes are in the whole cluster.

3. A peer-to-peer gossip network (networkDB) for each overlay network that records which physical node hosts each virtual IP, and related routing information.

Every node in the cluster joins the peer-to-peer networkDB at startup,
using its SwarmKit manager as the initial contact point.
SwarmKit provides the key material used to secure networkDB, and rotates the keys automatically from time to time.
The nodes ping each other (using the SWIM protocol) to track which nodes are up.
If a node fails to respond to these pings, it is removed from the networkDB database; it can apply to rejoin at any time.
Membership of the networkDB cluster is independent of membership of the SwarmKit cluster.

XXX: What is the point of this membership system?
Why not just declare the networkDB members to be the same as the SwarmKit members?

libnetwork can be used with or without SwarmKit, but for simplicity we will describe here how it is used with SwarmKit.

An overlay network can only be created using a SwarmKit manager node (e.g. `docker network create -d overlay foo`).
The manager assigns the network a unique ID and commits it to the raft store.
The SwarmKit allocator uses libnetwork to assign (subnets and?) VXLAN IDs to the new network, but does not cause any changes on worker nodes at this point.

XXX: `docker network inspect` only shows a VXLAN ID, not a subnet.
ovmanager.go:NetworkAllocate seems to return one VXLAN ID for each IPv4 network in `ipV4Data` (a list of the physical networks in the system?).
This list is created in networkallocator.go from `network.IPAM.Configs`.
But that appears to be empty by default.

When a service is created that uses the network, the SwarmKit manager:

1. Uses libnetwork to assign a VIP to the service (`driver.NetworkAllocate`).
2. Creates one task for each replica of the service.
3. Uses libnetwork to assign an IP to each task.
4. Assigns each task to a particular node.
5. Sends the task to the chosen worker node. The task includes details of the network.

If the worker node is not already a member of the overlay network, it will join the network's gossip group at this point.

The worker node contributes entries to two tables in the network's gossip database:

- The service discovery table maps service names to VIPs and task names to IP addresses.
- The routing table maps each task IP to its routing information: the host node's IP address and the VXLAN ID to use.

When the table is modified, the node sends the change to three other random nodes in the gossip group via UDP.
These nodes will in turn pass the changes on to three more nodes, and so on.
Every entry in a table is owned by a node and includes a version counter controlled by that node.
This allows peers to ignore updates when they already have a newer version of the information.

This process typically results in updates spreading throughout the gossip network within a few seconds.
In addition, the node will periodically connect to a random peer and sync the entire database over TCP.
This allows the system to recover even if all the UDP messages are missed for some reason.

A container on an overlay network will have one (virtual) ethernet device connected to the overlay network and
another ethernet device connected to the outside world via `default_gwbridge`.
The container's routing table will direct traffic to the appropriate device based on the destination network prefix.

Within a container, `resolv.conf` will be set to `127.0.0.11`.
An iptables DNAT rule causes DNS traffic sent to this address to the forwarded to the Docker daemon,
which uses the information in the gossip database to respond to DNS lookups within the network.

Once all the services using a network have been removed from SwarmKit's raft store, the network itself can be removed (e.g. `docker network rm foo` on a SwarmKit manager).
However, containers using the network may still exist on worker nodes at this point,
as it takes a while for them to be shut down.

XXX Why is it possible to remove an overlay network from a worker node? What does that actually do?

XXX Are there managers and workers in libnetwork? Are they the same as the managers and workers in SwarmKit?
What is a "serf"? (`drivers/overlay/ov_serf.go`)


### Multi-Host Overlay Driver Quick Start

This example is to provision two Docker Hosts with the **experimental** Libnetwork overlay network driver.

### Pre-Requisites

- Kernel >= 3.16
- Experimental Docker client

### Install Docker Experimental

Follow Docker experimental installation instructions at: [https://github.com/docker/docker/tree/master/experimental](https://github.com/docker/docker/tree/master/experimental)

To ensure you are running the experimental Docker branch, check the version and look for the experimental tag:

```
$ docker -v
Docker version 1.8.0-dev, build f39b9a0, experimental
```

### Install and Bootstrap K/V Store


Multi-host networking uses a pluggable Key-Value store backend to distribute states using `libkv`.
`libkv` supports multiple pluggable backends such as `consul`, `etcd` & `zookeeper` (more to come).

In this example we will use `consul`

Install:

```
$ curl -OL https://dl.bintray.com/mitchellh/consul/0.5.2_linux_amd64.zip
$ unzip 0.5.2_linux_amd64.zip
$ mv consul /usr/local/bin/
```

**host-1** Start Consul as a server in bootstrap mode:

``` 
$ consul agent -server -bootstrap -data-dir /tmp/consul -bind=<host-1-ip-address>
```

**host-2** Start the Consul agent:

``` 
$ consul agent -data-dir /tmp/consul -bind=<host-2-ip-address>
$ consul join <host-1-ip-address>
```


### Start the Docker Daemon with the Network Driver Daemon Flags

**host-1** Docker daemon:

```
$ docker -d --kv-store=consul:localhost:8500 --label=com.docker.network.driver.overlay.bind_interface=eth0
```

**host-2** Start the Docker Daemon with the neighbor ID configuration:

```
$ docker -d --kv-store=consul:localhost:8500 --label=com.docker.network.driver.overlay.bind_interface=eth0 --label=com.docker.network.driver.overlay.neighbor_ip=<host-1-ip-address>
```

### QuickStart Containers Attached to a Network

**host-1** Start a container that publishes a service svc1 in the network dev that is managed by overlay driver.

```
$ docker run -i -t --publish-service=svc1.dev.overlay debian
root@21578ff721a9:/# ip add show eth0
34: eth0: <BROADCAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:ec:41:35:bf brd ff:ff:ff:ff:ff:ff
    inet 172.21.0.16/16 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:ecff:fe41:35bf/64 scope link
       valid_lft forever preferred_lft forever
```

**host-2** Start a container that publishes a service svc2 in the network dev that is managed by overlay driver.

```
$ docker run -i -t --publish-service=svc2.dev.overlay debian
root@d217828eb876:/# ping svc1
PING svc1 (172.21.0.16): 56 data bytes
64 bytes from 172.21.0.16: icmp_seq=0 ttl=64 time=0.706 ms
64 bytes from 172.21.0.16: icmp_seq=1 ttl=64 time=0.687 ms
64 bytes from 172.21.0.16: icmp_seq=2 ttl=64 time=0.841 ms
```
### Detailed Setup

You can also setup networks and services and then attach a running container to them.

**host-1**:

```
docker network create -d overlay prod 
docker network ls
docker network info prod
docker service publish db1.prod
cid=$(docker run -itd -p 8000:8000 ubuntu)
docker service attach $cid db1.prod
```

**host-2**:

```
docker network ls
docker network info prod
docker service publish db2.prod
cid=$(docker run -itd -p 8000:8000 ubuntu)
docker service attach $cid db2.prod
```

Once a container is started, a container on `host-1` and `host-2` both containers should be able to ping one another via IP, service name, \<service name>.\<network name>


View information about the networks and services using `ls` and `info` subcommands like so:

```
$ docker service ls
SERVICE ID          NAME                  NETWORK             CONTAINER
0771deb5f84b        db2                   prod                0e54a527f22c
aea23b224acf        db1                   prod                4b0a309ca311

$ docker network info prod
Network Id: 5ac68be2518959b48ad102e9ec3d8f42fb2ec72056aa9592eb5abd0252203012
	Name: prod
	Type: overlay

$ docker service info db1.prod
Service Id: aea23b224acfd2da9b893870e0d632499188a1a4b3881515ba042928a9d3f465
	Name: db1
	Network: prod
```

To detach and unpublish a service:

```
$ docker service detach $cid <service>.<network>
$ docker service unpublish <service>.<network>

# Example:
$ docker service detach $cid  db2.prod
$ docker service unpublish db2.prod
```

To reiterate, this is experimental, and will be under active development.
