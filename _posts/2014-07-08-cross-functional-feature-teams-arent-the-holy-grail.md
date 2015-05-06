---
layout: post
title: Cross Functional Feature Teams aren’t the Holy Grail!
author: carlpearse
---
Cross function feature teams are supposed to be the be-all-and-end-all of agile teams. Everything you read on agile project management promotes feature teams, and recommends that you steer well clear of service or component teams. There are some huge benefits that can be realised from cross functional features teams. I am not going to regurgitate them here, [here is a good article if you need a refresher](http://featureteamprimer.org/).

At Wonga we follow the same approach. Product owners, scrum masters, co-located across functional teams pulling from single backlogs for each product stream, vanilla scrum. At the beginning, when scale was much smaller, it worked well for us, but when Wonga started to grow &nbsp;things were not quite as simple. More teams started to come online and the longer standing high performing teams were split to spread the knowledge and experience. New people came in as a continuous stream which was a challenge in itself and which I will leave for another blog.

Soon there were double digit teams. Software and features were being churned out of a growing development machine, all following feature teams and vanilla scrum. We continued to deliver high quality products into production but the effort to achieve this grew. At one point we fixed several hundred defects on a product before go live.

Wonga’s software is architected to massively scale. A fundamental principle of SOA is services, business components and messaging. Adopting cross function teams that focus on features with less focus on shared services led us down a route of teams changing the work of other guys who were working on the same service. The teams were aligned to the business functions (vertically) so had less understanding of the full platform infrastructure across all the product range (horizontally).

What about communities of practice? On paper this makes sense. Teams focus on product delivery but share knowledge and best practice across a common code base through communities of practice. We did this but it became too difficult to also scale as the code base grew as we continued to experience the same reoccurring problems.

This simply wasn’t sustainable or extensible. It took several man-years of development to manage the defects before go live. At this rate our development teams and code base was projected to grow at a similar rate to our infinitely scalable SOA architecture.

What does lean teach you to do when you’re a start-up and things aren’t what you expect? Answer - you pivot. And that is exactly what we did. We came to realise, amongst other things, that pure cross functional feature teams are not the be-all-and-end-all after all. A lesson there for agile writers! They obviously aren’t working in an environment or architecture like Wonga.

Over a short period of time we came to understand that there is actually a hybrid. There is still a place for cross functional teams in Wonga, a team focusing purely on value add led by a Product Owner. These teams consist of generalising specialists, guys who can work on multiple areas of the code base to deliver a number of business objectives within a sprint. Again, vanilla agile.

More importantly there is a role for specialist around the core elements of what Wonga is delivering across all its regions and products.&nbsp; Distributing this ownership, knowledge and value across product teams came at a cost.&nbsp; Very shortly after realigning some of the core largely agnostic business functions around that function, and service team, defects reduced. Now teams who own the service lived and breathed the service. The sense of ownership flourished and the service teams grew an identity and sense of purpose.

![Division of product and service teams](/images/2014-07-08-cross-functional-feature-teams-arent-the-holy-grail/Blog-image1.png)

There is no single answer to team design and optimisation. Sometimes it makes sense to optimise around features, coordination and delivery, and we still do this in our vanilla agile product teams. But often where a key business function or specialisation is required, optimisation around other dimensions can be more valuable.

So we found that you can’t have it all ways! Wonga continues to mature and learn on the optimal way to work and I’m sure as we learn we’ll continue to pivot so that we continue to deliver high quality products in a predictable way.
