+++
date = "2015-11-03T14:00:00-07:00"
title = "Neutron Subnet Pools"
Tags = ["Openstack Neutron L3 Development"]

+++

A new feature was added to Openstack Neutron in the Kilo release; it's
called subnet pools:  a simple feature that has the potential to improve
your workflow considerably.  It also provides a building block from
which other new features will be built in to Neutron.

Turns out, I'm not the first to blog about this new feature.  Click for
[mestery's blog].

[mestery's blog]: http://blog.siliconloons.com/posts/2015-04-28-subnetpools/

## Why you need them

Before Kilo, Neutron had no automation around the addresses used to
create a subnet.  To create one, you had to come up with the addresses
on your own without any help from the system.  There are at least two
major problems with this approach.

First, you need to manage your addresses on your own.  Wouldn't it be
nice if you could turn your pool of addresses over to Neutron to take
care of?  When you need to create a subnet, you just ask for addresses
to be allocated from the pool.

Second, imagine a cloud full of tenants all bringing their own
addresses.  The addresses are likely to overlap and there is no way to
tell if they're *routable* on an external network, or any other context
for that matter.  With this situation, there really is no feasible way
to allow routing to or from a tenant private network.  The system simply
doesn't know where addresses are compatible.

Neutron handles the second problem by requiring floating IPs between an
*external* network and any tenant private network.  So, no problem,
right?  Well, not really.  There are a couple of reasons why this is not
going to work in the long run:

1. Neutron has no IPv6 floating ips (and may never).
1. We can't route directly to a tenant network from an external network.
1. We can't easily work with routing domains like for VPNs or allowing
   routing between tenants.

## How they work

A subnet pool solves these issues by managing a pool of addresses from
which subnets can be allocated.  It ensures that there is no overlap
between two subnets allocated from the same pool.

As a regular tenant in an Openstack cloud, you can create a subnet pool
of your own and use it to manage your own pool of addresses.  This
doesn't require any admin privileges.  Your pool won't be visible to any
other tenant.

If you are an admin, you can create a pool which can be accessed by any
regular tenant.  Being a shared resource, there is a quota mechanism to
arbitrate access.

### Demo

If you have access to an Openstack Kilo or later based Neutron, you can
play with this feature now.  Let's give it a try.  This demo is adapted
from the [talk I gave] at the Openstack Liberty Conference in Vancouver.
Click to view the [slides] or the [video] using these links if you'd
like.

[talk I gave]: http://sched.co/2qco
[slides]: http://www.slideshare.net/carlbaldwin/subnet-pools-and-pluggable-ipam
[video]: https://www.openstack.org/summit/vancouver-2015/summit-videos/presentation/subnet-pools-and-pluggable-external-ip-management-in-openstack-kilo

First, as admin, I'll create a shared subnet pool:
```shell
admin> neutron subnetpool-create --shared --pool-prefix 203.0.113.0/24 \
           --default-prefixlen 26 demo-subnetpool4
Created a new subnetpool:
+-------------------+--------------------------------------+
| Field             | Value                                |
+-------------------+--------------------------------------+
| default_prefixlen | 26                                   |
| default_quota     |                                      |
| id                | 670eb517-4fd3-4dfc-9bed-da2f99f85c7a |
| ip_version        | 4                                    |
| max_prefixlen     | 32                                   |
| min_prefixlen     | 8                                    |
| name              | demo-subnetpool4                     |
| prefixes          | 203.0.113.0/24                       |
| shared            | True                                 |
| tenant_id         | c597484841ff4a8785804c62ba81449b     |
+-------------------+--------------------------------------+
```

I did essentially the same thing for ipv6 and so now I have two subnet
pools.  I can see them as a regular tenant.  (I trimmed the output a bit
for display so don't be surprised that it doesn't look exactly like
yours.)
```shell
$ neutron subnetpool-list
+---------+------------------+------------------------------------+-------------------+
| id      | name             | prefixes                           | default_prefixlen |
+---------+------------------+------------------------------------+-------------------+
| 670e... | demo-subnetpool4 | [u'203.0.113.0/24']                | 26                |
| 7b69... | demo-subnetpool  | [u'2001:db8:1:2', u'2001:db8:1:2'] | 64                |
+---------+------------------+------------------------------------+-------------------+
```

Now, let's use them.  It is easy to create a subnet from a pool:
```shell
$ neutron subnet-create --name demo-subnet1 --ip_version 4 \
      --subnetpool demo-subnetpool4 demo-network1
+-------------------+--------------------------------------+
| Field             | Value                                |
+-------------------+--------------------------------------+
| id                | 6e38b23f-0b27-4e3c-8e69-fd23a3df1935 |
| ip_version        | 4                                    |
| cidr              | 203.0.113.0/26                       |
| name              | demo-subnet1                         |
| network_id        | b5b729d8-31cc-4d2c-8284-72b3291fec02 |
| subnetpool_id     | 670eb517-4fd3-4dfc-9bed-da2f99f85c7a |
| tenant_id         | a8b3054cc1214f18b1186b291525650f     |
+-------------------+--------------------------------------+
```

If I run out, I can always load some more prefixes:
```shell
admin> neutron subnetpool-update --pool-prefix 203.0.113.0/24 \
           --pool-prefix 198.51.100.0/24 demo-subnetpool4
Updated subnetpool: demo-subnetpool4
admin> neutron subnetpool-show demo-subnetpool4
+-------------------+--------------------------------------+
| Field             | Value                                |
+-------------------+--------------------------------------+
| default_prefixlen | 26                                   |
| default_quota     |                                      |
| id                | 670eb517-4fd3-4dfc-9bed-da2f99f85c7a |
| ip_version        | 4                                    |
| max_prefixlen     | 32                                   |
| min_prefixlen     | 8                                    |
| name              | demo-subnetpool4                     |
| prefixes          | 198.51.100.0/24                      |
|                   | 203.0.113.0/24                       |
| shared            | True                                 |
| tenant_id         | c597484841ff4a8785804c62ba81449b     |
+-------------------+--------------------------------------+
```

## Horizon Support

Minimal support for this feature is being worked on in Horizon.  At
first, you'll only be able to use existing subnet pools when you create
a subnet.  You cannot manage your subnet pools in Horizon at this time.

## Subnet Pools in the Future

Subnet Pools are used as the basis of another new feature called
*address scopes* coming in the Openstack Mitaka release.  In short,
address scopes will allow Neutron to work with various routing domains.
Subnet pools can be placed in to an address scope.  When this is done,
Neutron routers will honor the scope and will know when addresses can be
routed directly because they are in the same scope.  This will be key to
routing public addresses on tenant private networks and setting up
virtual private networks.

When the address scopes feature is completed, another blog post will
describe it in detail.

<!-- vim:set tw=72 ft=markdown: -->
