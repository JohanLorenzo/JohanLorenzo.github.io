---
layout: post
title: "Part 2: How Mozilla publishes APKs onto Google Play Store, in a reasonably secure and automated way"
author: Johan Lorenzo
date:   2017-06-07 12:24:18 +0000
tags: releng
---
The Release Engineering team fully-automated the publication of Firefox for Android in version 53.0. Let's see what was already there and how things have changed since version 53.0.


> This blog post is a part of a serial. Checkout the other posts:
> 
> 1. [How did the project start?]({{ site.baseurl }}{% post_url 2017-06-06-part-1-how-mozilla-publishes-apks-onto-google-play-store-in-a-reasonably-secure-and-automated-way %})
1. Presentation of the solution [Here]
1. [5 things I would have loved knowing about Google Play]({{ site.baseurl }}{% post_url 2017-06-12-part-3-how-mozilla-publishes-apks-onto-google-play-store-in-a-reasonably-secure-and-automated-way %})
1. What's next? Want to contribute? & Special thanks [Coming soon]

# Presentation of the solution

## The needs

* Interact with the Play Store.
* Securely authenticate to it. It offers several ways to authenticate, including P12 certificates.
* Securely store the authentication credentials.
* Securely fetch the builds.

## The solution

### Based on Taskcluster

