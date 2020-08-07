---
title: 'Manual dependency injection in Scala and Play 2'
tags: [dependency injection, scala, play]
---

Using [Guice](https://github.com/google/guice) to wire application objects together is ubiquitous in the [Play framework](https://www.playframework.com/). In fact, Play is probably the main reason Scala developers are familiar with Guice. Runtime dependency injection frameworks such as Guice are widely used in the Java server world. They can save developers a lot of work by automatically constructing object dependency graphs and managing the lifecycle of objects. This is particularly useful for large enterprise applications and makes a lot of sense for big monolithic apps. Play, being popular among both Java and Scala developers, adopts this approach too. Although Guice is relatively lightweight compared to Java EE CDI or Spring, I would argue that in many Play 2 applications, it is better not to use any DI framework at all.

*Note: To see this process in action with real, runnable code, take a look at [this repository](https://github.com/tomas-milata/play-without-guice).*

Our data science workbench, [Faculty Platform](https://faculty.ai/products-services/platform/), consists of many backend microservices written in Scala and Play 2. They are mostly stateless and contain at most twenty classes. We don’t need any complicated life cycle. The apps usually load their configuration once at startup, create one instance for each business class (controllers, services, repositories etc) and wire these classes together; these objects exist until the app is shut down.

## Difficulties with Guice

### Errors at runtime

We often find ourselves in a situation where we have just finished implementing a new class, all unit tests are passing, and we’ve deployed changes to a dev environment – only to find that our app doesn’t start because of a single missing `@ImplementedBy` annotation. Of course, we could write integration tests to check that wiring of dependencies is correct, but this isn’t necessary; we don’t need to wait until runtime to catch errors. We can let the compiler do it and fail faster.

### Conditional dependencies and lazy singletons

Aside from the possibility of unexpected failures at runtime, Guice can also make it difficult to wire together applications with conditional dependencies. In our codebase, we sometimes need cloud-provider-specific implementations of the same trait. For example, we might need a client that talks to S3 when deployed to AWS or to Google Cloud Storage on GCP.

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

Now, what if we always want just 1 instance of the `StorageClient` (e.g. to keep some in-memory state) and don’t want to instantiate the `S3Client` when on GCP – and vice versa? For example, the `S3Client` might depend on configuration like reading `AWS_ACCESS_KEY_ID` from the environment, which is simply not present when on GCP.

We can’t use [`@Singleton` annotation](https://google.github.io/guice/api-docs/latest/javadoc/index.html?com/google/inject/Singleton.html) directly, because these singletons are eager, which means both clients would be instantiated at startup and crash the app. The only workaround we found (after many hours of frustration) to make singletons lazy was based on this subtle note found in [Guice docs](https://github.com/google/guice/wiki/Scopes#eager-singletons):

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

Even though singletons are marked eager here, they are instantiated lazily, because the conditional binding causes that Guice doesn’t know about them.

*Note: To save you from unpleasant surprises and wondering why your code works locally but breaks in prod, see the [Guice source code](https://github.com/google/guice/blob/11667ab03d90e0b90d7d2a60694e1a3d0eed458e/core/src/com/google/inject/internal/Scoping.java#L2420). Singletons can be lazy in development mode but eager in production.*

## Manual injection using constructors

Because of the overhead Guice introduces and the problems described above, we have decided to replace Guice with plain old constructors to pass dependencies around For example, if a `UserController` depends on a `UserService`, we declare the controller simply as:

```scala
class UserController(userService: UserService)
```

There’s usually no need to have a separate `trait` and its `class` implementation. We only use this approach when there’s a more complex instantiation logic that’s not directly related to the business logic – eg when we need to read parameters from configuration to instantiate the class, such as a default page size for pagination. In that case, we would define:

```scala
trait UserController {
  protected def defaultPageSize: Int
}

class UserControllerImpl(
  protected override val userService: UserService,
  configuration: Configuration
) {
  override protected def defaultPageSize =
    configuration.get[Int]("defaultPageSize")
}
```

*Note: Decoupling the business logic in the `UserController` trait from loading the configuration in the `Impl` class makes it easier to unit test the trait.*

Instantiation of the controller from an existing instance of the service and configuration is then as simple as:

```scala
val configuration: Configuration = ???
val userService: UserService = ???

val userController: UserController =
  new UserControllerImpl(userService, configuration)
```

## Structure of a Play 2 app

When Guice or no other DI frameworks are used in a Play 2 app, one needs to build an object tree manually. The gist is subclassing the [`ApplicationLoader`](https://www.playframework.com/documentation/2.8.x/api/scala/play/api/ApplicationLoader) and instantiating all your classes in there. The root of the hierarchy of dependencies is a Router which depends on controllers. Controllers typically depend on other business classes that are created from Play-provided building blocks like classes for database access, configuration or HTTP clients. Many of these can be obtained from [`BuiltInComponentsFromContext`](https://www.playframework.com/documentation/2.8.x/api/scala/play/api/BuiltInComponentsFromContext.html).

The official Play documentation provides a nice [guide with examples](https://www.playframework.com/documentation/2.8.x/ScalaCompileTimeDependencyInjection#Application-entry-point) on this. To see actual, runnable code for this, have a look at my [repository with an example](https://github.com/tomas-milata/play-without-guice). I will discuss the specifics of this example in my next post.

## Pros and cons of this approach

If you do choose to go with Guice in Play 2 apps, you will benefit from good documentation and the many working examples online. It’s a default choice in Play 2 and an easy start.

In simple scenarios, all dependencies are wired together by the framework for free. It also provides support for more complicated life cycles, such as adding [shutdown hooks](https://www.playframework.com/documentation/2.8.x/ScalaDependencyInjection#Stopping/cleaning-up). However, in some scenarios it lacks flexibility, as in the case of *lazy singletons*, forcing developers to use unintuitive workarounds which increase code complexity. The chance of getting runtime errors is also higher, because injection doesn’t happen at compilation time.

Choosing manual compile-time DI will probably lead to fewer runtime errors and make your application’s lifecycle easier to understand. Questions like how many instances of a controller are created can be answered by just looking at your code, with no need to dig into library code. Conditional injection can be as simple as an if/else statement.

On the other hand, there will be a little bit of extra boilerplate code to write and you will find fewer examples online, especially around resources provided by Play like database access or HTTP clients. You will lose support for an advanced application lifecycle, but will get more flexibility to tweak it yourself. Fewer dependencies is better.

Overall, using manual compile-time DI has some downsides, but we at Faculty believe they are outweighed by the understandability and straightforwardness of this approach.

*This article was also [posted on our company's tech blog](https://faculty.ai/blog/manual-dependency-injection-in-scala-and-play-2).*
