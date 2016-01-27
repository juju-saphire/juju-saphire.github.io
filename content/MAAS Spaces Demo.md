---
draft: true,
title: "MAAS Spaces Demo"
date: "2016-01-27"
---

In this post we are going to show you Juju 2.0s endpoint binding feature. By using endpoint binding you can deploy a bundle and have each service deployed into one or more spaces using a single command.

If that doesn't make much sense, don't worry, here is a quick run-down of what it means.

## Bundle
In Juju a Charm is a recipe for deploying a [service](https://jujucharms.com/docs/stable/charms-deploying). For example the [mariadb charm](https://jujucharms.com/mariadb/) tells Juju how to install mariadb and how it [relates](https://jujucharms.com/docs/stable/charms-relations) to other services. A bundle instructs Juju how to deploy several charms and how those relationships should be configured.

## Endpoint
Continuing to use mariadb for our examples, mariadb exposes several relations, one of which is "db". The combination of a service (mariadb) and a relation (db) is an endpoint.

## Space
A Juju space defines a collection of subnets in which each machine in those subnets can directly talk to each other as well as sharing equivalent firewall rules.

# Endpoint Binding - the new feature
By binding an endpoint to a space a bundle can configure, for example, all database traffic to be routed over one space while all public facing internet traffic can be kept in another. This traffic separation can be used to isolate different types of traffic for security and performance management.

# On with the show!
We will be using a customised version of the mediawiki bundle that deploys haproxy, mediawiki and mysql. It is pasted below and can be downloaded [here](/files/MAAS%20Spaces%20Demo/bundle.yaml). Traffic between haproxy and mediawiki is on a space called "internal" and traffic between mediawiki and mysql is in a space called "db". The haproxy website endpoint is bound to the public space. This is all set up using the bindings section of each service.

 You will need four machines to run this yourself. For this demonstration we will be placing each space in its own VLAN.

{{< highlight yaml >}}
services:
  haproxy:
    charm: cs:trusty/haproxy-13
    num_units: 1
    bindings:
        website: public
        reverseproxy: internal
        peer: internal
    expose: true
  mediawiki:
    charm: cs:trusty/mediawiki-3
    num_units: 1
    bindings:
      cache: internal
      website: internal
      db: db
    options:
      name: MAAS + Juju Network Spaces Wiki
  mysql:
    charm: cs:trusty/mysql-29
    num_units: 1
    bindings:
      db: db
      cluster: internal
series: trusty
relations:
- - haproxy:reverseproxy
  - mediawiki:website
- - mediawiki:db
  - mysql:db
{{< /highlight >}}

Setting up VLANs, subnets and spaces in MAAS isn't difficult, but it is time consuming to go through all the steps yourself so we have put together a script to do it for you. It starts by reading the charm bundle you give it, then:

 1. For each space it finds in the bundle it will:
   1. Create a VLAN in MAAS with a /24 subnet.
   1. Create a space, then place the VLAN in that space.
 1. Then, for each unit in the charm bundle it will place a node in each VLAN the charm requires.
 1. Put one node in all spaces for use as the bootstrap node.
 1. Bootstrap the environment.
 1. Deploy the charm.
 1. Check that each endpoint is where it should be.

The only prerequisite for the script is that the fabric that you will be creating VLANs on is called "managed":
  1. `maas maas fabrics read`
{{< highlight json >}}
[
    {
        "resource_uri": "/MAAS/api/1.0/fabrics/0/", 
        "vlans": [
            {
                "name": "untagged", 
                "vid": 0, 
                "mtu": 1500, 
                "fabric": "fabric-0", 
                "id": 0, 
                "resource_uri": "/MAAS/api/1.0/vlans/0/"
            }
        ], 
        "class_type": null, 
        "name": "fabric-0", 
        "id": 0
    }, 
    {
        "resource_uri": "/MAAS/api/1.0/fabrics/1/", 
        "vlans": [
            {
                "name": "untagged", 
                "vid": 0, 
                "mtu": 1500, 
                "fabric": "fabric-1", 
                "id": 5001, 
                "resource_uri": "/MAAS/api/1.0/vlans/5001/"
            }
        ], 
        "class_type": null, 
        "name": "fabric-1", 
        "id": 1
    }
  ]
{{< /highlight >}}
In my case I will be using the fabric that manages the subnet 192.168.1.0/24: `maas maas fabric update 1 name=managed`

... run the script ... the final output is:

```
mediawiki/0 website 192.168.10.102 in internal
mediawiki/0 db 192.168.12.102 in db
haproxy/0 peer 192.168.10.101 in internal
haproxy/0 reverseproxy 192.168.10.101 in internal
mysql/0 cluster 192.168.10.103 in internal
mysql/0 db 192.168.12.103 in db
```

You can now access your mediawiki instance on the public address of the haproxy unit!
