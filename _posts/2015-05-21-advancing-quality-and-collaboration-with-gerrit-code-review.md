---
layout: post
title: Advancing Quality And Collaboration With Gerrit Code Review
author: paulbourke
---
[As we mentioned on Twitter](https://twitter.com/wongatech/status/520588561079599104), we've recently started using [Gerrit](https://en.wikipedia.org/wiki/Gerrit_(software)) here at Wonga for source control, code review, and as a means to stop code which doesn't pass tests getting into our repositories, a pattern known as *gated commits*.  We've considered many other SCM options including GitHub, Gitlab and Stash, but none provided the same control and pre-commit review workflow that Gerrit allows for.

Gerrit provides a "change request" mechanism similar to GitHub's, but with some key differences that help maintain strong quality standards in an enterprise environment. These have been outlined in other presentations (see [Luca Milanesio](http://www.slideshare.net/lucamilanesio/gerrit-codereviewgit-hubplugin) or [Sean Dague's](https://dague.net/2013/05/23/github-vs-gerrit/) posts on the subject).  Some major open source projects such as [OpenStack](https://review.openstack.org/#/) and the [AOSP project](https://android-review.googlesource.com) are using Gerrit as their primary code review tool purportedly for this reason, with GitHub serving as a backup mirror.

Here are a few things that can be done with this fantastic tool, and how it can be used to provide a robust and open source Git solution.

## Audit trail

ACLs (access control lists) in Gerrit are extremely flexible. Projects can be configured to enforce peer review for every commit targeted at a repo, which means key stakeholders can have a bird's eye view of what is going into their code base at all times. Customisable labels can be added to mark a commit as having been through various workflow stages, such as 'reviewed face to face', 'verified by QA', etc.

## Let Jenkins be your reviewer!

The [Gerrit Trigger plugin](https://wiki.jenkins-ci.org/display/JENKINS/Gerrit+Trigger) for Jenkins makes it very quick to get Gerrit and Jenkins talking to each other. Jenkins monitors events from Gerrit and specific jobs can be run at each point in the review lifecycle. Here at Wonga we're using this functionality to have CI run for every review, automatically rejecting code that has not passed tests, violates style guides, etc. It's a really useful way of catching problems early and saving time reviewing.

## Contributor/maintainer model

One of the main things that makes Github great is the social aspect, where members are encouraged to submit a fix or pull request rather than simply filing a bug. This is the kind of culture we like to encourage at Wonga, so we're glad to report Gerrit works to foster this very effectively. The review board is open to all teams, so it wasn't long after we had it set up that people began to chip in on other team's reviews, and submit code that could be reviewed by project owners, which is subsequently verified and merged by Jenkins.

Of course no tool is perfect, so far some of the weaknesses we've found with Gerrit includes the "functional not pretty" UI, and the complexity of certain ACL configurations.  [Multi-master support](https://code.google.com/p/gerrit/wiki/MultiMaster) is also in it's infancy, which limits you to the [replication plugin](https://gerrit.googlesource.com/plugins/replication/) for high availability.

There's much more that can be written about Gerrit including advanced workflows, extending functionality via Java or Groovy plugins, etc. I'll leave that for another time. Until then you can check out the [project on Google Code](https://code.google.com/p/gerrit/), or get involved in the [mailing list](https://groups.google.com/forum/#!forum/repo-discuss).
