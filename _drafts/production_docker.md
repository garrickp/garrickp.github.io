---
layout: post
title: "Taking Docker to Production"
---

# The Goal #

The goal of this was pretty simple, take a NodeJS application from our
developers and put it into production. The challenge of such a seemingly simple
deploy was to match the infrastructure built out for the rest of our backend as
well - we were to use Docker to push the containers to production.

There are many who would consider this to be stupidly simple, after all, the
promise of Docker is that what our developers create is what goes to
production. However, the compromises which had to be made to create high performance
Docker installs ended up requiring more work than we expected.

# The Challenges #

## 1. Disk and Network Performance ##

These problems are pretty well documented across the internet - network
performance suffers at the hand of NAT-ted traffic, and disk performance scales
inversely with the number if image layers in a container.

Fortunately, the solutions for these problems are equally easy to find. For the
network, we simply start the containers with `--net=host` and completely
obliterate the network isolation offered by Docker. On the plus side, there is
no NAT traversal cost, there is no opportunity for iptables rules to be lost,
and no chance for asymmetric TCP.

### An Aside: Asymmetric TCP ###

What's asymmetric TCP? It's when the two legs of a TCP transmission take
separate paths. In Docker, this means that you keep the `docker0` interface
running, (on, say, 172.17.42.1). However, your application discovery service
(say, etcd) is not aware of this interface, just the IP of the host running the
service (on eth0 which holds the IP 10.0.0.1). So, for Node to talk to Redis,
the best case scenario is that Redis is actually running on 10.0.0.2. This
makes the TCP traffic flow in a predictable pattern:

    nodejs -> docker0 -> eth0 -> eth0 (10.0.0.2) -> docker0 -> Redis
    
And it comes back in exactly the opposite direction:

    Redis -> docker0 -> eth0 - eth0 (10.0.0.1) -> docker0 -> nodejs

However, if your Redis container also lives on 10.0.0.1, things get funky. The
traffic out goes as follows:

    nodejs -> docker0 -> eth0 -> docker0 -> Redis

The problem is, Redis got a source address of 172.17.42.1 for the incoming TCP
packets, and so the packets take a different route back:

    Redis -> docker0 -> nodejs

Uh, oh. That's the 'asymmetric' in asymmetric TCP. TCP, being terribly
sensitive to traversal time, positively freaks out. It freaks out so badly that
your latency and bandwidth can be cut by a factor of 10. 1mbps becomes
100kbps... Ouch.

Want to test this for yourself? Spin up two docker containers on your
boot2docker machine, with one end of a bandwidth testing tool in each. Do two
tests - one over the docker0 interface, and one via the host eth0 interface.
Shake your head in wonder, and pull out `--net=host`.

## 1.5 Disk ##

Now then, about the disk. The solution for avoiding the IO penalties of disk
isolation is also pretty straightforward, mount external volumes anywhere your
application writes. This will require a bit of forethought when deploying the
containers, but it's nothing that can't be handled with a bit of planning.

## 2: Devicemapper Woes ##


ref. https://github.com/docker/docker/issues/3182
