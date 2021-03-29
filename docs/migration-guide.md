---
id: migration-guide
title: Migration guide
---

## Summary

Here is a general view of the steps you should take to migrate your application to Cats Effect 3:

1. [Make sure your dependencies have upgraded](#make-sure-your-dependencies-have-upgraded)<!-- don't remove this comment - this ensures the vscode extension doesn't make this a ToC -->
1. [Run the Scalafix migration](#run-the-scalafix-migration) (optional)
1. [Upgrade dependencies and Cats Effect itself](#upgrade-dependencies)
1. [Fix remaining compilation issues](#fix-remaining-compilation-issues)
1. [Test your application](#test-your-application).

### Before you begin: this isn't a "quick start" guide

This guide is meant for existing users of Cats Effect 2 who want to upgrade their applications
to 3.0.0.

> If you haven't used Cats Effect before and want to give it a try,
> please follow the [getting started guide](./getting-started.md) instead!

### 🤔 Need help?

If any point of the migration turns out to be difficult and you feel like you need help, feel free to [explain your problem on Gitter](https://gitter.im/typelevel/cats-effect) and we will do our best to assist you.
If you spot a mistake in the guide or the library itself, you can [report an issue on GitHub](https://github.com/typelevel/cats-effect/issues/new).

### Context: what's changed, what's the same?

Cats Effect 3 (CE3 for short) is a complete redesign of the library.
Some abstractions known from Cats Effect 2 (CE2) have been removed, others changed responsibilities, and finally, new abstractions were introduced.

The `cats.effect.IO` type known from CE2 is still there, albeit with a different place in the type class hierarchy - namely, it doesn't appear in it.

The new set of type classes has been designed in a way that deliberately avoids coupling with `IO`, which makes the library more modular,
and allows library authors (as well as users of other effect types) to omit that dependency from their builds.

## Make sure your dependencies have upgraded

Before you make any changes to your build or your code, you should make sure all of your direct and transitive dependencies have made releases compatible with Cats Effect 3.

There isn't any automated way to do this, but you can just go ahead and [try to upgrade the dependencies](#upgrade-dependencies), then stash the changes and return to here.

If you're using an open source library that hasn't made a compatible release yet, [let us know - we are keeping track of the efforts of library authors](https://github.com/typelevel/cats-effect/issues/1330) to publish compatible releases as soon as possible when 3.0.0 final is out.

## Run the Scalafix migration

Many parts of this migration can be automated by using the [Scalafix][scalafix] migration.

> Note: In case of projects using Scala Steward, the migration should automatically be applied when you receive the update.

If you want to trigger the migration manually, you can follow [the instructions here](https://github.com/typelevel/cats-effect/blob/series/3.x/scalafix/README.md). Remember to run it *before* making any changes to your dependencies' versions.

Now is the time to update `cats-effect` and **every dependency using it** to a CE3-compatible version.

## Upgrade dependencies

At this point, if you've run the Scalafix migration, your code will not compile. However, you should hold off going through the list of errors and fixing the remaining issues yourself at this point.

If you're an [sbt][sbt] user, it is recommended that you upgrade to at least `1.5.0-RC2` before you proceed:

In your `project/build.properties`:

```diff
- sbt.version = 1.4.9
+ sbt.version = 1.5.0-RC2
```

This will enable eviction errors, which means your build will only succeed if all your dependencies
use compatible versions of each library (in the case of `cats-effect`, this will require your dependencies
all use either the 2.x.x versions or the 3.x.x versions).

Having upgraded sbt, you can try to upgrade Cats Effect:

### Which modules should I use?

Cats Effect 3 splits the code dependency into multiple modules. If you were previously using `cats-effect`, you can keep doing so, but if you're a user of another effect system (Monix, ZIO, ...), or a library author, you might be able to depend on a subset of it instead.

The current non-test modules are:

```scala
"org.typelevel" %% "cats-effect-kernel" % "3.0.0",
"org.typelevel" %% "cats-effect-std"    % "3.0.0",
"org.typelevel" %% "cats-effect"        % "3.0.0",
```

- `kernel` - type class definitions, simple concurrency primitives
- `std` - high-level abstractions like `Console`, `Semaphore`, `Hotswap`, `Dispatcher`
- `core` - `IO`, `SyncIO`

```diff
libraryDependencies ++= Seq(
  //...
-  "org.typelevel" %% "cats-effect" % "2.4.0",
+  "org.typelevel" %% "cats-effect" % "3.0.0",
  //...
)
```

Then run `update` or `evicted` in sbt. You should see something like the following:

```scala
sbt:demo> update
[error] stack trace is suppressed; run last core / update for the full output
[error] (core / update) found version conflict(s) in library dependencies; some are suspected to be binary incompatible:
[error]
[error] 	* org.typelevel:cats-effect_2.13:3.0.0 (early-semver) is selected over {2.3.1, 2.1.4}
[error] 	    +- com.example:core-core_2.13:0.0.7-26-3183519d       (depends on 3.0.0)
[error] 	    +- io.monix:monix-catnap_2.13:3.3.0                   (depends on 2.1.4)
[error] 	    +- com.github.valskalla:odin-core_2.13:0.11.0         (depends on 2.3.1)
[error]
[error]
[error] this can be overridden using libraryDependencySchemes or evictionErrorLevel
[error] Total time: 0 s, completed 27 Mar 2021, 17:51:52
```

This tells you that you need to upgrade both `monix-catnap` and `odin-core` before proceeding. Make sure `update` of all your project's modules passes before proceeding to the next point.

> Note: that some of the libraries listed might be transitive dependencies, which means
> you're depending on other projects that depend on them.
> Upgrading your direct dependencies should solve the transitive dependencies' incompatibilities as well.

## Fix remaining compilation issues

Here's the new type class hierarchy. It might be helpful in understanding some of the changes:

<a href="https://raw.githubusercontent.com/typelevel/cats-effect/series/3.x/images/hierarchy.svg" target="_blank">
  <img src="https://raw.githubusercontent.com/typelevel/cats-effect/series/3.x/images/hierarchy.svg" alt="hierarchy" style="margin: 30px 0"/>
</a>

Most of the following are handled by [the Scalafix migration](#run-the-scalafix-migration). If you can, try that first!

> Note: package name changes were skipped from this guide. Most type classes are now in the `cats.effect.kernel` package,
> but you can access them through `cats.effect.<the typeclass>` too thanks to type aliases.

### Async

| Cats Effect 2.x                   | Cats Effect 3                                      |
| --------------------------------- | -------------------------------------------------- |
| `Async[F].async`                  | `Async[F].async_`                                  |
| `Async[F].asyncF(f)`              | `Async[F].async(f(_).as(none))`                    |
| `Async.shift`                     | nothing / `Spawn[F].cede` - See [below](#shifting) |
| `Async.fromFuture`                | `Async[F].fromFuture`                              |
| `Async.memoize`                   | `Concurrent[F].memoize`                            |
| `Async.parTraverseN`              | `Concurrent[F].parTraverseN`                       |
| `Async.parSequenceN`              | `Concurrent[F].parSequenceN`                       |
| `Async[F].liftIO`, `Async.liftIO` | `LiftIO[F].liftIO`                                 |
| `Async <: LiftIO`                 | No subtyping relationship                          |

#### `async` signature

In Cats Effect 2, `Async` was able to lift asynchronous effects into `F`. `Concurrent` extended that ability
to allow canceling them - in CE3,
these two capabilities have been merged into `Async`, so all `Async` types
have to be cancelable to pass the laws.

A major outcome of this decision is that the `async` method supports cancelation. Here's the new signature:

```scala mdoc
trait Async[F[_]] {
  def async[A](k: (Either[Throwable, A] => Unit) => F[Option[F[Unit]]]): F[A]
}
```

We can divide the parameter `k` into the following:

- `Either[Throwable, A] => Unit` - the callback that will complete or fail this effect when called. This is identical as in CE2.
- `=> F[...]` (outer effect) - the effect of registering the callback. This would be e.g. `delay { window.setTimeout(() => cb(...)) }`.
- `Option[F[Unit]]` - an optional effect that will run if the action is canceled. Passing `None` here is equivalent to `Some(F.unit)`

The most similar method to this in CE2 would be `Concurrent.cancelableF`:

```scala mdoc
// CE2!
def cancelableF[F[_], A](k: (Either[Throwable, A] => Unit) => F[F[Unit]]) = ???
```

The only difference being that there was always an effect for cancelation - now it's optional (the `F` inside `F`).

#### Relationship with `LiftIO`

`LiftIO` is no longer part of the "kernel" type classes (being unlawful and strictly coupled to `IO`, which isn't part of `kernel` either), and has been moved to the `core` module (the `cats-effect` dependency).

`Async` is in `kernel` (`cats-effect-kernel`), so it can't depend on `LiftIO`, and thus doesn't extend it.
If you need `LiftIO` functionality, use it directly (or use the methods on `IO` like `to[F[_]: LiftIO]`).

#### Implementing Async

Types that used to implement `Async` but not `Concurrent` from CE2 might not be able to implement `Async` in CE3 -
this has an impact on users who have used polymorphic effects with an `F[_]: Async` constraint in their applications, where `F` was one of:

- [doobie](https://tpolecat.github.io/doobie/)'s `ConnectionIO`,
- [slick](http://scala-slick.org/)'s `DBIO` with [slick-effect](https://github.com/kubukoz/slick-effect),
- [ciris](https://cir.is)'s `ConfigValue`, or
- possibly others that we don't know of

Please refer to each library's appropriate documentation/changelog to see how to adjust your code to this change.

### Blocker

| Cats Effect 2.x           | Cats Effect 3                                                  | Notes                                   |
| ------------------------- | -------------------------------------------------------------- | --------------------------------------- |
| `Blocker.apply`           | -                                                              | blocking pool is provided by runtime    |
| `Blocker.delay`           | `Sync[F].blocking`                                             | `Blocker` was removed                   |
| `Blocker(ec).blockOn(fa)` | `Async[F].evalOn(fa, ec)`                                      | You can probably use `Sync[F].blocking` |
| `Blocker.blockOnK`        | For blocking actions on a specific pool, use `Async[F].evalOn` |                                         |

<!-- todo notes -->

### Bracket

| Cats Effect 2.x               | Cats Effect 3                          | Notes                       |
| ----------------------------- | -------------------------------------- | --------------------------- |
| `Bracket[F].bracket`          | `MonadCancel[F].bracket`               |                             |
| `Bracket[F].bracketCase`      | `MonadCancel[F].bracketCase`           | `ExitCase` is now `Outcome` |
| `Bracket[F].uncancelable(fa)` | `MonadCancel[F].uncancelable(_ => fa)` |                             |
| `Bracket[F].guarantee`        | `MonadCancel[F].guarantee`             |                             |
| `Bracket[F].guaranteeCase`    | `MonadCancel[F].guaranteeCase`         | `ExitCase` is now `Outcome` |
| `Bracket[F].onCancel`         | `MonadCancel[F].onCancel`              |                             |

### Clock

| Cats Effect 2.x                           | Cats Effect 3                           |
| ----------------------------------------- | --------------------------------------- |
| `Clock[F].realTime: TimeUnit => F[Long]`  | `Clock[F].realTime: F[FiniteDuration]`  |
| `Clock[F].monotonic: TimeUnit => F[Long]` | `Clock[F].monotonic: F[FiniteDuration]` |
| `Clock.instantNow`                        | `Clock[F].realTimeInstant`              |
| `Clock.create`, `Clock[F].mapK`           | -                                       |

<!-- todo why `create` and mapK are gone (because it's a typeclass now)  -->

### Concurrent

| Cats Effect 2.x                             | Cats Effect 3                   | Notes                                          |
| ------------------------------------------- | ------------------------------- | ---------------------------------------------- |
| `Concurrent[F].start`                       | `Spawn[F].start`                |                                                |
| `Concurrent[F].background`                  | `Spawn[F].background`           | Value in resource is now an `Outcome`          |
| `Concurrent[F].liftIO`, `Concurrent.liftIO` | `LiftIO[F].liftIO`              | `LiftIO` is in the `cats-effect` module        |
| `Concurrent <: LiftIO`                      | No subtyping relationship       | `LiftIO` is in the `cats-effect` module        |
| `Concurrent[F].race`                        | `Spawn[F].race`                 |                                                |
| `Concurrent[F].racePair`                    | `Spawn[F].racePair`             |                                                |
| `Concurrent[F].cancelable`                  | `Async.async`                   | Wrap side effects in F, cancel token in `Some` |
| `Concurrent[F].cancelableF`                 | `Async.async(f(_).map(_.some))` | `Some` means there is a finalizer to execute.  |
| `Concurrent[F].continual`                   | -                               | see [below](#continual)                        |
| `Concurrent.continual`                      | -                               | see [below](#continual)                        |
| `Concurrent.timeout`                        | `Temporal[F].timeout`           |                                                |
| `Concurrent.timeoutTo`                      | `Temporal[F].timeoutTo`         |                                                |
| `Concurrent.memoize`                        | `Concurrent[F].memoize`         |                                                |
| `Concurrent.parTraverseN`                   | `Concurrent[F].parTraverseN`    |                                                |
| `Concurrent.parSequenceN`                   | `Concurrent[F].parSequenceN`    |                                                |


#### continual

<!-- todo explain -->

```scala mdoc:reset
import cats.effect.kernel.MonadCancel
import cats.effect.kernel.MonadCancelThrow
import cats.syntax.all._

def continual[F[_]: MonadCancelThrow, A, B](fa: F[A])(
  f: Either[Throwable, A] => F[B]
): F[B] = MonadCancel[F].uncancelable { poll =>
  poll(fa).attempt.flatMap(f)
}
```

<!-- todo -->
### Effect, ConcurrentEffect, SyncEffect

| Cats Effect 2.x       | Cats Effect 3 |
| --------------------- | ------------- |
| `ConcurrentEffect[F]` | `Dispatcher`  |
| `Effect[F]`           | `Dispatcher`  |
| `Effect.toIOK`        | `Dispatcher`  |
| `SyncEffect[F]`       | `Dispatcher`  |


#### Dispatcher

todo - Gavin wrote about this

<!-- todo -->

### ContextShift

| Cats Effect 2.x          | Cats Effect 3             |
| ------------------------ | ------------------------- |
| `ContextShift[F].shift`  | nothing / `Spawn[F].cede` |
| `ContextShift[F].evalOn` | `Async[F].evalOn`         |

<!-- todo CS is gone! 🦀🌴🦀🌴🦀🌴🦀🌴🦀🌴🦀🌴🦀🌴🦀🌴🦀🌴🦀🌴🦀🌴🦀🌴🦀🌴🦀🌴🦀🌴 -->

#### Shifting

The `IO.shift` / `ContextShift[F].shift` methods are gone, and they don't have a fully compatible counterpart.

In CE2, `shift` would ensure the next actions in the fiber would be scheduled on the `ExecutionContext` instance (or the `ContextShift` instance) provided in the parameter.
This was used for two reasons:

- to switch back from a thread pool not managed by the effect system (e.g. a callback handler in a Java HTTP client)
- to reschedule the fiber on the given `ExecutionContext`, which would give other fibers a chance to run on that context's threads. This is called yielding to the scheduler.

There is no longer a need for shifting back, because interop with callback-based libraries is done through methods in `Async`, which now **switch back to the appropriate thread pool automatically**.

Yielding back to the scheduler can now be done with `Spawn[F].cede`.

### ExitCase, Fiber

| Cats Effect 2.x          | Cats Effect 3                  |
| ------------------------ | ------------------------------ |
| `ExitCase[E]`            | `Outcome[F, E, A]`             |
| `Fiber[F, A]`            | `Fiber[F, E, A]`               |
| `Fiber[F, A].join: F[A]` | `Fiber[F, E, A]`.joinWithNever |

#### Outcome

<!-- todo -->

### Sync

| Cats Effect 2.x   | Cats Effect 3   |
| ----------------- | --------------- |
| `Sync[F].suspend` | `Sync[F].defer` |

### IO

| Cats Effect 2.x                   | Cats Effect 3                                  | Notes                                                  |
| --------------------------------- | ---------------------------------------------- | ------------------------------------------------------ |
| `IO#as`                           | `IO.as` / `IO.map`                             | the argument isn't by-name anymore                     | <!-- todo https://github.com/typelevel/cats-effect/issues/1824 --> |
| `IO.runAsync`, `IO.runCancelable` | Unsafe variants or [`Dispatcher`](#dispatcher) |                                                        |
| `IO.unsafe*`                      | The same or [`Dispatcher`](#dispatcher)        | Methods that run an IO require an implicit `IORuntime` |
| `IO.unsafeRunAsyncAndForget`      | `IO.unsafeRunAndForget`                        |                                                        |
| `IO.unsafeRunCancelable`          | `start.unsafeRunSync.cancel`                   |                                                        |
| `IO.unsafeRunTimed`               | -                                              |                                                        |
| `IO.background`                   | The same                                       | Value in resource is now an `Outcome`                  |
| `IO.guaranteeCase`/`bracketCase`  | The same                                       | [`ExitCase` is now `Outcome`](#outcome)                |
| `IO.parProduct`                   | `IO.both`                                      |                                                        |
| `IO.suspend`                      | `IO.defer`                                     |                                                        |
| `IO.shift`                        | See [below](#shifting)                         |                                                        |
| `IO.cancelBoundary`               | `IO.cede`                                      | Also [performs a yield](#shifting)                     |

### Resource

| Cats Effect 2.x                      | Cats Effect 3                | Notes                                                    |
| ------------------------------------ | ---------------------------- | -------------------------------------------------------- |
| `Resource.parZip`                    | `Resource.both`              |                                                          |
| `Resource.liftF`                     | `Resource.eval`              |                                                          |
| `Resource.fromAutoCloseableBlocking` | `Resource.fromAutoCloseable` | The method always uses `blocking` for the cleanup action |

### Timer

| Cats Effect 2.x  | Cats Effect 3       | Notes |
| ---------------- | ------------------- | ----- |
| `Timer[F].clock` | `Clock[F]`          |       |
| `Timer[F].sleep` | `Temporal[F].sleep` |       |

### TODO: IOApp

### Tracing

### Deferred

<!-- todo - complete doesn't fail anymore -->

<!-- also check if https://github.com/scala-steward-org/scala-steward/pull/1940 has anything we don't -->

<!--  -->

Currently, improved stack traces are not implemented.
There is currently [work in progress](https://github.com/typelevel/cats-effect/pull/1763) to bring them back.

## Test your application

If you followed this guide, all your dependencies are using the 3.x releases of Cats Effect, your code compiles and your tests pass,
the process is probably done - at this point you should do the usual steps you make after major changes in your application:
running integration/end to end tests, manual testing, canary deployments and any other steps that might
typically be done in your environment.

Enjoy using Cats Effect 3!

[sbt]: https://scala-sbt.org
[scalafix]: https://scalacenter.github.io/scalafix/