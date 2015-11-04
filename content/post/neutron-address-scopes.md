+++
date = "2016-02-09T09:00:00-07:00"
title = "Neutron Address Scopes"
Tags = ["Openstack", "Neutron", "L3", "Development"]

+++

An exciting new feature was [just merged][merged] to Openstack Neutron
in the Mitaka release; it's called *address scopes*.  Address scopes
build from [subnet pools][subnet-pools-post] added in Kilo.  While
subnet pools give us a mechanism for controlling the allocation of
addresses to subnets, address scopes give Neutron a way to know where
addresses are viable.  They are also the thing within which addresses
are not allowed to overlap.

If you're unfamiliar with them, you might want to review [subnet
pools][subnet-pools-post] before you read on.

## Why you need them

When Neutron began, users were expected to specify addresses on their
own when creating subnets.  I call this "bring your own addressing" or
BYOA.  This is not ideal for a couple of reasons.

1. It put a burden on end users to manage their address usage.
1. It made it impossible to know where addresses are viable.

The first limitation was solved with the addition of [subnet
pools][subnet-pools-post] in Kilo.

The second limitation effectively led to the decision to NAT everything
between tenant networks and the *external network* in Neutron routers.
Think about it this way, the cloud operator has no way of knowing where
the addresses on the tenant networks came from, so the only valid
assumption to make is that they're all private addresses and are not
viable anywhere else.

So, what if I want to route tenant networks on the external network?
First tenants have to disable NAT on their routers.  Second, I have to
trust that tenants are all using the right addresses.

With address scopes, I can set up the address scopes for tenants to pull
addresses from.  Then, since Neutron routers understand address scopes,
they won't NAT between these networks and my external network as long as
the scopes match.  They'll just do straight routing.

## How it works

Anyone can create an address scope.  Admins can create shared address
scopes seen by all tenants.

Access to addresses in a scope is managed through subnet pools.  You can
create a subnet pool in an address scope or you can update existing
subnet pools to belong to a scope.

It may be useful to add more than one subnet pool to an address scope if
the pools have different owners.  This allows delegation of parts of the
address scope.  Address overlap is prevented across the whole scope so
you will get an error if two pools have some of the same address ranges
in them.

A Neutron router connects at least a couple of networks.  Each router
interface is associated with an address scope by looking at the subnets
on the network its connected to.  The router internally marks all
traffic connections originating from each interface with the
corresponding address scope to track it.  If traffic tries to leave an
interface in the wrong scope, it is blocked.

When a router connects to two networks with the same address scope, it
knows that these networks can be routed without any kind of address
translation.  Also, since subnet pools are part of the foundation of
address scopes, Neutron knows that all of the addresses in use within an
address scope are unique and legitimate from the address scope owner's
point of view.

### No Scope

Neutron preserves backwards compatibility with pre-Mitaka Neutron.  You
won't notice any difference until you decide to begin using them so you
won't be forced to change your behavior.

When subnets are not explicitly part of an address scope, they are
considered to be part of what I am now calling the "no scope" scope.
This scope is different in a few ways to preserve backwards
compatibility.

1. Unlimited address overlap is allowed.
1. Neutron routers, by default, will NAT traffic from internal networks
   to external networks even if they are all in this scope (unless snat
   is disabled for the router.)
1. This scope isn't visible through the API.  It won't show up when you
   list address scopes and you can't show details.  It exists only
   implicitly to catch all addresses which aren't explicitly scoped.

### Demo

Even though Mitaka has not been released, you can play with this new
feature as long as you have a recent pre-release copy of Neutron code
running.  A new devstack should do but you might need to update your
neutron client to something on the bleeding edge.

Let's give it a try...

    NOTE: I trim a lot of irrelevant fields from the output of these
    commands just for brevity and to avoid distracting you with too many
    details.

#### Admin Commands

