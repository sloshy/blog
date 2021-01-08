---
layout: post
title: The Journey of Moving to Pure Functional Scala
---
Today, January 8th, I am leaving my position at [Rewards Network](https://www.rewardsnetwork.com/), a Chicago-based company that primarily develops in Scala, to work with [47 Degrees](https://www.47deg.com/), a functional programming consultancy.
During my time at Rewards Network for the past few years I've worked on several different styles of projects and been introduced to innumerable libraries and frameworks.
These tools all take advantage of the Scala language's many advanced features, some more than others, and in many cases can enable entirely separate models of programming.
One model that particularly fascinates me is that of effectful, pure-functional programming.
Since starting the Scala portion of my career at Rewards Network, I've spearheaded an effort within my team to promote this style of programming internally.
Today, as a result, every single new application that we develop is now written in this style, by group consensus, something that I wouldn't have dreamed was possible when I was just a new hire at this company.

I would like to write about my experiences being a part of this effort, what it meant for us, and what lessons I've learned from trying to switch out our previous stack for a pure-functional one, in Scala.

The purpose of this article is not to convince you that pure functional programming, specifically, is a good idea, or that you should be doing it (you should be, and I could argue this for days, but I won't belabor that point).
Instead, I would like to illustrate exactly how we were able to succeed in the efforts to convert my team's entire programming methodology.
In the event that you are interested in introducing some other paradigm, technique, or programming practice to the company you work for, I hope my experiences will help make your own endeavors a success, or at least give you a reference point to figure out where things can go wrong if you are not careful.

## ELI5: Pure Functional Programming in Scala
Before going forward, there's a chance you've stumbled upon this article without knowing much about either "pure functional programming" or the Scala language.
As mentioned just above, I am not trying to convert you to coding in this style, much less the specific language of choice, but to talk about how I went from convincing my team of an idea, and how we followed through.
For reference, here is a brief description of pure functional programming (henceforth "pure FP") and Scala both:

**Scala** is a programming language that borrows the object-oriented style from Java, and the functional style from languages like ML.
This means that, like Java, all values are "objects", or instances of a "class" that defines certain "methods".
It also means that functions, like in all functional programming languages, are first-class citizens of the language, able to be defined and passed around as arguments to other functions without ceremony.
Scala makes a number of innovations and design decisions on top of these to make both styles of programming more approachable, including features such as "case classes" that are a simplified way to make low-boilerplate, data-oriented classes with some standard functionality.
Another feature you might see referenced frequently is the notion of "implicit parameters", which the compiler is able to pass to functions or class constructors as-requested, in a manner similar to that of "dependency injection" but at compile-time.
Many advanced features of Scala use implicits, such as "extension methods" on objects by having an "implicit (wrapper) class" in scope that the compiler can transparently use without needing to wrap objects in this class yourself, or by allowing you to write "typeclasses" (a feature found in many other functional programming languages) by defining standalone "instances" of them for certain types that can be implicitly summoned when they are in scope.

**Pure functional programming** is a discipline of functional programming whereby you eliminate the possibility of "side-effects" in your code.
If your code does anything other than what the type signature indicates, such as accepting input from the user, talking to the network, or producing some form of output as a file or logging, then that code is said to be side-effecting.
In pure FP, Instead of immediately performing these actions, you instead construct a data structure that contains the intention to perform these actions, encoded somehow.
You can do this indirectly, using (for example) a list or tree of specially-typed values where each item in the data structure represents an action you wish to perform.
This is then passed to an interpreter that looks at the instructions in the order they are listed, and actually performs your side-effects.
More commonly these days, specific data types will directly encode your side-effecting functions into a value that can be chained into other values of the same wrapping type.
In pure functional Scala, and some other languages like Haskell, this is referred to as the type `IO`.
`IO` is some kind of "monad", meaning (among many other things) that you can sequence multiple `IO` values together to form a single "program" as a value.
You can then run this value directly to execute all of your stored up steps, or pass it to the runtime or some upper layer to run your program for you.

Some languages are themselves pure, and do not allow you to compile or run "impure" code, but Scala is impure by design.
This means that interacting with the pure and impure parts of Scala code often requires either "wrapping" impure code so that it becomes pure, or explicitly running pure code at the moments you wish it to be evaluated when used in impure code.
In practice, this is not so difficult to reason about, and even if you have never done pure functional programming you are probably already familiar with similar "patterns" in imperative code, such as the "builder pattern" where you have to explicitly build something before it can be used.
It does create a bit of an inconsistency that will force you to prefer one or the other style as much as realistically possible, however, so while you can adopt a library written in this style, you will not see the full benefits until most of your code is written this way.

## Our Initial Stack
Now that I've gotten a lot of the formalities out of the way, lets get into the meat of why we're here: moving from one stack to another, and the problems we hoped to solve.
Our stack at first was primarily using a tool in Scala called "Akka", which is an implementation of the "actor model" of computing.
Essentially, instead of manually managing threads and concurrent tasks, you modeled them as "actors" which are distinct entities that can send and receive messages, as well as maintain their own state.
Upon receiving a message, and actor can act on it by performing some action and/or modifying its internal state.
This model of programming is, I should note, not incompatible with a pure functional style in the slightest, and in fact you can implement your own actor system fairly easily with modern pure-FP Scala libraries.
I should also note, for the record, that Akka is also a very widely-used platform for developing distributed applications that can scale very easily in a cluster.
For these cases, I would highly recommend it, but from the start I knew it was not a good fit for our team or what we were intending to use it for.

When I say that Akka can "scale" easily, please take a moment to internalize that I am talking specifically about cluster computing, and very large workloads such as online multiplayer gaming where availability is paramount.
This is because the actor model itself, not specifically Akka, makes it possible to abstract your code away from the particulars of any one machine or thread, as each actor will be running "somewhere on your cluster", on a thread you cannot control, and at times that are naturally unpredictable.
The notion of "eventual consistency", a concept observed in large distributed systems where atomic operations are not always possible, is baked-in to the style of programming here.
Sending and receiving a message is entirely asynchronous, meaning that you cannot guarantee when and where something occurs, only that *if* something happens, your actors will behave a certain way in response.
It follows that, if your system requires eventual consistency (which most do at a large enough scale), you have programming primitives that work that way by-design.
The downside of this is that if you don't need that, or your only points of eventual consistency are *between different applications* and not between asynchronous tasks in a cluster, you can end up over-complicating what are otherwise fairly straightforward tasks.

The business problems that my team was trying to solve, and, chances are, the business problems of the team you may find yourself on at this moment, not only did not require designing with this in mind, but it actively made it more complicated every single step of the way.

Imagine you are writing an application that does some simple, single-purpose function.
For example, an app that reads from a database table or file once a day, validates that data, and publishes the results somewhere like maybe another table or an Apache Kafka topic.
Or, lets even make this more complicated and say you are doing a *lot* of simple things at once, but in a way that is easily partitioned across multiple instances of an application; say, subscribing to a partitioned Kafka topic in a shared consumer group (if you're familiar with that) or processing uploads to a service that is partitioned by-key to balance the load.
In my experiences, the vast majority of anything anybody is doing in software is going to be written along these lines, and in these cases I would say that actors are not only overkill but they overcomplicate the solution to each problem.
Instead of writing a function that, when evaluated, writes the contents of a file as a single logical operation, you might be tempted to write a `FileWriterActor` that receives messages containing the file contents and then writes them.
This is significantly more complicated, has more boilerplate, and is, quite frankly, a misuse of the actor model that does nothing to help you solve business problems effectively.

Over time, this resulted in code that I personally found difficult to read, test, and reason about.
The feeling was also shared amongst many of my fellow team members, and nearly every other day I would hear about some new, strange way that something was broken, or certain operations were not occurring when we expected them to.
This also extended, to an extent, to Akka Streams, a library that builds on top of Akka to provide a robust push-based streaming solution with adapters for several common integrations available as part of the "Alpakka" project (a pun name, meant to evoke Apache Camel, another integrations library for Java).
None of these libraries had any notion of controlling side effects, and it was generally seen as a feature of the platform that your code may not be very predictable at runtime, given its elasticity requirements.


At the end of the day, it was a fundamental issue of software architecture, and the tools that we used shaped the way that we wrote a lot of our code.
It shaped our expectations of "what Scala development was" in a sense, but the language is full of opportunity, and I intended to see if things would be better for us a different way.

## The Big Migration
My first real exposure to the pure functional side of Scala was with an excellent library called [Monix](https://monix.io/) by Alexandru Nedelcu.
I was attracted to it because I was a fan of "reactive streaming", and Monix was a straightforward and easy-to-understand implementation of that model of programming.
To make this work, and as part of its implementation, it defined something similar to `IO` named `Task`, which is like a computation that is suspended as a value, not yet executed.
While I did not pay much attention to this part of the library at the time, instead focusing more on the streaming capabilities the library offered, this type would later be my formal introduction to the world of pure Scala.

Some time within a few months of being brought on, I pitched `Observable`, the streaming type of the library (which should be familiar to those of you who have used a Reactive Extensions library such as RxJS or RxJava) as an alternative to Akka Streams.
I personally found it much less intimidating to get started with, and I felt that it had a much nicer set of operators available in its public API.
While there was some limited interest in using better tooling, the main concern amongst team members was that we had already invested a lot of time and energy into Akka Streams, and rewriting code or otherwise supporting two different sets of applications that use their own stacks, was just not something we were comfortable doing at the time.
While disappointing, this effectively illustrates the first major takeaway of my experiences:

### A Clear Path to Buy-In

Over time and as my personal interest in the more pure-functional side of Scala increased, I started using projects from the community organization that Monix belongs to, [Typelevel](https://typelevel.org/), in my own personal work.
I was able to experiment with different libraries, find out which ones seemed to work the best for the  type of work I wanted to do, and get more of a handle on the underlying concepts behind them.
This is also when I got more familiar with [Cats Effect](https://typelevel.org/cats-effect/), the library that provided the foundation for the `Task` portion of Monix.
This library also provided its own `IO` monad, usable everywhere `Task` was, and I was hooked.
No longer was I just interested in getting my team to use a different streaming library, but I wanted my team to experience the full benefits of pure functional programming for themselves.

The biggest hurdle to convincing my team to switch from one stack of software to another was not going to be solved by evangelizing its virtues relentlessly as if my team were a bunch of unconverted heathens.
No, to get people on board, the most effective way to do that was to find some way that clearly illustrates the problems faced, with a solution that, ideally, wows people.
I say "ideally" because realistically you may get a response along the lines of, "that's nice but things are working fine and we don't want to rethink our entire programming style for benefits we cannot directly observe."
If I was going to propose writing our applications in a pure-functional style, I would need to not only make the case as to why you should write in that style in the first place, but I should do so with clear, directed examples of the benefits we would see.
As I mentioned earlier, we had a serious problem with overly complicated code that did simple things, and this would be a prime candidate problem to focus my attention on as I tried to promote pure FP.

To attack the problems my team had been facing, I decided to use a three-pronged approach:
1. Disavow the "actors for everything" model that we had been up-to-that-point been using in every application. Instead, favor simple, linear function composition and only break out the concurrency toolbox when absolutely necessary.
2. Stress the dependability, testability, and assurance that pure FP gives you. By removing side-effects from the equation, and building your entire programs this way, it makes it so much easier to both model and control the effects of different parts of your programs.
3. Focus on the libraries, what they are good at, and what they give us *besides* the ability to write pure functional code. So many libraries that we ended up taking advantage of had such nice, user-friendly APIs that felt understandable, and less "magical" than some other libraries. If the virtues of whatever dogmatic programming philosophy I was spewing at them were not going to make any converts, I at least wanted to make sure that the actual libraries that follow these concepts are impressive and immediately valuable.

Over the next several months, I developed an internal presentation that went over the Cats Effect library and its immediate ecosystem.
By the way, if you learn any single library in the Scala ecosystem, the single most all-encompasingly-useful library there is, is Cats, followed very closely by Cats Effect (which is a separate submodule specifically targeting pure-FP).
The Cats family of projects implements a host of useful typeclasses for Scala and other implicit syntax enhancements.
Many libraries are not only built with them, but of the libraries that don't that implement similar functionality, it is usually trivial to interoperate with them and get the best of both worlds.

Anyway, for this presentation, I wanted to prepare a set of examples based on actual problems we faced using design patterns that I felt were clear improvements on what we had been using up to that point.
I did not write full applications, of course, but I wrote the basic skeleton of about two or three, and I walked my teammates through on the decision process and what benefits each decision gives you.
The most important thing I tried to keep in mind during this process, was to make it seem just as easy, if not easier, compared to what we were currently doing, so I made sure to leave more advanced techniques that required significant explanation at home.
The presentation was an amazing success and, within a couple months and after several internal discussions afterward, I got cleared to use Cats Effect on a new project that was starting up as a way of test-running this style of programming.

Of course, I got rather lucky, and even if you go through all the effort to make demos and presentations for your own teams, there's a good chance you might not get much response.
Once, at a Java job, I wrote my own `Either` monad (called `ErrorOrSuccess`) for a config-template-parser app, and I gave a lengthy presentation on the virtues of monadic error handling.
To my team at the time, this just flew way over their heads, not because they didn't understand it as-explained but because the problem that it solved was so far removed from their immediate vision.
Functional programmers tend to know the benefits of error handling in this way by heart, and sing its praises often, but if the "Java way" is to just throw and check exceptions everywhere, and it "works fine", what are you really gaining?
The solutions that I hoped to provide at the time were so far removed from the types of problems that they were having, and that instance turned into a bit of an unconvincing failure.

So when trying to sell your ideas to your team, however large or small they are, you not only need to be convincing that it solves a problem, but you need to present it as something that has a direct connection to the stressful programming problems they deal with on the job daily.
Of course, you can't just hop right over.
Any series of changes to the *entire way your team writes software* can't, nor shouldn't, be done overnight.
What you need is a plan, even if it's a bit ad-hoc, to introduce elements of this new style in a way that makes everybody feel comfortable.

### Swapping the Stack

So at this point we had a general idea of where we wanted to go, which was away from our old Akka-based stack and onto the Typelevel stack, which is the sort of organization that houses libraries like Cats.
This included, but was not limited to:

* Scala Standard Library w/ [Akka](https://akka.io/) -> [Cats](https://typelevel.org/cats/) and [Cats Effect](https://typelevel.org/cats-effect/) as the "baseline" of all programs
* [Akka Streams](https://doc.akka.io/docs/akka/current/stream/index.html) -> [FS2](https://fs2.io/), a different streaming library built on top of Cats Effect
* [Slick](http://scala-slick.org/) -> [Doobie](https://tpolecat.github.io/doobie/) for database access
* [Akka HTTP](https://doc.akka.io/docs/akka-http/current/index.html) -> [http4s](https://http4s.org/), for hosting HTTP servers
* [Lightbend/Typesafe Config](https://github.com/lightbend/config) -> [Decline](http://ben.kirw.in/decline/) for config and argument loading
* [Json4s](https://json4s.org/) -> [Circe](https://circe.github.io/circe/), an excellent functional JSON library

You would be right to assume that is a big set of changes for any team to go through.
It took a long time for everybody to get to grips with each layer in this new stack, some more than others.
On each project, we would try out a new library, one or two at a time, figure out the quirks and things to be aware of relative to our old solutions, and figure out if it was worth the switch.
On older projects that were already written in our older stack, it became a matter of necessity.
"Do I need to rewrite all of this code this new way? Does it need to be rewritten anyway for some other reason? Is the old way of doing things actively causing us a problem? Is there some way to quarantine our older code and only allow new code to be made with our newer styles and libraries?"

Moving over to these new libraries, as a gradual process, was mostly straightforward.
There were many changes and investigations into what libraries we should use that fit with this style over time.
After a while of working in this style and using these libraries, it became natural for most team members to not only use these tools but also to help teach each other how to use them.
Before we could get to that point however, we had to first overcome the initial difficulties with changing our programming style.
For some team members, this was not so hard, but over time it became much easier.

But in the beginning, things were not always so easy.

### Sometimes Technology Problems Are People Problems

On my team we had people along the full range you would expect as a result of major changes like this, from outright skepticism-turned-antagonism to evangelizing it alongside me.
Most people were just about as cautiously positive as you would expect, but the biggest difficulty I had was dealing with conflicts as they popped up.

My most significant conflict was dealing with an engineer significantly more senior than I, who had enthusiastically latched onto the stack and went off immediately to write his next application in that style.
Every team member was new to writing code in this style, and I had only written code like this on personal projects at the time.
So when this senior engineer ran into problems that nobody else on the team could solve, it became a point of contention for the entire team and it carried a huge risk of making me look bad.
As a less experienced engineer who had just convinced his team, full of people much more senior, to *completely change their programming style going forward* on the promise of more maintainable code, I was suddenly staring a huge possible career failure in the face.

Reaching this point, I think, is inevitable if you make it this far.
Rare is the significant technical overhaul that goes off without a serious hitch.
Somebody is going to disagree, possibly very loudly and directed at the people who originally championed these decisions, and you need a way of dealing with them and their particular problems.
Or worse, people will *want* to agree and even enthusiastically so, but if they keep running into difficulties that nobody can help with, that's a huge net productivity loss that also damages team morale.
In my case, if I wanted to make this migration succeed, somebody had to do everything they could in order to assuage the worries and concerns of my team members.
That meant that someone, in this case myself, had to clock into overdrive and help lift the rest of my team members up.

### An Expert Is You

There are so many more people who not only know more than I do about pure FP, but they have a much better grasp on the tools and techniques that pure functional programmers employ.
What I realized very quickly was that even if I was not an expert at the time, somebody had to be, or at least good enough with all of these tools to be able to assist my fellow team members.
If you are introducing some concept or library to your team, you'll probably find that you have to do what I ended up doing for the better part of two years: filling the shoes of the hypothetical expert you wanted on your team.

What that meant in practice, was not just being someone to give advice and do the research for people, but you have to be willing to sit down with people, work through their problems, sometimes on your own time, and figure things out along with them.
It's a lot of work, and I spent a good chunk of my time making sure that everybody who had questions, everybody who was new and unfamiliar, had not just examples to follow or documentation to read, but also somebody to sit down with and talk things through.
You can't be distant with your team, only focusing on your work, if you are trying to get them to adopt practices or tools that you're promoting.
Along the way, you also need to find out what your weaknesses are, what parts of the language, tooling, and other features are foreign to you, so you can be more well-equipped the next time "that one problem" comes up.
In due time, whether you like it or not, you'll end up becoming the "expert" that your team needs.

Because I had to be the main point of contact for all sorts of problems we ran into over the past two years, this naturally exposed me to all sorts of new ways things can go wrong.
By documenting and getting exposure to all of these problems, it will only reinforce your status as the person on the team with the most familiarity and knowledge with the problem domain.

(For an example of the specific, technical kinds of problems faced, see the section at the bottom of this article.)

Some problems, like the one described at the end of this article, had solutions that helped automate away the pain and enforce good programming discipline.
Others required more project-specific expertise, and that meant that I had to really dig into how these tools all worked together.
For example, FS2 as a streaming library is not just about streaming data, but about composing `IO` programs that do several operations repeatedly, have multiple sub-programs inside of them, or do various tasks concurrently with safe resource acquisition.
In the end, FS2 became a kind of cornerstone of our tech stack, and almost every app we wrote benefitted from using it somewhere.
This meant that in all of our applications, certain patterns would emerge, and what better way to encode those than to write them down?

Scala's primary build tool, SBT, allows you to create project templates using a tool called Gitter8, which is a sort of template language.
While helping my fellow team members with pair programming or them generally asking me for advice was enjoyable, it was not a very efficient use of my time, so I used this Gitter8 tool to create special templates for starting new projects.
As we started new projects a lot, this greatly reduced the amount of boilerplate necessary as certain common things like an HTTP health check, common config variables, and general project structure setup were just a single command away.
As we had different projects with different needs, these templates established the "baseline" of common libraries used, such as Cats, Cats Effect, Decline, and FS2.
Other libraries with more specific usages, such as fd4s/fs2-kafka for our apps that uses Apache Kafka, were able to be "toggled" during the template init process, and with that toggle came some example code for getting started with that library.

In addition to using templates to share example code and project structure, I wrote a large list of "best practices" into a document that I would reference to my coworkers.
I made it a personal rule that every single time I gave somebody pointed advice about writing pure functional code, or anything with these libraries in particular, I had to write it down in here.
It served not only as a central repository for advice for my team, but also a great way to remind myself of things I've said, or given out as advice.
While it's not the point of this article to tell you every single one of them, there's a possibility some version of that will be published here on my personal website, or as a GitHub Gist someday.

But of course, I never would have been able to compile these things together, or gain the project knowledge that I had, without a little help from my friends, so-to-speak.

### Never Hurts to Ask For Help

None of anything I am discussing here was solely on me.
While it's true that, if you are going to sell ideas to your team, you need to be willing to put in the effort, it is not a one-person job.
Sometimes you are able to have other team members be just as interested as you are, and they are able to take some of the burden off your shoulders.
While I was not quite so lucky, I was able to find a lot of help in your friendly neighborhood Scala community.

Each of these projects we worked with, from the official Scala-affiliated projects like the language itself and our build tool SBT, to the Typelevel-stack projects we worked with daily, has a Gitter chatroom that you can join.
Joining these, even if you are not outright saying anything yourself frequently, is a great way to increase the surface area of your own knowledge by exposing yourself to new concepts and ideas daily.
If you are interested in a particular project, one thing you can also do is seek out the maintainers, follow them around online, and look at the talks, blog posts, gists, or other materials they are associated with, if any.

As much as I wanted to always be there to help my coworkers out, sometimes the best thing to do for both you and them is to refer them elsewhere.
Point them at the community, and make sure that everybody knows this is an acceptable thing to do.
You get a break for the good of your mental health, and they get more exposure to the problems that other people in the community are also having, same as you.
I can tell you from experience that every person I've worked with who has taken this advice to seek out project communities has gone on to become so much more proficient with them on a daily basis than before.
There are practically no downsides, only making sure to invest a little time here and there.

Finally, it just so happened that a lot of what I was going for was not only desirable within my team, but the outside help that we brought in was just itching for an opportunity to dig into pure functional programming as well.
We had hired several consultants over the years, and two consultants that we hired (named below in the Acknowledgements section) were instrumental in providing additional assistance wherever possible.
There are many consultants in the Scala world who are dying to get to work on projects with this type of stack, and they took the opportunity to not only help us build out our apps in a pure-functional style beyond what I was capable of myself, but they also wrote some training documentation for us to go through to make sure everybody was on the same page.

As mentioned at the very beginning of this article, and worth repeating to make clear my biases, I, too, will be taking my experiences learned here and elsewhere to my new position with 47 Degrees, which offers consulting services for Scala and many other languages.
I want to help enable people to make the best Scala software they possibly can, and being familiar with most major areas in the Scala ecosystem including working on a transition of this size, I hope that my services will be useful to the people I end up working with in any area.

If I have a single take-away from my experiences listed here, it's that any kind of programming technology change, especially one wher you're swapping out your entire paradigm, is a lot of effort, and it is not a one-man job.
You not only need buy-in from your team, but you need a community full of people who have put in effort of their own to make things easy.
We should not be afraid to reach out and ask for help, from the small things like fixing and understanding bugs, to the larger things like training materials, availability for answering design or implementation-level questions, and being able to listen and take people and their problems seriously.

## In Conclusion
* Have a clear goal, no matter how large, and make sure you have a path to get there. Try to focus on a real problem that your team is having, such as testability, code flexibility, or being able to depend on the software you write without feeling like there's a layer of "magic" underneath it all.
* Always have more than one reason for why a change should be made. For me, the libraries and frameworks were ergonomic and very usable out of the box, in addition to solving underlying problems that were hard to illustrate in presentations. Cover your weaknesses and it will be much easier to convince people.
* You need to put the work into materials. Presentations, working code, documentation, templates, you name it. If it doesn't exist, you're asking the rest of your team to pick up *your* slack, and that doesn't look good on you when you're trying to "sell" your team on something.
* Always try to be available. I was passionate about this, so I always tried to respond to questions and concerns from my coworkers even at odd hours. I would not say that is a requirement, but sometimes even just being there during working hours is more than enough. Listen to people, their problems, and take them seriously. Work to overcome them together.
* Write everything down. All of the tips, tricks, techniques, patterns, you-name-it, that you come across and implement as part of your journey. You'll not only thank yourself later for cataloging it, but you'll cement the knowledge in your head, only making you more effective at helping your team succeed.
* Ask for help. This is not a one-person job, nor should it be.
Get involved with the community of people making the tools you're using, and ask them any questions you come up with.
Sometimes this also means investing in training for your team, whether through internal or external means.

Most importantly, watch out for your own mental health.
Any kind of shift of this magnitude on your team requires being able to do a lot of work to accomodate everybody, their fears and confusions alike, but at the end of the day you are a person with your own needs.
Don't be afraid to say no, or refer people who need help to other resources when you can't be there.

Overall, my time working on all of these projects has been an immense learning experience, not just in how to write effective pure functional code, but as a case-study in the kind of things that need to happen to pull off a transition like this over the years.
I am happy that it has not only gone so well, but I believe that it will last a long time as the seeds of properly believing in this style of programming have been planted across my team.
It's no longer just some fringe theory of software development that I happen to employ on the weekends for personal projects, but through lots of help and perseverence we were all able to make this a reality for our professional work as well.

If you find yourself in these shoes, with big ideas and an eye for change at your workplace, you may not always succeed but do know it is possible with the right attitude, approach, and application of discipline.

## Acknowledgements
I would like to acknowledge my amazing team that I've worked with until now at [Rewards Network](https://www.rewardsnetwork.com/) in Chicago, IL.
If you are looking for a full-time Scala position, and perhaps want to get your hands on projects using these fancy libraries, I can wholeheartedly recommend them.
They are remote-friendly (not just from the COVID-19 pandemic) and are very accomodating to any person's unique needs in my experience.
They are also a very safe, friendly, and accommodating place to work for developers with a generally progressive outlook, so you can be sure that if you don't fit the stereotypical bill of a "white cis-het male developer" that you will be welcomed and fit right in.

Special thanks also goes to all of the Gitter channels I frequented for most of this time, including all of the Typelevel projects and their maintainers.
I could have never figured out as much as I have without the constant (really, constant) guidance and assistance I have received from every one of them.

We would not have accomplished everything if it weren't also for the amazing assistance from Francis Toth and Calvin Lee Fernandes, two consultants who we worked with during parts of the transitionary phase.
They put together training materials as well as developed applications with us in a pure functional style, and overall I was very impressed with their quality.
If you would like to contact either of them, here are their websites:
* Calvin Lee Fernandes - <a href="https://www.kaizen-solutions.io/">https://www.kaizen-solutions.io/</a>
* Francis Toth - <a href="https://contramap.dev/">https://contramap.dev/</a>

And of course, if you are reading this, this is my personal website.
You can access the sidebar of this site to see the most up-to-date information on contacting me personally, or as mentioned you can also get to me through [47 Degrees](https://www.47deg.com/).

## Further Reading
If you are actually interested in pure functional programming in Scala, besides offering my own services 

* [Functional Programming with Effects](https://www.youtube.com/watch?v=30q6BkBv5MY), presentation by Rob Norris (aka tpolecat, author of Doobie).
  * Understanding pure-functional programming means understanding the concept of "effects", and this is explained excellently here.
* [The FS2 Docs](https://fs2.io/guide.html)
  * I firmly believe that FS2 is one of the most powerful and most generally useful libraries for composing programs in Scala. The fact that it's pure-functional is an amazing bonus, and it has a thriving ecosystem.
* [Practical Functional Programming in Scala](https://leanpub.com/pfp-scala) by Gabriel Volpe
* ["The Red Book" - Functional Programming in Scala](https://www.manning.com/books/functional-programming-in-scala) by Paul Chiusano and Runar Bjarnason
  * Side-note: this is the book that got me into Scala in the first place and demystified a lot of what functional programming is, in a nutshell.
  * Also, the authors of this book are now developing the [Unison](https://www.unisonweb.org/) programming language. It looks fascinating and might be good for you to look into in your spare time.

##  That One Example Problem in Pure FP
One of the common problems we ran into very early on was that certain operations were not happening when we expected them to, or at all.
This was also a problem we ran into with Akka, due to its eventually consistent nature, but the problem here was much simpler to solve and was simply a matter of programming discipline rather than being an eventual consistency problem.
You see, in pure FP, you never execute anything at all until the "end of the world", which is the location at which your code *needs* to be ran.
So for most programs, this is the end of your "main method", or if you are writing code that is passed to a callback in some Java library, you might need to manually run your code inside of there.
The problem we ran into was simply a matter of making sure that, instead of trying to "run" code like before (e.g. calling a method and doing nothing with the result), we had to make sure that *every single operation* chained into another operation, and the result was returned.

Here is a small code sample of this type of problem, for reference:

```scala
import cats.effect.IO

//In Scala, Unit is what many other languages call "void".
//It is used when your result value is useless, such as when printing to your console.
//This function is supposed to upload some data to a repository.
//Then, it is supposed to post a success message using the client.
//Here is generally how you would write it using blocking APIs in a non-pure-functional style:
def myBuggyFunction(client: MyClient[IO], repo: MyRepository[IO]): IO[Unit] = {
  val exampleData = List("Hello", "World", "!")
  repo.uploadSomeData(exampleData) //Result: IO[Unit]
  client.postSuccess //Result: IO[Unit]
}

//The above function has a bug, can you spot it?
//The only thing that happens is the client posts "success".
//The database is never accessed at all. Why is that?
//This is because the operations, IO values, are not executed until the "end of the world"
//To properly execute them, we need to return the right IO value.
//Here is a version where we make sure that our operations are properly sequenced
//flatMap here means, chain this first IO value into the second one
//The underscore _ means "ignore the result of the uploadData operation"
//Normally you would use flatMap to create "dependent" IO values that use the result of the first one.
//Because we don't care about the result, it is disregarded with the underscore.
def myFunctionWithoutBugs(client: MyClient[IO], repo: MyRepository[IO]): IO[Unit] = {
  val exampleData = List("Hello", "World", "!")
  val uploadData = repo.uploadSomeData(exampleData) //Result: IO[Unit]
  val postSuccess = client.postSuccess //Result: IO[Unit]

  uploadData.flatMap(_ => postSuccess) //Result: IO[Unit]
}
```

As the comments above explain, this particular problem can be a bit tricky to realize you are doing unless the compiler is helping you out.
So, to get into the habit, we enabled a compiler warning for "discarded values", meaning that every single (non-Unit) value in a function has to be either used to compute some other value or returned from the function.
This single change eliminated the source of the error and helped enforce good programming practices.
To use this yourself, I would recommend the "sbt-tpolecat" plugin which enables this and many other useful (I'd say necessary) compiler warnings that enforce a good programming style.