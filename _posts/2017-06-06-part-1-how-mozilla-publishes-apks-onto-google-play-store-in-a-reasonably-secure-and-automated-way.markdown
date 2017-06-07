---
layout: post
title: "Part 1: How Mozilla publishes APKs onto Google Play Store, in a reasonably secure and automated way"
author: Johan Lorenzo
date:   2017-06-06 13:25:59 +0000
tags: releng
---
The Release Engineering team fully-automated the publication of Firefox for Android in version 53.0. Let's see what was already there and how things have changed since version 53.0.


> This blog post is a part of a series. Checkout the other posts:

> 1. How did the project start?
1. Presentation of the solution [Coming soon]
1. 5 things I would have loved knowing about Google Play [Coming soon]
1. What's next? Want to contribute? & Special thanks [Coming soon]

# How did the project start?

## Mozilla ships Firefox every day

This is true for desktop (Windows, Linux, Mac) and Android. However, we don't ship that often to every user. We have different channels, receiving updates at different frequencies:

* Firefox Nightly (and Aurora until we stopped it) gets updated usually every day (unless an important breakage happens).
* Firefox Beta and Developer Edition get two updates every week on Desktop. On Android, Beta is usually shipped once a week.
* Firefox Release (also known as simply "Firefox") gets one every six weeks.

### About Firefox Aurora

You may have heard, [Firefox Aurora has been discontinued](http://release.mozilla.org/firefox/release/2017/04/17/Dawn-Project-FAQ.html) in April 2017. Although, these blog posts will talk about it. The main reason is: Most of the experiments were done on Aurora, before it was stopped.

Today, the Android users who were on Aurora have been migrated to Nightly. New users are also given Nightly.

## Why do we need Firefox for Android on app stores?

Unlike Firefox for desktop, Android apps have to be uploaded onto application stores (like Google Play Store). Otherwise, they have very low visibility. For instance, Firefox for Android Aurora (codenamed “Fennec Aurora”) was not on Google Play [until September 2016](https://bugzilla.mozilla.org/show_bug.cgi?id=1241114#c31), but it was downloadable from [our official website](https://www.mozilla.org/en-US/firefox/android/aurora/all/) (now redirected to Nightly). After we started publishing Aurora on Google Play, we increased our number of users by 5x.

## Why are we automating the publication today?

Google didn't offer a way to script a publication on Play Store, [before July 2014](https://www.programmableweb.com/news/google-play-store-api-to-automate-app-distribution-and-production/2014/07/29). It had to be done manually, from their website. Around that time, a few people from Release Management implemented a first script. One person from the [Release Management team](https://wiki.mozilla.org/Release_Management) ran it every time Beta or Release was ready, from his/her own machine. With Aurora being out, we now have several APKs (one per processor architecture/Android API level, which translates to 2 at the moment: one for x86 processors, the other for ARM) to publish each day.

The daily frequency was new for Fennec. It led to 2 concerns:

1. A human has to repeat the same task every day.
1. Pushing every day from a workstation increases the surface of security threats.

That is why we decided to make APK publication a part of the automated release workflow.

# Presentation of the solution

See [the next post]({{ page.next }}) [Coming soon].
