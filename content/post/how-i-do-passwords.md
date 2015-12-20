+++
title = "How I Do Passwords"
Description = "A run-down of how I manage my passwords"
Tags = ["Passwords", "Security"]
date = "2015-12-19T22:10:27-00:00"

+++

I'm not paranoid, I just like to think paranoid.  I'm actually a pretty
trusting person.  But, I know the reality that if I let my guard down,
someone will take advantage.  I've had two bikes stolen because I let my
guard down, I know this.  But seriously, thinking paranoid, or
defensively takes some serious thought, learning, and practice.  And, I
find it fun.

With that in mind, I'd like to share how I handle passwords.  I've
already shared a little bit of my [frustration with having to come up
with usernames for every little site](../no-more-usernames).  Passwords
are different.  I insist on coming up with a unique password for each
and every site.  If a password is compromised on one site, it must be
completely useless in any other context.  Really!  You shouldn't be able
to learn anything about me from it that could ever be used to take
advantage of me elsewhere.

I'm a major fan of multi-factor authentication but I absolutely cannot
stand those security questions that many sites come up with.  I won't
really go in to these two subjects much in this post.  I just wanted to
mention them.

## Entropy

For me, the first key to a good password is lots of entropy.  I tend to
equate the word entropy with a bunch of random bits that are sourced in
a way that no one could ever possibly reproduce or guess any of them.
I don't remember the last time I included anything in a password that
had anything to do with me.  No dates, dictionary words, favorite
numbers, or anything like that.  So, forget about it.  You're not going
to crack my password by going through Falken's Maze.

I actually -- no exaggeration -- once created a pass-phrase by flipping
a coin a bunch of times and mapping the result onto a character set of
commonly acceptable characters.  I knew without a doubt that no one
could ever influence the outcome without my knowing it.  I used it for a
while but the thing was so hard to remember and so insanely difficult to
type that I decided I needed a different strategy.  Random bits are
still key but I needed a way to make it work for me.

## Password Hashing

A very difficult part of employing a new password management scheme is
integrating in to your routine.  This makes it hard to go off the beaten
path and come up with your own scheme.  The path I chose was blazed by a
couple of others.  I had [some code to get me started](hasher_github).
The most notable is the [Password Hash Plus](passwordhasherplus)
extension for Chrome.

Since I insist on different passwords for every site and I believe that
the strongest passwords are virtually impossible to remember, it stands
to reason that I'm going to need some apps to help me out.

At this point, I had multi-factor on my mind and I had the idea that I
could combine bits from a few different sources to create a password.
One component would be a password that I could remember and one that was
chosen to be easy to type.  Over time, I've taken to using a handful of
different passwords in different contexts like home, work, etc.

The second component is another chunk of entropy.  In fact, it is too
much entropy for me to remember and easily enter at the keyboard.  I
don't have it memorized.  Instead, I have a couple of apps -- [one for
Chrome](passwordfortifier) and [another for Android](hashit) -- that
remember it for me.  I store it in my safe and I manually enter it in to
each instance of the apps.

When I need a password the apps help me out.

[Password Fortifier](passwordfortifier) detects password fields on web
pages and does all of this automatically.  It helpfully turns the
password field a shade of gray indicating that I had asked it to hash
the password for that field before, I type my password in the field and
submit the form.  It does the rest.

[Hashit](hashit) isn't quite as well integrated but it isn't too bad.
When I need a password, I "share" the page with the app.  This passes
the url over to it and presents me with a field to type my password.
Then, I hit done and the app automatically copies the password to the
clipboard and sends me back over to the browser where I paste it.  Not
too bad.  I can use this app without a browser by manually typing the
relevant part of the url (the "site key") in to a field and my password
in another.  This is helpful if I need to enter my password in to
another Android app.

## How a Password is Created

The three essential components that go in to my hashed passwords are the
"site key", the secret seed, and my master password.  The apps
essentially concatenate these three components and run a strong hash
algorithm over it.  It then maps the result to a character set that can
optionally contain special characters.

### Master Password

I keep this in my head and provide it to the app each time I need to
sign in somewhere.  I randomly generated mine but it isn't super long.
If it were too long it would be a chore to type it in.  Too short, and
it would be easy to brute force.  (But, you'd have to have access to the
secret seed too.)  I went through a number of candidates before I chose
one that was pretty natural for me to type.