First, as admin, I'll create a couple of shared address scopes:
```shell
admin> neutron address-scope-create --shared address-scope-ip6 6
Created a new address_scope:
+------------+--------------------------------------+
| Field      | Value                                |
+------------+--------------------------------------+
| id         | 13b83fb2-beb4-4533-9e12-4bf9a5721ef5 |
| ip_version | 6                                    |
| name       | address-scope-ip6                    |
| shared     | True                                 |
+------------+--------------------------------------+
```
```shell
admin> neutron address-scope-create --shared address-scope-ip4 4
Created a new address_scope:
+------------+--------------------------------------+
| Field      | Value                                |
+------------+--------------------------------------+
| id         | 97702525-e145-40c8-8c8f-d415930d12ce |
| ip_version | 4                                    |
| name       | address-scope-ip4                    |
| shared     | True                                 |
+------------+--------------------------------------+
```

Next, we'll create subnet pools much like in [my subnet pool
post][subnet-pools-post].  The one important difference here is that I
specify the name of the address scope that the subnet pool should belong
to.  If you have existing subnet pools, you can use the
subnet-pool-update command to put them in to a new address scope.
```shell
admin> neutron subnetpool-create --address-scope address-scope-ip6 \
       --shared --pool-prefix 2001:db8:a583::/48 --default-prefixlen 64 \
       subnet-pool-ip6
Created a new subnetpool:
+-------------------+--------------------------------------+
| Field             | Value                                |
+-------------------+--------------------------------------+
| address_scope_id  | 13b83fb2-beb4-4533-9e12-4bf9a5721ef5 |
| default_prefixlen | 64                                   |
| id                | 14813344-d11a-4896-906c-e4c378291058 |
| ip_version        | 6                                    |
| name              | subnet-pool-ip6                      |
| prefixes          | 2001:db8:a583::/48                   |
| shared            | True                                 |
+-------------------+--------------------------------------+
```
```shell
admin> neutron subnetpool-create --address-scope address-scope-ip4 \
       --shared --pool-prefix 203.0.113.0/21 --default-prefixlen 26 \
       subnet-pool-ip4
Created a new subnetpool:
+-------------------+--------------------------------------+
| Field             | Value                                |
+-------------------+--------------------------------------+
| address_scope_id  | 97702525-e145-40c8-8c8f-d415930d12ce |
| default_prefixlen | 26                                   |
| id                | e2c4f12d-307f-4616-a4df-203a45e6cb7f |
| ip_version        | 4                                    |
| name              | subnet-pool-ip4                      |
| prefixes          | 203.0.112.0/21                       |
| shared            | True                                 |
+-------------------+--------------------------------------+
```

Now that these are created, we can create subnets on our external
network.  (Note:  In playing with this early on, I actually used the DB
to set the subnet pool of the external network.  Your milleage may vary.)
```shell
$ neutron subnet-show ipv6-public-subnet
+-------------------+------------------------------------------------------------------+
| Field             | Value                                                            |
+-------------------+------------------------------------------------------------------+
| cidr              | 2001:db8::/64                                                    |
| enable_dhcp       | False                                                            |
| gateway_ip        | 2001:db8::2                                                      |
| id                | 8e9299bf-5c48-4143-b081-010ba26636a2                             |
| ip_version        | 6                                                                |
| name              | ipv6-public-subnet                                               |
| network_id        | d2ac8578-7e86-4646-849a-afdf5a05fff0                             |
| subnetpool_id     | 14813344-d11a-4896-906c-e4c378291058                             |
+-------------------+------------------------------------------------------------------+
```
```shell
$ neutron subnet-show public-subnet
+-------------------+------------------------------------------------+
| Field             | Value                                          |
+-------------------+------------------------------------------------+
| cidr              | 172.24.4.0/24                                  |
| enable_dhcp       | False                                          |
| gateway_ip        | 172.24.4.1                                     |
| id                | 3c3029d2-8081-4e56-9842-6007ce742860           |
| ip_version        | 4                                              |
| name              | public-subnet                                  |
| network_id        | d2ac8578-7e86-4646-849a-afdf5a05fff0           |
| subnetpool_id     | e2c4f12d-307f-4616-a4df-203a45e6cb7f           |
+-------------------+------------------------------------------------+
```

This completes the portion of the demo that requires admin privileges.

#### Non-admin Tenant Commands

Let's start by creating a couple of networks.  We're going to create one
the old way, without address scopes.  We'll create the second one using
address scopes.  Later, this will let us see how routing between the two
works.

