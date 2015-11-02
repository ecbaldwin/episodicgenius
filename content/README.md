+++
Categories = ["Development"]
Description = ""
Tags = ["Development"]
date = "2015-11-02T13:25:32-07:00"
menu = "main"
title = "README"

+++

Episodic Genius
===============

Episodic Genius Blog Site

Howto
=====

To generate this site, follow the instructions below:

1. Install Hugo with [these instructions](http://gohugo.io/overview/installing/).
1. Clone this repository
   * `user="carl-baldwin"`
   * `git clone --recursive ssh://$user@gerrit.cobaldwins.net:29418/episodicgenius`
   * `cd episodicgenius`
   * `git config --add gitreview.username "$user"`
   * `git review -s`
1. Build the site using hugo
   * `hugo`
1. Sync the content out to your hosting provider:
   * `# TODO rsync -avz -e ssh public/* carlbaldwin@sadira.dreamhost.com:blog.episodicgenius.com/`
1. You're done!

To create new posts and run a local server ...

*  `hugo new post/welcome-to-hugo.md`
*  `hugo server --watch`

References
==========
The following are links for references for working with Hugo and Markdown:

* Markdown Cheatsheet: https://github.com/adam-p/markdown-here/wiki/Markdown-Cheatsheet
* VIM spell checking: http://robots.thoughtbot.com/vim-spell-checking
