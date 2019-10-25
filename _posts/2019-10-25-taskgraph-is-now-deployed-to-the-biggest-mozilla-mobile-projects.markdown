---
layout: post
title: "Taskgraph is now deployed to the biggest mozilla-mobile projects"
author: Johan Lorenzo
date:   2019-10-24 12:24:01 +0000
tags: releng
---

# The 3-minute-video

<iframe src="https://drive.google.com/file/d/1WnCR6blDtTGr_nAslXi53SQPWJ17OfLq/preview" width="640" height="360"></iframe>

See the [video there](https://drive.google.com/file/d/1WnCR6blDtTGr_nAslXi53SQPWJ17OfLq/view), if doesn't display up here.

# Context

## Why Taskcluster?

It’s a fairly common practice to build and test every time someone makes a change in the code. In the industry, we call this process “Continuous Integration” (CI). Another good practice is to automate the deployment of your builds to end-users. This process is called “Continuous Deployment” (CD). There are several CI/CD products on the market. Mozilla has used some of them for years and still uses them in some projects. About 6 years ago, Mozilla needed a CI/CD product that did more than what was available and started Taskcluster. If you’re interested in knowing more about why we made Taskcluster, please let me know in the comments and I’ll write a dedicated post.

Since then, we’ve entirely migrated many Android projects like Firefox[1], Firefox Focus[2], Firefox for Amazon's Fire TV, Android-Components, and the upcoming Firefox Preview to Taskcluster. The rationale for migrating off another CI/CD was usually the following: we want to ensure what we’re shipping to end-users comes from Mozilla, while still providing the development team an easy and consistent way to configure their tasks.


## Taskcluster and task definitions

A task is an arbitrary piece of code that is executed by Taskcluster. We tell Taskcluster to execute a task by submitting some configuration, that we call the “task definition”. A task definition contains data about version control repository it deals with or what commands to run. As your project grows, it will likely need more than a single task to perform all actions. In this case, you need a graph of task definitions. You can submit a graph in different ways:
either by manually entering each task on https://tools.taskcluster.net/task-creator/,
by defining them all in a .taskcluster.yml file in your repository https://docs.taskcluster.net/docs/reference/integrations/taskcluster-github/docs/taskcluster-yml-v1
by defining a single task in .taskcluster.yml and letting this task generate the rest of the graph. We call this single task a “decision task”. It can be implemented:
either by using one of the Taskcluster libraries (the Python one for instance https://pypi.org/project/taskcluster/),
or by using a higher-level framework: taskgraph.

The above graph submission options are ordered by complexity. You may not want to start with option 3 immediately. Although, if your CI/CD pipeline deals with multiple platforms, multiple types of tests, and/or multiple types of pipelines, you may want solution 3b. More on this below.

![Solution 1]({{ site.baseurl }}{% link /images/posts/taskgraph-android/taskcluster-simple.svg %})
Solution 1

![Solution 2]({{ site.baseurl }}{% link /images/posts/taskgraph-android/taskcluster-yml.svg %})
Solution 2

![Solution 3]({{ site.baseurl }}{% link /images/posts/taskgraph-android/taskcluster-decision-task.svg %})
Solution 3

![Solution 3a]({{ site.baseurl }}{% link /images/posts/taskgraph-android/taskcluster-decision-task-custom.svg %})
Solution 3a: What’s inside the decision task

![Solution 3b]({{ site.baseurl }}{% link /images/posts/taskgraph-android/taskcluster-decision-task-taskgraph.svg %})
Solution 3b: Inside the decision task


Taskgraph was originally built for the most complex project, Firefox itself, and it was strongly tied to it. So, when a simpler project - Firefox Focus - came up a year and a half ago, we - the Development Team and Release Engineering - agreed on going with solution 2 and later on then 3a. Firefox Focus went great and became the base of a lot of Android projects. They have grown quite big since Firefox Focus started. The way CI/CD is configured there has grown on top of a code which wasn't meant to be that big, and which was duplicated across projects.


## Taskcluster and people

Moreover, people started to come from our main historical repository https://hg.mozilla.org/mozilla-central (where Firefox for Android is) to our other Android projects. With each project, they had to figure out how things work and Release Engineering has had to be involved in many changes, so we were losing the “provide the development team an easy way to configure their tasks” feature.


## Here enters taskgraph

We know Android projects are becoming more and more complex, in terms of CI/CD. We know taskgraph is able to support thousands of jobs by splitting the complexity in small and dedicated files. We know Firefox developers have more or less knowledge about taskgraph to be able to add/modify their job without compromising the rest of the graph. So we decided to make taskgraph a more generic framework so Android projects can reuse it.


# Taskgraph, the tour


## In a few words

Like its name infers, taskgraph generates graphs of tasks for all types of events. Events can be “someone just pushed a new commit on the repository” or “someone just triggered a release”. Taskgraph knows what type of event happened and submits the corresponding graph to Taskcluster.


## Main features

1. It’s fast! Tens of thousands of tasks are generated in under a minute
2. It deals with default values. Oftentimes you have to provide the same data for each task, taskgraph knows about the right defaults, so you can focus on what’s really important to your task
3. It validates data before submitting anything to Taskcluster.
4. It’s deterministic. You’ll get the exact same result by reusing a set of parameters from a run on another machine.
5. It’s both configuration-oriented and programming-oriented. Want to avoid repeating the same configuration lines over and over? You can write a python function in a separate file

On Android projects, when we first implemented our simple solution based off of taskcluster libraries, we did have feature #1 (easy, when you deal with a handful of tasks), but we started to wait minutes to get a hundred tasks submitted. At some point, we needed feature #2, so we had to implement our own default values. We’ve never had feature #3 and #4. Depending on the Android project, feature #5 was more or less implemented.


## Other features I like

1. Docker images! We can create our build environments directly on Taskcluster as docker images and have our build tasks using them. We don’t need to publish the images somewhere, like on Docker hub.
2. Cached tasks. Sometimes you just want to rebuild something when a subset of the code changes (for instance a Dockerfile). Taskgraph knows where to find these tasks and reuse them in the graph it submits
3. Graphs that depend on other graphs. Cached tasks are for single tasks, but you can reuse entire graphs. This is useful in Firefox. We generate all the shippable builds whenever a push to the repository happens. Then, if we think they’re good enough, we promote them to be actually shipped.

To me, re-using taskgraph instead of reimplementing a new framework is today a huge win just for the sake of these features, even for small projects. We didn’t do it a year and a half ago, because taskgraph wasn’t a self-served module. Tom Prince put a lot of effort to make taskgraph generic enough to serve other projects than Firefox.


# Introduction to taskgraph


## Data flow, part 1

![First part of the dataflow]({{ site.baseurl }}{% link /images/posts/taskgraph-android/taskgraph-flow.svg %})
Hacking taskgraph is usually dealing with that part of the data flow


### Kind.yml

In order to help taskgraph to tell if a task should end up in a graph, tasks are grouped in “kinds”. Kinds group tasks that are similar.
For instance: You’d like to have 2 compilation tasks, one for a debug build and the other for an optimized one. Chances are both builds are generated with the same command, modulo a flag. In taskgraph, you will define both tasks under the same kind.
Each kind is stored in its own kind.yml file. It’s the configuration-oriented part of taskgraph. You usually put raw data in them. There are some bits of simple logic that you can put in them.
For example: `job-defaults` provides some default values for all jobs defined in the kind, it can be useful if the build tasks are generated with the same command.


### Loader

The loader is in charge of taking every task in the kind, applying `job-defaults` and finding what are the right upstream dependencies. The dependency links are what make the graphs of tasks.
For instance: In addition to having common values, both of your build tasks may depend on Docker images, where your full build environment is stored. The loader will output the 2 build tasks and set the docker image as their dependencies,


### Transforms

Here comes the programming oriented part of taskgraph. The configuration defined in kind.yml is usually defined to be as simple as possible. Transforms take them and translate them into actual task definitions. Transforms can be shared among kinds, which gives a way to factorize common logic. Some transforms validate their input, ensuring the dataflow is sane.
For example: For the sake of cost, Taskcluster enforces tasks to be deleted after a certain amount of time. The last transform ensures this value is valid and if it’s not defined (which usually happens) sets it to a year from now.


## Data flow, part 2

![Second part of the dataflow]({{ site.baseurl }}{% link /images/posts/taskgraph-android/taskgraph-flow-2.svg %})
You may not need it, but that’s how optimization happens!


### Target tasks

Taskgraph always generates the full graph, then it filters out what’s not needed. That’s what the target task phase does.
For instance: If you want to ship a nightly build, target_tasks will only select the right tasks to build, sign, and publish this build.


### Optimized tasks

Some tasks may have been already scheduled. If so, taskgraph takes them out of the target tasks.
For example: If the nightly build task was made at the same time as the debug one, taskgraph can reuse the task and just submit the signing and publishing tasks.


### Taskcluster client

At this point, taskgraph knows exactly what subgraph to submit. It delegates the submission to taskcluster via one of its client libraries. Tasks are then generated.


# What’s next?

I really enjoyed “taskgraph-ifying” the most important mobile projects. We have leveraged a codebase that can scale while being able to handle edge cases. We’ve been able to factorize some of the common logic, which wasn’t possible with our initial solution. There’s still some improvement we can, and Mitch, Tom and I are working on improving the quality of life.
Moreover, having taskgraph enables a better release workflow for Firefox Preview: we may have finer grained permission on who can ship a release. It requires more work and more blog posts, but taskgraph is a necessary stepping stone.


# Further reading

* The official documentation for Firefox: https://firefox-source-docs.mozilla.org/taskcluster/taskcluster/index.html
* How tos depending on what you want to do: https://firefox-source-docs.mozilla.org/taskcluster/taskcluster/how-tos.html
* “Task Configuration at Scale”, a video that explains the scaling problems taskgraph solves: https://ahal.ca/casts/2019/task-configuration-at-scale/
* The Gecko Hacker's Guide to Taskcluster: https://staktrace.com/spout/entry.php?id=870


# Need any help?

If you’re stuck on something or Taskcluster isn’t behaving as it should, feel free to reach out to Release Engineering in #releaseduty-mobile (Slack).


# Special thanks

* Tom Prince for extracting taskgraph out of the Firefox codebase, and converting the very first Android project: Reference-Browser
* Mitch Hentges for converting Firefox-TV and Application-Services, in addition to reviewing my patches
* Mihai Tabara for swiftly reviewing my other patches
* Jordan Lund for coordinating this project, including its communication
* Sebastian Kaspari for helping me to get the patches landed and for reporting any regression he noticed
* Richard Pappalardo and Kate Glazko for making sure the UI tests and the nimbledroid submission are working fine
* Paul Adenot for helping understand and fix the audio in the 3-minute-video.
* The reviewers of this article and the video.
* And anyone I missed above


# Comments

You can read and leave comments on [this Github issue](https://github.com/JohanLorenzo/blog/issues/5).
