---
layout: post
title:  "A path to flexible emergency-aware architecture"
date:   2023-11-23 00:00:00
description: ''
---

A samurai has no goal, only path...

### Introduction
If you want to have a productive discussion about system design, it's better to start from requirements. Most articles in this field focus on scalability, fault-tolerance and deployability, but it's far from covering all applications. What we needed was a website, a file storage, an admin panel, and a lot of different things to help us with our ideas that constantly appear and disappear. In other words, a lot of functionality, but cheap, highly configurable, and hopefully workable. And it takes some time to figure out how to do it the best way.

### AWS: Apparently Very Simple
So, we decided to build our first functional prototype using an AWS serverless stack:
- `AWS Amplify` for frontend hosting
- `Lambda` as a backend replacement
- `DynamoDB` to store data
- `S3` to store files
- `API Gateway` to connect all these services together  

It was fairly easy to set up all components and make them work, but also this easiness was kind of concerning: we questioned ourselves how much control over the application we actually have and how flexible it was. Particularly we had somewhat high latency without visible explanation and ability to debug/fix it. Answers on different forums were mostly suggesting to change the hosting region. That's all you can do.
Another questionable aspect was pricing. AWS is well known for their "pay for what you use" policy, but it doesn't mean that the usage is cheap. Free tier was barely enough to test the application for free during development (not very active development, we were doing it in our free time). With increased amounts of content and additional functionality it was going to become a significant expense item.
And the final reason that forced us to seek a different approach was service dependance. Our functionality at that time fitted pretty well into a small set of AWS services. But what if we wanted to run our own Minecraft Server? Or VPN? Or some other experimental project? We could use an EC2 service for example, but it would not be so cheap and easy. Clearly, AWS infrastructure did not provide enough *Flexibility* for us.

### Real server for a real company
As we didn't need horizontal scaling of our application, an option of running it on our own physical server seemed pretty good. With a bunch of unused PC components, the latest stable ubuntu server distributive and a place with good internet connection, we felt a real freedom and a sense of control. Porting the application took some time, but was not so painful:
- Frontend, written on `Angular` didn't require almost any changes
- Lambda functions were easily translatable into simple python scripts, powered by `Flask`
- Data from `DynamoDB` can be easily migrated to `MongoDB`
- To manage these data we used a self-hosted `mongo-express` instance
- For file storage any `S3` compatible self-hosted service just works
- For routing we used a simple `NGINX` server, configured as a reverse-proxy  

With every component containerized and easily manageable, this stack requires close to no effort to support, upgrade, or change. Everything worked fast, new components were added, it seemed like a happy end to this story.

### Emergency replication
But here came the blackouts and the main flaw of our design became uncovered. Having one (or even multiple) physical machines in one location makes them incredibly vulnerable to the influence of outer world, thus reducing the *Reliability*. Before choosing an appropriate solution of this problem we need to understand what functionality is critical and what is not. Let's take for example this two scenarios:
1. User tries to join the minecraft server and it appears offline - Not very good, but understandable  
2. User open the website and gets ERR_CONNECTION_TIMED_OUT - Company reputation is totally ruined  

.Such differentiation of scenarios is how real world entities handle the state of emergency. Hospitals switch to emergency lights and cut off all unnecessary electricity consumers to be able to sustain the critical infrastructure on diesel generators. In our case electricity is traffic/computational time/money and diesel generator is AWS.
Take note, that at this point all base infrastructure is AWS compatible by design. MongoDB is replaceable by DynamoDB, file storage is S3 compatible and the frontend is pretty universal, it just needs the right parameters to be inserted. Relationship between self-hosted infrastructure and AWS based is similar to leader-follower replication with few differences:
- Only critical data and functionality is replicated
- Writes are accepted only by *true leader*
- Reads are accepted only by *true leader* unless it is unavailable
- In case of *true leader* fail, follower becomes an *emergency leader*  

.This design is not perfect by any means, but fully satisfies our requirements. Most of the time cheap, but unreliable, *true leader* serves the users and during the blackouts, an expensive, but very reliable follower takes charge to serve critical functionality of the web application. Design of such system is demonstrated below:


![img](https://eknm-hub-public.s3.eu-central-1.amazonaws.com/building_hub/building_hub_dark.jpg)

On this scheme you can also see a *Health Proxy* layer, which we implemented on a client side. When a user opens our website, the AWS hosted web page is loaded. This page instantly tries to ping the *true leader* and decides where to route the request depending on leader availability. Statelessness of this proxy allowed us to make it extremely lightweight and always have client-specific availability data, approaching close to zero downtime. 

### Overengineering? Never heard of that
It's easy to be tempted by a huge beautiful scheme where everithing interacts with everything and all situations are handled. It may even be fun to implement. What is not fun is supporting this kind of architecture. If you take the cost of labour in account, the emergency replication scheme depicted above just doesn't save any money if you need to change things frequently. Current state of the project is transition to some kind of micro-site architecture, where each independent part is a separate site, server or whatever it should be. Will it be the final chapter? Time will tell.