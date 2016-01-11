+++
date = "2016-01-11T09:58:53-07:00"
Tags = ["Development", "Java"]
title = "I don't use Java/C asserts"

+++

## Year 2000

I was an intern working on timing tools for the PA-RISC and Itanium
processor projects at HP.  We were starting a new C++ project.  My
mentor -- who is a great engineer and I still admire in many ways --
suggested that we use asserts to check certain invariants on  entering
and exiting methods.  We could even do complex checks that reduced
performance because once we finished all of our testing, we could just
turn them off and it would all be great.  I bought in; it sounded great!

## Year 2008

I started learning Java and found that it had asserts too.  Without
thinking, I started using them.

A bit later, I found that Eclipse does not enable asserts (-ea option)
by default.  I didn't like that my tests could be running without
asserts so I wrote a unit test that would fail if they were not enabled.

## Year 2011

I started a personal hobby project using Java and I wrote a little bit
of code that used asserts.  One piece of code was a matcher meant to
assert that an XML document had the correct structure.  For a particular
list of properties in the XML, the higher level matcher would load
little  matchers to check each property.  At the end, there was one
assert to check that all of the properties were validated by a matcher.

I lost steam on this project and didn't work on it for some time.
Mostly because I changed jobs to one that truely interested me and took
my time and attention away from my own projects.

## Year 2015

I had a couple of weeks off work at the end of the year and I started to
rekindle interest in my old project.  For one thing, I had a new use
case that I thought it could help with.  I had never really lost
interest; I just did not have the energy to apply to it.

I got the project out, blew off the dust, updated its dependencies, and
ran the unit tests.  They all passed!  I even enabled assertions in
Eclipse because my one test that they were enabled failed.  I thought I
was in business.

When I went to run the server, it wouldn't run.  I was a bit sad but I
dove in to figure out what went wrong.  I did so by beginning to write a
suite of integration tests that exposed the problem with the server.
The integration tests ran under a whole new runtime that included server
and client jars so that real requests could be made and validated.

I realized that once I unmarshalled the XML response from the server, I
could reuse the same matchers that I had written for the unit tests.
Score!  I was excited about that.  Soon, I had the server running and a
small suite of integration tests that would ensure that I wouldn't have
to go through this again.

Well, next came the task of automating my tests.  I'd never used Maven
before and decide to create a simple maven project for my code so that I
could get some true CI going on.  I was getting even more excited.  But,
the integration tests failed!  What?!  It didn't make any sense to me
for a few hours and I was stumped.  I had to sleep on it.

It took a bit of digging but I finally figured out that my integration
tests were not properly decoding responses.  The test failures were
legit.  So, why didn't Eclipse catch this?  And, thanks to Maven for
enabling assertions so that I did catch it.

### Asserts are Harmful

The same problem happened under Eclipse but I forgot to enable
assertions for integration tests.  Remember I had them enabled for unit
tests but it didn't occur to me to do it again for integration tests.

The XML did not get properly unmarshalled to JaxB objects, my matchers
failed to validate the properties, and my assertion failed to fire.  The
tests passed.  Yuck!  I'm embarrassed to even admit all this.  The whole
thing should've been avoided.

I fixed it in two steps:

1. Fix the real unmarshalling problem and ensure asserts are enabled.
1. Remove all use of asserts in my project.  Exceptions are better.

## What is this project?

You're probably curious.  I'll give you the overview of the project.
Basically, it starts with a WebDav server and client.  That's the easy
part, it has been done before.  But, from there, I have some thoughts
around how some particular caching strategies could be applied to
utilize as much local storage on the client as is available to keep the
recent stuff local and keep the client from having to ask the server for
things.

I'm not the first person to try to apply caching to WebDav.  I realize
that it has been done before.  However, my ideas would extend WebDav in
ways that would make it much more efficiently cacheable.  It would
require a server and client that both understand the extensions to
perform optimally.  But, they would maintain backward compatibility with
current WebDav compliant servers and clients.  I hope to blog a bit more
about this as it develops.  I'm just playing around with it in my spare
time for now.  I don't have much of that.

<!-- vim:set tw=72 ft=markdown: -->
