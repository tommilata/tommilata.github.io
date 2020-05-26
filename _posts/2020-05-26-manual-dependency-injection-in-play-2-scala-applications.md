---
title: '[WIP] Manual Dependency Injection in Play 2 Scala Applications'
tags: [dependency injection, scala, play]
---

Using [Guice](https://github.com/google/guice) to wire application objects together is a default choice in [Play framework](https://www.playframework.com/). Runtime dependency injection frameworks are widely used in Java server world. They can save developers a lot of work by automatically constructing object dependency graphs and managing lifecycle of objects. This is particularly useful in large, enterprise, applications and made a lot of sense for big monolithic apps. Unsurprisingly, Play adopts this approach too. Although Guice is relatively lightweight compared to Java EE CDI or Spring, I would argue in many Play 2 applications it is better not to use any DI framework at all.

Our [Faculty Platform](https://faculty.ai/products-services/platform/) consists of many backend microservices written in Scala and Play. We do not need any complicated lifecycle. They are mostly stateless. The app usually loads its configration once at startup, creates one instance for each business class (controllers, services, repositories, ...) and wires them together. And these objects exist until the app is shut down.

## Pros & Cons

- 
| +                            | -                |
| ---------------------------- | ---------------- |
| explicit number of instances | no session scope |
|                              |                  |



## Structure

- conf
- Loader
- Components
    - 
- DB
- Configuration
- Controllers
- Router 
- 
## Example


## Conclusion
