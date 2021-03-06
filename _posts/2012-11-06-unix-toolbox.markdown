---
layout: post
title: "Unix Toolbox"
date: 2012-11-06 00:00:00
preview: commit to learning the basic programming tools
---

Software bugs occur for myriad reasons, oftentimes following a common pattern. Bugs introduced by the developer her/himself, however, may have exceptionally idiosyncratic origins. I have no right and certainly not the expertise to speak of other developers bug-generating mistakes -- but, like most younger developers, I am becoming painfully aware of my own bad habits.

The most pernicious has been a tendency to misread the stacktrace when tracking down an actual error. (Cue the trolls, who will tell me, "that's just a sign of your _insert moderately insulting epitaph_". Perhaps; but a bad habit is a bad habit, and the point here is to identify and expunge it.)

For example, let's say I'm working on a Ruby on Rails application, and I get an error that reads "undefined method Nil for NilClass". Recently, I thought this error might have been due to a broken query in the page's controller. Well, as it turned out, the controller performed its action just fine -- it was in reloading the view that a partial expected a variable when there had been none.

Leaving aside the details of this particular bug, focus on the fact that, had I paid closer attention to the stacktrace, I would have reached the solution much swifter than I had.

For my first 1UP challenge, and over the next two weeks, I will devote at minimum 30 minutes a day on using the Unix programs *tail*, *grep*, and the Perl script *ack* to track and pinpoint software bugs in real-time. In daily practice, I use these commands all the time; the 1UP challenge thus will be to use these tools less as the blunt instruments I've cruelly treated them as, and more like the delicate scalpel of an expert.
