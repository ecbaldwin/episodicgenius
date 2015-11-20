+++
Tags = ["Development", "Quality", "Testing"]
date = "2015-11-30T16:45:00-07:00"
title = "Validate before you Merge"

+++

I was chatting with my brother last Saturday while we were watching a
soccer game.  He was frustrated with the lack of discipline in the
developers at his new job.  After a lengthy conversation, we agreed that
much of the problem boiled down to something I feel very strongly about.

I think this is a such a fundamental and obvious best practice that I
would say if you're doing it wrong then you should make it your highest
priority task to go fix it now.

Basically, you should be doing all of the validation you can possibly do
before you merge changes or new code.  If you hear someone say "let's
get it merged so that I can take a look at it" or "... so that we can
start testing it" or anything similar then your development flow is
seriously flawed.  You'll find that you can't just seem to pull anything
together.  You should be working to grow your automated test suite with
every bug fix and new feature that goes in.

This also includes code review.  Your changes should be seen by at least
two other pairs of eyes *before* it merges.

There is some testing that can be done after merging.  In some cases, it
just isn't feasible to do it before merge.  In my opinion, if you don't
have the automated testing in place to gate merges then it doesn't make
sense to try moving on to these types of tests.

<!-- vim:set tw=72 ft=markdown: -->
