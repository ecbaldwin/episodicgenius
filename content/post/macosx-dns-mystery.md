+++
date = "2016-03-02T13:10:52-07:00"
Tags = ["Apple", "Mac", "DNS"]
title = "OS X DNS Mystery"

+++

DNS on my MacBook Pro has been a mystery to me.  But, I can figure it
out most of the time after a little bit of digging.  This time, I have
given up for the moment after spending too much time on this issue.

![MacBook Pro Specs](/images/os-x-yosemite.png)

## The Problem

This started sometime during the day on March 1, 2016.  Everything was
working before.  I'm not aware of any updates to the OS or anything I
did.

DNS works in general but certain private addresses that I can access
through a Tunnelblick VPN have stopped working.  The DNS names can be
resolved through public DNS but I've changed the domains for privacy.
Take a look:

```shell
$ ssh mybox.example.net 
ssh: Could not resolve hostname mybox.example.net: nodename nor servname provided, or not known
```
```shell
$ ping mybox.example.net 
ping: cannot resolve mybox.example.net: Unknown host
```
```shell
$ host mybox.example.net
mybox.example.net has address 10.224.24.147
mybox.example.net has IPv6 address 2601:282:8001:ffba:eeb1:d7ff:fe2c:9c5f
```
```shell
$ cat /etc/resolv.conf
domain wireless.example.com
nameserver 10.20.0.1
```

In case you're wondering, the IP address is reachable from this laptop
but if you know anything about this problem then you'll know that this
has nothing to do with it.  But, I'll show you anyway.  This is not a
connectivity problem.

```shell
$ ping -c 1 10.224.24.147
PING 10.224.24.147 (10.224.24.147): 56 data bytes
64 bytes from 10.224.24.147: icmp_seq=0 ttl=62 time=164.032 ms

--- 10.224.24.147 ping statistics ---
1 packets transmitted, 1 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 164.032/164.032/164.032/0.000 ms
```

## Digging in a Little Deeper

