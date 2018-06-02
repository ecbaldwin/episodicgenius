+++
date = "2018-06-01T20:09:00-06:00"
title = "Go dep inputs-digest"
Tags = ["golang", "vendoring", "dep"]
+++

I've been playing around with go for a few years now and vendoring for a couple.
Like about everyone else, I have some strong opinions about how vendoring should
work. I think the newish `dep` tool is my favorite so far (good thing, because
we're all going to be stuck with it) but there is one thing that is really
bothering me about it.

Part of my philosophy around vendoring is that you should only manually update
what you have to to make progress in your own code. Sometimes you need a newer
version of a library for a bug fix or some new functionality.

In my opinion, the rest should be updated automatically. So, I attempted to
write some automation around discovering libraries that can be updated and
proposing commits to update them for review. This way, the updates can run
through the CI gauntlet to be vetted before being merged. If our project
is in a place where we want to accept updates, then it is a simple matter of
approving and merging it (that's the first human interaction). If we're not in a
position to accept updates, we just let them hang out until we are.

My automation generates a handful of independent proposals. Normally, these
proposals could be accepted and merged in any order without conflict but
occasionally, one might depend on another one going in first in order to work
correctly. However, the `inputs-digest` field in the `Gopkg.lock` file
complicates everything. As soon as I merge anything that changes vendoring, all
of the other proposals go into a state of merge conflict and have to be updated.
It really impedes a process that would otherwise be pretty smooth and pleasant.

I found a [proposal to discontinue the use of the inputs-digest][discontinue],
which I fully support, and I commented with the following:

> The `inputs-digest` field ensures that the the `Gopkg.lock` file has had the
> correct value for it -- based on the source files -- written to it at some
> point. From this I might extrapolate that all of the contents of the lock
> file are correct given those inputs. From that -- if I'm really brave -- I
> might extrapolate that the vendor directory also contains all the right
> contents. These are leaps of faith that I'm not comfortable making.
>
> It causes merge conflicts because it is the single hottest point of merge
> contention in all of my projects that use vendoring. Besides that, I'm
> having trouble figuring out how it is helping me.
>
> Let's say I clone a project and I run `dep hash-inputs | tr -d "\n" | shasum
> -a256` on it and it doesn't match? Say I run `dep ensure` and all it changes
> is this hash. This probably just means that someone didn't commit something
> completely. So what? Why is it important to me that this hash is correct
> enough to create a new commit to fix it and post it for review?
>
> Even then, it is still a big leap of faith to say that if these values match
> then the entire contents of the vendor directory are good. The only way that
> I know how to be sure that is true is to run `dep ensure` on a clean working
> tree and check that nothing gets modified.
>
> I'd suggest doing one of the following.
>
> 1. Just get rid of this field. To be backward compatible, add a Gopkg.toml
>    setting to stop writing it. Seriously, I don't think anyone will miss it.
>    I don't even think we need to do any of the "list of checks" outlined in
>    the previous comment. Those can be done later.
>
> 1. Replace it with merge friendly toml encoding of `dep hash-inputs` (i.e.
>    not a single value squashed into one line that will cause a merge
>    conflict on every change). This would serve the same purpose but me merge
>    friendly.

Go [give it a thumbs up][discontinue], I hope to get some time soon to propose a
code change to address this. But, I started a new job and my time is mostly
consumed.

[discontinue]: https://github.com/golang/dep/issues/1496#issuecomment-392568459
