+++
title = "Merge or rebase? Neither!"
Tags = ["Openstack","Development","git","merge","rebase"]
date = "2017-02-03T16:40:00-07:00"

+++

This post proposes an alternative to merging and rebasing that will require
some enhancement to git to work. As far as I know, this isn't possible with git
in its current form but I think it would be worth the effort to make it happen.

I've been using git since about a week after it was first available for
download way back when BitKeeper pulled the rug out from under the Linux
kernel. If I recall, it didn't take long for the "merge vs rebase" debate to
start up. At first, I was firmly on the merge side because I didn't like
rewriting history and it is easy to get in trouble by rebasing under the wrong
circumstances.

Merge commits are important. I recall a few cases where I ran `git bisect` to
find where a problem was introduced and it landed on a merge commit! This told
me that the problem wasn't caused by either code change by itself. Instead, the
problem was introduced by bringing the two together. This is an important
insight that can be lost by rebasing.

## A Simple Feature

Consider the following scenario. Anna branches from master and begins working
on a feature. She finishes the feature and proposes it to be merged back to
master. While her feature is in review, John finishes some work on master which
conflicts with hers. It is Anna's responsibility to resolve the conflicts.
Let's see how she can accomplish this with various workflows.

### Branch / Merge

Using branch/merge, Anna's and John's work both branch from the same commit on
master. Upon discovering the conflict, Anna pulls the change from master
creating a merge commit and she reproposes the change. At this point, master
can be fast-forwarded to Anna's merge commit; another merge isn't necessary.

![Simple branch/merge flow](/images/basic-branch-merge.svg)

**Note:** Right-click and open the image in a new tab to see tooltips on each
of the commits. The final merge to master is a fast-forward. The master branch
ends up pointing to the last blue commit on the right.

### Rebase

I joined the Openstack project over four years ago and had an abrupt change in
perspective. It was my first exposure to the gerrit code review model. In that
model, Anna proposes a change and humans and automated bots both review it. When
they find issues, she addresses them. She runs `git commit --amend` to fix it up
and resubmits it for another round of review.

In general, the flow only allows trivial automatic merges (if at all). When it
encounters a merge conflict, Anna must rebase and resubmit. Let's see how this
flow works with a picture.

![Simple rebase flow](/images/basic-rebase.svg)

The master branch excludes the faded branch so the history looks very clean. In
this simple case, it ends up linear but that's not always the case if it allows
a trivial merge. It will end up simpler in every case because it either loses
the history or keeps it on private branches tucked away somewhere. Imagine
implementing 100 features this way. The cumulative effect is a much simpler git
history.

To make the history of the change available, the server can store old revisions
on private branches. There is still some information loss. The private history
is invisible to `git bisect` for example because the old branches are disjoint
from master. So, if conflict resolution was botched in a subtle way, it is
difficult to discern. If git bisect had the change history, it could help Anna
pinpoint where someone introduced the problem.

### Replay

Now for the new approach. Instead of rebasing and losing information, Anna
replays her proposal onto master. Replay leaves a `replaces` pointer back to the
original in the new commit. The picture shows this with a dashed line. This is
different than the `parent` pointer that git has always had. Git commands like
`git log` and `git bisect` skip these pointers **by default**.

Git commands like `git fetch`, `git push`, `git gc`, and others treat anything
reachable through them just like any other history that must be preserved and
passed around between repositories.

There must be a formal sense of what's upstream and what's downstream. No matter
which side the command is run from, the downstream must always be replayed onto
the upstream.

The result is a hybrid between merge and rebase. The history -- at least what
`git log` follows by default -- is linear like in rebase but the deeper history
of the change is available by following `replaces` pointers from the commit.

![Simple replay flow](/images/basic-replay.svg)

## More than One Commit

If you haven't caught the vision yet, that's okay. This simple example doesn't
show all the benefits of this approach. Let's have a look at a more complex
scenario. We'll see how replay solves some other issues that come up in more
sophisticated scenarios.

