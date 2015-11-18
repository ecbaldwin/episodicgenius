+++
date = "2015-11-18T13:22:00-07:00"
title = "A Neutron Layer 3 Model"
Tags = ["Openstack", "Neutron", "L3", "Development"]

+++

Since the beginning of Openstack Neutron (then Quantum), its logical
model has centered around a networking layer 2 construct named the
Network.

The semantics of this construct have not always been clear.  In fact,
there have been some attempts to model layer 3 networks with it.  They
have resulted in some awkward implementations and don't clearly
communicate to the end user the capabilities of the network.  Recent
discussions on the mailing list and examining the code base have made it
clear to me that the Network is meant to represent an L2 broadcast
domain.

As operators are trying to scale Neutron to larger datacenters, they're
looking to layer 3 networking to avoid issues around large layer 2
broadcast domains.  At the same time, fewer of us are interested in
anything more than just getting an IP address.  I'm not suggesting we
leave behind the layer 2 use cases; I'm suggesting that we enable a new
model which will work for the rest of us and better enable large scale
deployers.

Let's consider an example.  Say my cloud is built with 32 racks of
compute servers.  I want it to support a flat network model where VMs
boot straight to the external network.  I'd also like to route to the
top of each rack for scalability.  I could model this using a single
Network.  In that case, I'd be breaking the broadcast domain semantics
and I'd still have a problem assigning an IP from the right Subnet given
the location of the instance.  I'd soon run into problems with migration
and possibly other things.

Alternatively, I can model each rack's network with its own Network.
The broadcast domain semantics are preserved, the subnets line up with
the broadcast domains as expected and migration to another host in the
same network works as expected.  The problems now are that there are 32
Networks for the user to choose from and each of them is only available
to a subset of the hosts.  Why would the user pick one of these networks
over the other?  It is bad practice to present a user with such an
option without the context or need to choose one over the other.  For
this to work, we need a new construct which aggregates these networks in
to one logical thing from the user's perspective.

## Neutron Model

Here is a picture of some of the relevant parts of the Neutron model.

![Current Neutron Model](/images/NeutronL2Model.svg)

Notice that the Subnet and the FloatingIP are both attached directly to
the Network.  Since the Network provides a broadcast domain it is
implied that the Subnets are bound to it.  An IP subnet per broadcast
domain is a good starting point and it has been seen by many as good
design practice.  However, it doesn't work with every deployment.  It
precludes the possibility of having a routed network that is decoupled
from the broadcast domain.

## Subnet

The Subnet construct may initially have been meant to represent the
layer 3 part of the network.  However, a Subnet is really just a single
cidr with a gateway.  Today's networks often make use of more than one
subnet together because of IP fragmentation, prefix migration, or other
reasons.  The only way the model allows more than one is to group them
by attaching them to a common Network.  This means the Network is really
acting as the main layer 3 construct.

This is supported by the fact that floating IPs associate with the
Network.  However, moving floating IPs to the Subnet isn't quite right
either.  Consider when there are multiple subnets; we don't want the
user to be concerned with multiple equivalent subnets.  We want them to
be able to select the group of subnets.  The Network is the thing that
groups Subnets.

## Router

You might be wondering whether the Neutron Router object is the answer.
Afterall, it is the basis of the original layer 3 extension in Neutron.

The Router is a simple device which itself has ports that connect to
Networks.  From the API consumer's point of view.  The Router is a
virtual device that we create to connect our virtual Networks.  It isn't
something to connect another port to directly.

On the other hand, the Network is more of a device independent
abstraction representing a broadcast domain.  We connect our ports to
Networks because what we really care about is what they can reach.  We
need something that is more analogous to a Network but in layer 3:  a
device independent abstraction representing a routing reachability
domain.

In my opinion, we shouldn't try to extend the Neutron Router to fill
these use cases.

## IpNetwork

I propose that the model needs a new construct to model an L3 network in
Neutron.  For now, I'm going to call it the IpNetwork.  The name is up
for discussion but I like IpNetwork because it reflects the analog to
the existing Network construct and distinguishes it as a pure IP
construct.

The way I see it, we should add this construct to Neutron soon so that
end users can begin to shift their focus toward it and away from the
Network when they aren't interested in the broadcast domain semantics of
the network.  This allows operators the freedom to deploy L3-only
network architectures.

![Neutron Model with L3](/images/NeutronL3Model.svg)

This essentially inserts IpNetwork between Network and Subnet.  An
IpNetwork can be associated with a Network to fill today's use case
where Subnets are associated directly to the Network.  An IpNetwork
could also stand alone without any associated Network to represent a
pure routed network like [Calico].

[Calico]: http://www.projectcalico.org/

This is a big change.  It may take more than a single cycle to
accomplish it.  It will take a pretty hefty migration and some extra
code to make the new API compatible with the old one (at least in
existing configurations).

### Work going on in Neutron

I have written several attempts to address this.  The most recent [spec]
is up for review.  A [similar spec] has been written by Neil from
[Calico].  We've talked and plan to converge on a single strategy.
We've already made a lot of progress on it.

[spec]: https://review.openstack.org/#/c/225384/
[similar spec]: https://review.openstack.org/#/c/238895/

<!-- vim:set tw=72 ft=markdown: -->
