+++ 
date = 2020-02-12T20:37:41+01:00
title = "How I failed using Google Cloud Functions"
+++

This post is a follow-up to my previous article about [Quarkus, Kotlin and GraalVM](/posts/quarkus-kotlin-and-graalvm).

I was proud to have finished the Snake game, which was working in multiplayer mode. However, I wanted to improve the gaming experience by addressing two areas: the interface and scalability. I decided to focus on scalability, but unfortunately, I failed.

I have used [Google Cloud Functions](https://firebase.google.com/docs/functions), [Firebase](https://firebase.google.com/), [Firestore](https://firebase.google.com/docs/firestore) and [Google Pub/Sub](https://cloud.google.com/pubsub).

# Motivation
In the original project, I used [VertX](https://vertx.io/) for in-memory PubSub to manage the game events. This is a powerful and versatile library that allowed me to write less coupled code, use resources efficiently, and simplify my work by providing an easy way to use EventSource and Server Sent Events. As a result, the interface was truly reactive.

However, this was not a scalable solution because everything was done in memory. There are thousands of ways to do it, but I wanted to try Cloud Functions and see how it performs, if it has good performance, and if the code is easier to maintain and understand.

# What has been done?

Coding the functions was the fastest part because the idea was already there thanks to the previous project. However, the learning curve took more time because I had to learn how to use Firebase to publish the functions, Firestore to persist the matches, and Pub/Sub to broadcast the events.

The original code can be found [here](https://github.com/brunopacheco1/worms), and the serverless code can be found [here](https://github.com/brunopacheco1/worms-serverless).

# What have I found?

If you are brave enough to compare the code, you can check the repos yourself, but here are my findings:

Certainly, the code is much smaller using lambda functions. The code in Kotlin/Quarkus was more verbose because I had to code all the layers by myself (domain, dtos, repositories, services, controllers, exception handling). With lambda functions, I only needed the domain classes (specifically to initialize default values), the state evaluators, and controllers. I believe this is due to the language - Javascript is simpler and easier to manipulate objects, and Firebase provided me with easier ways to expose the endpoints and connect to all the components.

Using Firebase and other Google Cloud products was easy and cost-effective, with clear documentation and smooth integration. However, Google Cloud Functions underperformed. In comparison, hosting the Worms game on a small instance provided a satisfactory gaming experience, while Google Cloud Functions made it impossible to play.

# What should I have done?

- I should have added stress tests to have a reliable way to compare both projects.
- I should have identified what needed to be improved in both projects. My initial thoughts were that the reduced amount of CPU power and latency when connecting to Firestore and Pub/Sub were issues. The slower first calls could also have been due to resource allocation.
- I should have properly documented the benchmark of both solutions to have concrete evidence to share.

**See you | Bis geschwënn | Até mais | À bientôt**
