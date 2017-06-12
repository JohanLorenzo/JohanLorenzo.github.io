---
layout: post
title: "Part 3: How Mozilla publishes APKs onto Google Play Store, in a reasonably secure and automated way"
author: Johan Lorenzo
date:   2017-06-12 9:17:32 +0000
tags: releng
---
The Release Engineering team fully-automated the publication of Firefox for Android in version 53.0. Let's see what was already there and how things have changed since version 53.0.


> This blog post is a part of a serial. Checkout the other posts:
> 
> 1. [How did the project start?]({{ site.baseurl }}{% post_url 2017-06-06-part-1-how-mozilla-publishes-apks-onto-google-play-store-in-a-reasonably-secure-and-automated-way %})
1. [Presentation of the solution]({{ site.baseurl }}{% post_url 2017-06-07-part-2-how-mozilla-publishes-apks-onto-google-play-store-in-a-reasonably-secure-and-automated-way %})
1. 5 things I would have loved knowing about Google Play [Here]
1. What's next? Want to contribute? & Special thanks [Coming soon]

# 5 things I would have loved knowing about Google Play
 
This part is more oriented to personal takeaways and a couple of questions that remain unanswered.
 
## It is easy to publish an APK that is not localized
 
A few checks done in pushapk_scriptworker are actually because of previous errors. The number of locales is one of them. A few weeks after Fennec Aurora was shipped to Google Play, some users started to see their browser in English, even though their phone is set in a different locale and Aurora *used to be* in their own locales.
 
Like said previously, APKs were originally uploaded via a script. This was true also for the first Aurora APKs. Furthermore, automation on Taskcluster was first tried on Aurora. When I started to roll these bits out, there was a configuration issue. Pushapk_scriptworker picked the en-US-only-APK, instead of the multi-locale one. The fix was fairly simple: just change the APK locations.
 
Google Play has a lot of ways to detect wrong APKs: by signatures, by package name (none of the Firefox versions share the same one), by version code, and some others. Although, it doesn't warn about a big change in:
 
* Size. There is approximately a 10-MB-difference between a single locale build and a multi-locale one. Multi-locale APK are usually less than 40 MB. That APK shrunk by 25%, but not for good reasons.
* The directory list of the archive. Manifest files were smaller, there were 90 times less files in some directories.
 
Of course, from one stable version to another, a lot of files may change in the archive. Asking Google to watch out for everything doesn't seem reasonable. Although, when I started working on Google Play, it left me the feeling of being well-hardened. At that time, I thought Google Play did check the locales within an APK.
 
The consequence I take away: If your app has different build flavors (like single vs multi-locales), I recommend you write your own sanity checks, first.
 
## Locales on the Play Store are independent from locales in APK
 
It might sound obvious after explaining of the previous issue, but this error message confused several Mozillians:
 
```
Tried to set recent changes text for APK version 12345678 for language es-US. Language is not associated with the app.
```
 
We hit it with a couple of locales, at pace of 1 per month, approximately. Like explained in the architecture part, locales are defined in an external service, stores_l10n. We have had many theories about it:
 
* This locale is not supported by Google Play
* The locale code (for instance "es-US") expected by Google Play is not the one we provide. We may want to find the list of the locales officially supported
* We don't ship this locale within the APK, and the Play Store detects it
 
Actually, the fix ended up being simple. "Recent changes" is something we want to update on every new APK. But because the descriptions were more set in stone, they were not a part of the automated workflow. Actually, stores_l10n released had recently released new locales each time we hit the problem. That error message was actually telling us the descriptions of these new locales had never been uploaded. Once we figured this out, it became a part of the regular update workflow.
 
## You cannot catch everything when you don't commit transactions
 
The dry-run feature in MozApkPublisher, which just doesn't commit the Google Play transaction, helps in detecting early failures. For instance: wrong version codes, wrong package names. Nevertheless, we have hit cases where dry runs went smoothly and we had to diagnose new issues at commit time.
 
* Locales mismatch. The previous issue was not dectected at the upload time, likely for the reason that listings and the recent changes are  [2 different](https://developers.google.com/android-publisher/api-ref/edits/listings) [API calls](https://developers.google.com/android-publisher/api-ref/edits/apklistings). I assume, because you can call these API in whatever order within the same transaction, Google doesn't check until the final state is fully known.
* Permissions not granted to the account. Due to Firefox Aurora being stopped, we restricted the Google Play account in charge of Aurora to not be able to upload any APK anymore. Yet, we decided to publish Nightly to the same product as Aurora (in order to not strand users on a unmaintained version). Our tests went fine, Google Play accepted our APKs. But the day of the go-live, a new error came up saying we are not able to upload APKs after all. I don't have any theory for this scenario, but if you do, I would love to hear it!
 
## User fractions can also be specified on other tracks (but that's not a feature)
 
Fennec Release 53.0 is the first version which was entirely published via Taskcluster. Mozilla also uses the rollout track on Release only. Beta is pushed to the production one (and that is actually something we are reviewing). Sadly, there was another configuration error: even though, the user fraction was specified, the configured track was the production one. Google Play didn't raise any error (even at commit time), starting a full-throttled release. At that time, I contacted the Google Play support, to ask if it was possible to switch back to rollout. The person was very courteous. He explained they were not able to perform that kind of action. This is why they transmitted my request to the tech team, who will follow up by email.
 
In parallel, we have fixed the configuration error and implemented our own check in MozApkPublisher.
 
 
## There is no way to rollback to previous APKs, even if you ask a human
 
The previous configuration error could have remained brieve, if somebody didn't report what seemed like an important crash, 1 hour later. At that point, the Release Management team wanted to stall updates, in order to not spread the regression too much. We were still waiting on the support's answer, but I reached out to them again since our request became different. I told the new contact I had about the previous issue, the new one, and the fact we were running against the clock. Sadly for us, the person only gave us the option to wait on the email follow up.
 
About 16 hours later, we got the email with the official answer:
 
> Unfortunately, we cannot remove the APK in question, neither can we claw back the APK from the users that have already installed this update.
> 
> If you want to stop further users from installing this APK, then you need to make another release that deactivates this APK and add the APK that you want users to install instead.
 
With that in mind, we have been thinking of another way to use Google Play, where the [rollout track would be more extensively used](https://bugzilla.mozilla.org/show_bug.cgi?id=1366820).

# What's next? Want to contribute? & Special thanks

See the next post [Coming soon].

# Comments

You can read and leave comments on [this Github issue](https://github.com/JohanLorenzo/blog/issues/3).