There are a lot of utilities out there for this.  Like with the secret
seed, be sure that your utility is generating a password using random
data from a good fresh entropy source.

```shell
$ pwgen
pheeFie4 chee2Aip Quini2wu kaey8Voo aeHoh8po ou0Inuiv eim4AiTh Luewah3p
...
$ apg

Please enter some random data (only first 16 are significant)
(eg. your old password):>
VusByffAw2 (Vus-Byff-Aw-TWO)
thugItGuOm7 (thug-It-Gu-Om-SEVEN)
iardInboc1 (iard-In-boc-ONE)
AthGeac0 (Ath-Geac-ZERO)
8quejEawOidd (EIGHT-quej-Eaw-Oidd)
wurIOc6OcA (wur-I-Oc-SIX-Oc-A)
```

### Secret Seed

I generated this using 128 bits out of /dev/random.  This secret should
be well guarded.  For this reason, [Password
Fortifier](passwordfortifier) is careful not to sync this value to
Chromesync.  It also does not send this value over to the context script
to mingle with the rest of the javascript on the login page.

[Password Fortifier](passwordfortifier) allows me to change the secret
key used for new passwords.  It can handle an number of them at the same
time.  It simply remembers which one it used for a given site key.
[Hashit](hashit) doesn't allow me to do this yet.

Please, don't use an online UUID generator!  That's just wrong.  There
is one built in to [Password Fortifier](passwordfortifier).  You can use
it if it you're comfortable with that.  I didn't.  You could also try
one of these, just be sure you're getting a random data based uuid:

```shell
$ cat /proc/sys/kernel/random/uuid 
e17693bf-b7fc-4f98-886e-801daf8fe42d
$ uuidgen -r
c87eb4d3-a597-4c6f-be30-847e33e27616
$ uuid -v4
290aa813-aa02-4be5-a1e0-9b0a22c18950
```

### Site Key

This is the piece that ensures the password is unique for each site.
It is derived from the hostname of the page I'm logging in to.  For
most sites it is just the second component of the hostname.  Here are
some examples:

```
byu.edu -> byu
www.netflix.com -> netflix
byu.net -> byu
google.com -> google
```

You might notice that a few of these map to the same key.  This means if
everything else is the same then you could end up with the same password
for two sites.  Indeed, this is something to watch out for.  To me, it
isn't a problem.  I need to always be aware of where I'm signing in.

## The future

I'd like to continue making it more convenient for me while still
protecting the entropy that makes my passwords secure.  I have been
through all of the source code for the Chrome and Android apps that I
use and I've begun to develop them.

My version of Chrome app [is available here](passwordfortifier).  It has
been working well for me and enhances [the original](passwordhasherplus)
in a few ways.  Most notably, it uses Chrome sync to copy some of the
metadata between your different browser instances.  For example, if you
decide to change the settings for your bank passwords to make them
longer, it will remember that setting across Chrome instances.  Don't
worry, it doesn't send the secrets to Chrome sync.  You still have to
sync that manually between browsers.  That's the way I like it.

If I had some extra time, I'd like to get the Chrome app and the Android
app to sync metadata between them.  That will take something other than
Chrome sync since I haven't found a way to get access to Chromesync data
from an Android app.  I'd also like to make it a bit more user friendly.
Realistically, I don't have time to hack more on this.  I got it to the
point where I'm very happy with it and now I'm moving on.  It is all
open source software, maybe you could do something with it.  I'd be
happy to give some guidance.

I'd really like to see some standards developed around the area of
password management.  For example, it would be nice if Android password
management apps were well integrated in to the apps that need them.
Also, the url munging that I use to get the "site key" could be
ambiguous.  k

[passwordhasherplus]: https://chrome.google.com/webstore/detail/password-hasher-plus-pass/glopbmohkffbnplcjbbbfmmimfhfnhgd?hl=en
[hasher_github]: https://github.com/ericwoodruff/passwordhasherplus
[hashit]: http://android.ginkel.com/
[passwordfortifier]: https://chrome.google.com/webstore/detail/password-fortifier/kgmpfgadlkgfogigokibmgpckflefokd/related?hl=en-US&gl=US

<!-- vim:set tw=72 ft=markdown: -->