Consider this scenario. John has a little bit bigger feature to implement. He
decides it is going to take a couple of commits to fully implement it. He
creates the first commit and then discovers that Anna has checked in a
conflicting feature on master. John merges this into his branch before moving
onto the second change in his series.

John proposes his series of changes for review and reviewers find an issue in
the first change. He fixes this issue with a third commit. Finally, he discovers
that Anna checked in another conflicting feature and he needs to merge it before
his branch can be merged to master.

### Branch / Merge

The following picture illustrates what this will look like with the branch /
merge workflow.

![Intermediate branch/merge flow](/images/intermediate-branch-merge.svg)

### Rebase

In the rebase workflow, the history looks more complex but remember that all of
the old branches are hidden from the master branch. The resulting public branch
has a much simpler linear history.

John folded the fix directly into a new version of the first commit. This
further simplifies the history and makes the first commit a better candidate for
cherry-picking. It is easier to revert it if a serious flaw is discovered later
on.

In my experience, this is friendlier to code reviewers than merging. To review
the change series in the merge workflow, one has to keep in mind that the first
commit is broken but then fixed in a later commit. Imagine how adding more
fixups and iterations to the review cycle make reviewing more complex. Even
those who prefer the branch / merge workflow will often result to some rebasing
to add some sanity to reviewing code. Difficulty comes when others pull changes
to try them out, their life is complicated by rewriting the history.

![Intermediate rebase flow](/images/intermediate-rebase.svg)

### Replay

The replay method looks a lot like the rebase one but adds explicit links to
former revisions. Note that he cherry-picked the fixup into a new revision of
the first commit and so it appears obsolete in the final history graph.

If you followed along up to this point, you might be thinking that this method
has all the complexity of merging plus the complexity of rebasing and maybe
even a little more. How can making things more complex be an improvement.

Keep in mind that all of the complexity behind the `replaces` pointers is
hidden from view unless you explicitly ask to drill into it. To the casual
observer, the history is as simple as in the rebase workflow but it makes the
history of each commit easier to access. For example, it could be made
selectively available to `git bisect`.

![Intermediate replay flow](/images/intermediate-replay.svg)

## Collaboration

Collaboration is something that any good version control system should
facilitate. This section is specifically about collaborating with others on
proposed changes using the rebase model.

### One Change

Consider a scenario where multiple people collaborate on one change. This
scenario comes up naturally in a workflow where thorough code review is
required. Say Anna is reviewing John's proposal. She spots a minor flaw that is
easy to fix and asks John's permission to fix it up herself and resubmit. She's
hoping to save John the trouble of having to plan time to review her feedback
and make the change later.

There are reasons to collaborate more formally with someone on a single change.
If the change is complicated, the collaboration can easily be more involved.

This is where the rebase workflow breaks down. The history of a change is
chronological and therefore linear. When a new version of a change is uploaded
it automatically supersedes all previous proposals for the change. This works ok
when there is one author working in one workspace. It doesn't work as well when
there is more than one of either.

Watch what happens when John (in blue) starts a change and then John and Anna
(in orange) both amend the change to fix it up. As far as git knows, they are
working on unrelated branches. The last change proposed is the latest revision.
Say Anna proposes hers and then John proposes his without first updating to
Anna's revision. John's change clobbers Anna's. The tool doesn't have the
information to sort this out.

![Collaboration on One Change](/images/collaborate-one-change.svg)

When faced with a simple scenario like this, It might be tempting to think this
is no big deal, they just need to communicate. The truth is that this happens
quite often and can easily escape notice.

Replay keeps the metadata necessary to sort this out.

![Collaboration on One Change with Replay](/images/collaborate-one-change-replay.svg)

### More than One Change

Anna and John were able to get the last feature done but made a few mistakes
along the way. Now, they have a more complicated feature to work on together.
They want to get smarter and try to avoid the pitfalls from last time.

They divide the feature development into a few parts. The first change is
groundwork that will enable them to each work independently on second and
third changes. Anna gets a head start on the groundwork and gets that up for
review quickly as a starting point for John to dive in. They each create a new
change with a dependency on the first.

