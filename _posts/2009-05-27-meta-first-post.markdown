---
layout: post
bargo: 2009-05-27 19:27:32
title: Obligatory meta first post
tags: ['blogging', 'meta']
---

Not that it should matter, but this blog is running under
[Bryar](http://search.cpan.org/dist/Bryar/).  There is, as you can see, a
very small tale of woe to go with that.

A friend of mine suggested I get to the blogging again, presumably
rather than just unloading my brain in his direction.  Eventually I buckled
and installed wordpress because the last thing I needed was the effort or
or distraction of maintaining a blogging platform to suck the time up.

Then I upgraded my box from etch to lenny, and the php4-mysql package got
conflicted out taking wordpress down with it before I even got going
(the upgrade ate my homework).

So of course, I decided to install something that'd cope with flat files
at the backend, maybe be written in perl in case of dire emergency, but still
nothing I had to write or understand that much to use.  Guess what,
that didn't go well either.  Burned about a week of time, and about two weeks
of good intentions in trying to make
[Angerwhale](http://search.cpan.org/dist/Angerwhale/) install.

Going back to my friend with my tale of woe he just suggested I cut the knot
and run bloxsom, which rang bells, not least because I remembered that Simon
had written Bryar as a more modular replacement.

It installed quickly and cleanly.

It worked.

So of course I had to [hack on it][plugin-github] (probably on CPAN in
due course)

[plugin-github]: http://github.com/richardc/perl-bryar-datasource-flatfile-dated-markdown/tree/master
 

Bugger.

Going to be a couple more entries to queue up, and some html mangling, before
I officially abandon the idea again, so watch the RSS I guess.
