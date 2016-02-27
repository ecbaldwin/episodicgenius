+++
date = "2016-02-26T18:55:00-07:00"
title = "Neutron Service Subnets"
Tags = ["OpenStack", "Neutron", "L3", "Development" ]

+++

I'm just returning from the latest OpenStack Neutron mid-cycle in
Rochestor, MN.  It turned out to be great mid-cycle for me.  I wanted to
share a new idea that came out of it.

## The Problem

One thing that has been on my mind for a while now is wasted IPv4
address space by Neutron.  I guess I was waiting for someone else to
bring the solution because I wasn't personally feeling the pain yet but
it is time to solve it once and for all.

### An Address for Every Router

When using the L3 agent in Neutron, a tenant can create a virtual router
and connect its default gateway to an "external" network.  You can then
allocate a floating IP address from the network and associate it with a
VM on your private network to make it accessible externally.  This is a
common usage model with Neutron.

When this happens, the router gets a port and that port is assigned an
IP address of its own from the external address.  The address *might* be
used to provide SNAT service for VMs that don't have a floating IP but
it might not.  It depends on whether the 'default-snat' option is
enabled and whether there are VMs that need it.  The IP address is
allocated regardless.

### Another Address for Every Hypervisor Host (DVR)

If you use the distributed virtual router (DVR) option, then there is an
additional way that IP addresses are wasted.  Most people find this a
bit more subtle and surprising.  When DVR is in use, every hypervisor
host in the cloud is allocated an IP address from the external address.
These add up.  What makes it worse is that these IP addresses are not,
in any way, externally visible.  Normal operation of the router does not
produce or consume any traffic with this address.  This is what makes it
so surprising and disturbing.

For the first case, some operators would like to save the IP address by
turn off SNAT without a floating IP.  Others have asked if they could
use private addresses so that they can run their own gateway box to do
SNAT.  In the second case, no one wants these addresses, ever.

## The Fix

At lunch on Thursday, I was talking to a group of regular Neutron
contributors about the problem and asked if we could just provide a way
to distinguish some subnets from others and reserve them for a
particular use.  Neutron can tell if you're asking for the IP address
for a VM, a router port, a DVR gateway, or a floating IP (among others).
What if we had a way to say "addresses from this subnet should only be
used for DVR gateways?"

If we had such a thing, we could create that subnet with private
addresses, size the subnet appropriately, and all of the dvr gateway
ports would come out of the that subnet.  Problem solved.  Of course, we
have to write the code but this is something that I think can get in to
the Newton cycle.

The current plan is to add a service type field to the subnet.  To start
with, we'll have two possible values other than the current default of
null:  dvr_gateway, and dvr_and_router_ports.  The names aren't written
in stone.  The choice between the two depends on if you want your
routers to get public addresses for SNAT.

Any kind of port can get an address from a subnet without a specified
service type.  This fallback ensures that legacy behavior is always
preserved and there are no surprises when you upgrade.

Let's go get it done in the Newton cycle!

<!-- vim:set tw=72 ft=markdown: -->
