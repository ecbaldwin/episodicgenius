+++
Categories = ["Development", "GoLang"]
Description = ""
Tags = ["Openstack", "Neutron", "L3", "Development"]
date = "2016-01-15T10:28:44-07:00"
menu = "main"
title = "100.64.0.0/10"

+++

Back in 2012, a new IP allocation was made called "shared address
space."  At first glance, you might think that this allocation is just
like the [old RFC 1918][rfc1918] allocation.  Maybe the IANA decided
that we could use a fresh allocation of these things because our
address space is feeling a little cramped.

That wasn't it.  ISPs and other service providers needed an address
space that can be reused in different geographies, doesn't overlap with
any global IP addresses and, just as importantly, won't overlap with any
private IP address usage on customers' private networks.

I shared my thoughts on how this address space should be used in a
[recent email to the OpenStack-operators mailing list][ml-post].  Here
is a condensed version of that email:

I'd discourage the use of 100.64.0.0/10 for any tenant networks.
The [introduction of the RFC][rfc6598] says that there are limits to
this address space that existing private address spaces don't have.

> In particular, Shared Address Space can only be used in Service
> Provider networks or on routing equipment that is able to do address
> translation across router interfaces when the addresses are identical
> on two different interfaces.

The closest this would get to a customer is the router at the customer's
edge an address from this range (like your home router's external
address).  It may also include equipment that is provided and managed by
the provider.  These devices pass packets and use the shared address
space to communicate with other devices in the service provider's
network but they are never seen by the end user in the packets that
reach their devices.

If tenants and other people in general start getting the idea that these
addresses are just a fresh allocation of new [RFC 1918][rfc1918]
equivalent addresses and they start using them then this will limit
availability for the infrastructure networks where they were meant to be
used.  This defeats the purpose for allocating this space.

Don't use these addresses on your home, corporate or other networks
thinking that this is just an addition to the existing [RFC
1918][rfc1918] addresses!  This address space can be used carefully to
reduce pressure or your address space but use it properly, please.  Use
it for your routers and other infrastructure to talk to each and other
isolated, confined areas but don't expose these addresses to your
end-users and be sure the scope where these addresses are viable within
your organization's network is strictly limited.

If you decide to strictly use NAT between the tenant networks and the
rest of the network, then it is isolated but I still worry about
exposing the addresses to tenants.  It is a small step to go from this
to further use of [RFC 6598][rfc6598] addresses by the customer in other
places.  As an example, say the customer wants a VPN to their tenant
network.  Now, these addresses are routing in to their networks.
Pretty soon, the customer is using the address space on their own in a
way that conflicts with a provider and we're back at square one.
Service providers should always have the right-of-way with this space.

My $0.02

[rfc1918]: https://tools.ietf.org/html/rfc1918
[rfc6598]: https://tools.ietf.org/html/rfc6598#section-1
[ml-post]: http://lists.openstack.org/pipermail/openstack-operators/2016-January/009318.html

<!-- vim:set tw=72 ft=markdown: -->