When John or Anna rebases their change, it they have to rebase the groundwork
change with it. This means they're both rebasing that change independently of
each other. Any changes to it on either side will be clobbered by changes on
the other. In this case, John doesn't notice that he clobbered a change Anna
made to the base image until later on when they encounter the bug while
demonstrating the new enhancement.

![Collaboration on Three Changes](/images/collaborate-three-changes.svg)

This is getting pretty complicated for the rebase workflow. But, shouldn't this
be easy? This is git afterall, it excels at this sort of thing. We should be
able to do this. It can be tempting to revert back to branch / merge. In fact,
a feature branch was often suggested in Openstack when things got much more
complicated than this. I always resisted this because I thought that creating a
development branch in Openstack incurred a lot of overhead, reviewers quickly
lost interest in them, and developers got fast and loose with their commits.
When they declared the branch ready to reincorporate into master, the reviewers
would suddenly have renewed interest and would be overwhealmed trying to
understand all of the changes coming in one giant merge.

It turns out that the replay flow will handle these scenarios very well. Since
replay doesn't destroy any history, the common revisions of the base change are
preserved as John and Anna diverge. They serve as common ancestors for merging
just like git normally does today. In the end, they end up with the three
changes lined up nicely with `replaces` pointers pointing to the history.

![Collaboration on Three Changes with Replay](/images/collaborate-three-changes-replay.svg)

It has an added bonus of being able to track distinct contributions from both
authors on any of the changes.

You might notice that there are still three proposed changes here. Merging this
work together isn't quite the same as merging regular old git branches together.
We still want to preserve the distinct changes. It ends up being more of a weave
of commits from both branches. First merge the various revisions of Anna's
groundwork patch. Then, replay Anna's (upstream) and then John's (downstream)
other changes onto that result. It helps to know that the upstream branch should
be replayed before the downstream.

I know this looks pretty complicated. But, having had experience trying to
manage this kind of collaboration on my own, I'm confident that it will be
better to have the tool do it automatically. I'll admit that with the rebase in
the middle of the groundwork revisions, it makes this merge a little bit tricky.
But, the tool now has the information to do it.

### Squash Replay

This is an optional extension of the multiple changes scenario. Some developers
prefer to take a set of changes like Anna and John's feature and squash them all
together when finally merging into master.

This makes me think of times when I created multi-commit pull requests in github
-- something I always wished gerrit could handle in one merge. Some reviewers on
the other end of the pull request insisted that I squash all of them together
before approving the merge. They did this as a matter of course. I always
resisted because I had very good reasons for making them separate commits. Mine
were never haphazard, stream of consciousness commits. I felt it added value to
keep them separate and made them easier to understand and review. Otherwise, I'd
be happy to squash them.

With replay, I would be happy to do the final squash. This kind of squash leaves
`replaces` pointers back to the original commits. I see squashing with replay as
a nice way to group commits that logically belong together into a pull request.
They would appear as one commit by default but I could still drill into see how
they were broken out during development.

![Collaboration on Three Changes with Replay](/images/collaborate-three-changes-replay-squash.svg)

There is an advantage to doing this. If these commits really belong together, it
is easy to cherry-pick them together as a single commit. Reverting as a single
entity would also be easy -- this might be good or bad.

### Still rewrite history?

There may be cases where it will be nice to clean up history before proposing
changes upstream. If Anna goes through lots of tiny iterations privately she may
appreciate the option to automatically minimize changes from upstream when
proposing them. It would start by fetching the latest from upstream and then
finding all of the local iterations that don't appear upstream. These could be
optionally squashed into the simplest form possible and then submitted.

### Gerrit Change Ids

In gerrit's implementation of the rebase flow, it wrote a change id to a line in
the commit message to track the change across the otherwise disjoint private
branches holding the change's history.

    Change-Id: Ib9871b6d00dca82adc450207be0867cf384022da

The replay flow with `replaces` pointers effectively obsoletes this. The history
of a change can always be constructed by following the pointers. When new
revisions are uploaded, they can be easily linked to existing changes by
traversing `replaces` pointers. Since this history can be non-linear, it is more
powerful. There could even be mini proposals posted against changes that are up
for review.

