---
layout: post
title:  "Update dependencies with Renovate"
date:   2027-09-13 17:00:00 +0200
categories: dependencies renovate
---

All projects have dependencies.
To manage them is quite difficult.
It is also difficult to keep them up to date.

Today I want to write about [Renovate](https://github.com/renovatebot/renovate) which helps you with the latter.

The easiest way to use Renovate is via GitHub App.
The basic configuration keeps track of all discovered dependencies.
Maybe you want to group them.
Or handle major, minor and patch updates differently.
There are many examples on the internet to match your needs.

I would stronlgy suggest to have some sorts of tests, a CI build for each pull or merge request.
In that way you can "safely" opt in to automerge minor and patch updates (as many dependencies use semantic versioning and will not break APIs during that updates).

Renovate will work with all kinds of dependencies: NPM, Docker, Nuget, Bundler, GitHub Workflow Actions, ... .
Just to name a few.

This site uses also Renovate to keep it up to date.
Just have a look in my [renovate.json](https://github.com/mrdavidkovacs/davidkovacs.net/blob/main/renovate.json) and the [issue](https://github.com/mrdavidkovacs/davidkovacs.net/issues/6) which Renovate manages.

It is not that complicated in first place.
To adapt it to your needs it may get trickier.
But its up to you!

Use some tools which helps you to keep track of updates.
