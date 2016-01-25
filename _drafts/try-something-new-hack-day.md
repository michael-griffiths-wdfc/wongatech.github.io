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

![Sophie Taeuber-Arp's Cercles et barres](http://stuff.aftnn.org/wongatech-hack-day-jan-2016/sophie-taeuber-arp-cercles-et-barres.jpg)

It's composed of a set of non-overlapping circles of reducing size and a couple
of line elements, some at right angles, some arranged into fans.

And here's a screenshot of my program:

![My attempt at Cercles et barres](http://stuff.aftnn.org/wongatech-hack-day-jan-2016/hack-day-2016-cercles-et-barres.png)

The [code is on
GitHub](https://github.com/afternoon/geometric-abstraction-with-three.js) and
you can [see it running
here](http://afternoon.github.io/geometric-abstraction-with-three.js/).

Nabil Boag - Help Centre Search
-------------------------------

For my hack day I wanted to implement a search box that would allow users to
type human readable questions and get back a list of help pages that were
relevant to the question.

Our help centre content is currently stored in a MySQL database in text tables.
I did some research and found out that MySQL has a handy natural language
search that was perfect for what I needed.

I built a JSON API that takes a search term and returns up to 5 relevant results:
```json
[
  {
    "title":"How do I add a new card?",
    "href":"\/help\/how-do-i-add-new-card"
  },
  {
    "title":"Do I need a bank account?",
    "href":"\/help\/do-i-need-a-bank-account""
  },
  {
    "title":"Can I pay with someone else's card?",
    "href":"\/help\/can-i-pay-someone-else\'s-card"
  },
  {
    "title":"What do I do if I\'ve been a victim of fraud?",
    "href":"\/help\/what-do-i-do-if-i\'ve-been-victim-fraud"
  },
  {
    "title":"What alternative payment methods do you accept?",
    "href":"\/help\/what-alternative-payment-methods-do-you-accept"
  }
]
```

I then built an Angular text input widget that returns results as the user types.
The input takes advantage of Angular input's debounce feature that only updates
the model value when the user has stopped typing for a certain length of time.
This stops us from querying the API too much.

![The search bar after entering a question](http://i.imgur.com/DZCDtIj.png)

Douglas Eggleton and Pete Watts - Foodie Slack Bot
--------------------------------------------------

Kim is a Slack bot and a real foodie, she knows all the best places in town!
She is integrated with both Foursquare and OpenWeather to provide
recommendations for local lunch and coffee houses.

Cold outside? She doesn’t want you to get cold and so will suggest
places which are closer. If the weather is nice, she will suggest places
further away but still within walking distance.

![Kim](http://i.imgur.com/5isHsN1.png)

Fergus Weldon - MRI Deep Learning and Capturing Medical Results
---------------------------------------------------------------

My hack day was split in two:

The morning was getting my feet wet with Google's Deep Learning library,
Tensorflow. I worked through the advanced convolutional model example and
then started to apply this technique to the
[Annual data science bowl challenge](https://www.kaggle.com/c/second-annual-data-science-bowl).
The task is to develop an algorithm that will aid in the diagnosis of heart
disease from patient MRI images. Below is an MRI image of a patients
abdomen with the areas of interest highlighted.

![Highlighted MRI image](http://i.imgur.com/D46M0ps.png)

My afternoon was used to start to develop a front end application that would
record blood culture results over time for a patient. The backend is to be
powered by Firebase which is a NoSQL backend.

Chris Conolly - Mood Monitor
----------------------------

For my hackday project I choose to create a mood monitor. This was inspired by
the smiley face buttons you get at check in at the airport. I was enthralled
in the airport lounge watching this data come in in real time and thought
it would be fun to make it at work.

The app was quite a simple one so I choose as many new technologies as I
could possibly squeeze into one project. I decided to create the project
using the Flux architecture, which Facebook created to rebuild their
notification system, and used Webpack, React, ES7 and Firebase as the backend.

The new tech stack took a little getting used to but I got it doing exactly
what I wanted to using the patterns I wanted. Next up I'd like to add graphs
using the real time data then its ready to let loose on the office!

![Mood Monitor](http://i.imgur.com/8F6R78v.png)

Bryan Watt - Raspberry Pi Camera Integration
--------------------------------------------

While perhaps this is nothing "new", my hack day goal was to tackle a
Raspberry Pi and Camera Module, and experiment with a few of the options.
This actually serves a longer term goal to build a app controlled self-driving
[Big Trak](https://s-media-cache-ak0.pinimg.com/736x/dd/69/77/dd6977b28ca1850b929c8d8e0aecf899.jpg),
and one of the key pieces being the camera. This was a brief introduction into
development on a Raspberry Pi.

![Raspberry Pi Camera](http://i.imgur.com/TsyGbqq.jpg)

After setting up the Pi basics, connecting the Camera unit, I pulled down the
PiCamera python module. Initial integration was actually way easier than I
thought it would be, and I tested it with a few of the supported Bash
commands for taking video (.h264) and still images.

![Script Running](http://i.imgur.com/ERZYG96.jpg)

I then started with several python scripts, to experiment with the camera
settings, and in the end had a script that configured the camera, took a
preview to evaluate attributes like White Balance and Exposure, and then
keep the settings static for a series of timelapse photos. I then set the
camera up on a windowsill and took a series of photos over an hour of sunset
at 5 second intervals, and created a video from that by FTPing the photos over
while they were being shot to my mac.

![Timelapse Running](http://i.imgur.com/m6fkeun.jpg)

My hack showed me the ease of the Pi Camera integration, and was an
introduction to Python which I've never really used before. There should
be no problem with the camera in the coming BigTrak project.

Hans Doyekee - A voice-responsive server-metrics dashboard
----------------------------------------------------------

Tech used: Javascript, Twitter Bootstrap, CSS, Asp.net MVC, REST

“Talking Thomas” is a simple infra dashboard which I built during the hack
day in an attempt to use voice and sound interactively. The application was
built on top of an Asp.net MVC model with a Twitter Bootstrap theme as the UI.
The data source that I decided to use came from New Relic. So, using
New Relic's REST APIs, I pulled the list of servers in our Production
environment and populated the “servers” section of the application.
Once a server is selected, its up-to-date CPU usage stats are fetched from
New Relic again for the last 30 mins. The data is requested and retrieved
from the “controller” section of the application, i.e. the back end. This
returns a model containing the server data as JSON to the UI, which is then
transformed and plotted on screen in a line chart, using “D3 JS”.  A
JavaScript text to speech library then parses the data and narrates a summary
of the on-screen data back to the user in the following manner:
“The average total CPU usage for server XXX was Y% with a peak usage of
Z% at DD-MM-YYYY HH:mm”. To interact with the UI, the user has the choice
between the conventional point & click approach or use Chrome voice commands.
The latter relies on the SmartVoice API and works on all browsers that allow
access to microphones. The API recognises pre-defined commands which in turn
fires JavaScript events. In this instance, I defined the common user
scenarios of such a dashboard, such as navigating between servers, showing
the servers list, making the browser go fullscreen as well as navigation
controls (back, forward, refresh).