Mozilla, and more specifically the Release Engineering team, uses [Taskcluster](http://docs.taskcluster.net/) to implement the Firefox release workflow. The workflow can be summed up as:

1. Build Firefox with all supported locales (languages)
1. Sign these builds
1. Publish them everywhere (on https://archive.mozilla.org/, on https://www.mozilla.org/firefox/, via updates, etc.)

Each step is defined by its own set of tasks. These tasks are processed by [specialized workers](http://escapewindow.dreamwidth.org/tag/scriptworker) (represented by worker types). Those workers basically run a script against parameters given in the task definition.

Therefore, publishing to Google Play was a matter of creating a new Taskcluster task, which will be processed by a dedicated script and executed by its own worker type.

### With some extra-security features

The aforementioned script must be bootstrapped to be integrated to the rest of Taskcluster. There are several ways to bootstrap scripts for Taskcluster. One of them is to create a docker image which Taskcluster pulls and run.

However, because of the needs stated above, we decided to go with a security-focused framework: [scriptworker](https://github.com/mozilla-releng/scriptworker). Scriptworker was initially created to perform one of the most critical operation security-wise: sign builds. The framework has some great interesting features:

* It securely downloads artifacts. Files are forced to be downloaded over https. Checksums and signatures are checked.
* It validates that the task definitions were not changed between the time of creation and the time of execution. This prevents some tasks to be duplicated and edited to introduce extra-commands, which may be used to tamper a build, for instance.
* It abstracts some of the Taskcluster details, thus you just have write a script (language-agnostic) that does what your worker has to do.


### How pieces are wired together

#### 0. Overview

Here's a general view of how things are wired together:

![Architecture overview]({{ site.baseurl }}{% link /images/posts/push-apk/architecture.svg %})

* 1\. The “decision task” creates a task for pushapk_scriptworker
* 2/3\. Scriptworker polls for pending tasks and check their scopes. It downloads APKs via Chain of Trust. Scriptworker checks if the upstream tasks were altered.
* 4\. Scriptworker defers valid tasks to pushapkscript. The latter validates APKs signatures, makes sure every APK architecture is present.
* 5\. Pushapkscript calls MozApkPublisher with credentials and on-disk locations of APKs
* 6/7\. MozApkPublisher verifies whether APKs contain several locales. It fetches localized strings displayed on Google Play Store (aka “listings” and “what’s new section”)
* 8\. MozApkPublisher opens the Google play credentials.
* 9\. MozApkPublisher publishes APKs, listings and “what’s new”

#### 1. Task creation

There are many ways to submit the definition of a task to Taskcluster. For example, you can:

* Use the [web task creator](https://tools.taskcluster.net/task-creator/) (useful for debugging)
* Call the Taskcluster API via one of the libraries (interesting if you want a bot that spawns tasks)
* Generate the task via [taskcluster/taskgraph](https://dxr.mozilla.org/mozilla-central/source/taskcluster/taskgraph/) that lives "in-tree", that is to say, alonside the Firefox code.

Each of them was used at some point, but the ultimate solution relies on the last one. The taskgraph is a graph generator which, depending on given parameters, creates a graph of builds, tests and deployment tasks. Taskgraph generation is run on Taskcluster too, under what we commonly call "the decision task". This solution benefits from being on hg.mozilla.org: it is versioned and only vouched people are able to modify it.

Moreover, taskgraph generates what is necessary for scriptworker to validate the task definitions and artifacts. To do so, taskgraph:
* Creates a JSON representation of the graph,
* Creates another JSON file that describes the artifacts generated (including the JSON of the graph) and signs it.

#### 2. Scriptworker and new tasks

Scriptworker polls tasks from Taskcluster queue. That is actually one of the great things about Taskcluster: workers don't have to open inbound (listening) ports. This reduces the potential surface of attack. Fetching new tasks is done via [this bit of REST API](https://docs.taskcluster.net/reference/platform/taskcluster-queue/references/api#claimWork) which workers can poll. Speaking of which, workers are authenticated to Taskcluster, which prevents them from claiming a task that it isn't meant to take.

Secure download of artifacts is done by the "Chain of Trust" feature of scriptworker. [Once set up](http://scriptworker.readthedocs.io/en/latest/chain_of_trust.html), if you define `upstreamArtifacts` within the [task definition](https://tools.taskcluster.net/task-group-inspector/#/URGuaVTFTO2QAEXZmNsmgg/LXAaGWSYRM6VroGjIYpFKA?_k=adkey1), scriptworker will:

1. Make sure the current task and its dependencies have not changed since its definition. This is done by comparing the JSON the taskgraph generated and the actual definition.
1. Check the signatures of every dependency, by looking at a special artifact Chain of Trust creates. This helps to verify no rogue worker processed a upstream task.
1. Download artifacts on the worker and verify the checksums.

If all goes well, scriptworker will call [pushapkscript](https://github.com/mozilla-releng/pushapkscript)


#### 3. Pushapkscript and APKs

Here starts the Android-specific bits. Pushapkscript performs some extra checks on the APKs:

* APKs are signed with the correct certificates. In the previous steps, we have only checked the origin of the tasks. Now, we verify the APK itself. This may not sound extremely important because Google Play is vigilant about APK signatures and will refuse any APK for which the signature is not valid. However, it is safer to bail out before any outbound traffic is done to Google Play. Besides, with this check, Google acts as a second factor instead of being the only actor accountable for signatures.

* No required processor architecture is missing, in order to upload them all in the same request. We have to publish them at the same time because some Android devices support several architectures. We have already had [one big crash](https://bugzilla.mozilla.org/show_bug.cgi?id=1337290) on these devices because an x86 APK was overseeded by its "brother in ARM".

Pushapkscript knows about the location of the Google Play credentials (P12 certificates). It finally gives all the files (checked APKs and credentials) to MozApkPublisher.


#### 4. MozApkPublisher, locales and Google Play

To be honest, MozApkPublisher could have been implemented within pushapkscript, but the split exists for historical reasons and still has a meaning today: this was the script Release Management used before this project got started. It also remains a way to let a human publish, in case of emergency.

It checks that APKs are multi-locale. We serve the same APK, which includes (almost) every locale in it. That's a verification Google doesn't do.

It also fetches the latest strings to display on the Play Store (like the localized descriptions). These strings are then posted on Google Play, alongside the APKs.

MozApkPublisher provides a dry-run mode thanks to the transaction mechanism exposed by Google's API. Nothing is effectively published until the transaction is committed.

#### 5. Pushapk_scriptworker: Scriptworker, Pushapkscript, and MozApkPublisher on the same machine

The 3 pieces live on the same Amazon EC2 instance, under the name pushapk_scriptworker. The configuration of this instance is managed by Puppet. The entire Puppet configuration is [public on hg.mozilla.org](https://hg.mozilla.org/build/puppet/file/8238d014f1f1/modules/pushapk_scriptworker), with the exception of secrets (Tascluster credentials, P12 certificates) which are encrypted on a seperate machine. Like the main Firefox repository, only vouched people can submit changes to the Puppet configuration.

# 5 things I would have loved knowing about Google Play

See [the next post]({{ site.baseurl }}{{ page.next.url }}).

# Comments

You can read and leave comments on [this Github issue](https://github.com/JohanLorenzo/blog/issues/2).
