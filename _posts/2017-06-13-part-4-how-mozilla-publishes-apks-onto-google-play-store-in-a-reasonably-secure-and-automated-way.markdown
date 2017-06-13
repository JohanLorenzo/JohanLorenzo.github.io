---
layout: post
title: "Part 4: How Mozilla publishes APKs onto Google Play Store, in a reasonably secure and automated way"
author: Johan Lorenzo
date:   2017-06-13 8:56:29 +0000
tags: releng
---
The Release Engineering team fully-automated the publication of Firefox for Android in version 53.0. Let's see what was already there and how things have changed since version 53.0.


> This blog post is a part of a serial. Checkout the other posts:
> 
> 1. [How did the project start?]({{ site.baseurl }}{% post_url 2017-06-06-part-1-how-mozilla-publishes-apks-onto-google-play-store-in-a-reasonably-secure-and-automated-way %})
1. [Presentation of the solution]({{ site.baseurl }}{% post_url 2017-06-07-part-2-how-mozilla-publishes-apks-onto-google-play-store-in-a-reasonably-secure-and-automated-way %})
1. [5 things I would have loved knowing about Google Play]({{ site.baseurl }}{% post_url 2017-06-12-part-3-how-mozilla-publishes-apks-onto-google-play-store-in-a-reasonably-secure-and-automated-way %})
1. What's next? Want to contribute? & Special thanks [Here]

# What's next? Want to contribute?
 
The initial publication was the first big step of the project. The next big one will be to expand the release workflow to simplify [what percentage of the user base is getting the latest update](https://bugzilla.mozilla.org/show_bug.cgi?id=1366820).
 
If you want to contribute, there are also [a few](https://github.com/mozilla-releng/mozapkpublisher/issues) [open](https://github.com/mozilla-releng/pushapkscript/issues) [bugs](https://github.com/mozilla-releng/scriptworker/issues). You might also want to take look at [MozApkPublisher](https://github.com/mozilla-releng/mozapkpublisher) and [pushapkscript](https://github.com/mozilla-releng/pushapkscript) to see how we added our checks.

Finally, if you have some stories on how you use Google Play you would like to share, questions or comments about our tools, please [leave a comment](https://github.com/JohanLorenzo/blog/issues/4).
 
# Special thanks
 
A lot of people were involved in a way or another in this project. By alphabetical order:
 
* Rail Aliiev, for the help spawning nightly tasks against the correct events and more hints
* Chris AtLee, for guiding me during the entire project and always pointing me in the right direction
* Chris Cooper, for organizing the Taskcluster migration work week, which allowed to put the last bits for Firefox 53.0
* Julien Cristau, for giving feedback from Release Management's point of view at almost every step we made
* Guillaume Destuynder, for reviewing the security risks
* Ben Hearsum, for reading and testing out the documentation in place
* Liz Henry, for helping with Google Play permissions
* Sylvestre Ledru, for the initial solution and the overall administration of Google Play
* Francesco Lodolo, for detecting and helping diagnose locales-related bugs
* Jordan Lund, for the help in the 52 release duty cycle, while the taskgraph was being finished up in 53
* Ritu Kothari, for initiating the new rollout process
* Amy Rich, for watching my back with Puppet
* Aki Sasaki, for helping me with scriptworker and reviewing so many of my patches across several components
* Mihai Tabara, for wiring the last parts of the Taskcluster graph together
* Justin Woods, for the tips and tricks of Taskcluster's taskgraph
* Anybody involved in the Android Taskcluster migration, and who might not be in this list
* The reviewers of these articles
* And anyone I missed above

# Comments

You can read and leave comments on [this Github issue](https://github.com/JohanLorenzo/blog/issues/4).
