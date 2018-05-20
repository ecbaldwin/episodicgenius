+++
date = "2018-05-19T18:00:00-06:00"
title = "Moved to Netlify"
Tags = ["CDN", "Publishing"]
+++

I have a weekend between jobs. I left my job yesterday and I'll fly out to NYC
to start a new one on Tuesday. As I leave, I'm realizing it was stifling my
creativity. Now that I'm free, I feel it returning and I hope to post with a
little more frequency.

With some time on my hands this weekend, I looked for a better way to host this
site; you know more responsive, easier to manage, and all that. My friend over
at [Silicon Loons][siliconloons] turned me on to [netlify] and I thought it
looked like a pretty good way to go. It turned out to be super easy and I was
able to use it to check another todo off my list: add https with [Let's
Encrypt][letsencrypt].

The highest hurdle I had to jump was my use of git [submodules] to bring in my
theme from github. It would've worked if [netlify] cloned with --recursive but
since they don't, I had to work around it. I'm a fan of [submodules] but I know
I'm in the minority. It wasn't a big deal so I decided to use [git subtree]
instead.

After that, everything was really easy. There really wasn't much to it. Just [a
few changes to my git repo][gitchanges] and some clicks through the UI to
connect [netlify] with my blog's source repo. Oh, and a new CNAME in DNS. Once
it propogated, I was in business. The [https certificate][letsencrypt] came
literally with a click or two.

[netlify]: https://www.netlify.com/
[letsencrypt]: https://letsencrypt.org/
[siliconloons]: https://www.siliconloons.com/
[submodule]: https://git-scm.com/book/en/v2/Git-Tools-Submodules
[git subtree]: https://github.com/git/git/blob/master/contrib/subtree/git-subtree.txt
[gitchanges]: https://github.com/ecbaldwin/episodicgenius/commit/6c2570cf4594a1becd8de22755959393af61dca3
