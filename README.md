
# Code Review Checklist: Java Concurrency

Design
 - [Concurrency is rationalized?](#rationalize)
 - [Can use patterns to simplify concurrency?](#use-patterns)
   - Immutability/Snapshotting
   - Divide and conquer
   - Producer-consumer
   - Instance confinement
   - Thread/Task/Serial thread confinement
   - Active object

Documentation
 - [Thread safety is justified in comments?](#justify-document)
 - [Class (method, field) has concurrent access documentation?](#justify-document)
 - [Threading model of a subsystem (class) is described?](#threading-flow-model)
 - [Concurrent control flow (or data flow) of a subsystem (class) is described?
 ](#threading-flow-model)
 - [Class is documented as immutable, thread-safe, or not thread-safe?](#immutable-thread-safe)
 - [Used concurrency patterns are pronounced?](#name-patterns)
 - [`ConcurrentHashMap` is *not* stored in a variable of `Map` type?](#concurrent-map-type)
 - [`compute()`-like methods are *not* called on a variable of `ConcurrentMap` type?](#chm-type)
 - [`@GuardedBy` annotation is used?](#guarded-by)
 - [Safety of a benign race (e. g. unbalanced synchronization) is explained?](#document-benign-race)
 - [Each use of `volatile` is justified?](#justify-volatile)
 - [Each field that is neither `volatile` nor annotated with `@GuardedBy` has a comment?
 ](#plain-field)

Insufficient synchronization
 - [Static methods and fields are thread-safe?](#static-thread-safe)
 - [Thread *doesn't* wait in a loop for a non-volatile field to be updated by another thread?
 ](#non-volatile-visibility)
 - [Read access to a non-volatile, concurrently updatable primitive field is protected?
 ](#non-volatile-protection)
 - [Servlets, Controllers, Filters, Handlers, `@Get`/`@Post` methods are thread-safe?
 ](#server-framework-sync)
 - [Calls to `DateFormat.parse()` and `format()` are synchronized?](#dateformat)

Excessive thread safety
 - [No "extra" (pseudo) thread safety?](#pseudo-safety)
 - [No atomics on which only `get()` and `set()` are called?](#redundant-atomics)
 - [Class (method) needs to be thread-safe?](#unneeded-thread-safety)
 - [`ReentrantLock` (`ReentrantReadWriteLock`, `Semaphore`) needs to be fair?](#unneeded-fairness)

Race conditions
 - [No `put()` or `remove()` calls on a `ConcurrentMap` (or Cache) after `get()` or
 `containsKey()`?](#chm-race)
 - [No point accesses to a non-thread-safe collection outside of critical sections?
 ](#unsafe-concurrent-point-read)
 - [Iteration over a non-thread-safe collection doesn't leak outside of a critical section?
 ](#unsafe-concurrent-iteration)
 - [A non-thread-safe collection is *not* returned wrapped in `Collections.unmodifiable*()` from
 a getter in a thread-safe class?](#unsafe-concurrent-iteration)
 - [Non-trivial object is *not* returned from a getter in a thread-safe class?
 ](#concurrent-mutation-race)
 - [No separate getters to an atomically updated state?](#moving-state-race)
 - [No *check-then-act* race conditions (state used inside a critical section is read outside of
 it)?](#read-outside-critical-section-race)
 - [`coll.toArray(new E[coll.size()])` is *not* called on a synchronized collection?
 ](#read-outside-critical-section-race)
 - [No race conditions with user or programmatic input or interop between programs?
 ](#outside-world-race)
 - [No check-then-act race conditions with file system operations?](#outside-world-race)
 - [No concurrent `invalidate(key)` and `get()` calls  on Guava's loading `Cache`?
 ](#guava-cache-invalidation-race)
 - [`Cache.put()` is not used (nor exposed in the own Cache interface)?](#cache-invalidation-race)
 - [Concurrent invalidation race is not possible on a lazily initialized state?
 ](#cache-invalidation-race)
 - [Iteration, Stream pipeline, or copying a `Collections.synchronized*()` collection is protected
 by a lock?](#synchronized-collection-iter)

Testing
 - [Unit tests for thread-safe classes are multi-threaded?](#multi-threaded-tests)
 - [A shared `Random` instance is *not* used from concurrent test workers?](#concurrent-test-random)
 - [Concurrent test workers coordinate their start?](#coordinate-test-workers)
 - [There are more test threads than CPUs (if possible for the test)?](#test-workers-interleavings)
 - [Assertions in parallel threads and asynchronous code are handled properly?](#concurrent-assert)

Locks
 - [Can use some concurrency utility instead of a lock with conditional `wait` (`await`) calls?
 ](#avoid-wait-notify)
 - [Can use Guava’s `Monitor` instead of a lock with conditional `wait` (`await`) calls?
 ](#guava-monitor)
 - [Can use `synchronized` instead of a `ReentrantLock`?](#use-synchronized)
 - [`lock()` is called outside of `try {}`? No statements between `lock()` and `try {}`?
 ](#lock-unlock)

Avoiding deadlocks
 - [Can avoid nested critical sections?](#avoid-nested-critical-sections)
 - [Locking order for nested critical sections is documented?](#document-locking-order)
 - [Dynamically determined locks for nested critical sections are ordered?](#dynamic-lock-ordering)
 - [No extension API calls within critical sections?](#non-open-call)
 - [No calls to `ConcurrentHashMap`'s methods (incl. `get()`) in `compute()`-like lambdas on the
 same map?](#chm-nested-calls)

Improving scalability
 - [Critical section is as small as possible?](#minimize-critical-sections)
 - [Can use `ConcurrentHashMap.compute()` or Guava's `Striped` for per-key locking?
 ](#increase-locking-granularity)
 - [Can replace blocking collection or a queue with a concurrent one?](#non-blocking-collections)
 - [Can use `ClassValue` instead of `ConcurrentHashMap<Class, ...>`?](#use-class-value)
 - [Considered `ReadWriteLock` (or `StampedLock`) instead of a simple lock?](#read-write-lock)
 - [`StampedLock` is used instead of `ReadWriteLock` when reentrancy is not needed?
 ](#use-stamped-lock)
 - [Considered `LongAdder` instead of an `AtomicLong` for a "hot field"?
 ](#long-adder-for-hot-fields)
 - [Considered queues from JCTools instead of the standard concurrent queues?](#jctools)
 - [Considered Caffeine cache instead of other caching libraries?](#caffeine)
 - [Can apply speculation (optimistic concurrency) technique?](#speculation)

Lazy initialization and double-checked locking
 - [Lazy initialization of a field should be thread-safe?](#lazy-init-thread-safety)
 - [Considered double-checked locking for a lazy initialization to improve performance?
 ](#use-dcl)
 - [Double-checked locking follows the SafeLocalDCL pattern?](#safe-local-dcl)
  - [Considered eager initialization instead of a lazy initialization to simplify code?
  ](#eager-init)
 - [Can do lazy initialization with a benign race and without locking to improve performance?
 ](#lazy-init-benign-race)
 - [Holder class idiom is used for lazy static fields rather than double-checked locking?
 ](#no-static-dcl)

Non-blocking and partially blocking code
 - [Non-blocking code has enough comments to make line-by-line checking as easy as possible?
 ](#check-non-blocking-code)
 - [Can use immutable POJO + compare-and-swap operations to simplify non-blocking code?
 ](#swap-state-atomically)
 - [Boundaries of non-blocking or benignly racy code are identified with WARNING comments?
 ](#non-blocking-warning)
 - [Busy waiting (spin loop), all calls to `Thread.yield()` and `Thread.onSpinWait()` are justified?
 ](#justify-busy-wait)

Threads and Executors
 - [Thread is named?](#name-threads)
 - [Can use `ExecutorService` instead of creating a new `Thread` each time some method is called?
 ](#reuse-threads)
 - [No network I/O in a CachedThreadPool?](#cached-thread-pool-no-io)
 - [No blocking (incl. I/O) operations in a `ForkJoinPool` or in a parallel Stream pipeline?
 ](#fjp-no-blocking)
 - [Can execute non-blocking computation in `FJP.commonPool()` instead of a custom thread pool?
 ](#use-common-fjp)
 - [`ExecutorService` is shutdown explicitly?](#explicit-shutdown)
 - [Callback is attached to a `CompletableFuture` (`SettableFuture`) in non-async mode only if
 either:](#cf-beware-non-async)
   - the callback is lightweight and non-blocking; or
   - the future is completed and the callback is attached from the same thread pool?
 - [Adding a callback to a `CompletableFuture` (`SettableFuture`) in non-async mode is justified?
 ](#cf-beware-non-async)

Parallel Streams
 - [Parallel Stream computation takes more than 100us in total?](#justify-parallel-stream-use)
 - [Comment before a parallel Streams pipeline explains how it takes more than 100us in total?
 ](#justify-parallel-stream-use)
 
Thread interruption and `Future` cancellation
 - [Interruption status is restored before wrapping `InterruptedException` with another exception?
 ](#restore-interruption)
 - [`InterruptedException` is swallowed only in the following kinds of methods:
 ](#interruption-swallowing)
   - `Runnable.run()`, `Callable.call()`, or methods to be passed to executors as lambda tasks; or
   - Methods with "try" or "best effort" semantics?
 - [`InterruptedException` swallowing is documented for a method?](#interruption-swallowing)
 - [Can use Guava's `Uninterruptibles` to avoid `InterruptedException` swallowing?
 ](#interruption-swallowing)
 - [`Future` is canceled upon catching an `InterruptedException` or a `TimeoutException` on `get()`?
 ](#cancel-future)

Time
 - [`nanoTime()` values are compared in an overflow-aware manner?](#nano-time-overflow)
 - [`currentTimeMillis()` is *not* used to measure time intervals and timeouts?
 ](#time-going-backward)
 - [Units for a time variable are identified in the variable's name or via `TimeUnit`?](#time-units)
 - [Negative timeouts and delays are treated as zeros?](#treat-negative-timeout-as-zero)

Thread safety of Cleaners and native code
 - [`close()` is concurrently idempotent in a class with a `Cleaner` or `finalize()`?
 ](#thread-safe-close-with-cleaner)
 - [Method accessing native state calls `reachabilityFence()` in a class with a `Cleaner` or
 `finalize()`?](#reachability-fence)
 - [`Cleaner` or `finalize()` is used for real cleanup, not mere reporting?](#finalize-misuse)
 - [Considered making a class with native state thread-safe?](#thread-safe-native)

<hr>

### Design

<a name="rationalize"></a>
[#](#rationalize) Dn.1. If the patch introduces a new subsystem with concurrent code, is **the
necessity for concurrency or thread safety rationalized in the patch description**? Is there a
discussion of alternative design approaches that could simplify the concurrency model of the code
(see the next item)?

<a name="use-patterns"></a>
[#](#use-patterns) Dn.2. Is it possible to apply one or several design patterns (some of them are
listed below) to significantly **simplify the concurrency model of the code, while not considerably
compromising other quality aspects**, such as overall simplicity, efficiency, testability,
extensibility, etc?

**Immutability/Snapshotting.** When some state should be updated, a new immutable object (or a
snapshot within a mutable object) is created, published and used, while some concurrent threads may
still use older copies or snapshots. See [EJ Item 17], [JCIP 3.4], [RC.5](#moving-state-race) and
[NB.2](#swap-state-atomically), `CopyOnWriteArrayList`, `CopyOnWriteArraySet`, [persistent data
structures](https://en.wikipedia.org/wiki/Persistent_data_structure).

**Divide and conquer.** Work is split into several parts that are processed independently, each part
in a single thread. Then the results of processing are combined. [Parallel
Streams](#parallel-streams) or `ForkJoinPool` (see [TE.4](#fjp-no-blocking) and
[TE.5](#use-common-fjp)) can be used to apply this pattern.

**Producer-consumer.** Pieces of work are transmitted between worker threads via queues. See
[JCIP 5.3], [Dl.1](#avoid-nested-critical-sections), [CSP](
https://en.wikipedia.org/wiki/Communicating_sequential_processes), [SEDA](
https://en.wikipedia.org/wiki/Staged_event-driven_architecture).

**Instance confinement.** Objects of some root type encapsulate some complex hierarchical child
state. Root objects are solitarily responsible for the safety of accesses and modifications to the
child state from multiple threads. In other words, composed objects are synchronized rather than
synchronized objects are composed. See [JCIP 4.2, 10.1.3, 10.1.4].

**Thread/Task/Serial thread confinement.** Some state is made local to a thread using top-down
pass-through parameters or `ThreadLocal`. See [JCIP 3.3]. Task confinement is a variation of the
idea of thread confinement that is used in conjunction with the divide-and-conquer pattern. It
usually comes in the form of lambda-captured "context" parameters or fields in the per-thread task
objects. Serial thread confinement is an extension of the idea of thread confinement for the
producer-consumer pattern, see [JCIP 5.3.2].

**Active object.** An object manages its own `ExecutorService` or `Thread` to do the work. See the
[article on Wikipedia](https://en.wikipedia.org/wiki/Active_object) and [TE.6](#explicit-shutdown).

### Documentation

<a name="justify-document"></a>
[#](#justify-document) Dc.1. For every class, method, and field that has signs of being thread-safe,
such as the `synchronized` keyword, `volatile` modifiers on fields, use of any classes from
`java.util.concurrent.*`, or third-party concurrency primitives, or concurrent collections: do their
Javadoc comments include

 - **The justification for thread safety**: is it explained why a particular class, method or field
 has to be thread-safe?

 - **Concurrent access documentation**: is it enumerated from what methods and in contexts of
 what threads (executors, thread pools) each specific method of a thread-safe class is called?

Wherever some logic is parallelized or the execution is delegated to another thread, are there
comments explaining why it’s worse or inappropriate to execute the logic sequentially or in the same
thread? See [PS.1](#justify-parallel-stream-use) regarding this.

See also [NB.3](#non-blocking-warning) and [NB.4](#justify-busy-wait) regarding justification of
non-blocking code, racy code, and busy waiting.

<a name="threading-flow-model"></a>
[#](#threading-flow-model) Dc.2. If the patch introduces a new subsystem that uses threads or thread
pools, are there **high-level descriptions of the threading model, the concurrent control flow (or
the data flow) of the subsystem** somewhere, e. g. in the Javadoc comment for the package in
`package-info.java` or for the main class of the subsystem? Are these descriptions kept up-to-date
when new threads or thread pools are added or some old ones deleted from the system?

Description of the threading model includes the enumeration of threads and thread pools created and
managed in the subsystem, and external pools used in the subsystem (such as
`ForkJoinPool.commonPool()`), their sizes and other important characteristics such as thread
priorities, and the lifecycle of the managed threads and thread pools.

A high-level description of concurrent control flow should be an overview and tie together
concurrent control flow documentation for individual classes, see the previous item. If the
producer-consumer pattern is used, the concurrent control flow is trivial and the data flow should
be documented instead.

Describing threading models and control/data flow greatly improves the maintainability of the
system, because in the absence of descriptions or diagrams developers spend a lot of time and effort
to create and refresh these models in their minds. Putting the models down also helps to discover
bottlenecks and the ways to simplify the design (see [Dn.2](#use-patterns)).

<a name="immutable-thread-safe"></a>
[#](#immutable-thread-safe) Dc.3. For classes and methods that are parts of the public API or the
extensions API of the project: is it specified in their Javadoc comments whether they are (or in
case of interfaces and abstract classes designed for subclassing in extensions, should they be
implemented as) **immutable, thread-safe, or not thread-safe**? For classes and methods that are (or
should be implemented as) thread-safe, is it documented precisely with what other methods (or
themselves) they may be called concurrently from multiple threads? See also [EJ Item 82],
[JCIP 4.5], [CON52-J](
https://wiki.sei.cmu.edu/confluence/display/java/CON52-J.+Document+thread-safety+and+use+annotations+where+applicable).

If the `@com.google.errorprone.annotations.Immutable` annotation is used to mark immutable classes,
[Error Prone](https://errorprone.info/) static analysis tool is capable to detect when a class is
not actually immutable (see a relevant [bug
pattern](https://errorprone.info/bugpattern/Immutable)).

<a name="name-patterns"></a>
[#](#name-patterns) Dc.4. For subsystems, classes, methods, and fields that use some concurrency
design patterns, either high-level, such as those mentioned in [Dn.2](#use-patterns), or low-level,
such as [double-checked locking](#lazy-init) or [spin looping](#justify-busy-wait): are the used
**concurrency patterns pronounced in the design or implementation comments** for the respective
subsystems, classes, methods, and fields? This helps readers to make sense out of the code quicker.

Pronouncing the used patterns in comments may be replaced with more succinct documentation
annotations, such as `@Immutable` ([Dc.3](#immutable-thread-safe)), `@GuardedBy`
([Dc.7](#guarded-by)), `@LazyInit` ([LI.5](#lazy-init-benign-race)), or annotations that you define
yourself for specific patterns which appear many times in your project.

<a name="concurrent-map-type"></a>
[#](#concurrent-map-type) Dc.5. Are `ConcurrentHashMap` and `ConcurrentSkipListMap` objects stored
in fields and variables of `ConcurrentHashMap` or `ConcurrentSkipListMap` or **`ConcurrentMap`
type**, but not just `Map`?

This is important, because in code like the following:

    ConcurrentMap<String, Entity> entities = getEntities();
    if (!entities.containsKey(key)) {
      entities.put(key, entity);
    } else {
      ...
    }

It should be pretty obvious that there might be a race condition because an entity may be put into
the map by a concurrent thread between the calls to `containsKey()` and `put()` (see
[RC.1](#chm-race) about this type of race conditions). While if the type of the entities variable
was just `Map<String, Entity>` it would be less obvious and readers might think this is only
slightly suboptimal code and pass by.

It’s possible to turn this advice into [an inspection](
https://github.com/apache/incubator-druid/pull/6898/files#diff-3aa5d63fbb1f0748c146f88b6f0efc81R239)
in IntelliJ IDEA.

<a name="chm-type"></a>
[#](#chm-type) Dc.6. An extension of the previous item: are ConcurrentHashMaps on which `compute()`,
`computeIfAbsent()`, `computeIfPresent()`, or `merge()` methods are called stored in fields and
variables of `ConcurrentHashMap` type rather than `ConcurrentMap`? This is because
`ConcurrentHashMap` (unlike the generic `ConcurrentMap` interface) guarantees that the lambdas
passed into `compute()`-like methods are performed atomically per key, and the thread safety of the
class may depend on that guarantee.

This advice may seem to be overly pedantic, but if used in conjunction with a static analysis rule
that prohibits calling `compute()`-like methods on `ConcurrentMap`-typed objects that are not
ConcurrentHashMaps (it’s possible to create such inspection in IntelliJ IDEA too) it could prevent
some bugs: e. g. **calling `compute()` on a `ConcurrentSkipListMap` might be a race condition** and
it’s easy to overlook that for somebody who is used to rely on the strong semantics of `compute()`
in `ConcurrentHashMap`.

<a name="guarded-by"></a>
[#](#guarded-by) Dc.7. Is **`@GuardedBy` annotation used**? If accesses to some fields should be
protected by some lock, are those fields annotated with `@GuardedBy`? Are private methods that are
called from within critical sections in other methods annotated with `@GuardedBy`? If the project
doesn’t depend on any library containing this annotation (it’s provided by [`jcip-annotations`](
https://search.maven.org/artifact/net.jcip/jcip-annotations/1.0/jar), [`error_prone_annotations`](
https://search.maven.org/search?q=a:error_prone_annotations%20g:com.google.errorprone), [`jsr305`](
https://search.maven.org/search?q=g:com.google.code.findbugs%20a:jsr305), and other libraries) and
for some reason it’s undesirable to add such dependency, it should be mentioned in Javadoc comments
for the respective fields and methods that accesses and calls to them should be protected by some
specified locks.

See [JCIP 2.4] for more information about `@GuardedBy`.

Usage of `@GuardedBy` is especially beneficial in conjunction with [Error Prone](
https://errorprone.info/) tool which is able to [statically check for unguarded accesses to fields
and methods with @GuardedBy annotations](https://errorprone.info/bugpattern/GuardedBy).

<a name="document-benign-race"></a>
[#](#document-benign-race) Dc.8. If in a thread-safe class some **fields are accessed both from
within critical sections and outside of critical sections**, is it explained in comments why this
is safe? For example, unprotected read-only access to a reference to an immutable object might be
benignly racy (see [RC.5](#moving-state-race)). Answering this question also helps to prevent
problems described in [IS.2](#non-volatile-visibility), [IS.3](#non-volatile-protection), and
[RC.2](#unsafe-concurrent-point-read).

Instead of writing a comment explaining that access to a *lazily initialized field* outside of a
critical section is safe, the field could just be annotated with [`@LazyInit`](
http://errorprone.info/api/latest/com/google/errorprone/annotations/concurrent/LazyInit.html) from
[`error_prone_annotations`](
https://search.maven.org/search?q=a:error_prone_annotations%20g:com.google.errorprone) (but make
sure to read the Javadoc for this annotation and to check that the field conforms to the
description; [LI.3](#safe-local-dcl) and [LI.5](#lazy-init-benign-race) mention potential pitfalls).

Apart from the explanations why the partially blocking or racy code is safe, there should also be
comments justifying such error-prone code and warning the developers that the code should be
modified and reviewed with double attention: see [NB.3](#non-blocking-warning).

<a name="justify-volatile"></a>
[#](#justify-volatile) Dc.9. Regarding every field with a `volatile` modifier: **does it really need
to be `volatile`**? Does the Javadoc comment for the field explain why the semantics of `volatile`
field reads and writes (as defined in the [Java Memory Model](
https://docs.oracle.com/javase/specs/jls/se11/html/jls-17.html#jls-17.4)) are required for the
field?

Similarly to what is noted in the previous item, justification for a lazily initialized field to be
`volatile` could be omitted if the lazy initialization pattern itself is identified, according to
[Dc.4](#name-patterns).  When `volatile` on a field is needed to ensure *safe publication* of
objects written into it (see [JCIP 3.5] or [here](
https://shipilev.net/blog/2014/safe-public-construction/#_safe_publication)), then just mentioning
"safe publication" in the Javadoc comment for the field is sufficient, it's not needed to elaborate
the semantics of `volatile` which ensure the safe publication.

<a name="plain-field"></a>
[#](#plain-field) Dc.10. Is it explained in the **Javadoc comment for each mutable field in a
thread-safe class that is neither `volatile` nor annotated with `@GuardedBy`**, why that is safe?
Perhaps, the field is only accessed and mutated from a single method or a set of methods that are
specified to be called only from a single thread sequentially (described as per
[Dc.1](#justify-document)). This recommendation also applies to `final` fields that store objects of
non-thread-safe classes when those objects could be mutated from some methods of the enclosing
thread-safe class. See [IS.2](#non-volatile-visibility), [IS.3](#non-volatile-protection),
[RC.2](#unsafe-concurrent-point-read), [RC.3](#unsafe-concurrent-iteration), and
[RC.4](#concurrent-mutation-race) about what could go wrong with such code.

### Insufficient synchronization

<a name="static-thread-safe"></a>
[#](#static-thread-safe) IS.1. **Can non-private static methods be called concurrently from multiple
threads?** If there is a non-private static field with mutable state, such as a collection, is it an
instance of a thread-safe class or synchronized using some `Collections.synchronizedXxx()` method?

Note that calls to `DateFormat.parse()` and `format()` must be synchronized because they mutate the
object: see [IS.5](#dateformat).

<a name="non-volatile-visibility"></a>
[#](#non-volatile-visibility) IS.2. There is no situation where some **thread awaits until a
non-volatile field has a certain value in a loop, expecting it to be updated from another thread?**
The field should at least be `volatile` to ensure eventual visibility of concurrent updates. See
[JCIP 3.1, 3.1.4], and [VNA00-J](
https://wiki.sei.cmu.edu/confluence/display/java/VNA00-J.+Ensure+visibility+when+accessing+shared+primitive+variables)
for more details and examples.

Even if the respective field if `volatile`, busy waiting for a condition in a loop can be abused
easily and therefore should be justified in a comment: see [NB.4](#justify-busy-wait).

[Dc.10](#plain-field) also demands adding explaining comments to mutable fields which are neither
`volatile` nor annotated with `@GuardedBy` which should inevitably lead to the discovery of the
visibility issue.

<a name="non-volatile-protection"></a>
[#](#non-volatile-protection) IS.3. **Read accesses to non-volatile primitive fields which can be
updated concurrently are protected with a lock as well as the writes?** The minimum reason for this
is that reads of `long` and `double` fields are non-atomic (see [JLS 17.7](
https://docs.oracle.com/javase/specs/jls/se11/html/jls-17.html#jls-17.7), [JCIP 3.1.2], [VNA05-J](
https://wiki.sei.cmu.edu/confluence/display/java/VNA05-J.+Ensure+atomicity+when+reading+and+writing+64-bit+values)).
But even with other types of fields, unbalanced synchronization creates possibilities for downstream
bugs related to the visibility (see the previous item) and the lack of the expected happens-before
relationships.

As well as with the previous item, accurate documentation of benign races
([Dc.8](#document-benign-race) and [Dc.10](#plain-field)) should reliably expose the cases when
unbalanced synchronization is problematic.

See also [RC.2](#unsafe-concurrent-point-read) regarding unbalanced synchronization of read accesses
to mutable objects, such as collections.

<a name="server-framework-sync"></a>
[#](#server-framework-sync) IS.4. **Is the business logic written for server frameworks
thread-safe?** This includes:
 - `Servlet` implementations
 - `@(Rest)Controller`-annotated classes, `@Get/PostMapping`-annotated methods in Spring
 - `@SessionScoped` and `@ApplicationScoped` managed beans in JSF
 - `Filter` and `Handler` implementations in various synchronous and asynchronous frameworks
 (including Jetty, Netty, Undertow)
 - `@GET`- and `@POST`-annotated methods (resources) in JAX-RS (RESTful APIs)

It's easy to forget that if such code mutates some state (e. g. fields in the class), it must be
properly synchronized or access only concurrent collections and classes.

<a name="dateformat"></a>
[#](#dateformat) IS.5. **Calls to `parse()` and `format()` on a shared instance of `DateFormat` are
synchronized**, e. g. if a `DateFormat` is stored in a static field? Although `parse()` and
`format()` may look "read-only", they actually mutate the receiving `DateFormat` object.

### Excessive thread safety

<a name="pseudo-safety"></a>
[#](#pseudo-safety) ETS.1. An example of excessive thread safety is a class where every modifiable
field is `volatile` or an `AtomicReference` or other atomic, and every collection field stores a
concurrent collection (e. g. `ConcurrentHashMap`), although all accesses to those fields are
`synchronized`.

**There shouldn’t be any "extra" thread safety in code, there should be just enough of it.**
Duplication of thread safety confuses readers because they might think the extra thread safety
precautions are (or used to be) needed for something but will fail to find the purpose.

The exception from this principle is the `volatile` modifier on the lazily initialized field in the
[safe local double-checked locking pattern](
http://hg.openjdk.java.net/code-tools/jcstress/file/9270b927e00f/tests-custom/src/main/java/org/openjdk/jcstress/tests/singletons/SafeLocalDCL.java#l71)
which is the recommended way to implement double-checked locking, despite that `volatile` is
[excessive for correctness](https://shipilev.net/blog/2014/safe-public-construction/#_correctness)
when the lazily initialized object has all `final` fields[*](
https://shipilev.net/blog/2014/safe-public-construction/#_safe_initialization). Without that
`volatile` modifier the thread safety of the double-checked locking could easily be broken by a
change (addition of a non-final field) in the class of lazily initialized objects, though that class
should not be aware of subtle concurrency implications. If the class of lazily initialized objects
is *specified* to be immutable (see [Dc.3](#immutable-thread-safe)) the `volatile` is still
unnecessary and the [UnsafeLocalDCL](
http://hg.openjdk.java.net/code-tools/jcstress/file/9270b927e00f/tests-custom/src/main/java/org/openjdk/jcstress/tests/singletons/UnsafeLocalDCL.java#l71)
pattern could be used safely, but the fact that some class has all `final` fields doesn’t
necessarily mean that it’s immutable.

See also [the section about double-checked locking](#lazy-init).

<a name="redundant-atomics"></a>
[#](#redundant-atomics) ETS.2. Aren’t there **`AtomicReference`, `AtomicBoolean`, `AtomicInteger` or
`AtomicLong` fields on which only `get()` and `set()` methods are called?** Simple fields with
`volatile` modifiers can be used instead, but `volatile` might not be needed too; see
[Dc.9](#justify-volatile).

<a name="unneeded-thread-safety"></a>
[#](#unneeded-thread-safety) ETS.3. **Does a class (method) need to be thread-safe?** May a class
be accessed (method called) concurrently from multiple threads (without *happens-before*
relationships between the accesses or calls)? Can a class (method) be simplified by making it
non-thread-safe?

See also [Dc.9](#justify-volatile) about potentially unneeded `volatile` modifiers. 

This item is a close relative of [Dn.1](#rationalize) (about rationalizing concurrency and thread
safety in the patch description) and [Dc.1](#justify-document) (about justifying concurrency in
Javadocs for classes and methods, and documenting concurrent access). If these actions are done, it
should be self-evident whether the class (method) needs to be thread-safe or not. There may be
cases, however, when it might be desirable to make the class (method) thread-safe although it's not
supposed to be accessed or called concurrently as of the moment of the patch. For example, thread
safety may be needed to ensure memory safety (see [CN.4](#thread-safe-native) about this).
Anticipating some changes to the codebase that make the class (method) being accessed from multiple
threads may be another reason to make the class (method) thread-safe up front.

<a name="unneeded-fairness"></a>
[#](#unneeded-fairness) ETS.4. **Does a `ReentrantLock` (or `ReentrantReadWriteLock`, `Semaphore`)
need to be fair?** To justify the throughput penalty of making a lock fair it should be demonstrated
that a lack of fairness leads to unacceptably long starvation periods in some threads trying to
acquire the lock or pass the semaphore. This should be documented in the Javadoc comment for the
field holding the lock or the semaphore. See [JCIP 13.3] for more details.

### Race conditions

<a name="chm-race"></a>
[#](#chm-race) RC.1. Aren’t **`ConcurrentMap` (or Cache) objects updated with separate
`containsKey()`, `get()`, `put()` and `remove()` calls** instead of a single call to
`compute()`/`computeIfAbsent()`/`computeIfPresent()`/`replace()`?

<a name="unsafe-concurrent-point-read"></a>
[#](#unsafe-concurrent-point-read) RC.2. Aren’t there **point read accesses such as `Map.get()`,
`containsKey()` or `List.get()` outside of critical sections to a non-thread-safe collection such as
`HashMap` or `ArrayList`**, while new entries can be added to the collection concurrently, even
though there is a happens-before edge between the moment when some entry is put into the collection
and the moment when the same entry is point-queried outside of a critical section?

The problem is that when new entries can be added to a collection, it grows and changes its internal
structure from time to time (HashMap rehashes the hash table, `ArrayList` reallocates the internal
array). At such moments races might happen and unprotected point read accesses might fail with
`NullPointerException`, `ArrayIndexOutOfBoundsException`, or return `null` or some random entry.

Note that this concern applies to `ArrayList` even when elements are only added to the end of the
list. However, a small change in `ArrayList`’s implementation in OpenJDK could have disallowed data
races in such cases at very little cost. If you are subscribed to the concurrency-interest mailing
list, you could help to bring attention to this problem by reviving [this thread](
http://cs.oswego.edu/pipermail/concurrency-interest/2018-September/016526.html).

See also [IS.3](#non-volatile-protection) regarding unbalanced synchronization of accesses to
primitive fields.

<a name="unsafe-concurrent-iteration"></a>
[#](#unsafe-concurrent-iteration) RC.3. A variation of the previous item: isn’t a non-thread-safe
collection such as `HashMap` or `ArrayList` **iterated outside of a critical section**, while it may
be modified concurrently? This could happen by accident when an `Iterable`, `Iterator` or `Stream`
over a collection is returned from a method of a thread-safe class, even though the iterator or
stream is created within a critical section. Note that **returning unmodifiable collection views
like `Collections.unmodifiableList()` from getters wrapping collection fields that may be modified
concurrently doesn't solve this problem.** If the collection is relatively small, it should be
copied entirely, or a copy-on-write collection (see [Sc.3](#non-blocking-collections)) should be
used instead of a non-thread-safe collection.

Note that calling `toString()` on a collection (e. g. in a logging statement) implicitly iterates
over it.

Like the previous item, this one applies to growing ArrayLists too.

This item applies even to synchronized collections: see [RC.10](#synchronized-collection-iter) for
details.

<a name="concurrent-mutation-race"></a>
[#](#concurrent-mutation-race) RC.4. Generalization of the previous item: aren’t **non-trivial
objects that can be mutated concurrently returned from getters** in a thread-safe class (and thus
inevitably leaking outside of critical sections)?

<a name="moving-state-race"></a>
[#](#moving-state-race) RC.5. If there are multiple variables in a thread-safe class that are
**updated at once but have individual getters**, isn’t there a race condition in the code that calls
those getters? If there is, the variables should be made `final` fields in a dedicated POJO, that
serves as a snapshot of the updated state. The POJO is stored in a field of the thread-safe class,
directly or as an `AtomicReference`. Multiple getters to individual fields should be replaced with
a single getter that returns the POJO. This allows avoiding a race condition in the client code by
reading a consistent snapshot of the state at once.

This pattern is also very useful for creating safe and reasonably simple non-blocking code: see
[NB.2](#swap-state-atomically) and [JCIP 15.3.1].

<a name="read-outside-critical-section-race"></a>
[#](#read-outside-critical-section-race) RC.6. If some logic within some critical section depends on
some data that principally is part of the internal mutable state of the class, but was read outside
of the critical section or in a different critical section, isn’t there a race condition because the
**local copy of the data may become out of sync with the internal state by the time when the
critical section is entered**? This is a typical variant of check-then-act race condition, see
[JCIP 2.2.1].

An example of this race condition is calling `toArray()` on synchronized collections with a sized
array:
```java
list = Collections.synchronizedList(new ArrayList<>());
...
elements = list.toArray(new Element[list.size()]);
```
This might unexpectedly leave the `elements` array with some nulls in the end if there are some
concurrent removals from the list. Therefore, `toArray()` on a synchronized collection should be
called with a zero-length array: `toArray(new Element[0])`, which is also not worse from the
performance perspective: see "[Arrays of Wisdom of the
Ancients](https://shipilev.net/blog/2016/arrays-wisdom-ancients/)".

See also [RC.9](#cache-invalidation-race) about cache invalidation races which are similar to
check-then-act races.

<a name="outside-world-race"></a>
[#](#outside-world-race) RC.7. Aren't there **race conditions between the code (i. e. program
runtime actions) and some actions in the outside world** or actions performed by some other programs
running on the machine? For example, if some configurations or credentials are hot reloaded from
some file or external registry, reading separate configuration parameters or separate credentials
(such as username and password) in separate transactions with the file or the registry may be racing
with a system operator updating those configurations or credentials.

Another example is checking that a file exists (or not exists) and then reading, deleting, or
creating it, respectively, while another program or a user may delete or create the file between the
check and the act. It's not always possible to cope with such race conditions, but it's useful to
keep such possibilities in mind. Prefer static methods from [`java.nio.file.Files`](
https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/nio/file/Files.html) class and
NIO file reading/writing API to methods from the old `java.io.File` for file system operations.
Methods from `Files` are more sensitive to file system race conditions and tend to throw exceptions
in adverse cases, while methods on `File` swallow errors and make it hard even to detect race
conditions. Static methods from `Files` also support `StandardOpenOption.CREATE` and `CREATE_NEW`
which may help to ensure some extra atomicity.

<a name="guava-cache-invalidation-race"></a>
[#](#guava-cache-invalidation-race) RC.8. If you are **using Guava Cache and `invalidate(key)`, are
you not affected by the [race condition](https://github.com/google/guava/issues/1881)** which can
leave a `Cache` with an invalid (stale) value mapped for a key? Consider using [Caffeine cache](
https://github.com/ben-manes/caffeine) which doesn't have this problem. Caffeine is also faster and
more scalable than Guava Cache: see [Sc.9](#caffeine).

<a name="cache-invalidation-race"></a>
[#](#cache-invalidation-race) RC.9. Generalization of the previous item: isn't there a potential
**cache invalidation race** in the code? There are several ways to get into this problem:
 - Using `Cache.put()` method concurrently with `invalidate()`. Unlike
 [RC.8](#guava-cache-invalidation-race), this is a race regardless of what caching library is used,
 not necessarily Guava. This is also similar to [RC.1](#chm-race).
 - Having `put()` and `invalidate()` methods exposed in the own Cache interface. This places the
 burden of synchronizing `put()` (together the preceding "checking" code, such as `get()`) and
 `invalidate()` calls on the users of the API which really should be the job of the Cache
 implementation.
 - There is some [lazily initialized state](#lazy-init) in a mutable object which can be invalidated
 upon mutation of the object, and can also be accessed concurrently with the mutation. This means
 the class is in the category of [non-blocking concurrency](#non-blocking): see the corresponding
 checklist items. A way to avoid cache invalidation race in this case is to wrap the primary state
 and the cached state into a POJO and replace it atomically, as described in
 [NB.2](#swap-state-atomically).

<a name="synchronized-collection-iter"></a>
[#](#synchronized-collection-iter) RC.10. **The whole iteration loop over a synchronized collection
(i. e. obtained from one of the `Collections.synchronizedXxx()` static factory methods), or a
Stream pipeline using a synchronized collection as a source is protected by `synchronized (coll)`?**
See [the Javadoc](
https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/Collections.html#synchronizedCollection(java.util.Collection))
for examples and details. This also applies to passing synchronized collections into copy
constructors or static factory methods of other collections because they implicitly iterate over the
source collection.

See also [RC.3](#unsafe-concurrent-iteration) about unprotected iteration over non-thread-safe
collections.

### Testing

<a name="multi-threaded-tests"></a>
[#](#multi-threaded-tests) T.1. **Are unit tests for thread-safe classes multi-threaded?**
Single-threaded tests don't really test the thread safety and concurrency.

<a name="concurrent-test-random"></a>
[#](#concurrent-test-random) T.2. **Isn't a shared `java.util.Random` object used for data
generation in a concurrency test?** `java.util.Random` is synchronized internally, so if multiple
test threads (which are conceived to access the tested class concurrently) access the same
`java.util.Random` object then the test might degenerate to a mostly synchronous one and fail to
exercise the concurrency properties of the tested class. See [JCIP 12.1.3]. `Math.random()` is a
subject for this problem too because internally `Math.random()` uses a globally shared
`java.util.Random` instance. Use `ThreadLocalRandom` instead. 

<a name="coordinate-test-workers"></a>
[#](#coordinate-test-workers) T.3. Do **concurrent test workers coordinate their start using a latch
such as `CountDownLatch`?** If they don't much or even all of the test work might be done by the
first few started workers. See [JCIP 12.1.3] for more information.

<a name="test-workers-interleavings"></a>
[#](#test-workers-interleavings) T.4. Are there **more test threads than there are available
processors** if possible for the test? This will help to generate more thread scheduling
interleavings and thus test the logic for the absence of race conditions more thoroughly. See
[JCIP 12.1.6] for more information. The number of available processors on the machine can be
obtained as [`Runtime.getRuntime().availableProcessors()`](
https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/lang/Runtime.html#availableProcessors()).

<a name="concurrent-assert"></a>
[#](#concurrent-assert) T.5. Are there **no regular assertions in code that is executed not in the
main thread running the unit test?** Consider the following example:
```java
@Test public void testServiceListener() {
  // Missed assertion -- Don't do this!
  service.addListener(event -> Assert.assertEquals(Event.Type.MESSAGE_RECEIVED, event.getType()));
  service.sendMessage("test");
}
```
Assuming the `service` executes the code of listeners asynchronously in some internally-managed or
a shared thread pool, even if the assertion within the listener's lambda fails, JUnit and TestNG
will think the test has passed.

The solution to this problem is either to pass the data (or thrown exceptions) from a concurrent
thread back to the main test thread and verify it in the end of the test, or to use
[ConcurrentUnit](https://github.com/jhalterman/concurrentunit) library which takes over
the boilerplate associated with the first approach.

<a name="replacing-locks-with-concurrency-utilities"></a>
### Locks

<a name="avoid-wait-notify"></a>
[#](#avoid-wait-notify) Lk.1. Is it possible to use concurrent collections and/or utilities from
`java.util.concurrent.*` and **avoid using locks with `Object.wait()`/`notify()`/`notifyAll()`**?
Code redesigned around concurrent collections and utilities is often both clearer and less
error-prone than code implementing the equivalent logic with intrinsic locks, `Object.wait()` and
`notify()` (`Lock` objects with `await()` and `signal()` are not different in this regard). See
[EJ Item 81] for more information.

<a name="guava-monitor"></a>
[#](#guava-monitor) Lk.2. Is it possible to **simplify code that uses intrinsic locks or `Lock`
objects with conditional waits by using Guava’s [`Monitor`](
https://google.github.io/guava/releases/27.0.1-jre/api/docs/com/google/common/util/concurrent/Monitor.html)
instead**?

<a name="use-synchronized"></a>
[#](#use-synchronized) Lk.3. **Isn't `ReentrantLock` used when `synchronized` would suffice?**
`ReentrantLock` shoulndn't be used in the situations when none of its distinctive features
(`tryLock()`, timed and interruptible locking methods, etc.) are used. Note that reentrancy is *not*
such a feature: intrinsic Java locks support reentrancy too. The ability for a `ReentrantLock` to be
fair is seldom such a feature: see [ETS.4](#unneeded-fairness). See [JCIP 13.4] for more information
about this.

This advice also applies when a class uses a private lock object (instead of `synchronized (this)`
or synchronized methods) to protect against accidental or malicious interference by the clients
acquiring synchronizing on the object of the class: see [JCIP 4.2.1],  [EJ Item 82], [LCK00-J](
https://wiki.sei.cmu.edu/confluence/display/java/LCK00-J.+Use+private+final+lock+objects+to+synchronize+classes+that+may+interact+with+untrusted+code).

<a name="lock-unlock"></a>
[#](#lock-unlock) Lk.4. **Locking (`lock()`, `lockInterruptibly()`, `tryLock()`) and `unlock()`
methods are used strictly with the recommended [try-finally idiom](
https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/concurrent/locks/Lock.html)
without deviations?**
 - `lock()` (or `lockInterruptibly()`) call goes *before* the `try {}` block rather than within it?
 - There are no statements between the `lock()` (or `lockInterruptibly()`) call and the beginning of
 the `try {}` block?
 - `unlock()` call is the first statement within the `finally {}` block?

This advice doesn't apply when locking methods and `unlock()` should occur in different scopes, i.
e. not within the recommended try-finally idiom altogether. The containing methods could be
annotated with Error Prone's [`@LockMethod`](
https://errorprone.info/api/latest/com/google/errorprone/annotations/concurrent/LockMethod.html) and
`@UnlockMethod` annotations.

There is a "Lock acquired but not safely unlocked" inspection in IntelliJ IDEA which corresponds to
this item.

See also [LCK08-J](
https://wiki.sei.cmu.edu/confluence/display/java/LCK08-J.+Ensure+actively+held+locks+are+released+on+exceptional+conditions).

### Avoiding deadlocks

<!-- Preserving former anchor with a typo. -->
<a name="avoid-nested-critial-sections"></a>
<a name="avoid-nested-critical-sections"></a>
[#](#avoid-nested-critical-sections) Dl.1. If a thread-safe class is implemented so that there are
nested critical sections protected by different locks, **is it possible to redesign the code to get
rid of nested critical sections**? Sometimes a class could be split into several distinct classes,
or some work that is done within a single thread could be split between several threads or tasks
which communicate via concurrent queues. See [JCIP 5.3] for more information about the
producer-consumer pattern.

<a name="document-locking-order"></a>
[#](#document-locking-order) Dl.2. If restructuring a thread-safe class to avoid nested critical
sections is not reasonable, was it deliberately checked that the locks are acquired in the same
order throughout the code of the class? **Is the locking order documented in the Javadoc comments
for the fields where the lock objects are stored?**

See [LCK07-J](
https://wiki.sei.cmu.edu/confluence/display/java/LCK07-J.+Avoid+deadlock+by+requesting+and+releasing+locks+in+the+same+order)
for examples.

<a name="dynamic-lock-ordering"></a>
[#](#dynamic-lock-ordering) Dl.3. If there are nested critical sections protected by several
(potentially different) **dynamically determined locks (for example, associated with some business
logic entities), are the locks ordered before the acquisition**? See [JCIP 10.1.2] for more
information.

<a name="non-open-call"></a>
[#](#non-open-call) Dl.4. Aren’t there **calls to some callbacks (listeners, etc.) that can be
configured through public API or extension interface calls within critical sections**? With such
calls, the system might be inherently prone to deadlocks because the external logic executed within
a critical section may be unaware of the locking considerations and call back into the logic of the
system, where some more locks may be acquired, potentially forming a locking cycle that might lead
to a deadlock. Also, the external logic could just perform some time-consuming operation and by
that harm the efficiency of the system (see [Sc.1](#minimize-critical-sections)). See [JCIP 10.1.3]
and [EJ Item 79] for more information.

When public API or extension interface calls happen within lambdas passed into `Map.compute()`,
`computeIfAbsent()`, `computeIfPresent()`, and `merge()`, there is a risk of not only deadlocks (see
the next item) but also race conditions which could result in a corrupted map (if it's not a
`ConcurrentHashMap`, e. g. a simple `HashMap`) or runtime exceptions.

Beware that a [`CompletableFuture`](
https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/concurrent/CompletableFuture.html
) or a [`ListenableFuture`](
https://guava.dev/releases/28.1-jre/api/docs/com/google/common/util/concurrent/ListenableFuture.html
) returned from a public API opens a door for performing some user-defined callbacks from the place
where the future is completed, deep inside the library (framework). If the future is completed
within a critical section, or from an `ExecutorService` whose threads must not block, such as a
`ForkJoinPool` (see [TE.4](#fjp-no-blocking)) or the worker pool of an I/O library, consider either
returning a simple `Future`, or documenting that stages shouldn't be attached to this
`CompletableFuture` in the *default execution mode*. See [TE.7](#cf-beware-non-async) for more
information.

<a name="chm-nested-calls"></a>
[#](#chm-nested-calls) Dl.5. Aren't there **calls to methods on a `ConcurrentHashMap` instance
within lambdas passed into `compute()`-like methods called on the same map?** For example, the
following code is deadlock-prone:
```java
map.compute(key, (String k, Integer v) -> {
  if (v == null || v == 0) {
    return map.get(DEFAULT_KEY);
  }
  return v;
});
```
Note that nested calls to non-lambda accepting methods, *including read-only access methods like
`get()`* create the possibility of deadlocks as well as nested calls to `compute()`-like methods
because the former are not always lock-free. 

### Improving scalability

<a name="minimize-critical-sections"></a>
[#](#minimize-critical-sections) Sc.1. **Are critical sections as small as possible?** For every
critical section: can’t some statements in the beginning and the end of the section be moved out of
it? Not only minimizing critical sections improves scalability, but also makes it easier to review
them and spot race conditions and deadlocks.

This advice equally applies to lambdas passed into `ConcurrentHashMap`’s `compute()`-like methods.

See also [JCIP 11.4.1] and [EJ Item 79].

<a name="increase-locking-granularity"></a>
[#](#increase-locking-granularity) Sc.2. Is it possible to **increase locking granularity**? If a
thread-safe class encapsulates accesses to map, is it possible to **turn critical sections into
lambdas passed into `ConcurrentHashMap.compute()`** or `computeIfAbsent()` or `computeIfPresent()`
methods to enjoy effective per-key locking granularity? Otherwise, is it possible to use
**[Guava’s `Striped`](https://github.com/google/guava/wiki/StripedExplained)** or an equivalent? See
[JCIP 11.4.3] for more information about lock striping.

<a name="non-blocking-collections"></a>
[#](#non-blocking-collections) Sc.3. Is it possible to **use non-blocking collections instead of
blocking ones?** Here are some possible replacements within JDK:

 - `Collections.synchronizedMap(HashMap)`, `Hashtable` → `ConcurrentHashMap`
 - `Collections.synchronizedSet(HashSet)` → `ConcurrentHashMap.newKeySet()`
 - `Collections.synchronizedMap(TreeMap)` → `ConcurrentSkipListMap`. By the way,
 `ConcurrentSkipListMap` is not the state of the art concurrent sorted dictionary implementation.
 [SnapTree](https://github.com/nbronson/snaptree) is [more efficient](
 https://github.com/apache/incubator-druid/pull/6719) than `ConcurrentSkipListMap` and there have
 been some research papers presenting algorithms that are claimed to be more efficient than
 SnapTree.
 - `Collections.synchronizedSet(TreeSet)` → `ConcurrentSkipListSet`
 - `Collections.synchronizedList(ArrayList)`, `Vector` → `CopyOnWriteArrayList`
 - `LinkedBlockingQueue` → `ConcurrentLinkedQueue`
 - `LinkedBlockingDeque` → `ConcurrentLinkedDeque`

Also consider using queues from JCTools instead of concurrent queues from the JDK: see
[Sc.8](#jctools).

<a name="use-class-value"></a>
[#](#use-class-value) Sc.4. Is it possible to **use [`ClassValue`](
https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/lang/ClassValue.html) instead of
`ConcurrentHashMap<Class, ...>`?** Note, however, that unlike `ConcurrentHashMap` with its
`computeIfAbsent()` method `ClassValue` doesn’t guarantee that per-class value is computed only
once, i. e. `ClassValue.computeValue()` might be executed by multiple concurrent threads. So if the
computation inside `computeValue()` is not thread-safe, it should be synchronized separately. On the
other hand, `ClassValue` does guarantee that the same value is always returned from
`ClassValue.get()` (unless `remove()` is called).

<a name="read-write-lock"></a>
[#](#read-write-lock) Sc.5. Was it considered to **replace a simple lock with a `ReadWriteLock`**?
Beware, however, that it’s more expensive to acquire and release a `ReentrantReadWriteLock` than a
simple intrinsic lock, so the increase in scalability comes at the cost of reduced throughput. If
the operations to be performed under a lock are short, or if a lock is already striped (see
[Sc.2](##increase-locking-granularity)) and therefore very lightly contended, **replacing a simple
lock with a `ReadWriteLock` might have a net negative effect** on the application performance. See
[this comment](
https://medium.com/@leventov/interesting-perspective-thanks-i-didnt-think-about-this-before-e044eec71870)
for more details.

<a name="use-stamped-lock"></a>
[#](#use-stamped-lock) Sc.6. Is it possible to use a **[`StampedLock`](
https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/concurrent/locks/StampedLock.html)
instead of a `ReentrantReadWriteLock`** when reentrancy is not needed?

<a name="long-adder-for-hot-fields"></a>
[#](#long-adder-for-hot-fields) Sc.7. Is it possible to use **[`LongAdder`](
https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/concurrent/atomic/LongAdder.html)
for "hot fields"** (see [JCIP 11.4.4]) instead of `AtomicLong` or `AtomicInteger` on which only
methods like `incrementAndGet()`, `decrementAndGet()`, `addAndGet()` and (rarely) `get()` is called,
but not `set()` and `compareAndSet()`?

Note that a field should be really updated from several concurrent threads steadily to justify using
`LongAdder`. If the field is usually updated only from one thread at a time (there may be several
updating threads, but each of them accesses the field infrequently, so the updates from different
threads rarely happen at the same time) it's still better to use `AtomicLong` or `AtomicInteger`
because they take less memory than `LongAdder` and their updates are cheaper.

<a name="jctools"></a>
[#](#jctools) Sc.8. Was it considered to **use one of the array-based queues from [the JCTools
library](https://www.baeldung.com/java-concurrency-jc-tools) instead of `ArrayBlockingQueue`**?
Those queues from JCTools are classified as blocking, but they avoid lock acquisition in many cases
and are generally much faster than `ArrayBlockingQueue`.

See also [Sc.3](#non-blocking-collections) regarding replacing blocking queues (and other
collections) with non-blocking equivalents within JDK.

<a name="caffeine"></a>
[#](#caffeine) Sc.9. Was it considered to **use [Caffeine](https://github.com/ben-manes/caffeine)
cache instead of other Cache implementations (such from Guava)**? [Caffeine's performance](
https://github.com/ben-manes/caffeine/wiki/Benchmarks) is very good compared to other caching
libraries.

Another reason to use Caffeine instead of Guava Cache is that it avoids an invalidation race: see
[RC.8](#guava-cache-invalidation-race).

<a name="speculation"></a>
[#](#speculation) Sc.10. When some state or a condition is checked, or resource is allocated within
a critical section, and the result is usually expected to be positive (access granted,
resource allocated) because the entity is rarely in a protected, restricted, or otherwise special
state, or because the underlying resource is rarely in shortage, is it possible to **apply
speculation (optimistic concurrency) to improve scalability**? This means to use lighter-weight
synchronization (e. g. shared locking with a `ReadWriteLock` or a `StampedLock` instead of exclusive
locking) sufficient just to detect a shortage of the resource or that the entity is in a special
state, and fallback to heavier synchronization only when necessary.

This principle is used internally in many scalable concurrent data structures, including
`ConcurrentHashMap` and JCTools's queues, but could be applied on the higher logical level as well.

See also the article about [Optimistic concurrency control](
https://en.wikipedia.org/wiki/Optimistic_concurrency_control) on Wikipedia.

<a name="lazy-init"></a>
### Lazy initialization and double-checked locking

Regarding all items in this section, see also [EJ Item 83] and "[Safe Publication this and Safe
Initialization in Java](https://shipilev.net/blog/2014/safe-public-construction/)".

<a name="lazy-init-thread-safety"></a>
[#](#lazy-init-thread-safety) LI.1. For every lazily initialized field: **is the initialization code
thread-safe and might it be called from multiple threads concurrently?** If the answers are "no" and
"yes", either double-checked locking should be used or the initialization should be eager.

Be especially wary of using lazy-initialization in mutable objects. They are prone to cache
invalidation race conditions: see [RC.9](#cache-invalidation-race).

<a name="use-dcl"></a>
[#](#use-dcl) LI.2. If a field is initialized lazily under a simple lock, is it possible to use
double-checked locking instead to improve performance?

<a name="safe-local-dcl"></a>
[#](#safe-local-dcl) LI.3. Does double-checked locking follow the [SafeLocalDCL](
http://hg.openjdk.java.net/code-tools/jcstress/file/9270b927e00f/tests-custom/src/main/java/org/openjdk/jcstress/tests/singletons/SafeLocalDCL.java#l71)
pattern, as noted in [ETS.1](#pseudo-safety)?

If the initialized objects are immutable a more efficient [UnsafeLocalDCL](
http://hg.openjdk.java.net/code-tools/jcstress/file/9270b927e00f/tests-custom/src/main/java/org/openjdk/jcstress/tests/singletons/UnsafeLocalDCL.java#l71)
pattern might also be used. However, if the lazily-initialized field is not `volatile` and there are
accesses to the field that bypass the initialization path, the value of the **field must be
carefully cached in a local variable**. For example, the following code is buggy:
```java
private MyImmutableClass lazilyInitializedField;

void doSomething() {
  ...
  if (lazilyInitializedField != null) {       // (1)
    lazilyInitializedField.doSomethingElse(); // (2) - Can throw NPE!
  }
}
```
This code might result in a `NullPointerException`, because although a non-null value is observed
when the field is read the first time at line 1, the second read at line 2 could observe null.

The above code could be fixed as follows:
```java
void doSomething() {
  MyImmutableClass lazilyInitialized = this.lazilyInitializedField;
  if (lazilyInitialized != null) {
    // Calling doSomethingElse() on a local variable to avoid NPE:
    // see https://github.com/code-review-checklists/java-concurrency#safe-local-dcl
    lazilyInitialized.doSomethingElse();
  }
}
```
See "[Wishful Thinking: Happens-Before Is The Actual Ordering](
https://shipilev.net/blog/2016/close-encounters-of-jmm-kind/#wishful-hb-actual)" and
"[Date-Race-Ful Lazy Initialization for Performance](
http://jeremymanson.blogspot.com/2008/12/benign-data-races-in-java.html)" for more information.

<a name="eager-init"></a>
[#](#eager-init) LI.4. In each particular case, doesn’t the **net impact of double-checked locking
and lazy field initialization on performance and complexity overweight the benefits of lazy
initialization?** Isn’t it ultimately better to initialize the field eagerly?

<a name="lazy-init-benign-race"></a>
[#](#lazy-init-benign-race) LI.5. If a field is initialized lazily under a simple lock or using
double-checked locking, does it really need locking? If nothing bad may happen if two threads do the
initialization at the same time and use different copies of the initialized state then a benign race
could be allowed. The initialized field should still be `volatile` (unless the initialized objects
are immutable) to ensure there is a happens-before edge between threads doing the initialization and
reading the field. This is called *a single-check idiom* (or *a racy single-check idiom* if the
field doesn't have a `volatile` modifier) in [EJ Item 83].

Annotate such fields with [`@LazyInit`](
http://errorprone.info/api/latest/com/google/errorprone/annotations/concurrent/LazyInit.html) from
[`error_prone_annotations`](
https://search.maven.org/search?q=a:error_prone_annotations%20g:com.google.errorprone). The place
in code with the race should also be identified with WARNING comments: see
[NB.3](#non-blocking-warning).

<a name="no-static-dcl"></a>
[#](#no-static-dcl) LI.6. Is **[lazy initialization holder class idiom](
https://en.wikipedia.org/wiki/Initialization-on-demand_holder_idiom) used for static fields which
must be lazy rather than double-checked locking?** There are no reasons to use double-checked
locking for static fields because lazy initialization holder class idiom is simpler, harder to make
mistake in, and is at least as efficient as double-checked locking (see benchmark results in "[Safe
Publication and Safe Initialization in
Java](https://shipilev.net/blog/2014/safe-public-construction/)").

<a name="non-blocking"></a>
### Non-blocking and partially blocking code

<a name="check-non-blocking-code"></a>
[#](#check-non-blocking-code) NB.1. If there is some non-blocking or semi-symmetrically blocking
code that mutates the state of a thread-safe class, was it deliberately checked that if a **thread
on a non-blocking mutation path is preempted after each statement, the object is still in a valid
state**? Are there enough comments, perhaps before almost every statement where the state is
changed, to make it relatively easy for readers of the code to repeat and verify the check?

<a name="swap-state-atomically"></a>
[#](#swap-state-atomically) NB.2. Is it possible to simplify some non-blocking code by **confining
all mutable state in an immutable POJO and update it via compare-and-swap operations**? This pattern
is also mentioned in [RC.5](#moving-state-race). Instead of a POJO, a single `long` value could be
used if all parts of the state are integers that can together fit 64 bits. See also [JCIP 15.3.1].

<a name="non-blocking-warning"></a>
[#](#non-blocking-warning) NB.3. Are there **visible WARNING comments identifying the boundaries of
non-blocking code**? The comments should mark the start and the end of non-blocking code, partially
blocking code, and benignly racy code (see [Dc.8](#document-benign-race) and
[LI.5](#lazy-init-benign-race)). The opening comments should:

 1. Justify the need for such error-prone code (which is a special case of
 [Dc.1](#justify-document)).
 2. **Warn developers that changes in the following code should be made (and reviewed) extremely
 carefully.**

<a name="justify-busy-wait"></a>
[#](#justify-busy-wait) NB.4. If some condition is awaited in a (busy) loop, like in the following
example:
```java
volatile boolean condition;

// in some method:
    while (!condition) {
      // Or Thread.sleep/yield/onSpinWait, or no statement, i. e. a pure spin wait
      TimeUnit.SECONDS.sleep(1L);
    }
    // ... do something when condition is true
```
**Is it explained in a comment why busy waiting is needed in the specific case**, and why the
costs and potential problems associated with busy waiting (see [JCIP 12.4.2] and [JCIP 14.1.1])
either don't apply in the specific case or are outweighed by the benefits?

If there is no good reason for spin waiting, it's preferable to synchronize explicitly using a tool
such as [`Semaphore`](
https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/concurrent/Semaphore.html),
[`CountDownLatch`](
https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/concurrent/CountDownLatch.html
), or [`Exchanger`](
https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/concurrent/Exchanger.html),
or if the logic for which the spin loop awaits is executed in some `ExecutorService`, it's better to
add a callback to the corresponding `CompletableFuture` or [`ListenableFuture`](
https://github.com/google/guava/wiki/ListenableFutureExplained) (check [TE.7](#cf-beware-non-async)
about doing this properly).

Since `Thread.yield()` and `Thread.onSpinWait()` are rarely, if ever, useful outside of spin loops,
this item could also be interpreted as that there should be a comment to every call to either of
these methods, explaining either why they are called outside of a spin loop, or justifying the spin
loop itself.

In any case, the field checked in the busy loop must be `volatile`: see
[IS.2](#non-volatile-visibility) for details.

The busy wait pattern is covered by IntelliJ IDEA's inspections "Busy wait" and "while loop spins on
field".

### Threads and Executors

<a name="name-threads"></a>
[#](#name-threads) TE.1. **Are Threads given names** when created? Are ExecutorServices created with
thread factories that name threads?

It appears that different projects have different policies regarding other aspects of `Thread`
creation: whether to make them daemon with `setDaemon()`, whether to set thread priorities and
whether a `ThreadGroup` should be specified. Many of such rules can be effectively enforced with
[forbidden-apis](https://github.com/policeman-tools/forbidden-apis).

<a name="reuse-threads"></a>
[#](#reuse-threads) TE.2. Aren’t there threads created and started, but not stored in fields, a-la
**`new Thread(...).start()`**, in some methods that may be called repeatedly? Is it possible to
delegate the work to a cached or a shared `ExecutorService` instead?

<a name="cached-thread-pool-no-io"></a>
[#](#cached-thread-pool-no-io) TE.3. **Aren’t some network I/O operations performed in an
`Executors.newCachedThreadPool()`-created `ExecutorService`?** If a machine that runs the
application has network problems or the network bandwidth is exhausted due to increased load,
CachedThreadPools that perform network I/O might begin to create new threads uncontrollably.

Note that completing some `CompletableFuture` or `SettableFuture` from inside a cached thread pool
and then returning this future to a user might expose the thread pool to executing unwanted actions
if the future is used improperly: see [TE.7](#cf-beware-non-async) for details.

<a name="fjp-no-blocking"></a>
[#](#fjp-no-blocking) TE.4. **Aren’t there blocking or I/O operations performed in tasks scheduled
to a `ForkJoinPool`** (except those performed via a [`managedBlock()`](
https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/concurrent/ForkJoinPool.html#managedBlock(java.util.concurrent.ForkJoinPool.ManagedBlocker)
) call)? Parallel `Stream` operations are executed in the common `ForkJoinPool` implicitly, as well
as the lambdas passed into `CompletableFuture`’s methods whose names end with "Async" but not
accepting a custom executor.

Note that attaching blocking or I/O operations to a `CompletableFuture` as stage in *default
execution mode* (via methods like `thenAccept()`, `thenApply()`, `handle()`, etc.) might also
inadvertently lead to performing them in `ForkJoinPool` from where the future may be completed: see
[TE.7](#cf-beware-non-async).

This advice should not be taken too far: occasional transient IO (such as that may happen during
logging) and operations that may rarely block (such as `ConcurrentHashMap.put()` calls) usually
shouldn’t disqualify all their callers from execution in a `ForkJoinPool` or in a parallel `Stream`.
See [Parallel Stream Guidance](http://gee.cs.oswego.edu/dl/html/StreamParallelGuidance.html) for the
more detailed discussion of these tradeoffs.

See also [the section about parallel Streams](#parallel-streams).

<a name="use-common-fjp"></a>
[#](#use-common-fjp) TE.5. An opposite of the previous item: **can non-blocking computations be
parallelized or executed asynchronously by submitting tasks to `ForkJoinPool.commonPool()` or via
parallel Streams instead of using a custom thread pool** (e. g. created by one of the static factory
methods from `ExecutorServices`)? Unless the custom thread pool is configured with a `ThreadFactory`
that specifies a non-default priority for threads or a custom exception handler (see
[TE.1](#name-threads)) there is little reason to create more threads in the system instead of
reusing threads of the common `ForkJoinPool`.

<a name="explicit-shutdown"></a>
[#](#explicit-shutdown) TE.6. Is every **`ExecutorService` treated as a resource and is shutdown
explicitly in the `close()` method of the containing object**, or in a try-with-resources of a
try-finally statement? Failure to shutdown an `ExecutorService` might lead to a thread leak even if
an `ExecutorService` object is no longer accessible, because some implementations (such as
`ThreadPoolExecutor`) shutdown themselves in a finalizer, [while `finalize()` is not guaranteed to
ever be called](
https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/lang/Object.html#finalize()) by
the JVM.

<a name="cf-beware-non-async"></a>
[#](#cf-beware-non-async) TE.7. Are **non-async stages attached to a `CompletableFuture` simple and
non-blocking** unless the future is [completed](
https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/concurrent/CompletableFuture.html#complete(T)
) from a thread in the same thread pool as the thread from where a `CompletionStage` is attached?
This also applies when asynchronous callback is attached using Guava's
`ListenableFuture.addListener()` or [`Futures.addCallback()`](
https://guava.dev/releases/28.1-jre/api/docs/com/google/common/util/concurrent/Futures.html#addCallback-com.google.common.util.concurrent.ListenableFuture-com.google.common.util.concurrent.FutureCallback-java.util.concurrent.Executor-
) methods and [`directExecutor()`](
https://guava.dev/releases/28.1-jre/api/docs/com/google/common/util/concurrent/MoreExecutors.html#directExecutor--
) (or an equivalent) is provided as the executor for the callback.

Non-async execution is called *default execution* (or *default mode*) in the documentation for
[`CompletionStage`](
https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/concurrent/CompletionStage.html):
these are methods `thenApply()`, `thenAccept()`, `thenRun()`, `handle()`, etc. Such stages may be
executed *either* from the thread adding a stage, or from the thread calling `future.complete()`.

If the `CompletableFuture` originates from a library, it's usually unknown in which thread
(executor) it is completed, therefore, chaining a heavyweight, blocking, or I/O operation to such a
future might lead to problems described in [TE.3](#cached-thread-pool-no-io),
[TE.4](#fjp-no-blocking) (if the future is completed from a `ForkJoinPool` or an event loop executor
in an asynchronous library), and [Dl.4](#non-open-call).

Even if the `CompletableFuture` is created in the same codebase as stages attached to it, but is
completed in a different thread pool, default stage execution mode leads to non-deterministic
scheduling of operations. If these operations are heavyweight and/or blocking, this reduces the
system's operational predictability and robustness, and also creates a subtle dependency between the
components: e. g. if the component which completes the future decides to migrate this action to a
`ForkJoinPool`, it could suddenly lead to the problems described in the previous paragraph.

A lightweight, non-blocking operation which is OK to attach to a `CompletableFuture` as a non-async
stage may be something like incrementing an `AtomicInteger` counter, adding an element to a
non-blocking queue, putting a value into a `ConcurentHashMap`, or a logging statement.

It's also fine to attach a stage in the default mode if preceded with `if (future.isDone())` check
which guarantees that the completion stage will be executed immediately in the current thread
(assuming the current thread belongs to the proper thread pool to perform the stage action).

The specific reason(s) making non-async stage or callback attachment permissible (future completion
and stage attachment happening in the same thread pool; simple/non-blocking callback; or
`future.isDone()` check) should be identified in a comment.

See also the Javadoc for [`ListenableFuture.addListener()`](
https://guava.dev/releases/28.1-jre/api/docs/com/google/common/util/concurrent/ListenableFuture.html#addListener-java.lang.Runnable-java.util.concurrent.Executor-
) describing this problem.

### Parallel Streams

<a name="justify-parallel-stream-use"></a>
[#](#justify-parallel-stream-use) PS.1. For every use of parallel Streams via
`Collection.parallelStream()` or `Stream.parallel()`: **is it explained why parallel `Stream` is
used in a comment preceding the stream operation?** Are there back-of-the-envelope calculations or
references to benchmarks showing that the total CPU time cost of the parallelized computation
exceeds [100 microseconds](http://gee.cs.oswego.edu/dl/html/StreamParallelGuidance.html)?

Is there a note in the comment that parallelized operations are generally I/O-free and non-blocking,
as per [TE.4](#fjp-no-blocking)? The latter might be obvious momentary, but as codebase evolves the
logic that is called from the parallel stream operation might become blocking accidentally. Without
comment, it’s harder to notice the discrepancy and the fact that the computation is no longer a good
fit for parallel Streams. It can be fixed by calling the non-blocking version of the logic again or
by using a simple sequential `Stream` instead of a parallel `Stream`.

### Thread interruption and `Future` cancellation

<a name="restore-interruption"></a>
[#](#restore-interruption) IF.1. If some code propagates `InterruptedException` wrapped into another
exception (e. g. `RuntimeException`), is **the interruption status of the current thread restored
before the wrapping exception is thrown?**

Propagating `InterruptedException` wrapped into another exception is a controversial practice
(especially in libraries) and it may be prohibited in some projects completely, or in specific
subsystems.

<a name="interruption-swallowing"></a>
[#](#interruption-swallowing) IF.2. If some method **returns normally after catching an
`InterruptedException`**, is this coherent with the (documented) semantics of the method? Returning
normally after catching an `InterruptedException` usually makes sense only in two types of methods:

 - `Runnable.run()` or `Callable.call()` themselves, or methods that are intended to be submitted as
 tasks to some Executors as method references. `Thread.currentThread().interrupt()` should still be
 called before returning from the method, assuming that the interruption policy of the threads in
 the `Executor` is unknown.
 - Methods with "try" or "best effort" semantics. Documentation for such methods should be clear
 that they stop attempting to do something when the thread is interrupted, restore the interruption
 status of the thread and return. For example, `log()` or `sendMetric()` could probably be such
 methods, as well as `boolean trySendMoney()`, but not `void sendMoney()`.

If a method doesn’t fall into either of these categories, it should propagate `InterruptedException`
directly or wrapped into another exception (see the previous item), or it should not return normally
after catching an `InterruptedException`, but rather continue execution in some sort of retry loop,
saving the interruption status and restoring it before returning (see an [example](
http://jcip.net/listings/NoncancelableTask.java) from JCIP). Fortunately, in most situations, it’s
not needed to write such boilerplate code: **one of the methods from [`Uninterruptibles`](
https://google.github.io/guava/releases/27.0.1-jre/api/docs/com/google/common/util/concurrent/Uninterruptibles.html
) utility class from Guava can be used.**

<a name="cancel-future"></a>
[#](#cancel-future) IF.3. If an **`InterruptedException` or a `TimeoutException` is caught on a
`Future.get()` call** and the task behind the future doesn’t have side effects, i. e. `get()` is
called only to obtain and use the result in the context of the current thread rather than achieve
some side effect, is the future [canceled](
https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/concurrent/Future.html#cancel(boolean))?

See [JCIP 7.1] for more information about thread interruption and task cancellation.

### Time

<a name="nano-time-overflow"></a>
[#](#nano-time-overflow) Tm.1. Are values returned from **`System.nanoTime()` compared in an
overflow-aware manner**, as described in [the documentation](
https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/lang/System.html#nanoTime()) for
this method?

<a name="time-going-backward"></a>
[#](#time-going-backward) Tm.2. **Isn't `System.currentTimeMillis()` used for time comparisons,
timed blocking, measuring intervals, timeouts, etc.?** `System.currentTimeMillis()` is a subject to
the "time going backward" phenomenon. This might happen due to a time correction on a server, for
example.

`System.nanoTime()` should be used instead of `currentTimeMillis()` for the purposes of time
comparision, interval measurements, etc. Values returned from `nanoTime()` never decrease (but may
overflow — see the previous item). Warning: `nanoTime()` didn’t always uphold to this guarantee in
OpenJDK until 8u192 (see [JDK-8184271](https://bugs.openjdk.java.net/browse/JDK-8184271)). Make sure
to use the freshest distribution.

In distributed systems, the [leap second](https://en.wikipedia.org/wiki/Leap_second) adjustment
causes similar issues.

<a name="time-units"></a>
[#](#time-units) Tm.3. Do **variables that store time limits and periods have suffixes identifying
their units**, for example, "timeoutMillis" (also -Seconds, -Micros, -Nanos) rather than just
"timeout"? In method and constructor parameters, an alternative is providing a [`TimeUnit`](
https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/concurrent/TimeUnit.html)
parameter next to a "timeout" parameter. This is the preferred option for public APIs.

<a name="treat-negative-timeout-as-zero"></a>
[#](#treat-negative-timeout-as-zero) Tm.4. **Do methods that have "timeout" and "delay" parameters
treat negative arguments as zeros?** This is to obey the principle of least astonishment because all
timed blocking methods in classes from `java.util.concurrent.*` follow this convention.

### Thread safety of Cleaners and native code

<a name="thread-safe-close-with-cleaner"></a>
[#](#thread-safe-close-with-cleaner) CN.1. If a class manages native resources and employs
`java.lang.ref.Cleaner` (or `sun.misc.Cleaner`; or overrides `Object.finalize()`) to ensure that
resources are freed when objects of the class are garbage collected, and the class implements
`Closeable` with the same cleanup logic executed from `close()` directly rather than through
`Cleanable.clean()` (or `sun.misc.Cleaner.clean()`) to be able to distinguish between explicit
`close()` and cleanup through a cleaner (for example, `clean()` can log a warning about the object
not being closed explicitly before freeing the resources), is it ensured that even if the **cleanup
logic is called concurrently from multiple threads, the actual cleanup is performed only once**? The
cleanup logic in such classes should obviously be idempotent because it’s usually expected to be
called twice: the first time from the `close()` method and the second time from the cleaner or
`finalize()`. The catch is that the cleanup *must be concurrently idempotent, even if `close()` is
never called concurrently on objects of the class*. That’s because the garbage collector may
consider the object to become unreachable before the end of a `close()` call and initiate cleanup
through the cleaner or `finalize()` while `close()` is still being executed.

Alternatively, `close()` could simply delegate to `Cleanable.clean()` (`sun.misc.Cleaner.clean()`)
which is concurrently idempotent itself. But then it’s impossible to distinguish between explicit
and automatic cleanup.

See also [JLS 12.6.2](https://docs.oracle.com/javase/specs/jls/se11/html/jls-12.html#jls-12.6.2).

<a name="reachability-fence"></a>
[#](#reachability-fence) CN.2. In a class with some native state that has a cleaner or overrides
`finalize()`, are **bodies of all methods that interact with the native state wrapped with
`try { ... } finally { Reference.reachabilityFence(this); }`**,
including constructors and the `close()` method, but excluding `finalize()`? This is needed because
an object could become unreachable and the native memory might be freed from the cleaner while the
method that interacts with the native state is being executed, that might lead to use-after-free or
JVM memory corruption.

`reachabilityFence()` in `close()` also eliminates the race between `close()` and the cleanup
executed through the cleaner or `finalize()` (see the previous item), but it may be a good idea to
retain the thread safety precautions in the cleanup procedure, especially if the class in question
belongs to the public API of the project because otherwise if `close()` is accidentally or
maliciously called concurrently from multiple threads, the JVM might crash due to double memory free
or, worse, memory might be silently corrupted, while the promise of the Java platform is that
whatever buggy some code is, as long as it passes bytecode verification, thrown exceptions should be
the worst possible outcome, but the virtual machine shouldn’t crash. [CN.4](#thread-safe-native)
also stresses on this principle.

`Reference.reachabilityFence()` has been added in JDK 9. If the project targets JDK 8 and Hotspot
JVM, [any method with an empty body is an effective emulation of `reachabilityFence()`](
http://mail.openjdk.java.net/pipermail/core-libs-dev/2018-February/051312.html).

See the documentation for [`Reference.reachabilityFence()`](
https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/lang/ref/Reference.html#reachabilityFence(java.lang.Object))
and [this discussion](http://cs.oswego.edu/pipermail/concurrency-interest/2015-December/014609.html)
in the concurrency-interest mailing list for more information.

<a name="finalize-misuse"></a>
[#](#finalize-misuse) CN.3. Aren’t there classes that have **cleaners or override `finalize()` not
to free native resources**, but merely to return heap objects to some pools, or merely to report
that some heap objects are not returned to some pools? This is an antipattern because of the
tremendous complexity of using cleaners and `finalize()` correctly (see the previous two items) and
the negative impact on performance (especially of `finalize()`), that might be even larger than the
impact of not returning objects back to some pool and thus slightly increasing the garbage
allocation rate in the application. If the latter issue arises to be any important, it should better
be diagnosed with [async-profiler](https://github.com/jvm-profiling-tools/async-profiler) in the
allocation profiling mode (-e alloc) than by registering cleaners or overriding `finalize()`.

This advice also applies when pooled objects are direct ByteBuffers or other Java wrappers of native
memory chunks. [async-profiler -e malloc](
https://stackoverflow.com/questions/53576163/interpreting-jemaloc-data-possible-off-heap-leak/53598622#53598622)
could be used in such cases to detect direct memory leaks.

<a name="thread-safe-native"></a>
[#](#thread-safe-native) CN.4. If some **classes have some state in native memory and are used
actively in concurrent code, or belong to the public API of the project, was it considered making
them thread-safe**? As described in [CN.2](#reachability-fence), if objects of such classes are
inadvertently accessed from multiple threads without proper synchronization, memory corruption and
JVM crashes might result. This is why classes in the JDK such as [`java.util.zip.Deflater`](
http://hg.openjdk.java.net/jdk/jdk/file/a772e65727c5/src/java.base/share/classes/java/util/zip/Deflater.java)
use synchronization internally despite `Deflater` objects are not intended to be used concurrently
from multiple threads.

Note that making classes with some state in native memory thread-safe also implies that the **native
state should be safely published in constructors**. This means that either the native state should
be stored exclusively in `final` fields, or [`VarHandle.storeStoreFence()`](
https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/lang/invoke/VarHandle.html#storeStoreFence())
should be called in constructors after full initialization of the native state. If the project
targets JDK 9 and `VarHandle` is not available, the same effect could be achieved by wrapping
constructors’ bodies in `synchronized (this) { ... }`.

<hr>

<a name="forbid-jdk-internally-synchronized"></a>
[#](#forbid-jdk-internally-synchronized) Bonus: is [forbidden-apis](
https://github.com/policeman-tools/forbidden-apis) configured for the project and are
`java.util.StringBuffer`, `java.util.Random` and `Math.random()` prohibited? `StringBuffer` and
`Random` are thread-safe and all their methods are synchronized, which is never useful in practice
and only inhibits the performance. In OpenJDK, `Math.random()` delegates to a global static `Random`
instance. `StringBuilder` should be used instead of `StringBuffer`, `ThreadLocalRandom` or
`SplittableRandom` should be used instead of `Random` (see also [T.2](#concurrent-test-random)).

## Reading List

 - [JLS] Java Language Specification, [Memory Model](
 https://docs.oracle.com/javase/specs/jls/se11/html/jls-17.html#jls-17.4) and [`final` field
 semantics](https://docs.oracle.com/javase/specs/jls/se11/html/jls-17.html#jls-17.5).
 - [EJ] "Effective Java" by Joshua Bloch, Chapter 11. Concurrency.
 - [JCIP] "Java Concurrency in Practice" by Brian Goetz, Tim Peierls, Joshua Bloch, Joseph Bowbeer,
 David Holmes, and Doug Lea.
 - Posts by Aleksey Shipilёv:
    - [Safe Publication and Safe Initialization in Java](
    https://shipilev.net/blog/2014/safe-public-construction/)
    - [Java Memory Model Pragmatics](https://shipilev.net/blog/2014/jmm-pragmatics/)
    - [Close Encounters of The Java Memory Model Kind](
    https://shipilev.net/blog/2016/close-encounters-of-jmm-kind/)
 - [When to use parallel streams](http://gee.cs.oswego.edu/dl/html/StreamParallelGuidance.html)
 written by Doug Lea, with the help of Brian Goetz, Paul Sandoz, Aleksey Shipilev, Heinz Kabutz,
 Joe Bowbeer, …
 - [SEI CERT Oracle Coding Standard for Java](
 https://wiki.sei.cmu.edu/confluence/display/java/SEI+CERT+Oracle+Coding+Standard+for+Java):
    - [Rule 08. Visibility and Atomicity (VNA)](
    https://wiki.sei.cmu.edu/confluence/pages/viewpage.action?pageId=88487824)
    - [Rule 09. Locking (LCK)](
    https://wiki.sei.cmu.edu/confluence/pages/viewpage.action?pageId=88487666)
    - [Rec. 18. Concurrency (CON)](
    https://wiki.sei.cmu.edu/confluence/pages/viewpage.action?pageId=88487352)

## Concurrency checklists for other programming langugages

 - C++: [Concurrency and parallelism](
 http://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#cp-concurrency-and-parallelism) section
 in C++ Core Guidelines.
 - Go: [Concurrency](https://golang.org/doc/effective_go.html#concurrency) section in *Effective
 Go*.

<hr>

## Authors

This checklist was originally published as a [post on Medium](
https://medium.com/@leventov/code-review-checklist-java-concurrency-49398c326154).

The following people contributed ideas and comments about this checklist before it was imported to
Github:

 - [Roman Leventov](https://github.com/leventov)
 - [Marko Topolnik](https://stackoverflow.com/users/1103872/marko-topolnik)
 - [Matko Medenjak](https://github.com/mmedenjak)
 - [Chris Vest](https://github.com/chrisvest)
 - [Simon Willnauer](https://github.com/s1monw)
 - [Ben Manes](https://github.com/ben-manes)
 - [Gleb Smirnov](https://github.com/gvsmirnov)
 - [Andrey Satarin](https://github.com/asatarin)
 - [Benedict Jin](https://github.com/asdf2014)
 - [Petr Janeček](https://stackoverflow.com/users/1273080/petr-jane%C4%8Dek)

The ideas for some items are taken from "[Java Concurrency Gotchas](
https://www.slideshare.net/alexmiller/java-concurrency-gotchas-3666977)" presentation by [Alex
Miller](https://github.com/puredanger) and [What is the most frequent concurrency issue you've
encountered in Java?](https://stackoverflow.com/questions/461896) question on StackOverflow (thanks
to Alex Miller who created this question and the contributors).
 
At the moment when this checklist was imported to Github, all text was written by Roman Leventov.
 
The checklist is not considered complete, comments and contributions are welcome!

## No Copyright

This checklist is public domain. By submitting a PR to this repository contributors agree to release
their writing to public domain.