I tried tcpdump to see if the requests were going out.  They go out and
correct responses come back immediately!  Yet, the ssh command still
hangs for a while and eventually times out.  This suprised me a bit.  (I
won't even ask why ssh needs an MX record.)

```shell
$ sudo tcpdump -i en0 -vnnt "udp port 53"
tcpdump: listening on en0, link-type EN10MB (Ethernet), capture size 65535 bytes
IP (tos 0x0, ttl 64, id 21181, offset 0, flags [none], proto UDP (17), length 63)
    10.20.1.162.55765 > 10.20.0.1.53: 13474+ A? mybox.example.net. (35)
IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto UDP (17), length 275)
    10.20.0.1.53 > 10.20.1.162.55765: 13474 1/5/5 mybox.example.net. A 10.224.24.147 (247)
IP (tos 0x0, ttl 64, id 55914, offset 0, flags [none], proto UDP (17), length 63)
    10.20.1.162.55248 > 10.20.0.1.53: 16230+ AAAA? mybox.example.net. (35)
IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto UDP (17), length 287)
    10.20.0.1.53 > 10.20.1.162.55248: 16230 1/5/5 mybox.example.net. AAAA 2601:282:8001:ffba:eeb1:d7ff:fe2c:9c5f (259)
IP (tos 0x0, ttl 64, id 53373, offset 0, flags [none], proto UDP (17), length 63)
    10.20.1.162.49590 > 10.20.0.1.53: 64095+ MX? mybox.example.net. (35)
IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto UDP (17), length 137)
    10.20.0.1.53 > 10.20.1.162.49590: 64095 0/1/0 (109)
```

I know that the [host and dig] commands don't go through the same
channels as other commands.  This boggles my mind but whatever.

I've tried clearing the cache, bouncing the DNS responder, and
rebooting.  None of it made a difference.  But, I figured it couldn't
hurt.

```shell
$ dscacheutil -flushcache
$ sudo launchctl unload -w /System/Library/LaunchDaemons/com.apple.mDNSResponder.plist
$ sudo launchctl load -w /System/Library/LaunchDaemons/com.apple.mDNSResponder.plist
```

Let's turn on logging in the responder according to the [man page].

```shell
$ sudo killall -USR1 mDNSResponder
$ sudo syslog -c mDNSResponder -d
$ tail -f system.log | grep mDNSResponder
Mar  2 12:14:36 My-MacBook-Pro.local mDNSResponder[3922]:  18: Adding FD for uid 501
Mar  2 12:14:36 My-MacBook-Pro.local mDNSResponder[3922]:  18: DNSServiceCreateConnection START PID[4128](ssh)
Mar  2 12:14:36 My-MacBook-Pro.local mDNSResponder[3922]:  18: Error socket 19 created 00000000 00000001
Mar  2 12:14:36 My-MacBook-Pro.local mDNSResponder[3922]:  18: DNSServiceQueryRecord(15000, 0, mybox.example.net., Addr) START PID[4128]()
Mar  2 12:14:36 My-MacBook-Pro.local mDNSResponder[3922]:  18: Error socket 19 closed  00000000 00000001 (0)
Mar  2 12:14:36 My-MacBook-Pro.local mDNSResponder[3922]:  18: Error socket 22 created 00000000 00000002
Mar  2 12:14:36 My-MacBook-Pro.local mDNSResponder[3922]:  18: DNSServiceQueryRecord(15000, 0, mybox.example.net., AAAA) START PID[4128]()
Mar  2 12:14:36 My-MacBook-Pro.local mDNSResponder[3922]:  18: Error socket 22 closed  00000000 00000002 (0)
Mar  2 12:15:06 My-MacBook-Pro.local mDNSResponder[3922]:  18: DNSServiceQueryRecord(mybox.example.net., Addr) ADD    0 mybox.example.net. Addr 
Mar  2 12:15:06 My-MacBook-Pro.local mDNSResponder[3922]:  18: DNSServiceQueryRecord(mybox.example.net., AAAA) ADD    0 mybox.example.net. AAAA 
Mar  2 12:15:06 My-MacBook-Pro.local mDNSResponder[3922]:  18: Cancel 00000000 00000001
Mar  2 12:15:06 My-MacBook-Pro.local mDNSResponder[3922]:  18: DNSServiceQueryRecord(mybox.example.net., Addr) STOP PID[4128]()
Mar  2 12:15:06 My-MacBook-Pro.local mDNSResponder[3922]:  18: Cancel 00000000 00000002
Mar  2 12:15:06 My-MacBook-Pro.local mDNSResponder[3922]:  18: DNSServiceQueryRecord(mybox.example.net., AAAA) STOP PID[4128]()
Mar  2 12:15:06 My-MacBook-Pro.local mDNSResponder[3922]:  18: DNSServiceCreateConnection STOP PID[4128](ssh)
Mar  2 12:15:06 My-MacBook-Pro.local mDNSResponder[3922]:  18: Removing FD
```

I'm not sure what "Error socket ..." means but I wouldn't call this
logging helpful.  After turning on packet level logging, I found
messages like this:

```shell
$ sudo killall -USR2 mDNSResponder
$ tail -f system.log | grep mDNSResponder
...
Mar  2 12:21:06 My-MacBook-Pro.local mDNSResponder[3922]: -- Sent UDP DNS Query (flags 0100) RCODE: NoErr (0) RD ID: 389 23 bytes from port 58829 to 10.224.24.1:53 --
Mar  2 12:21:06 My-MacBook-Pro.local mDNSResponder[3922]:  1 Questions
Mar  2 12:21:06 My-MacBook-Pro.local mDNSResponder[3922]:  0 mybox.example.net. Addr
Mar  2 12:21:06 My-MacBook-Pro.local mDNSResponder[3922]:  0 Answers
Mar  2 12:21:06 My-MacBook-Pro.local mDNSResponder[3922]:  0 Authorities
Mar  2 12:21:06 My-MacBook-Pro.local mDNSResponder[3922]:  0 Additionals
```

If I run the tcpdump above on the VPN interface I do see DNS queries
being sent out with no replies.  I forgot to capture that output before
I worked around the problem and I don't want to go back to get it.  You
can imagine what it looks like.

## Making Progress

Now we're getting somewhere.  Why is it querying 10.224.24.1?!  This
*is* the IP address of a name server.  When I'm connected directly to my
home wireless, it is the correct name server.  But, I'm not connected to
my home wireless.  I'm connected from the road through my VPN.

So, where is mDNSResponder getting the idea that it should go ask this
DNS server?  Also, why does it send out queries on en0 and then ignore
the replies?  Why doesn't clearing the cache, or a reboot change
anything?

- Even if I disconnect my VPN, it tries to go there.
- I told Tunnelblick "Do not set nameserver".
- The VPN server is not configured to send any nameserver, let alone
  this one.
- This isn't the IP address of the VPN server
- This isn't in /etc/resolv.conf or anywhere else on the VPN server.
  The VPN server is running outside of my home network.  A VPN client
  inside my home network is also connected to it, that's how I have
  connectivity to it.
- I don't see it in my Network preferences under any active connection's
  DNS servers.
- It isn't in /etc/resolv.conf on the MacBook Pro.

This and similar hosts with private IP addresses accessible through my
VPN don't work in my Chrome, Safari, and Firefox browsers.  I get
DNS_PROBE_FINISHED_NXDOMAIN for all of them.

My solution for now is to enable DNS queries on the dnsmasq server at
10.224.24.1 from VPN clients by turning off "local-service".  But, this
is still a mystery that I'd like to solve.  That works but DNS queries
are a bit slower through the VPN.

[host and dig]: http://apple.stackexchange.com/questions/157410/dns-works-in-terminal-but-not-elsewhere-in-mac-os
[man page]: https://developer.apple.com/library/mac/documentation/Darwin/Reference/ManPages/man8/mDNSResponder.8.html

<!-- vim:set tw=72 ft=markdown: -->
