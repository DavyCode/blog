---
layout: post_extend
title: "Why Auth0 chose Serverless Extensibility"
description: ""
date: 2017-12-04 10:19
category: Extend, Business
author:
  name: "Bobby Johnson"
  url: "https://twitter.com/NotMyself"
  mail: "bobby.johnson@auth0.com"
  avatar: "https://cdn.auth0.com/website/blog/profiles/bobbyjohnson.png"
design:
  bg_color: "#3445DC"
  image: https://cdn.auth0.com/website/blog/extend/why-auth0-chose-serverless-extensibility/logo.png
tags:
  - extend
  - serverless
  - webhooks
  - extensibility
related:
  - 2017-05-16-introducing-auth0-extend-the-new-way-to-extend-your-saas
  - 2017-05-19-serverless-webhooks-with-auth0-extend
  - 2017-08-22-for-the-best-security-think-beyond-webhooks
---
Previously, we wrote about the emerging pattern of [Serverless Extensibility](https://auth0.com/blog/why-is-serverless-extensibility-better-than-webhooks/). Examples of the concept are popping up in many of the services you use daily like [Twilio](https://www.twilio.com/functions) and here at [Auth0](https://auth0.com/).

How did Auth0 get to the point of offering extensibility through a serverless platform? It has been a four-year journey that predates both [Amazon Lambda](https://aws.amazon.com/lambda/) and the term Serverless. Like all innovation, it starts with resource constraints, the need to give customers the features they wanted and making sales.

## The problem we were trying to solve

![NASA Engineers working on MAVEN](https://cdn.auth0.com/website/blog/extend/why-auth0-chose-serverless-extensibility/engineers_working_on_the_high-gain_antenna_of_the_MAVEN_spacecraft.jpg)

*Placed into the public domain by [NASA](http://mars.jpl.nasa.gov/multimedia/images/?s=1) using [Creative Commons](https://creativecommons.org/publicdomain/zero/1.0/)*

In the early days at Auth0, there were two groups: Core and Field Engineering. Core focused on the core functionality of the authentication product, and the field engineers helped customers use the product in their applications. The company was discovering what customers needed in the product.

Our customer's focus was on the authentication transaction. A lot of interesting features can attach to the process of someone trying to login:

- Profile Enrichment
- Progressive Profiling
- Authorization
- Claims Transformation

The possibilities are endless. We could not possibly build every feature at once.

Moreover, sometimes building features into the core product for customers is not the right approach. We needed a way to try ideas out. An idea may sound great, but in practice be problematic. Investing core engineering resources to these ideas is very expensive.

> **"The primary use case for extensibility at Auth0 was to empower field engineers to work on the last mile solutions for the customer without involving core engineering."**<br />
> Eugenio Pace - Co-Founder, VP Customer Success

If every interaction with a customer involved identifying an idea, putting it in the backlog and coming back to them at a later date, it would add friction to the process and turn customers away.

## Custom code extensibility

![NASA's "Webb-cam" Captures Engineers at Work on Webb at Johnson Space Center](https://cdn.auth0.com/website/blog/extend/why-auth0-chose-serverless-extensibility/34827899312_571fa2a976_k.jpg)

*[NASA](https://www.flickr.com/photos/gsfc/34827899312) licensed under [Creative Commons 2.0](https://creativecommons.org/licenses/by/2.0/)*

The inspiration for custom code extensibility as a solution came from spreadsheets. Excel has significant functionality out of the box, but there is always a function, macro or calculation that is not available. However, you can write them yourself directly in Excel removing the dependency on Microsoft engineers.

> **"We wanted a similar experience for our users. A user should be able to log in to the dashboard, write a small amount of node.js code that executes later during authorization transactions."**<br />
> Eugenio Pace - Co-Founder, VP Customer Success

Like Excel, we wanted to offer users a really simple experience for customizing our product through code. We believed a user should be able to write their logic in an editor, debug it in place and make it live in production, all without having to stand up a service. This differed from the prevalent approach at the time for dealing with customization through webhooks, which were hosted by the user.

We decided to go and implement an MVP of the concept using [node sandbox](https://www.npmjs.com/package/node-sandbox
). It was very similar to a CGI model; spinning up a separate node process, sending the custom code to execute in it and collecting the result. Every custom code execution was using a dedicated system process, which was resource intensive.

This implementation had some issues, primarily a lack of security. It was merely creating a process boundary between the core Auth0 stack and the customer's code. Node sandbox was primarily preventing well behaved, well-intentioned code from accidentally bringing down the authorization service or other sandboxed code. It was not preventing malicious code from accessing things it should not or corrupting the environment.

> **"It was a very expensive operation to run because each rule would spin up a new process."**<br />
> Sandrino Di Mattia - Engineering Lead

Although the MVP had its limitations, it proved that the user experience for customization could be greatly improved, aligning well with our philosophy of "Identity made simple for developers."

## How do you secure execution of untrusted code?

![NASA engineers working on Webb](https://cdn.auth0.com/website/blog/extend/why-auth0-chose-serverless-extensibility/nasa_engineers_conduct_low_light_test.jpg)

*[NASA](https://www.flickr.com/photos/gsfc/15119297052/) licensed under [Creative Commons 2.0](https://creativecommons.org/licenses/by/2.0/)*

> **"When Eugenio and I started talking in August of 2014, this turned out to be my interview question. We have this problem, this is what we want to do, how would you secure it? "**<br />
> Tomasz Janczuk - Chief Webtasks Architect

The answer to that question was a new technology we would create, Webtasks. The initial high-level goal for the Webtasks architecture was to create a protocol boundary around the core Auth0 stack and the custom code execution engine and create a multitenant sandbox behind that protocol.

The first prototype of the Webtasks architecture was an interview exercise. Extensibility was a pressing enough problem; it turned into an actual assignment. The implementation was relatively simple; it was a month of work to complete.

The stabilization effort however was something entirely different. You cannot solve a stabilization problem by adding people to the team. It's a long road of many sleepless nights to build a robust, stable and secure service; it took us several years to arrive on our current solution.

## Evolving the Webtasks stack

![NASA enginners working on GPM](https://cdn.auth0.com/website/blog/extend/why-auth0-chose-serverless-extensibility/10860353183_307529002e_k.jpg)

*[NASA](https://www.flickr.com/photos/gsfc/10860353183) licensed under [Creative Commons 2.0](https://creativecommons.org/licenses/by/2.0/)*

### Creating the platform

The node sandbox MVP suffered from a lack of isolation between our various customer's code, we decided to focus on isolation through container based sandboxing. The first version of Webtasks used early versions of [Core OS](https://coreos.com). Core OS at the time used three distinct technologies to provide a platform for creating distributed systems:

- Docker for managing containers
- etcd for distributed configuration management
- Fleet for distributed service management

Core OS promised that these three components complemented each other well. Docker organized compute, etcd provided configuration management in a cluster and Fleet decided what workload ran where and when.

This architecture maintained a named container on a single virtual machine within a cluster. When a request came in to execute some code in this container, we used etcd to find the VM the container was provisioned on and reverse proxy the request to that VM.

Becuase this architecture depended on distributed configuration management it had a higher exposure to failure. Also, add to this the fact that individual virtual machines had a certain probability of failure for random reasons. If a single container only lives on a single virtual machine, there is a single point of failure for that container.

Once in production, we realized we were relying on the cutting edge of three technologies. We had instances of the platform destabilizing at 2:00 AM in the morning in Australia one too many times. It felt like a constant game of whack-a-mole, chasing one stability issue after another. When we upgraded the stack to new versions of Docker, etcd or Fleet; some new issue would pop up.

![Webtasks V1](https://cdn.auth0.com/website/blog/extend/why-auth0-chose-serverless-extensibility/version_1.3.png)

### Stabilizing the platform

The focus of version two of Webtasks was to simplify the architecture and remove everything that was not absolutely needed to increase the fault tolerance of the overall system.

> **"We started with the assumption that the virtual machines should operate entirely independently making them private universes."**<br />
> Tomasz Janczuk - Chief Webtasks Architect

In terms of container management, the VMs have no business communicating with each other for anything but non-critical functionality like real-time logging.

This change removed the need for etcd because we no longer needed to exchange state between the virtual machines. It also removed the need for Fleet because a named container was allowed to exist on all the virtual machines in the cluster at any given time.

In the new model, when a request comes in, the load balancer decides to send it to a particular virtual machine. The virtual machine considers that request in complete isolation from the other VMs in the cluster. If the VM needs a container, it picks a new container from a pool of pre-created, unassigned containers on that VM.

At this point the only component of Core OS still in use was Docker. So, we dropped down to vanilla Ubuntu. The process of simplification was a metamorphosis of the stack that resulted in considerable improvements in stability.

![Webtasks V2](https://cdn.auth0.com/website/blog/extend/why-auth0-chose-serverless-extensibility/version_2.1.png)

### Stabilizing real-time logging

The third change to the Webtasks stack focused on real-time logs. To this day it is the only feature in the Webtasks architecture that requires virtual machines to be aware of each other's existence.

Real-time logs work by consolidating logging information into a single point. A client makes a management API request providing filtering information. Regardless of the virtual machine in the cluster that the client attaches to, it collects the real-time logging information from all other VMs and streams it back to the client on the HTTP response.

The original implementation of this used [Kafka](https://kafka.apache.org/). Kafka is a very high throughput message broker optimized for log aggregation scenarios. There are many success stories from places like Netflix using it.

> **"At the same time there was certainly some blood on the street with Kafka. There were some horror stories around management and so on."**<br />
> Tomasz Janczuk - Chief Webtasks Architect

Kafka builds on top of [ZooKeeper](https://zookeeper.apache.org/) for distributed configuration management, similar to how Core OS uses etcd. It turned out, at the time, ZooKeeper had a few skeletons in the closet regarding stability. The Kafka ZooKeeper components were destabilizing enough to cause virtual machine failures. As with etcd, tracking down the issues was never-ending after three months.

We started looking for alternatives and landed on [ZeroMQ](http://zeromq.org/). ZeroMQ has no storage involved at all. It is an in-memory system that provides messaging patterns over TCP. ZeroMQ's publish/subscribe pattern allowed the source of logging information to act as a publisher. We could have any number of subscribers providing filtering criteria and receiving messages. One nice feature is of ZeroMQ is it is stateless and does not require any storage, which helps with reliability. This was exactly what we needed for real-time logging.

![Webtasks V3](https://cdn.auth0.com/website/blog/extend/why-auth0-chose-serverless-extensibility/version_3.1.png)

Switching to ZeroMQ was the single most stabilizing change we made in the history of the Webtasks cluster. It was such an impressive improvement Tomasz wrote a [post about it](https://tomasz.janczuk.org/2015/09/from-kafka-to-zeromq-for-log-aggregation.html). That post received 10,000 views the first day it published. Others were apparently having similar issues.

## Evolving Webtasks features

![NASA engineers cleaning mirror with carbon dioxide snow](https://cdn.auth0.com/website/blog/extend/why-auth0-chose-serverless-extensibility/17402277412_5e2834517c_k.jpg)

*[NASA](https://www.flickr.com/photos/gsfc/17402277412) licensed under [Creative Commons 2.0](https://creativecommons.org/licenses/by/2.0/)*

### Feature parity with node sandbox

The first version of Webtasks functionality started as a better equivalent of node sandbox. The execution of custom code took only one HTTP request. The body of that request contained the code to execute.

When a new authorization request comes in the code authored by the customer is bundled up along with contextual information like the user object and request headers. This bundle is sent to the execution engine for execution. In this model, the Webtasks cluster is entirely stateless and unaware of any notion of code being stored anywhere.

Another improvement the Webtasks implementation realized compared to node sandbox approach was the reuse of the system process across several rule executions of the same tenant. Compared to the node sandbox model which was like CGI creating a new process for every request. This version of Webtasks and the way it was used was like FastCGI. We were still sending the code to execute every request, but the process persisted across many requests. This enhancement saved considerable time recreating the process.

### Moving to named webtasks

The next functional change in Webtasks was a move from a pure sandbox model to a named webtask model. We created a set of management APIs that allowed the creation of a webtask that could then be invoked separately.

> **"Imagine our biggest customers having thousands of logins per second. Performance-wise instead of having to send up to 100kb of code, directly sending the request to a webtask and getting a response back dramatically increased throughput."**<br />
> Sandrino Di Mattia - Engineering Lead

This change was a more traditional model bringing it conceptually in line with other FaaS providers like Lamda. Named webtasks had an advantage in allowing the system to optimize compilation of the code once and execute over and over. It also freed up the body of the request sent to the webtask making it much more useful for a large number of scenarios.

### Focus on startup latency

Auth0 is in a unique position from other FaaS providers in that our code executes in the UI path. With each execution, a user is sitting at a login dialog and watching a spinner spin. There is a brief window of time that we have to execute all webtasks.

Auth0 is also a highly multi-tenant platform. Some customers need to execute webtasks infrequently. A typical authentication scenario for them would be users come to work and log in then are done for the rest of the day. Since custom code must execute in isolation of rules for other customers, these users who come in later very frequently encounter a situation where our stack is cold.

> **"We have put considerable effort into optimizing webtask execution latency so that it is predictably low, regardless if the stack is warm or cold. Unpredictable or high latency would reflect poorly on the end user experience."**<br />
> Tomasz Janczuk - Chief Webtasks Architect

A choice was made to keep a pre-warmed pool of containers that are immediately ready to execute webtasks. When a request comes in execute code on behalf of a customer that has not been seen in a while, the Webtasks platform picks a container that is already running from a pool of unassigned containers and reverse proxies that request to it.

There is some overhead in making the assignment compared to a warm request, but it is considerably less than spinning up a new container. It is probably the most distinctive aspect of our platform allowing us to provide a low variance of latency between requests, regardless if they are warm or old; offering a high degree of predictability.

## The impact on our sales engineers and customers

![NASA engineers celebrating curiosity](https://cdn.auth0.com/website/blog/extend/why-auth0-chose-serverless-extensibility/celebrating_curiosity.jpg)

*Placed into the public domain by [NASA](https://commons.wikimedia.org/wiki/File:Celebrating_Curiosity.jpg) using [Creative Commons](https://creativecommons.org/publicdomain/zero/1.0/)*

Adding extensibility to the identity product opened up a window of customization where field engineers could work independently from core engineering. They could deliver customizations very quickly without waiting weeks or even days, enabling many successful sales engagements.

> **"In some cases, we were able to deliver demos where we implemented feature requests in the demo during the meeting. It was amazing for field engineers and our customers."**<br />
> Eugenio Pace - Co-Founder, VP Customer Success

Putting extensibility into the hands of customers also gives us great insights into where the market is going. If we see an extension being implemented again and again through the use of custom code, that is a validation of that feature and an opportunity to add in the core product.

Multi-Factor Authentication came about this way. MFA was not a switch on the dashboard initially. It was merely a rule you applied to your account. Over time, our customers' usage of this rule indicated MFA was an important feature to have in the core product.

For more examples, take a look at our [rules repository](https://github.com/auth0/rules) on Github. Every single one comes from a real-world customer use case.

> **"Every event in the life cycle of the user can be expressed through code and in that way we can support any backend out there in a flash. No matter how customers encrypt or hash passwords; whatever they do is supported because they can provide the details using custom code."**<br />
> Sandrino Di Mattia - Engineering Lead

## Summary

Serverless Extensibility is a logical evolution to Webhooks. It has taken us four years of experimentation and refinement, and the resulting Webtasks platform is a success that empowers our field engineers to close deals, simplifies our core architecture and delight our customers.

We have created a product called [Auth0 Extend](https://auth0.com/extend/) based on the pattern that allows other SaaS companies to offer serverless extensibility from their products quickly.

When [Jeff Lindsay](https://twitter.com/progrium) was introduced to Extend he certainly felt we were on to something. Jeff was the person who coined the phrase [Webhook](http://progrium.com/blog/2007/05/03/web-hooks-to-revolutionize-the-web/), which has become ubiquitous on the web as a way to offer extension points from applications online.

<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">This was the whole point of pushing webhooks in 2007. Literally built this as a prototype. <a href="https://t.co/Wyoz0qXO9P">https://t.co/Wyoz0qXO9P</a></p>&mdash; Jeff Lindsay (@progrium) <a href="https://twitter.com/progrium/status/864588610858881029?ref_src=twsrc%5Etfw">May 16, 2017</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

If you are interested in giving it a [try](https://auth0.com/extend/try?umt_source=extend_blog&umt_medium=utility-bar-try&umt_campaign=extend_blog), you can start today for free. Feel free to [contact us](https://auth0.com/extend/#support) if you have any questions.

I would like to thank [Eugenio Pace](https://twitter.com/eugenio_pace), [Sandrino Di Mattia](https://twitter.com/sandrinodm) and [Tomasz Janczuk](https://twitter.com/tjanczuk) for taking time out of their busy schedules to be interviewed for this post.
