---
title: '[WIP] Manual Dependency Injection in Play 2 Scala Applications'
tags: [dependency injection, scala, play]
---

Using [Guice](https://github.com/google/guice) to wire application objects together is a default choice in [Play framework](https://www.playframework.com/). Runtime dependency injection frameworks are widely used in Java server world. They can save developers a lot of work by automatically constructing object dependency graphs and managing lifecycle of objects. This is particularly useful in large, enterprise, applications and makes a lot of sense for big monolithic apps. Unsurprisingly, Play adopts this approach too. Although Guice is relatively lightweight compared to Java EE CDI or Spring, I would argue in many Play 2 applications, it is better not to use any DI framework at all.

Our [Faculty Platform](https://faculty.ai/products-services/platform/) consists of many backend microservices written in Scala and Play. They are mostly stateless.  We do not need any complicated lifecycle. The app usually loads its configration once at startup, creates one instance for each business class (controllers, services, repositories, ...) and wires them together. And these objects exist until the app is shut down.

## Difficulties with Guice

I often found myself in a situation where I had just finished implementing a new class, all unit tests were passing, deployed my changes to a dev environment, only to find my app did not start because of a single `@ImplementedBy` annotation missing. I could have written integration tests to check correct wiring, but the point is we do not need to wait for runtime. We can let the compiler do it and fail faster.

### Lazy Singletons

Sometimes we have cloud-provider-specific implementations of the same trait. E.g. a client that talks to S3 when deployed onto AWS and to Google Cloud Storage on GCP.

_TODO example StorageClient, S3Client, GcsClient ?_

```scala
trait StorageClient {
  def listBucket: List[String]
}

class S3Client extends StorageClient {
  override def listBucket: List[String] = ???
}

class GcsClient extends StorageClient {
  override def listBucket: List[String] = ???
}
```

What if we

- always want just 1 instance of `StorageClient` (e.g. to keep some in-memory state) and
- don't want to instantiate the `S3Client` when on GCP and vice versa? E.g. because the `S3Client` depends on configuration like reading `AWS_ACCESS_KEY_ID` from the environment which is simly not present when on GCP.

We cannot use [`@Singleton` annotation](https://google.github.io/guice/api-docs/latest/javadoc/index.html?com/google/inject/Singleton.html) directly because these singletons are _eager_, which means both clients would be instantiated at startup, crashing the app.

The only workaround we found (after many hours of frustration) to make singletons lazy was based on this subtle note found in [Guice docs](https://github.com/google/guice/wiki/Scopes#eager-singletons):

> Guice will only eagerly build singletons for the types it knows about. These are the types mentioned in your modules, plus the transitive dependencies of those types.

The trick is to [create a custom Guice module](https://www.playframework.com/documentation/2.6.x/ScalaPlayModules) with a conditional binding:

```scala
cloudProvider match {
  case "AWS" =>
    bind(classOf[StorageClient]).to(classOf[S3Client]).asEagerSingleton()
  case "GCP" =>
    bind(classOf[StorageClient]).to(classOf[GcsClient]).asEagerSingleton()
}
```

Even though singletons are marked eager here, they are instantiated lazily because Guice does not know about them (due to the conditional binding).

_Note: To save you from wondering why your code works locally but breaks in prod, banging your head against the wall, see the [the Guice source code](https://github.com/google/guice/blob/11667ab03d90e0b90d7d2a60694e1a3d0eed458e/core/src/com/google/inject/internal/Scoping.java#L2420). Singletons can be lazy in the development mode but are eager in production._

## Manual Injection Using Constructors

### Structure

- conf
- Loader
- Components
    - 
- DB
- Configuration
- Controllers
- Router 


## Example



## Pros & Cons


problems

- runtime failure - missing @ImplementedBy


- runtime errors, typically missing `@ImplementedBy` — need to redeploy



- Guice complexity
- easy to follow application lifecycle, e.g.
    - How many instances of each controller, service are created?
- no Guice support for lazy singletons
    - only hack with custom module
    - spent ~2 MD figuring this out
- less dependencies
- simplify code - no need for separate `trait` + `Impl` in many cases
- easy custom wiring based on e.g. `CloudProvider`


- Cons
  - no session scope
    - write constructors manually
        - ~ 100 lines in `ivory`, ~200 lines in `steve`
        - how to get things like `Database`, `WSClient`, `SecurityFilter` s or `ExecutionContext` from Play
        - breaking changes in `sherlockml-base`


## Conclusion
