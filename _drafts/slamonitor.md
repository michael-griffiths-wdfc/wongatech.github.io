---
layout: post
title: SLAMonitor
author: aidanmcginley
---

This is a post about our motivations for creating [SLAMonitor](https://github.com/wongatech/SLAMonitor) and some of the design decisions behind it.
Whilst developing some of our new services, [Suremaker]() and I wanted a mechanism to track how calls to external services were performing, we had a few simple requirements

* must be lightweight
* support for both synchronous and asynchronous request response patterns
* locally testable
* support some form of alerting
* ability to configure triggers based on different constraints

## Lightweight

There are a plethora of tools that can provide this capability by either 

+ Installing agents on your machine (Think NewRelic)
+ Monitoring the service externally (like [Uptrends](https://www.uptrends.com/) . 

The problem with 1 is it introduces a reasonably complex dependency on your service i.e.  you must have the appropriate software installed and configured on the environment you are running in.
And monitoring using approach 2 results in a loss of control as the setup and configuration of the monitoring is done outside the scope of your service. Not to mention the difficulties with configuring your network to allow a third party to monitor your internal services.

## Testable

The aim here was to ensure we could test the alert trigger from the unit/integration tests of our service. So by locally I mean within the context of the service itself and its deployment pipeline and not rely on any externally deployed components.

## Alerting

We went with the assumption that alerting can be triggered from entries in the application log. We figured this was the most flexible approach as it should be possible in any production environment to raise alerts from errors in your log files. If not that's something you should definitely invest in!
This also fly's a little in the face of our lightweight approach of not relying on externally deployed components, however it's balanced by the fact that implementing a robust alerting system would pollute this single responsibility library with a huge amount of functionality which is far better served elsewhere.

## constraints

I.e email generation 2 minutes, document generation 2 seconds same service
