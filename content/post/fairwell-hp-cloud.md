+++
date = "2016-01-01T19:45:10-07:00"
title = "Fairwell HP Public Cloud"
Tags = ["Openstack", "Neutron"]

+++

I got started in OpenStack when I took an unexpected job opportunity at
HP to help stand up Neutron in the second release of [HP Public
Cloud](http://www.hpcloud.com/).  At the time, it was among the largest
scale deployments of Neutron in existence.  I suspect it was the largest
at the time.

One of the perks of the job was access to free virtual machines to use
for work and even some personal use.  I didn't pay for cloud resources
for 3 years.  It was reliable and reasonably performant.  I think I
probably would have used it even if I had to pay.

Unfortunately, after its initial release in 2013, we didn't put effort
in to updating it.  While it was a great engineering accomplishment, it
was superseded and became obsolete.

A couple of months ago, as I had suspected would happen, HPE announced a
[new model to deliver public
cloud](http://community.hpe.com/t5/Grounded-in-the-Cloud/A-new-model-to-deliver-public-cloud/ba-p/6804409#.Voc9JZMrLYb)
that would include the sunset of the current deployment and deliver
public could by partnering with others.  This was a bittersweet moment
for me but, more practically, it meant that I had to find a new cloud
provider.

I decided to write this post now because I spent a decent part of the
day moving my personal stuff off of HP Public Cloud and I no longer have
anything running out there.  I thought it appropriate to mark the
occasion with this post.

I have to say that my transition was made a lot easier because I ran my
services in [Docker](http://docker.io) containers.  Moving them was
mostly a matter of booting VMs, installing docker, moving a bit of data
from one storage to another, running the Dockerfiles, and twiddling DNS.
I'll admit it took me a little longer than it needed to because I felt
the need to update software versions and reorganize my source files a
bit.

When all was said and done, I moved a personal gerrit server (hosting
personal projects including the source code for this blog), my
home-grown finance software, this blog server, a VPN server, my
Openstack development machines and devstacks, and some archives.

I'll always think fondly of the time that I spent on HP Public Cloud.
Despite its short-comings and eventual demise, I consider it a great
personal accomplish and something I'll always be proud of.

<!-- vim:set tw=72 ft=markdown: -->
