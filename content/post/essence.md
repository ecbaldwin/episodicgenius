+++
title = "Essence"
date = 2011-03-04T21:58:00Z
updated = 2011-05-27T12:36:08Z
tags = ["etag", "hash", "git"]
blogimport = true
aliases = [ "/2010/12/essence.html" ]
[author]
	name = "Carl Baldwin"
	uri = "https://www.blogger.com/profile/15809009204843590598"
+++

The essence of an entity is a number, sort of a handle, to its exact contents.  The only requirement is that if two representations of an entity are different then their essence values must be different.<br /><br />I came up with concept quite a while ago, probably 6 years ago, while I was working with very large hierarchical IC designs.  We were trying to figure out how to run tools more quickly on large designs.  I began to think of entire sub-hierarchies as entities.  I realized that the essence value for a whole hierarchy could be calculated by combining the essence values of its sub-hierarchies.<br /><br />This was around the same time that git was developed for Linux kernel development.  I watched the development of git from day one.  I was using git as an integral part of my development within a month or two.  I got inspiration from it and realized that its underlying concepts could be used in other ways.  Essentially, git uses a sha1 as an essence value for files (blobs) and directories (trees).  It does it hierarchically by including the essence values of files and sub-directories in calculating the essence value of a parent directory.  This was very much like what I needed.<br /><br />Recently, I've been trying to learn more about http.  I've read RFC 2616 (Hypertext Transfer Protocol -- HTTP/1.1) in its entirety and realized that it has a concept almost exactly like my essence concept.  They call it an entity tag.  More specifically a strong entity tag.  I was pretty excited to see this in there.  So much focus is given to cache validation using timestamps, ages and expiration that I didn't even realize that this was in there until I read it.  HTTP became more than just a necessary evil to me.  I think I will enjoy it.<br /><br />Obviously, I didn't come up with the essence concept but I see much more potential in this concept than has been tapped in the few applications that I've come across.  I hope to be responsible for tapping this potential in the near future.
