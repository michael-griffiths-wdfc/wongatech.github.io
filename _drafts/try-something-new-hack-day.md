Try Something New Hack Day
==========================

Ben Godfrey: Geometric Abstraction With Three.js
------------------------------------------------

My hack was definitely the most leftfield of the day. I'm a big fan of
the generative art of people like [John
Maeda](http://www.maedastudio.com/2001/maedamedia/index.php?category=all&next=exists&prev=exists&this=maedamedia),
[Casey Raes](http://reas.com/) and [Ben Fry](http://benfry.com/). I've built
simple graphics systems before and wanted to rekindle that for the hack day.
When I logged in to the wifi, the [Google Doodle of the day was for the 127th
birthday of Swiss artist Sophie
Taeuber-Arp](http://www.google.com/doodles/sophie-taeuber-arps-127th-birthday),
one of the most important artists of geometric abstraction, so I decided to try
rebuilding some of her pieces in code.

I wanted to try something new, so I decided to use WebGL in the browser to
generate simple geometric objects. I quickly switched to
[three.js](http://threejs.org/), which has a simpler programming model. After a
few test hacks I created a `Scene` abstraction to wrap three.js and then set
about writing a scene implementation to randomly generate pieces similar to
Cercles et barres, from 1934.

Here's the original:

!(https://s3-us-west-2.amazonaws.com/stuff.aftnn.org/wongatech-hack-day-jan-2016/hack-day-2016-cercles-et-barres.png)

It's composed of an set of non-overlapping circles of
reducing size and a couple of line elements, some at right angles, some arranged
into fans.

And here's a screenshot of my program:

!(https://s3-us-west-2.amazonaws.com/stuff.aftnn.org/wongatech-hack-day-jan-2016/sophie-taeuber-arp-cercles-et-barres.jpg)
