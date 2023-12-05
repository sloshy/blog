---
layout: post
title: Big, Big Changes
---
It's December already? Wow, how time flies, and wow, how few updates I've made about TinyDB over the past month.
There have been a lot of changes, mostly internal-facing, but I do have quite a few things going on behind-the-scenes as well that I'm working on.
Here's the most substantial changes you can expect as we go into this new year:

## 1. The App Formerly Known As TinyDB
Oh, well, this one is kind of embarassing.
When I came up with the name for this app, and when I *bought the domain name* for said app, I swear, I did my research, but somehow [a project with the name TinyDB already exists](https://tinydb.readthedocs.io/en/latest/).
I know, I know, it's easy to say "you should have googled it" and I could have sworn I did, but I did not.
I have absolutely no association with this project, so I will be retiring the name of "TinyDB" for my own similar project.

What is the new name?
Well...

That's a surprise.
I've already bought the domain name, and I'm trying to work on a visual style to suit it, but the domain name I bought will redirect to the new one for some time to be sure that anyone finding it through my ScalaCon presentation or any of my earlier tweets will wind up at the right place.

Oh, and speaking of...

## 2. The TinyDB website is temporarily down
I don't know if you've noticed but if you try to go to [http://tinydb.app/](http://tinydb.app/) right now, it just straight-up won't load.
I think it's DNS-related as I was trying to get something set up with web hosting a while back, but it seems to have totally borked the main site.

Fortunately, if you are one of the lucky few with private closed beta access, that link should still work for now.
So I've got that going I guess.

I will have a new homepage up once I am ready to show off the latest version of the app, with its new name.

Though, it wouldn't be particularly accurate to call this just an "app" anymore, because...

## 3. Going Modular
As I've been working on this app I keep coming up with so many new ideas and it's hard to stay focused on the core feature set.
In light of this I have been taking a few steps back and reevaluating what exactly I want this project to be, and I've come up with a resolution: this should be a platform, a framework, a collection of tools and libraries and solution-building utilities, rather than just "an app".

In practice, this means that there will still be an app, but the way features will, and can be, developed, will have to change.
I am currently working on porting a lot of the existing featureset into a kind of "plug-in" system.
I want it to be possible for you to write your own features for your own running instance of it.
That means you should be able to use different components, author your own UIs and forms, add entire buttons and custom code and who knows what else, without needing to rebuild the app from scratch.
The basic idea is that you should not only own your own data, but the way that you perceive it, the way you develop for it and use it, should also be totally portable.
You should be able to load community-developed features into the app on your own terms, modifying the app to work how you see fit.

I'd like to start with a Scala-based integration and work towards enabling a full JavaScript API that you can write for that exposes runtime hooks as well as common building blocks for writing your own integrations.

Another use-case would be to reuse components of the app in completely different contexts.
How cool would it be if, say, a browser add-on for scraping text from a webpage could output a fully-configured database file compatible with the app?
Or, what if you wanted to write an integration for importing things from a specific JSON-based HTTP API into database tables, making custom transformations and modifications along the way?
All of this should be as easy as possible without needing to go fully custom.

Be on the lookout for news about this in the future as it is a direction I am very excited about enabling.

## 4. Sneak Preview of Visual Updates
Not too much to show here, but I spent some time recently cleaning up the incredibly ugly UI for the app and have come up with this:

![New DB Look](/public/newlook.png)

Some of the changes:
* There is a "Table Toolbar" below the top bar that only appears when you've created at least one table. You can select a table or chart, and buttons relevant to that table appear on the bar.
* The top bar has been shrunk somewhat as a result and is a much more concise UI.
* The table buttons actually kind-of look tabular now.
* The sizing and shape of buttons has been adjusted
* Tables now appear in a little "floating" box

That last change is part of a bit of experimentation I'm doing into possible alternate table views. For example, viewing more than one table at once in their own sections.
This would be very useful for me so you know I'll eventually work on it, but for now, it just looks pretty nice to have some shadow in there.

Anyway, that's all for now, and stay tuned!
An update containing these visual changes and more will be pushed to the closed beta page soon for those of you with access.
Afterward, in 2024, we'll finally start to see some of the fruits of my labor and I hope you all will be very surprised and excited!
