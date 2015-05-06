---
layout: post
title: Remaining Agile Under Regulation
author: ronshteinberg
---
**How we built a release management system that satisfies regulatory needs while
keeping the agility of our teams**

Wonga's tech organisation has always implemented agile development methodologies. Most of the teams are scrum teams releasing a new version at the end of each sprint cycle (usually every 2 weeks).

With the recent introduction of our new regulator, the FCA, we asked ourselves the question: how can we remain agile while satisfying regulatory requirements and meeting customer expectations?

We wanted to identify and implement an approach that would provide for centralised governance and compliance assurance, while allowing individual product teams to use the release processes, cadence and tools they prefer.

## Our approach

We were glad to learn the regulatory requirements focus on customer outcomes rather than prescribing operational methodologies, and leave room for organisations to choose the most appropriate processes. Essentially we are required to ensure software releases meet quality criteria and are signed off by business owners. These requirements did not differ from our existing practices, but we saw an opportunity to better formalise and evidence adherence to these criteria, while allowing product development teams to maintain their release cadence and methodologies.

We set out to establish a software system to document the changes introduced with each release, ensure release checklists and procedures were completed, and provide an evidence base for quality and auditing purposes. Naturally, we were keen to minimise disruption to productive work so long as quality criteria were met.&nbsp;

## Our solution: ReMi

[![ReMi](/images/2014-10-22-remaining-agile-under-regulation/remi.png)](/images/2014-10-22-remaining-agile-under-regulation/remi.png)

[Click for larger
image](/images/2014-10-22-remaining-agile-under-regulation/remi.png)

We built ReMi, a simple web application that provides teams with a release scheduling, reporting and documentation system and ultimately a central repository for evidence of release policy adherence.

A product owner can schedule a release through ReMi by requesting a “release window” on a given date and time (our international product portfolio is supported in ReMi, so product owners can also see when other releases may take place). We integrate with JIRA using its API to extract the changes (typically user stories) included in the release.&nbsp; We also incorporate a list of stakeholders in the business that need to be notified about the release and potentially approve it. Finally, each requested release may also have a checklist of system and other tasks to be performed as part of the release.

With ReMi we create “release windows” for the various products. For each window, we capture a list of included features (generally user stories), approvals from stakeholders and a checklist of any manual steps to be performed during the release.

ReMi supports a wide variety of release processes, and accommodates almost any process the teams choose so long as criteria, set by our release management team and in line with a central release policy (such as passing tests), are met.

For example, several teams perform a coordinated release once a sprint. For these teams we orchestrate a manual process. We conduct an approval call before each release, review the features and changes included in the release and the detailed release and test plan. The approval of the release, along with the features and the release plan are stored in ReMi for reporting later.

On the other side of the spectrum, we have service teams that perform a completely automated process. Those teams use ReMi by making API calls to schedule release windows and push the feature lists.

Between these two extremes there are processes that use different degrees of automation. Teams can choose how much or how little of their release process they want to automate. ReMi makes sure they remain compliant either way.

ReMi still requires several checks to be confirmed by an individual, and those will be further automated through integration with our various software development lifecycle tools and testing systems. Already though, ReMi supports multiple efficient and traceable releases every week.

Our next goal is to further improve or process by formalising the use of BDD and common agile tools to ensure acceptance tests cover business requirements.
