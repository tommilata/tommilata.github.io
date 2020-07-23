---
title: 'Manual Dependency Injection in Play 2 Scala Applications'
tags: [dependency injection, scala, play]
---

Using [Guice](https://github.com/google/guice) to wire application objects together is a default choice in [Play framework](https://www.playframework.com/). Runtime dependency injection frameworks are widely used in the Java server world. They can save developers a lot of work by automatically constructing object dependency graphs and managing lifecycle of objects. This is particularly useful for large, enterprise, applications and makes a lot of sense for big monolithic apps. Play, being popular among both Java and Scala developers, adopts this approach too. Although Guice is relatively lightweight compared to Java EE CDI or Spring, I would argue that in many Play 2 applications, it is better not to use any DI framework at all.

Our [Faculty platform](https://faculty.ai/products-services/platform/) consists of many backend microservices written in Scala and Play 2. They are mostly stateless. We do not need any complicated lifecycle. The apps usually load their configuration once at startup, create one instance for each business class (controllers, services, repositories, ...) and wire these classes together. And these objects exist until the app is shut down.

## Difficulties with Guice

I often found myself in a situation when I had just finished implementing a new class, all unit tests were passing, deployed my changes to a dev environment, only to find my app did not start because of a single `@ImplementedBy` annotation missing. I could write integration tests to check correct wiring, but the point is we do not need to wait until runtime. We can let the compiler do it. And fail faster.

### Lazy Singletons

In our codebase, we sometimes need cloud-provider-specific implementations of the same trait. E.g. a client that talks to S3 when deployed onto AWS or to Google Cloud Storage on GCP.

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

Now, what if we

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

_Note: To save you from unpleasant surprises and wondering why your code works locally but breaks in prod, see the [the Guice source code](https://github.com/google/guice/blob/11667ab03d90e0b90d7d2a60694e1a3d0eed458e/core/src/com/google/inject/internal/Scoping.java#L2420). Singletons can be lazy in the development mode but are eager in production._

## Manual Injection Using Constructors

At Faculty, we pass dependencies using plain old constructors. E.g. if a `UserController` depends on a `UserService`, we declare the controller simply as

```scala
class UserController(userService: UserService)
```

There's usually no need to have a separate `trait` and its `class` implementation. We only use it sometimes when there's a more complex instantiation logic, not directly related to the business logic. E.g. when we need to read parameters from configuration to instantiate the class, such as a default page size for pagination. In that case, we would define

```scala
trait UserController {
  protected def defaultPageSize: Int
}

class UserControllerImpl(
  protected override val userService: UserService,
  configuration: Configuration
) {
  override protected def defaultPageSize = configuration.get[Int]("defaultPageSize")
}
```

_Note: Decoupling the business logic in the `UserController` trait from loading the configuration in the `Impl` class makes it e.g. easier to unit-test the trait._

Instantiation of the controller from an existing instance of the service and configuration is then as simple as

```scala
val configuration: Configuration = ???
val userService: UserService = ???

val userController: UserController = new UserControllerImpl(userService, configuration)
```

### Structure of a Play 2 App

To manually build an object tree for a Play 2 app, the gist is subclassing the [`ApplicationLoader`](https://www.playframework.com/documentation/2.8.x/api/scala/play/api/ApplicationLoader) and instantiate all your classes in there. The root of the hiearchy of dependencies is a `Router` which depends on controllers. Controllers typically depend on other business classes that are created from Play-provided building blocks like classes for database access, configuration or HTTP clients. Many of these can be obtained from [`BuiltInComponentsFromContext`](https://www.playframework.com/documentation/2.8.x/api/scala/play/api/BuiltInComponentsFromContext.html).

The official Play documentation provides a nice [guide with examples](https://www.playframework.com/documentation/2.8.x/ScalaCompileTimeDependencyInjection#Application-entry-point) on this.

To see actual, runnable, code, try my [repository with an example TODO](https://github.com/tomas-milata/???).

I will discuss specifics in the [next post in this series TODO](???).

## Pros & Cons

To summarise, if you choose to go with Guice in Play 2 apps, you will benefit from many working examples online and good documentation. It's a default choice in Play 2 and an easy start. In simple scenarios, all dependencies are wired together by the framework for free. It also provides support for more complicated lifecycles, e.g. adding [shutdown hooks](https://www.playframework.com/documentation/2.8.x/ScalaDependencyInjection#Stopping/cleaning-up). However, in some scenarios it lacks flexibility, e.g. for _lazy singletons_, forcing developers to use uninitutive workarounds and inceasing code complexity unnecessarily. Also, the chance of getting runtime errors is higher due to the fact that injection does not happen at compilation time.

Choosing manual compile-time DI will probably lead to fewer runtime errors. Your application's lifecycle is likely to get easier to reason about. Questions like how many instances of a controller are created can be anwered simly by just looking at your code -- no need to dig into library code. And conditional injection can be as simple as an _if/else_ statement. There will be a little bit of extra boilerplate code to write. And you will find fewer examples online, especially around resources provided by Play, like database access or HTTP clients. You will lose support for advanced application lifecycle, but will get more flexibility to tweak it yourself. Fewer dependencies is always better. Overall, using manual compile-time DI has some downsides, but we a Faculty believe they are outweighed by the understandability and straightforwardness of this approach.