```shell
$ neutron net-create network1
Created a new network:
+-------------------------+--------------------------------------+
| Field                   | Value                                |
+-------------------------+--------------------------------------+
| id                      | f5a980d9-5521-438e-b831-0ebacba2b372 |
| name                    | network1                             |
| subnets                 |                                      |
+-------------------------+--------------------------------------+
```
```shell
$ neutron net-create network2
Created a new network:
+-------------------------+--------------------------------------+
| Field                   | Value                                |
+-------------------------+--------------------------------------+
| id                      | 438e4f26-0e45-4b26-9797-57d0bd817953 |
| name                    | network2                             |
| subnets                 |                                      |
+-------------------------+--------------------------------------+
```

First, the old way...
```shell
$ neutron subnet-create --name subnet-ip4-1 network1 198.51.100.0/26
Created a new subnet:
+-------------------+---------------------------------------------------+
| Field             | Value                                             |
+-------------------+---------------------------------------------------+
| cidr              | 198.51.100.0/26                                   |
| id                | 48ed5c71-2a1d-4f73-b29e-371deec04d44              |
| name              | subnet-ip4-1                                      |
| network_id        | f5a980d9-5521-438e-b831-0ebacba2b372              |
| subnetpool_id     |                                                   |
+-------------------+---------------------------------------------------+
```
```shell
$ neutron subnet-create --name subnet-ip6-1 network1 \
  --ipv6-ra-mode slaac --ipv6-address-mode slaac \
  --ip_version 6 2001:db8:80d2:c4d3::/64
Created a new subnet:
+-------------------+-------------------------------------------------------------------------------------+
| Field             | Value                                                                               |
+-------------------+-------------------------------------------------------------------------------------+
| cidr              | 2001:db8:80d2:c4d3::/64                                                             |
| id                | c9f0bb79-1d7b-435f-b362-05a9a7259aa6                                                |
| name              | subnet-ip6-1                                                                        |
| network_id        | f5a980d9-5521-438e-b831-0ebacba2b372                                                |
| subnetpool_id     |                                                                                     |
+-------------------+-------------------------------------------------------------------------------------+
```

... and the new way.  Subnet pools make it nice, don't they?
```shell
$ neutron subnet-create --name subnet-ip4-2 \
  --subnetpool subnet-pool-ip4 network2
Created a new subnet:
+-------------------+-------------------------------------------------+
| Field             | Value                                           |
+-------------------+-------------------------------------------------+
| cidr              | 203.0.112.0/26                                  |
| id                | deb36645-8d46-4c13-a489-1135174d8a8c            |
| name              | subnet-ip4-2                                    |
| network_id        | 438e4f26-0e45-4b26-9797-57d0bd817953            |
| subnetpool_id     | e2c4f12d-307f-4616-a4df-203a45e6cb7f            |
+-------------------+-------------------------------------------------+
```
```shell
$ neutron subnet-create --name subnet-ip6-2 --ip_version 6 \
  --ipv6-ra-mode slaac --ipv6-address-mode slaac \
  --subnetpool subnet-pool-ip6 network2
Created a new subnet:
+-------------------+-----------------------------------------------------------------------------+
| Field             | Value                                                                       |
+-------------------+-----------------------------------------------------------------------------+
| cidr              | 2001:db8:a583::/64                                                          |
| id                | b157e288-748e-4c4b-9b2e-8b8e65241036                                        |
| name              | subnet-ip6-2                                                                |
| network_id        | 438e4f26-0e45-4b26-9797-57d0bd817953                                        |
| subnetpool_id     | 14813344-d11a-4896-906c-e4c378291058                                        |
+-------------------+-----------------------------------------------------------------------------+
```

We just need to connect up the router and we're ready to start looking
at what it did.
```shell
$ neutron router-interface-add router1 subnet-ip4-1
Added interface 73d832e1-e4a7-4029-9a66-f4e0f4ba0e76 to router router1.
$ neutron router-interface-add router1 subnet-ip4-2
Added interface 94b4cdb2-875d-4ab3-9a6e-803c3626c4d9 to router router1.
$ neutron router-interface-add router1 subnet-ip6-1
Added interface f35c4541-d529-4bd8-af4e-1b069269c263 to router router1.
$ neutron router-interface-add router1 subnet-ip6-2
Added interface f5904a4b-9547-4c08-bc7e-bc5fc71a8db9 to router router1.
```