## Open Issues

There are some open issues that I have thought through but not in quite as much
detail.

### Reordering changes

Rebase allows you to reorder, drop, and fixup commits. If you strictly use
`replaces` pointers for these scenarios, the graph gets quite convoluted.

I think that all of these cases can be expressed as some combination of
dropping and cherry picking. Can a change in the middle of other changes simply
be dropped? It will still be reachable through the history graph. But, it would
only be reachable by traversing a `replaces` pointer from one of the later
commits and then traversing a parent pointer. I think this might be enough for
the tool to figure out what is happening.

[//]: # (TODO A diagram of this would be nice)

You can think of reordering commits as dropping commits from earlier in the
chain and then cherry-picking them somewhere later in the chain. In this case,
I want a cherry-pick to leave a pointer to the original but I'm not sure if
this should be a regular `replaces` pointer or something different. It gets
complicated to think about `replaces` pointers crossing each other back in the
history. Again, it might actually be okay, I just need to know what the
scenarios are and think through them.

I think I'd like any cherry pick, even over to another branch to leave a trace
of where it came from. There may be opportunity here to solve the cherry-pick
case more generally.

### Carrying local changes

Here's some history behind how I started thinking up the idea for the replay
workflow. I didn't know it at the time but this is where my mind began working
on the problem.

When I first started working on Openstack, we made a lot of changes in a hurry
to stand up the public cloud we were building. Our intention was always to get
these changes upstream when we could and we eventually followed through with
them but there was time between when we made the changes and when we finally got
them merged upstream.

There is often a need to carry local changes to an upstream master. The ideal of
always fixing bugs upstream doesn't always pan out because reviewers don't
always share the same priorities.

We made a number of changes on a branch in a particular order. Then, we took
those changes and proposed them upstream individually, essentially
cherry-picking each one upstream and proposing it by itself. Some of them were
accepted more readily than others.

The problems came when we decided to update our local copy from upstream. Many
of our changes made it upstream, but others did not. Most of the remaining ones
needed changes to work with the new upstream code. A couple of them were even
rendered obsolete by other changes. We needed a way to handle all of this.

I was still pretty set on the branch / merge flow at this time and I thought
surely there must be a way to do all of this without rebasing. I spent three
whole days trying to work out a way to do this and eventually decided that my
efforts were futile. I tried something very close to the replay flow by
individually merging each proposal up to the latest upstream instead of doing a
single merge of our whole branch with upstream. One problem with this was that
the merge commit of a change was not a good substitute for the original commit.

I found it essential to repropose each change against the new upstream and run
all tests against it. A full code review wasn't essential but I did have
reviewers look over any merge conflicts that came up by going through the same
merge in their own sandboxes and comparing their results with mine. We used a
local gerrit server to essentially take each patch all the way through the
review process onto a new branch.

It was essential to have a way to obsolete any number of the changes that are no
longer needed with the new upstream. The goal was to eventually drive this
branch to nothing so I couldn't have all of the obsolete patches hanging around
after their time. This was a part of what made my attempt and doing this without
rebasing impossible.

I ended up taking the previous branch through a process that I called
delinearization where I tried to rebase each one back to the start of the branch
so that they were all independent changes hanging off of the old upstream. I
wrote a script to do this automatically. If there were merge conflicts, it would
try to rebase it on top of some of the other changes until the merge conflicts
went away. This left me with a bunch of independent branches that could be
vetted independently. It would have taken a lot longer to propose them one at a
time as dependent commits.

[//]: # (TODO Find the code for this would be nice.)

## Conclusion

Thanks for hearing me out. This could change the way that development is done in
git and solves lots of problems that I encountered with both of the prevailing
flows. My hope is that this can bring together both sides.

By the way, I actually wrote my own python code to render the illustrations.
I'm not usually very good at illustrations but I spent quite a bit of time
hacking on this stuff. Take a look. I'm sorry that it doesn't have any tests or
anything like that. I was thinking about it as throw-away code while I wrote it
but decided that it might prove useful when I was done. So, I threw it up [here
on github](https://github.com/ecbaldwin/simgit).