#### What Just Happened?

Okay, at this point you might be wondering what I've just done.  I'm
going to boot two vms, instance1 on network1 and instance2 on network2
and give them some ip addresses.  Be sure you adjust your security
groups to allow pings and ssh (both IPv4 and IPv6).
```shell
$ nova list
+--------------+-----------+---------------------------------------------------------------------------+
| ID           | Name      | Networks                                                                  |
+--------------+-----------+---------------------------------------------------------------------------+
| 97e49c8e-... | instance1 | network1=2001:db8:80d2:c4d3:f816:3eff:fe52:b69f, 198.51.100.3, 172.24.4.3 |
| ceba9638-... | instance2 | network2=203.0.112.3, 2001:db8:a583:0:f816:3eff:fe42:1eeb, 172.24.4.4     |
+--------------+-----------+---------------------------------------------------------------------------+
```

Let's see what we can do.  First, regardless of address scopes, I can
ping the floating IPs from my external network.
```shell
$ ping -c 1 172.24.4.3
1 packets transmitted, 1 received, 0% packet loss, time 0ms
$ ping -c 1 172.24.4.4
1 packets transmitted, 1 received, 0% packet loss, time 0ms
```

With just a little bit of routing help, I can actually ping directly in
to the internal network2 which is in the same address scope as the
external network.
```shell
$ sudo ip route add 203.0.112.0/26 via 172.24.4.2
$ ping -c 1 203.0.112.3
1 packets transmitted, 1 received, 0% packet loss, time 0ms
```
```shell
$ sudo ip route add 2001:db8:a583::/64 via 2001:db8::1
$ ping6 -c 1 2001:db8:a583:0:f816:3eff:fe42:1eeb
1 packets transmitted, 1 received, 0% packet loss, time 0ms
```

But, I can't do that with the other network because the scopes don't
match.
```shell
$ sudo ip route add 198.51.100.0/26 via 172.24.4.2
$ ping -c 1 198.51.100.3
1 packets transmitted, 0 received, 100% packet loss, time 0ms
```
```shell
$ sudo ip route add 2001:db8:80d2:c4d3::/64 via 2001:db8::1
$ ping6 -c 1 2001:db8:80d2:c4d3:f816:3eff:fe52:b69f
1 packets transmitted, 0 received, 100% packet loss, time 0ms
```

In general, you'll find that if you use address scopes and the scope
matches between networks then your pings (and other traffic) route right
through.  If the scopes don't match between networks then the router
either drops the traffic or it applies NAT to cross scope boundaries.
Go ahead and play with it.

## Address Scopes in the Future

### BGP Dynamic Routing

Another exciting new Neutron feature that is close to completion in
Mitaka is BGP dynamic routing announcement.  This new feature will
essentially turn Neutron in to a BGP speaker.  Through its API, you will
give Neutron just enough information about your physical routers and how
they connect to your Neutron networks.  It will then be able to compute
the routes and next hops necessary to route in to Neutron internal
networks through Neutron logical routers.  Address Scopes are a big part
of this.  Neutron will only consider internal networks which belong to
your external address scope for announcement.

It is difficult to imagine integrating BGP without address scopes.  With
BYOA, we couldn't just pick up addresses on internal networks and
announce them to physical routers because we don't know that they're
unique, viable on your network, or that tenants are permitted to use
them at all.

### Routed Networks

BGP will announce floating IPs since they are already intrinsically part
the external address scope.  You may wonder why we would need this at
all.  I mean, floating IPs already work in Neutron without BGP, right?
Yes, they do, but think about them in the context of the [up-coming
routed networks work in Neutron][routed-networks].  Connecting a Neutron
router's external gateway to a routed network will require routing to
the floating IP from the outside.  Also, consider how distributed
virtual routers fits in to all of this where a router's logical external
gateway port might actually be a number of ports spread across the
segments of the routed network.  More on that later.

[subnet-pools-post]: ../neutron-subnet-pools/
[spec]: http://specs.openstack.org/openstack/neutron-specs/specs/mitaka/address-scopes.html
[routed-networks]: https://review.openstack.org/#/c/225384
[merged]: https://review.openstack.org/#/c/270001

<!-- vim:set tw=72 ft=markdown: -->
