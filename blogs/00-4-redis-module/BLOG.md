# Six Containers, One Annotation: A Real Redis Sentinel Cluster Inside a JUnit Test — and a Failover You Can Assert On

*Macstab Chaos Stack, Layer 3 — annotation-driven Redis topologies for the failure modes your cache code has never actually survived.*

> 🎥 **Video walkthrough coming soon** — subscribe on **[Macstab Engineering](https://www.youtube.com/@MacstabEngineering)**.

![Macstab Chaos Stack, Layer 3 — a real Redis Sentinel cluster inside a JUnit test](./images/header.png)
*One annotation, six containers, and a promotion clock you can put an assertion around.*

---

**Your Redis is up. It answers `PING`. Its replicas are in sync, every dashboard is green — and it has refused every single write for the last forty minutes.**

A background snapshot failed. `stop-writes-on-bgsave-error` defaults to `yes`, so Redis now rejects every command that could modify the data set. Reads still work perfectly. And because the master still answers `PING` — which is exactly what Sentinel's health check watches — there is no SDOWN, no ODOWN, no election. **No failover. Ever.** The cluster will sit there, healthy by every measure you are monitoring, until a human reads the logs.

Infrastructure chaos cannot produce that: nothing died. A mocked `RedisTemplate` cannot produce it: a mock has no disk. And the code you wrote for the *other* failure — the one where a node actually dies — has never run either. Not once, not in CI, not on your laptop, because rehearsing it honestly means one master, two replicas, three Sentinels, a correct quorum, replication actually converged, and a kill at exactly the right instant. So it doesn't get written. The cache gets mocked, an embedded single-node Redis stands in for a highly-available cluster, and the first time `+switch-master` is ever observed by your client library is at 03:14 in production, in front of an audience.

This article is about the layer that removes that excuse. One class-level annotation:

```java
@RedisSentinel                       // 1 master + 2 replicas + 3 sentinels, quorum 2
class FailoverTest { }
```

Six containers on a private Docker network, in the correct startup order, with a Sentinel config tuned so that a failover completes inside a CI budget rather than a coffee break — and a `Duration triggerFailover()` you can put an `assertThat(...).isLessThan(...)` around. And the cheap end of the same annotation family:

```java
@RedisStandalone                     // one real Redis 7.4, random port, auto-torn-down
class CacheTest { }
```

That is the whole setup. No `@Testcontainers`, no `static GenericContainer<?> redis = ...`, no `@DynamicPropertySource`, no `@BeforeAll`, no `stop()` in an `@AfterAll` that you forgot to write.

This is the third layer of the Macstab Chaos Stack — beneath it sit [six C99 `LD_PRELOAD` libraries](https://github.com/macstab/macstab-chaos-testing-libraries) that rewrite the libc/kernel contract and a JVM bytecode agent that injects faults at the Java method level; this layer contributes the thing neither of those can, *a real production-shaped topology to inject them into*. Read it at whatever depth you need: §1 gets you a Redis container in a test in ninety seconds and shows the ninety lines of Testcontainers plumbing the annotation replaces; the rest goes all the way down — Sentinel's discovery protocol and why it fights Docker port mapping, quorum versus election majority, SDOWN/ODOWN, TILT mode, the snapshot failure from the top of this article reproduced in ten seconds, and the exact places where this module's abstraction is thinner than its Javadoc suggests.

---

## 1. The easy part — and the ninety lines it replaces

One dependency. The module is published from [`github.com/macstab/chaos-testing`](https://github.com/macstab/chaos-testing) under the `com.macstab.chaos` group:

```groovy
// Gradle
testImplementation 'com.macstab.chaos:macstab-chaos-redis:1.0.0'
```

```xml
<!-- Maven -->
<dependency>
    <groupId>com.macstab.chaos</groupId>
    <artifactId>macstab-chaos-redis</artifactId>
    <version>1.0.0</version>
    <scope>test</scope>
</dependency>
```

One annotation, one test:

```java
@RedisStandalone
class CacheTest {

    @Test
    void redisAnswers(StandaloneRedis redis) {
        RedisClient client = RedisClient.create(
                RedisURI.builder().withHost(redis.host()).withPort(redis.port()).build());
        try (var conn = client.connect()) {
            assertThat(conn.sync().ping()).isEqualTo("PONG");
            conn.sync().set("k", "v");
            assertThat(conn.sync().get("k")).isEqualTo("v");
        } finally {
            client.shutdown();
        }
    }
}
```

`StandaloneRedis` is a two-component [record](https://openjdk.org/jeps/395) — `host`, `port` — resolved as a test-method parameter. The container is a real `redis:7.4` image on a random host port, started once before the class and stopped after it. Nothing else is required: no `@Testcontainers` on the class, no static field, no `@ExtendWith`, no lifecycle code of your own.

### 1.1 What that one line replaces

The claim "insanely easy" only means something against the alternative, so here is the alternative. Hand-rolled Testcontainers, standalone — the version everyone has written at least once:

```java
@Testcontainers
class CacheTest {

    @Container
    static final GenericContainer<?> REDIS =
            new GenericContainer<>(DockerImageName.parse("redis:7.4"))
                    .withExposedPorts(6379)
                    .waitingFor(Wait.forLogMessage(".*Ready to accept connections.*\\n", 1));

    static String host() { return REDIS.getHost(); }
    static int    port() { return REDIS.getMappedPort(6379); }

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry r) {
        r.add("spring.data.redis.host", CacheTest::host);
        r.add("spring.data.redis.port", CacheTest::port);
    }

    @Test void redisAnswers() { /* … build a client from host()/port() … */ }
}
```

Twenty lines, repeated in every test class that needs a Redis, each copy free to drift — a different image tag here, a missing wait strategy there, a `@Container` someone made non-static so it restarts per method and triples the suite runtime. Versus:

```java
@RedisStandalone
class CacheTest {
    @Test void redisAnswers(StandaloneRedis redis) { /* … */ }
}
```

Now the Sentinel cluster. This is the honest hand-rolled equivalent of `@RedisSentinel` — condensed, with **no** error handling, **no** parallel startup, **no** cleanup-on-partial-failure, **no** role detection and **no** failover helper:

```java
Network net = Network.newNetwork();

GenericContainer<?> master = new GenericContainer<>("redis:7-alpine")
        .withNetwork(net).withNetworkAliases("redis-master")
        .withExposedPorts(6379)
        .withCommand("redis-server", "--protected-mode", "no")
        .waitingFor(Wait.forLogMessage(".*Ready to accept connections.*\\n", 1));
master.start();

List<GenericContainer<?>> replicas = new ArrayList<>();
for (int i = 1; i <= 2; i++) {
GenericContainer<?> r = new GenericContainer<>("redis:7-alpine")
        .withNetwork(net).withNetworkAliases("redis-replica" + i)
        .withExposedPorts(6379)
        .withCommand("redis-server", "--protected-mode", "no",
                "--replicaof", "redis-master", "6379")
        // NOT "port is open" and NOT PING — a loading replica answers both
        .waitingFor(Wait.forLogMessage(".*MASTER <-> REPLICA sync: Finished with success.*\\n", 1));
  r.start();
  replicas.add(r);
}

// Sentinels must be told an address that every OTHER participant can also use,
// because that address is gossiped and handed to clients — so: the internal IP,
// not the alias, not the mapped port.
String masterIp = master.getContainerInfo().getNetworkSettings()
        .getNetworks().values().iterator().next().getIpAddress();

int quorum = (3 / 2) + 1;
String conf = "printf \"port 26379\\n"
        + "sentinel monitor mymaster " + masterIp + " 6379 " + quorum + "\\n"
        + "sentinel down-after-milliseconds mymaster 2000\\n"   // NOT the 30000 default,
        + "sentinel parallel-syncs mymaster 1\\n"               // or no test can assert
        + "sentinel failover-timeout mymaster 5000\\n"          // a failover in CI time
        + "\" > /tmp/sentinel.conf && redis-server /tmp/sentinel.conf --sentinel";

List<GenericContainer<?>> sentinels = new ArrayList<>();
for (int i = 1; i <= 3; i++) {
GenericContainer<?> s = new GenericContainer<>("redis:7-alpine")
        .withNetwork(net).withNetworkAliases("sentinel" + i)
        .withExposedPorts(26379)
        .withCommand("sh", "-c", conf)
        .waitingFor(Wait.forSuccessfulCommand("redis-cli -p 26379 SENTINEL master mymaster"));
  s.start();
  sentinels.add(s);
}

        Thread.sleep(2000);   // __sentinel__:hello convergence — see §6.2

// … and now write the teardown, in reverse order, in an @AfterAll that must run
// even when startup threw half-way through. Then write role detection. Then write
// the failover helper. Then discover, on macOS, that none of it connects (§6.5).
```

That is roughly **ninety lines of infrastructure you now own and maintain** — every line of which encodes a piece of Redis operational knowledge (which log line means "synced", why the IP and not the alias, why 2 000 and not 30 000) that is invisible in the code and will be silently wrong the first time someone copy-pastes it. Versus:

```java
@RedisSentinel
class FailoverTest { }
```

Same six containers, same startup order, same tuned config — plus parallel startup, failure isolation, budget validation, automatic teardown, role detection and a failover clock. The ease is not that the annotation is short; it is that **the operational knowledge moved into the framework**, where it is written once, tested, and version-controlled, instead of being re-derived per test class from a Stack Overflow answer.

### 1.2 It runs

Everything below was executed while writing this article, on a macOS host with Docker Desktop 28.3.2, Java 25, against `macstab-chaos-redis` at `1.0.0`:

```sh
./gradlew :macstab-chaos-redis:test \
  --tests "*StandaloneExtensionLifecycleTest*" \
  --tests "*MultiInstanceStandaloneExtensionTest*"
```

```
RedisContainerExtension — Standalone Lifecycle Integration > Container is actually running Redis > should accept connections on mapped port PASSED
RedisContainerExtension — Standalone Lifecycle Integration > Container is actually running Redis > should support basic SET and GET PASSED
RedisContainerExtension — Standalone Lifecycle Integration > StandaloneRedis parameter injection > should inject StandaloneRedis parameter PASSED
RedisContainerExtension — Standalone Lifecycle Integration > StandaloneRedis parameter injection > should inject List<StandaloneRedis> parameter (single element) PASSED
RedisContainerExtension — Standalone Lifecycle Integration > INSTANCE programmatic access > RedisStandalone.INSTANCE.get(id) should return connection info PASSED
StandaloneStartupOrchestrator — Multi-Instance Integration > Parallel startup > should start all instances and return correct count PASSED
StandaloneStartupOrchestrator — Multi-Instance Integration > Parallel startup > should preserve annotation declaration order PASSED
StandaloneStartupOrchestrator — Multi-Instance Integration > Data isolation > should isolate data between instances PASSED
...
BUILD SUCCESSFUL in 16s
```

![FIG-T1 · the real run](./images/term-01.png)

*Captured output, not a mock-up: eleven tests, two test classes, three Redis containers.*

Eleven tests, two test classes, three Redis containers, sixteen seconds wall clock including compilation and the JaCoCo report — on a laptop, through a Docker Desktop VM, on the slowest supported platform.

---

## 2. What the annotation actually is

`@RedisStandalone` is not magic and it is worth knowing exactly what it is, because everything else in this module composes out of the same four lines:

```java
@ChaosTest                                   // meta: registers ChaosTestingExtension
@ExtendWith(RedisContainerExtension.class)   // the container lifecycle owner
@Repeatable(RedisStandalones.class)          // JLS §9.6.3 repeating annotation
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface RedisStandalone { ... }
```

Four decisions, each load-bearing.

**`@ExtendWith` on the annotation, not on your class.** JUnit 5 resolves `@ExtendWith` as a [meta-annotation](https://junit.org/junit5/docs/current/user-guide/#extensions-registration-declarative): declaring `@RedisStandalone` registers `RedisContainerExtension` transitively. Worth knowing *why* that is a framework feature and not a language one — Java has no annotation inheritance in this direction. [`@Inherited`](https://docs.oracle.com/javase/specs/jls/se21/html/jls-9.html#jls-9.6.4.3) (JLS §9.6.4.3) propagates an annotation from a superclass to its subclasses and nothing else; it says nothing about annotations *on* annotations. So JUnit implements the meta-annotation search itself, walking the annotation graph of the test class with cycle detection. Spring does the same thing for `@SpringBootTest`; so does Micronaut; so does Quarkus. Every "composed annotation" you have ever used is a framework working around that one gap in the JLS.

The gain is the [Extension Object pattern](https://junit.org/junit5/docs/current/user-guide/#extensions) in its intended form: your test class declares *what it needs*, never *how it is wired*, and the extension can never be registered without the configuration that parameterises it — because the configuration **is** the registration.

**`@ChaosTest` is the framework's single registration point.** It is an internal meta-annotation that pulls in `ChaosTestingExtension`, the stack-wide orchestrator shared by every chaos annotation in the framework (`@RedisStandalone`, `@RedisSentinel`, the L3 incident packs, the resource controls). Adding an eleventh container annotation is one line, not eleven copies of `@ExtendWith`. It also means the Redis annotations sit in the same lifecycle as syscall-level and JVM-level fault injection — the layers below can be switched on inside the same test class without a second lifecycle fighting the first.

**`@Target(ElementType.TYPE)` — class scope, deliberately.** A container is a ~200 ms Docker API call plus process start; per-method provisioning would multiply that by your test count. The container is created in `beforeAll` and shared by every method in the class. The corollary is the classic Testcontainers discipline: state leaks between test methods unless you clean it (`FLUSHALL` in `@AfterEach`, or namespaced keys per test). The module does not do this for you, and it should not — silently flushing a database the test may be asserting against is worse than an explicit line.

**`@Repeatable`.** Write the annotation twice and the *compiler* synthesizes the `@RedisStandalones` **containing annotation type** around both — [JLS §9.6.3](https://docs.oracle.com/javase/specs/jls/se21/html/jls-9.html#jls-9.6.3) defines the repeatable type, [JLS §9.7.5](https://docs.oracle.com/javase/specs/jls/se21/html/jls-9.html#jls-9.7.5) defines the rewrite of multiple annotations of the same type into that container. The spec imposes constraints the module has to satisfy: the containing type's `value()` must be an array of the repeatable type, its retention must be **at least as long** as the repeatable type's (both `RUNTIME` here, or the whole thing vanishes before the extension can read it), and its `@Target` must be a subset. Get any of those wrong and it is a compile error, not a runtime surprise — which is the good case.

The consequence for the reading code is that two representations exist for the same declaration, and reflection does not unify them. You never write `@RedisStandalones` yourself; the extension reads whichever form is present:

```java
private RedisStandalone[] extractAnnotations(final ExtensionContext context) {
    final Class<?> testClass = context.getRequiredTestClass();
    final RedisStandalones container = testClass.getAnnotation(RedisStandalones.class);
    if (container != null) {
        return container.value();          // 2..n annotations → compiler-synthesized container
    }
    final RedisStandalone single = testClass.getAnnotation(RedisStandalone.class);
    return single != null ? new RedisStandalone[] {single} : new RedisStandalone[0];
}
```

### The lifecycle, and the one part that is not where you'd expect

```
beforeAll(context)
  ├─ CURRENT_CONTEXT.set(context)                  ← ThreadLocal for static accessors
  ├─ extractAnnotations(context)                   ← 1..n @RedisStandalone
  ├─ ResourceBudget.validateStandaloneBudget(...)  ← fail fast, before any Docker call
  ├─ DependencyVerifier.requireCacheModule()       ← only if enableNetworkChaos=true
  ├─ DependencyVerifier.requireConnectionModule()  ← only if enableConnectionChaos=true
  ├─ StandaloneStartupOrchestrator.start(...)      ← parallel start, failure isolation
  └─ Store.put(CONTAINER_MAP_KEY, …)               ← CloseableStoreMap (CloseableResource)

afterAll(context)
  └─ CURRENT_CONTEXT.remove()                      ← that is ALL it does
```

![FIG-02 · extension lifecycle, and where teardown really happens](./images/redis-02-lifecycle.png)

*The meta-annotation search, the guard clause that runs before Docker is touched, and the two independent cleanup paths.*

Note what `afterAll` does *not* do: it does not stop containers. Cleanup rides on JUnit's [`ExtensionContext.Store.CloseableResource`](https://junit.org/junit5/docs/current/api/org.junit.jupiter.api/org/junit/jupiter/api/extension/ExtensionContext.Store.CloseableResource.html) contract — the store closes every `CloseableResource` it holds when its scope ends, in reverse insertion order, *after* all `afterAll` callbacks have run. The wrapper exists for a subtle reason spelled out in the source: JUnit only calls `close()` on objects stored *directly*, so a bare `Map<String, Store>` of containers would never be closed. Hence `CloseableStoreMap`, whose sole job is to fan `close()` out across the map:

```java
private static final class CloseableStoreMap implements ExtensionContext.Store.CloseableResource {
    @Override public void close() {
        log.info("Cleaning up {} standalone Redis container(s)", stores.size());
        stores.values().forEach(Store::close);       // → container.stop()
    }
}
```

Below that sits Testcontainers' own [Ryuk resource reaper](https://java.testcontainers.org/features/garbage_collector/) — a sidecar that holds a socket to the JVM and reaps every labelled container when that socket closes. So containers die when the test class ends, and they also die when you `kill -9` the Gradle worker, and when your IDE terminates the run. Two independent cleanup paths, which is the correct number for something that costs money in CI minutes.

---

## 3. Getting a handle: parameter injection, and the static back door

The extension implements [`ParameterResolver`](https://junit.org/junit5/docs/current/user-guide/#extensions-parameter-resolution), so four parameter shapes are injectable on any `@Test`, `@BeforeAll`, `@BeforeEach` or `@AfterEach` method:

| Parameter type | Resolves to |
|---|---|
| `StandaloneRedis` | the single instance (or the one with `id="default"`) |
| `List<StandaloneRedis>` | every instance, **in annotation declaration order** |
| `RedisConnectionInfo` | the same thing in the extension's internal type |
| `List<RedisConnectionInfo>` | same, all instances |

For Sentinel, symmetrically: `SentinelCluster`, `List<SentinelCluster>`, `SentinelRedis`, `List<SentinelRedis>`. `SentinelCluster` is the live handle (containers, control facade, failover); `SentinelRedis` is the immutable connection-info record you can hand to a client factory.

Declaration order is not an accident — it is preserved through a `LinkedHashMap` in the orchestrator and re-derived from the annotation array on read:

```java
for (final RedisStandalone annotation : annotations) {   // annotation order, not map order
final RedisConnectionInfo instance = instances.get(annotation.id());
  if (instance != null) { ordered.add(instance); }
        }
```

So `all.get(0)` is the first annotation you wrote. That is asserted by the module's own integration test, and it passed in the run above.

There is also a static accessor, for the code that is not a test method — a `@BeforeAll` helper, a Spring `@DynamicPropertySource`, a fixture builder three frames down:

```java
StandaloneRedis cache   = RedisStandalone.INSTANCE.get("cache");
SentinelRedis   session = RedisSentinel.INSTANCE.get("session");
List<Redis>     all     = Redis.getAll();   // sealed interface over both topologies
```

`Redis` is a `sealed interface ... permits StandaloneRedis, SentinelRedis`, so a switch over topologies is exhaustive without a default branch:

```java
String describe(Redis redis) {
    return switch (redis) {
        case StandaloneRedis s -> "standalone " + s.host() + ":" + s.port();
        case SentinelRedis   s -> "sentinel master " + s.masterName() + " via " + s.sentinels();
    };
}
```

**How the static accessor works, and what that costs.** The two access routes are the two classic answers to the same question, and the module ships both on purpose. Parameter injection is **dependency injection** — the container is handed to you, the dependency is visible in the method signature, and nothing resolves unless the framework is active. `INSTANCE.get(id)` is a **service locator** — you reach out and ask a global for it. Service locator is the [pattern Fowler argues against](https://martinfowler.com/articles/injection.html) for production wiring, for exactly the reason that applies here: it hides the dependency and moves the failure from compile time to run time.

It earns its place anyway, because a test fixture is not production wiring: a Spring `@DynamicPropertySource` is `static` and takes a `DynamicPropertyRegistry`, so no injection point exists for it — locator or nothing. The implementation is a `ContainerManager` whose lookup calls `RedisContainerExtension.getContainer(id)`, which reads a `ThreadLocal<ExtensionContext>` set in `beforeAll`: a **thread-confined ambient context**, the same shape as `SecurityContextHolder` or `TransactionSynchronizationManager`, and with the same failure mode. Stated plainly, because it is the kind of thing that costs an afternoon:

- Called outside a `@RedisStandalone` class you get `IllegalStateException: Called outside @RedisStandalone test context` — clear, not a `NullPointerException`.
- Called from a thread that is not the JUnit worker (an executor you spawned, a reactive scheduler), the `ThreadLocal` is empty and you get the same exception. Resolve the handle on the test thread and pass it in.
- The `ThreadLocal` is cleaned twice — in `afterAll`, and again by a `ThreadLocalCleanup` `CloseableResource` — because JUnit worker threads are pooled and a leaked context is a leaked `ExtensionContext` graph.
- `get("default")` is forgiving in exactly one direction: with a single instance it returns that instance whatever its id; with several instances and no `id="default"` it throws and *names the ids that do exist*.

---

## 4. Several Redises, started in parallel, with failure isolation

Repeat the annotation and you get independent instances:

```java
@RedisStandalone(id = "cache",        version = "7.4")
@RedisStandalone(id = "session",      version = "7.2")
@RedisStandalone(id = "rate-limiter")
class MultiStoreTest {

    @Test
    void storesAreIsolated() {
        var cache   = RedisStandalone.INSTANCE.get("cache");
        var session = RedisStandalone.INSTANCE.get("session");
        // same key, two Redises, two answers
    }
}
```

They start concurrently. The orchestrator is small enough to quote in full intent:

```java
final ExecutorService executor = Executors.newFixedThreadPool(annotations.length);
final List<Future<StartupResult>> futures = submitStartupTasks(executor, annotations);
// … collect with a per-instance timeout, then:
if (!failures.isEmpty()) {
final List<String> cleanupActions = cleanupInstances(toCleanup);   // stop the ones that DID start
  throw new ClusterStartupException(failures, cleanupActions);
}
```

That is **scatter–gather** with a **bulkhead**: fan the work out, collect every outcome independently, and never let one failing unit sink the others or the reporting. Four properties fall out of it, and each is a real-world CI failure mode someone has already paid for:

**Bounded, not unbounded, waiting.** `future.get(60, SECONDS)` per standalone instance (120 s per Sentinel cluster). A Docker daemon that has wedged on an image pull produces a named, timestamped failure — `Instance 'cache' startup timed out` — rather than a build that sits at 0 % until the CI job's global timeout kills it an hour later with no diagnosis.

**Failure isolation, expressed as a type.** `StartupResult` is a **sum type** — an algebraic data type in the ML sense, spelled in Java as a [sealed interface](https://openjdk.org/jeps/409) (JEP 409) permitting exactly two [records](https://openjdk.org/jeps/395) (JEP 395), consumed by [pattern matching for `switch`](https://openjdk.org/jeps/441) (JEP 441):

```java
public sealed interface StartupResult permits StartupResult.Success, StartupResult.Failure {
    String instanceId();
    record Success(String instanceId, RedisConnectionInfo connectionInfo, Store store)
            implements StartupResult {}
    record Failure(String instanceId, String errorMessage, Exception error)
            implements StartupResult {}
}
```

```java
switch (result) {
        case StartupResult.Success s -> { successful.put(s.instanceId(), s.connectionInfo()); … }
        case StartupResult.Failure f -> failures.add(new ClusterStartupFailure(f.instanceId(), …));
        }
```

Sealing is what makes that `switch` **exhaustive without a `default`** — the compiler can enumerate the permitted subtypes, so adding a third outcome (`Skipped`, say) turns every consuming switch into a compile error instead of a silently-taken default branch. That is the whole argument for ADTs over the `boolean success` + nullable-fields struct this replaced, and it is a genuine correctness property, not a style preference. One instance failing does not abort collection of the others; every failure is gathered and reported together, so you see *all three* broken configurations on the first run, not one per push.

**Cleanup on partial failure.** If instance 3 of 5 fails, instances 1 and 2 are already running and would otherwise leak until Ryuk noticed. They are stopped explicitly, and *what was stopped* is attached to the thrown `ClusterStartupException` alongside the failures.

**Ordering survives concurrency.** Results are collected by iterating the futures in annotation order, so parallel startup never reorders `List<StandaloneRedis>`.

The measured effect on the two-instance class in the run above: both containers came up concurrently, and the whole class — five tests, two Redis instances, including data-isolation assertions across both — contributed to a sixteen-second total build.

---

## 5. The budget that stops you before Docker does

Before a single Docker API call, `ResourceBudget` validates the request:

```java
public static final int MAX_SENTINEL_CLUSTERS    = 3;
public static final int MAX_STANDALONE_INSTANCES = 5;
public static final int MAX_TOTAL_CONTAINERS     = 20;
public static final int MAX_MEMORY_MB            = 100;
private static final int MEMORY_PER_CONTAINER_MB = 4;   // redis:7-alpine baseline
```

A sixth `@RedisStandalone` on one class, or a fourth `@RedisSentinel`, fails in `beforeAll` with a message that says what the limit is and what to do about it:

```
ResourceBudgetExceededException: Too many Sentinel clusters: 4 (max: 3).
  Each cluster uses 1 master + N replicas + M sentinels.
  Consider reducing cluster count or using standalone Redis for non-HA tests.
```

Mixed topologies are counted the way you'd hope — `1 + replicas + sentinels` per cluster, plus one per standalone — so `@RedisSentinel(replicas=2, sentinels=3)` alongside two `@RedisStandalone` is 8 of 20.

Two honest notes. The memory figure is a *nominal* accounting constant (4 MB × containers, ceiling 100 MB), not a measurement of the host and not a cgroup limit imposed on the containers; an empty `redis:7-alpine` process plus container overhead lands nearer 15–30 MB in practice. Treat the budget as what it is — a guard rail against the misconfiguration where a loop or a copy-paste asks for forty containers — not as a capacity planner. And the budget is *per test class*: ten classes at five instances each, running under Gradle `--parallel`, is fifty containers, and nothing in this module knows that. Your Docker daemon's `pids`, memory and network-per-bridge limits are still yours to respect.

---

## 6. `@RedisSentinel`: what is actually hard here

Everything up to this point is convenience. This is the part that is genuinely difficult, and the reason the module exists.

```java
@RedisSentinel(replicas = 2, sentinels = 3)
class FailoverTest { }
```

![FIG-01 · the topology one annotation provisions](./images/redis-01-topology.png)

*One master, two replicas, three Sentinels, one private bridge network — created for the test class and destroyed with it. Each Sentinel is configured with the master's **internal** address and a quorum; it discovers the other Sentinels and every replica by itself.*

### 6.1 The startup order is not negotiable

[Sentinel](https://redis.io/docs/latest/operate/oss_and_stack/management/sentinel/) monitors a master by address. Start the Sentinels first and they monitor nothing; start them before replication has converged and the first thing they may see is a master with zero healthy replicas — a cluster that cannot fail over. The factory therefore serialises four phases, each with a *semantic* wait strategy rather than a sleep:

| Phase | Command | Ready when |
|---|---|---|
| master | `redis-server --protected-mode no` | log matches `Ready to accept connections` |
| replicas | `redis-server --protected-mode no --replicaof redis-master 6379` | log matches `MASTER <-> REPLICA sync: Finished with success` |
| sentinels | `redis-server /tmp/sentinel.conf --sentinel` | `redis-cli -p 26379 SENTINEL master mymaster` exits 0 |
| stabilisation | — | fixed 2 s |

![FIG-03 · the startup sequence and its wait strategies](./images/redis-03-startup-order.png)

*Four phases, each gated on a semantic signal rather than a sleep — and the one honest `Thread.sleep` at the end.*

The replica wait strategy is the good one. `MASTER <-> REPLICA sync: Finished with success` is the log line Redis emits at the end of the [full synchronisation](https://redis.io/docs/latest/operate/oss_and_stack/management/replication/) handshake — `PSYNC`, RDB snapshot transfer, backlog catch-up. Waiting on the *port* being open would return true while the replica is still loading the RDB and answering `-LOADING Redis is loading the dataset in memory`. Waiting on `PING` would return true for the same reason. Only that line means "this replica now holds the master's dataset and is streaming the backlog".

The Sentinel wait strategy is `Wait.forSuccessfulCommand(...)` around `SENTINEL master mymaster` — the Sentinel process is up *and* has parsed its config *and* has resolved that master. Then a flat 2 000 ms sleep, honestly labelled in the source: after the `+monitor` event, Sentinels still need a moment to discover each other. Which brings us to the mechanism that makes that sleep necessary.

### 6.2 Why each Sentinel is told only about the master

Look at the config the module writes — it is the entire Sentinel configuration, built by `RedisCommandBuilder.buildSentinelStartCommand()` and `printf`'d into the container at launch:

```
port 26379
sentinel monitor mymaster 172.18.0.2 6379 2
sentinel down-after-milliseconds mymaster 2000
sentinel parallel-syncs mymaster 1
sentinel failover-timeout mymaster 5000
```

Three Sentinels, and not one of them is told about the other two. It doesn't need to be — because the topology is discovered, not configured, by a **publish–subscribe rendezvous** over the very resource being monitored. Sentinel's discovery is [pub/sub-based](https://redis.io/docs/latest/operate/oss_and_stack/management/sentinel/#adding-or-removing-sentinels): every Sentinel publishes a hello message to the `__sentinel__:hello` channel *on the monitored master and its replicas* every two seconds, carrying its own address, run-id, epoch and its view of the master. Every Sentinel subscribes to that channel. So the master itself is the rendezvous point: three Sentinels that each know only the master converge on a complete view of each other within a couple of publish intervals. Replicas are discovered the same way — from the master's `INFO replication` output, which is why the module never has to tell Sentinel about the replicas either.

![FIG-04 · hello-channel discovery](./images/redis-04-hello-discovery.png)

*Each Sentinel boots knowing one address. The master is the rendezvous: everything else — the other Sentinels, every replica — is discovered over `__sentinel__:hello` and `INFO replication`.*

That is the whole reason for the 2-second stabilisation wait — and the honest caveat about it. Redis's documentation says a newly started Sentinel acquires the list of other Sentinels and the replica set "within 10 seconds", so two seconds is a pragmatic floor stacked on top of the per-Sentinel `SENTINEL master mymaster` readiness probe, not a guarantee that mutual discovery has completed. Every Sentinel is verifiably monitoring the master when the cluster is handed to your test; whether all three have seen each other yet is a race you should not write assertions against in the first millisecond. Assert on the cluster's *behaviour* (a failover completes) rather than on `num-other-sentinels` at t=0.

It is also the reason the module passes the master's **internal Docker IP** rather than the network alias `redis-master`: the address in `sentinel monitor` is not just a connection target, it is the identity that gets gossiped to every other Sentinel and handed to every client that asks. An alias that only resolves inside one container's DNS view would propagate as an address other participants cannot use.

```java
static String getMasterIpAddress(final GenericContainer<?> master) {
    return master.getContainerInfo().getNetworkSettings().getNetworks().values().stream()
            .findFirst()
            .orElseThrow(() -> new RuntimeException("Master container has no network"))
            .getIpAddress();
}
```

### 6.3 The timing constants are the interesting part

Compare what the module writes with Redis's own defaults:

| Directive | Redis default | This module | Why |
|---|---|---|---|
| `down-after-milliseconds` | 30 000 | **2 000** | time to SDOWN; dominates failover latency |
| `failover-timeout` | 180 000 | **5 000** | bounds each failover state, and the retry interval |
| `parallel-syncs` | 1 | 1 | replicas allowed to resync with the new master *simultaneously* — 1 means one at a time, so the rest keep serving (stale) reads instead of all blocking on a full sync at once |
| `quorum` | — | `(sentinels / 2) + 1` | 3 → 2, 5 → 3 |

Those "Redis default" numbers are not from memory — here they are, read out of a real Redis 7.4.8 Sentinel that was given nothing but a `sentinel monitor` line:

![FIG-T3 · Sentinel's actual defaults](./images/term-03.png)

*`SENTINEL master mymaster` on an otherwise-unconfigured Sentinel. 30 000 ms to detection, 180 000 ms failover timeout.*

Thirty seconds of `down-after-milliseconds` is correct in production: it is the guard against a GC pause, a slow disk, or a two-second network blip triggering a real failover with real write loss. In a test it means a failover assertion cannot complete inside any sane CI budget, so nobody writes one. Two seconds is the number that makes the assertion possible — and this is exactly the kind of decision that has to be made *once, in the framework*, by someone who knows why the production default is what it is. A test that waits 35 seconds to observe a promotion gets deleted by the third engineer who runs the suite.

The trade is that a 2-second detection window is *tight*. A Docker Desktop VM under load, a laptop that swapped, a CI runner with noisy neighbours — any of those can stall a Redis process past two seconds and produce a spurious `+sdown`. In a failover test that is harmless (the cluster does what it is supposed to do). In a test that asserts *no* failover happened, it is a flake, and you should raise the window for that class rather than pretend it cannot happen.

### 6.4 Quorum is not the election majority — and mixing them up loses data

This is the single most misunderstood parameter in Redis HA, so it is worth being exact.

`quorum` is the number of Sentinels that must independently mark the master as subjectively down (**SDOWN**, one Sentinel's own opinion after `down-after-milliseconds` of no valid reply) before it is promoted to objectively down (**ODOWN**, an agreed verdict, which exists only for masters, never for replicas). ODOWN is what *authorises* a failover attempt.

It is **not** what performs one. The [Sentinel documentation](https://redis.io/docs/latest/operate/oss_and_stack/management/sentinel/#quorum) is unambiguous, and worth quoting exactly:

> However **the quorum is only used to detect the failure**. In order to actually perform a failover, one of the Sentinels need to be elected leader for the failover and be authorized to proceed. This only happens with the vote of the **majority of the Sentinel processes**.

So a failover needs *both*: quorum Sentinels agreeing the master is down (ODOWN), **and** a majority of all known Sentinels voting one of them leader — or more than the majority, if you set quorum higher than the majority. The elected leader receives a unique **configuration epoch** for that failover, which versions the resulting configuration so that two Sentinels can never propagate conflicting views of who the master is. If you have read Raft, the shape will be familiar — epoch as term, one leader per epoch, votes granted at most once — though Redis's own documentation describes the mechanism in its own terms rather than claiming the algorithm. From the [Sentinel documentation](https://redis.io/docs/latest/operate/oss_and_stack/management/sentinel/#quorum): the quorum only decides ODOWN; the majority is always required to elect a leader and start the failover.

The practical consequence: setting `quorum = 1` on five Sentinels does *not* give you a cluster that fails over when one Sentinel is lonely. It gives you fast ODOWN — and then, if only two Sentinels can see each other, no leader, no failover, and a cluster that has decided the master is dead but can do nothing about it. And in the opposite direction — a partition that isolates the master with a minority of Sentinels — the majority requirement is precisely what stops the minority side from promoting a second master and giving you two divergent datasets.

The module derives `quorum = (sentinels / 2) + 1` — 3 → 2, 5 → 3 — which makes ODOWN and the election majority coincide. That is the configuration you want unless you are deliberately testing a split.

**Sharp edge, stated plainly:** the `quorum()` attribute on `@RedisSentinel` is currently *not plumbed through* to the container configuration — the factory always computes the majority formula from `sentinels`. For the default (3 sentinels, quorum 2) the two agree, so nothing is observable; if you set `quorum = 1` expecting a fast-ODOWN cluster, you will not get one. Two neighbouring attributes have the same gap today: `version()` on `@RedisSentinel` is not applied to the cluster nodes (they are always `redis:7-alpine`; `version()` on `@RedisStandalone` *is* honoured), and `masterName()` is stored in the `SentinelCluster`/`SentinelRedis` records but the generated Sentinel config always monitors the name **`mymaster`**. So this compiles, runs, and then fails to connect:

```java
@RedisSentinel(masterName = "ha-master")            // ← record says "ha-master" …
class Wrong {
    @Test void t(SentinelRedis c) {
        var pool = new JedisSentinelPool(c.masterName(), sentinelAddrs(c));   // … Sentinel knows "mymaster"
    }
}
```

Use `"mymaster"` — or read the name from the cluster and pin it in one place — until that attribute is wired. It is a five-line fix in `RedisCommandBuilder`, and it is exactly the sort of thing that a Javadoc-only reading of the module will not tell you.

### 6.5 The address problem: why this is Linux-only, and why that is not laziness

Here is the deep reason Sentinel plus Testcontainers is hard, and why almost nobody ships it.

Sentinel is a **discovery protocol**, not a proxy. A Sentinel-aware client does not connect to Sentinel and get its traffic forwarded; it asks a Sentinel `SENTINEL get-master-addr-by-name mymaster`, receives an address, and then opens a *direct* connection to that address. It also subscribes to `+switch-master` events to learn the new address after a failover.

Inside the test's Docker network, the master's address is `172.18.0.2:6379`. That is what Sentinel knows, gossips, and returns. Your JVM is on the host, and reaches Redis through a *mapped* port — `localhost:54321`. So the sequence is:

```
JVM  ──▶ sentinel (localhost:54873)   "where is mymaster?"
JVM  ◀──                              "172.18.0.2:6379"
JVM  ──▶ 172.18.0.2:6379              ← routable from a Linux host. Not from a Docker Desktop VM.
```

On a **Linux host** with a native Docker daemon, the container bridge network is a real interface on the host, so `172.18.0.2:6379` is genuinely reachable and everything works. Inside a **dev container or CI container** sharing the daemon's network, likewise. On **macOS or Windows**, Docker Desktop runs the daemon inside a lightweight Linux VM; the bridge subnet lives in the VM, the host has no route to it, and every Sentinel-aware client fails at the second hop — after a successful Sentinel handshake, which makes the failure maximally confusing.

![FIG-05 · why the second hop dies on macOS](./images/redis-05-address-problem.png)

*The Sentinel handshake succeeds on both platforms. Only the third step — the direct connection to the address Sentinel returned — depends on whether the container bridge is a real host interface.*

None of this is a Testcontainers quirk — Redis documents it directly, under the heading [*Sentinel, Docker, NAT, and possible issues*](https://redis.io/docs/latest/operate/oss_and_stack/management/sentinel/#sentinel-docker-nat-and-possible-issues): "Docker uses a technique called port mapping: programs running inside Docker containers may be exposed with a different port compared to the one the program believes to be using… Remapping ports and addresses creates issues with Sentinel."

Redis's own answer to this is the announce directives — `replica-announce-ip` / `replica-announce-port` on data nodes and `sentinel announce-ip` / `announce-port` on Sentinels — which decouple *the address a node reports* from *the address it binds*. The module ships builders for exactly that (`SentinelCommandBuilder.configureMasterAnnouncement`, `configureReplicaAnnouncement`, `configureSentinelAnnouncement`) plus the `host.testcontainers.internal:host-gateway` extra host on every node for Docker-in-Docker reachability. Honest status: **those builders are not invoked on the current extension path.** The supported configuration is a Linux host or a container, and the module enforces that rather than letting you discover it through a nine-minute connection timeout:

```java
@DisabledOnNonLinuxHost(
        "Redis Sentinel tests require native Docker networking (Linux host or dev container)")
public @interface RedisSentinel { … }
```

The condition is smarter than JUnit's built-in `@EnabledOnOs(LINUX)`, and for a specific reason: inside a dev container running *on* macOS, `os.name` still reports `Mac OS X`. So it checks for `/.dockerenv` first — if you are in a container, you are on Linux by construction, whatever the JVM thinks — and only then falls back to `os.name`.

Run the Sentinel suite on macOS and you get this, which is the *correct* behaviour: skipped, not failed, not hung.

```sh
./gradlew :macstab-chaos-redis:test --tests "*SentinelFailoverTest*"
```

```
FailoverHelper — Sentinel Failover Integration > Failover simulation > triggerFailover() should complete and elect new master SKIPPED
FailoverHelper — Sentinel Failover Integration > Role detection > getMaster() should return a running container SKIPPED
FailoverHelper — Sentinel Failover Integration > Role detection > getReplicas() should return at least 1 replica container SKIPPED
FailoverHelper — Sentinel Failover Integration > Role detection > getContainer(MASTER) should return running master SKIPPED
BUILD SUCCESSFUL in 2s
```

![FIG-T2 · skipped, with the reason, in two seconds](./images/term-02.png)

*The same test class that runs a six-container failover in CI, on a macOS laptop.*

A red build on a developer laptop for an environmental reason nobody can fix locally is a build people learn to ignore. A skip with a printed reason is a build people keep. Write the Sentinel tests, run them in CI on Linux, and they simply do not execute on the macOS laptop — the same annotation, the same code, no `if` in your test.

---

## 7. Failover as an assertion

This is the payoff — the thing that was previously a week of plumbing:

```java
@RedisSentinel(replicas = 2, sentinels = 3)
class FailoverTest {

    @Test
    void promotesWithinSlo(SentinelCluster cluster) {
        Duration elapsed = cluster.getControl().triggerFailover(Duration.ofSeconds(30));

        assertThat(elapsed).isLessThan(Duration.ofSeconds(15));
        assertThat(cluster.getControl().getMaster().isRunning()).isTrue();
    }
}
```

![FIG-06 · what the returned Duration actually measures](./images/redis-06-failover-clock.png)

*SIGKILL → detection window → quorum → majority vote → promotion → the first poll that observes a new master.*

What `triggerFailover` does, precisely:

1. **Find the current master** — not the container that was created as master; the one that *reports* `role:master` right now. After a previous failover in the same class those differ.
2. **`kill`, not `stop`** — `TestcontainerController.kill()` issues `killContainerCmd` (SIGKILL). This matters, and the reason is in [`signal(7)`](https://man7.org/linux/man-pages/man7/signal.7.html): `SIGTERM` is catchable, `SIGKILL` is one of the two signals POSIX guarantees a process **cannot** catch, block or ignore. `docker stop` sends SIGTERM, Redis's signal handler runs a clean shutdown, sockets are closed properly — so the Sentinels observe FINs and a refused reconnect, which is a *different* and generally faster detection path than the one your production incident took. SIGKILL models a node death: no FIN, no `ECONNREFUSED`, just a peer that stops answering, which is precisely the case `down-after-milliseconds` exists to time out. Testing the wrong one gives you a green failover test that never exercised failure detection at all.
3. **Poll for a new master** every 500 ms up to the timeout, clearing the role cache on each iteration, because the whole point is that roles are changing underneath you.
4. **Return the elapsed `Duration`** — kill to first observation of a different container reporting `role:master`.
5. **Or throw `FailoverException`** carrying the timeout, the elapsed time and the retry count, rather than a bare `AssertionError`.

Role detection itself is worth a look, because "which container is the master" is not a question Docker can answer:

```java
try (RedisClient client = RedisClient.create(uri);
StatefulRedisConnection<String, String> conn = client.connect()) {
final String info = conn.sync().info("replication");
  if (info.contains("role:master"))      return ContainerRole.MASTER;
  else if (info.contains("role:slave"))  return resolveReplicaRole(container);   // → REPLICA_n
  else                                   return ContainerRole.UNKNOWN;
}
```

It asks Redis, over the wire, via `INFO replication` — the same field an operator greps during an incident. Three details make it work in the environments this has to run in: Sentinels are identified structurally (they expose 26379, so they are never asked for a replication role); the connection is made to the container's **internal** Docker IP with a fallback to `host:mappedPort`, which is what makes it correct inside Docker-in-Docker where containers cannot reach each other via the host; and results are cached in a `ConcurrentHashMap` with an explicit `clearCache()` — because caching a role across a failover is how you get a test that passes for the wrong reason.

The resulting `ContainerRole` is a semantic enum — `MASTER`, `REPLICA_0..8`, `SENTINEL_0..8`, `UNKNOWN` — so a test can say what it means:

```java
cluster.getControl().network().injectLatency(
        cluster.getContainer(ContainerRole.REPLICA_1), Duration.ofMillis(80));   // "the EU replica"
```

And what you should actually assert around a failover, since the clock is only half the story: Redis replication is **asynchronous**. The master acknowledges a write before any replica has it; a promotion therefore *can* lose the tail of the write stream. That is not a bug in Redis and it is not a bug in this module — it is a deliberate position in the [PACELC](https://en.wikipedia.org/wiki/PACELC_design_principle) space (*else, latency over consistency*: absent a partition, Sentinel-managed Redis chooses to ack fast rather than wait for replicas), and it is the thing your application code is silently assuming away. With a real cluster in a test you can finally measure it: write a monotonic sequence, trigger the failover, read the sequence off the new master, and assert your actual tolerance. If your answer is "we can't lose any", the conversation you now need is about [`WAIT`](https://redis.io/docs/latest/commands/wait/), `min-replicas-to-write` and `min-replicas-max-lag` — and you can test each of those settings the same way.

---

## 8. Now inject faults into it — two orthogonal layers

A real topology is the substrate. The stack's lower layers are the tools. Two independent flags on both annotations:

```java
@RedisSentinel(replicas = 2, enableNetworkChaos = true, enableConnectionChaos = true)
```

**`enableNetworkChaos = true` — the kernel packet path.** Adds the `NET_ADMIN` capability to every node in the cluster (never `--privileged`), then installs `iproute2` and `iptables` through a distribution-detecting package installer — the same code path resolves `apt-get`, `apk`, `dnf`, `yum`, `pacman` or `zypper` from `/etc/os-release`, so it works whether your image is `redis:7.4` (Debian) or `redis:7.4-alpine`. With that in place, `control.network()` gives you per-container [`tc`/`netem`](https://man7.org/linux/man-pages/man8/tc-netem.8.html) latency, jitter, packet loss, and `iptables`-based partitions. `NET_ADMIN` is scoped to the container's own network namespace: it cannot touch the host, your VPN, or the other containers.

```java
// one replica is "in another region"
control.network().injectLatency(cluster.getContainer(ContainerRole.REPLICA_0), Duration.ofMillis(80));
// … and now the split-brain question, with a real answer
        control.network().partitionFrom(replica, master);
```

**`enableConnectionChaos = true` — the libc socket layer.** Prepares `libchaos-net` on every container *before* `container.start()` — necessarily so, since the dynamic loader only honours `LD_PRELOAD` at process launch. From there you get per-syscall errno injection on `connect`, `bind`, `accept`, `send`, `recv` and `poll`, plus fault classes a TCP proxy structurally cannot reach: UDP, unix sockets, DNS resolution, `recv` corruption, file-descriptor exhaustion. Verbs it cannot model — bandwidth shaping — fall through automatically to a lazily-spawned Toxiproxy sidecar inside `CompositeConnectionChaos`.

![FIG-07 · two orthogonal fault layers](./images/redis-07-two-fault-layers.png)

*One flag shapes the kernel packet path; the other fails the libc socket call the process made. Different layers, different failures, both on the same cluster.*

The two are genuinely orthogonal and compose: `netem` reshapes packets on the wire, `libchaos-net` fails the syscall the process made. A retry loop that survives 5 % packet loss can still die on a single `ECONNRESET` from `send()`, and the point of having both is that you find out which.

The provider resolution deserves a paragraph, because it is three patterns stacked and the failure mode is otherwise mystifying. `control.connection()` performs a two-step lookup: first `CompositeConnectionChaos.standard()` reflectively, then [`ServiceLoader`](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/ServiceLoader.html) — the **service provider interface** mechanism specified by the [JAR file specification](https://docs.oracle.com/en/java/javase/21/docs/specs/jar/jar.html#service-provider) (`META-INF/services/<fully.qualified.Interface>`, one implementation class per line; this module ships exactly such a file for `ChaosPlugin`, which is how `RedisPlugin` is discovered without anyone importing it). What it returns is a **Composite** wrapping a **Chain of Responsibility**: every verb is offered to the `libchaos-net` layer first and falls through to Toxiproxy only for what the syscall layer cannot model. And the whole resolution sits behind **double-checked locking** over a `volatile` field — the idiom that was [broken before JSR-133](https://www.cs.umd.edu/~pugh/java/memoryModel/DoubleCheckedLocking.html) and correct after it, because the Java Memory Model ([JLS §17.4](https://docs.oracle.com/javase/specs/jls/se21/html/jls-17.html#jls-17.4)) gives a `volatile` write a happens-before edge to the subsequent read, so no thread can observe a partially-constructed provider:

```java
private volatile ConnectionChaos connectionChaos;      // volatile is load-bearing, not decoration

public ConnectionChaos connection() {
    ConnectionChaos local = connectionChaos;             // one volatile read on the hot path
    if (local == null) {
        synchronized (this) {
            local = connectionChaos;
            if (local == null) { local = resolveConnectionChaos(); connectionChaos = local; }
        }
    }
    return local;
}
```

If neither route resolves, the `IllegalStateException` names both fixes — set `enableConnectionChaos = true`, or add the `macstab-chaos-connection` dependency. And there is deliberately **no Null Object fallback** for syscall-level verbs: a container that was not prepared before start surfaces `LibchaosNotPreparedException` at the call site rather than silently doing nothing. A chaos framework that no-ops when misconfigured produces green builds that prove nothing, which is strictly worse than a red one.

### The incident packs: compound faults, one annotation

Above the primitives, `macstab-chaos-testpacks-l3-redis` ships method-level annotations that compose multiple domains into one named incident. These are not wrappers around a single toxic — they are the multi-domain shapes real outages have:

```java
@Test
@IncidentChaosRedisNetworkFlap
void sentinelSurvivesResetStorm(SentinelCluster cluster) { … }
```

| Annotation | What it actually applies |
|---|---|
| `@IncidentChaosRedisNetworkFlap` | `ECONNRESET` on **both** `CONNECT` and `RECV` at the configured toxicity — rapid TCP reset cycling, which is what drives an election storm rather than a clean failover |
| `@IncidentChaosRedisFailoverStorm` | `ECONNREFUSED` on `CONNECT` **+** `EAI_AGAIN` on every forward DNS lookup **+** a `CLOCK_REALTIME` offset — connectivity loss during the election window, concurrent with an NTP correction |
| `@IncidentChaosRedisOomEviction` | memory pressure driving aggressive key eviction |
| `@IncidentChaosRedisClockDrift` | inter-node clock skew — TTL expiry races and Sentinel timeout miscalculation |
| `@IncidentChaosRedisCacheAvalanche` | synchronised expiry storm under read concurrency |
| `@IncidentChaosRedisSlowlog` | injected latency that pushes operations into `SLOWLOG` territory |

The clock-drift ones deserve a note, because they hit a piece of Sentinel most engineers have never met: **[TILT mode](https://redis.io/docs/latest/operate/oss_and_stack/management/sentinel/#tilt-mode)**. Sentinel's timer interrupt fires ten times a second, so it expects roughly 100 ms between calls. It records the previous call time and compares — and, in the documentation's words, "if the time difference is negative or unexpectedly big (2 seconds or more) the TILT mode is entered". While in TILT a Sentinel "will continue to monitor everything, but: it stops acting at all" and "starts to reply negatively to `SENTINEL is-master-down-by-addr` requests as the ability to detect a failure is no longer trusted" — so it also stops helping *other* Sentinels reach ODOWN. It leaves TILT after 30 seconds of normality. The protection exists so that a suspended VM or an NTP step does not cause a cluster-wide phantom failover.

And it is directly assertable, which is the part that makes it a test rather than a lecture: `INFO` on a Sentinel exposes `sentinel_tilt` and `sentinel_tilt_since_seconds` (`-1` when not tilted). Skew `CLOCK_REALTIME` inside a Sentinel container by more than two seconds and you can assert that the HA layer entered its protective coma — and then assert what your application does during the half-minute when nothing will be promoted no matter what dies.

Which means: skew the realtime clock inside a Sentinel container and you can test whether your system survives the case where the HA layer has deliberately, correctly, and silently gone dormant for half a minute. That is a genuine production scenario — VM live-migration, an NTP correction after a long drift, a paused container — and there is essentially no other way to rehearse it. It is also why `libchaos-time` sits *below* the JVM: `System.currentTimeMillis()` is a JIT intrinsic inlined to the vDSO, and Redis's own `mstime()` bottoms out at the same place. You cannot fake this from inside the JVM. You can from underneath libc.

---

### The failure your Sentinel cluster will never fail over for

Which brings us back to the failure this article opened with — because everything between here and there assumed the interesting failure is a node *dying*, and that one isn't.

Redis's `stop-writes-on-bgsave-error` defaults to `yes`. If a background snapshot fails — a full disk, a read-only mount, an `EIO` on `write` from a single `libchaos-io` rule — Redis refuses every command that could modify the data set. Not the connection. Not the process. Just writes. It takes about ten seconds to reproduce, and here it is against a real Redis 7.4.8:

![FIG-T4 · MISCONF — reads fine, writes refused, no failover](./images/term-04.png)

*The disk went read-only under a running master. `GET` still answers. `SET` does not. `rdb_last_bgsave_status:err`.*

Now walk the topology. The master is **not** down: it accepts connections, answers `PING`, serves reads, and streams replication. Sentinel's health check is exactly that `PING` — the documentation describes it as remembering "the time of the latest successful reply to the `PING` command". So there is no SDOWN, therefore no ODOWN, therefore no election, therefore **no failover — ever**. Your liveness probe is green. Your replicas are in sync. Your dashboard is calm. And every write in the system has been failing since the disk filled up.

That is a class of incident that infrastructure chaos cannot produce (nothing died) and a mocked `RedisTemplate` cannot produce (the mock does not have a disk). It takes a real Redis, a real durability configuration, and a fault at the syscall layer — which is precisely the three-layer stack this article sits at the top of. And the assertion is one line: after injecting the fault, your write path must surface a specific, actionable error rather than silently dropping the write or retrying forever.

---

## 9. Proving what your client library actually did

A topology that runs is one thing. Evidence about what happened inside it is another, and it is where "we have a Redis integration test" turns into "we have a proof".

### `MONITOR`, wired into assertions

`RedisCommandTracker` is the **Observer** pattern in its purely passive form — it changes nothing about the system under test, it only watches. It attaches to a container's [`MONITOR`](https://redis.io/docs/latest/commands/monitor/) stream (a [RESP](https://redis.io/docs/latest/develop/reference/protocol-spec/) stream of simple status replies, one line per command, carrying timestamp, client address, database index and the command with its arguments), consumed frame-by-frame on a daemon thread:

```java
var tracker = new RedisCommandTracker(cluster.getReplicaContainers().get(0));
tracker.start();
service.loadUserProfiles(1_000);      // the code under test
tracker.stop();

tracker.assertThat()
    .hasCommand("GET").atLeast(900)   // reads really did land on the replica
    .hasNoDangerousCommands();        // and nothing issued KEYS / FLUSHALL
```

That is the assertion nobody can write against a mock: **did my read/write splitting actually route reads to the replica**, or is the `ReadFrom.REPLICA_PREFERRED` setting in my config silently ineffective because of how the connection was built? Run one tracker on the master and one on a replica, count `GET` on each, and the answer is arithmetic instead of belief.

Around that core the tracker exposes counting by command, counting by key pattern (`countCommandsMatchingKeyPattern("GET", "user:*")`), category rollups (READ / WRITE / ADMIN / PUBSUB / TRANSACTION / SCRIPTING / STREAM) and a read/write ratio, hot-key detection (top-N by access count — the tenant that will hotspot a shard), inter-arrival timing per command, and a static `measureReplicationLag(master, replica)`.

Three honest caveats. The capture buffer is a `CopyOnWriteArrayList`, which makes concurrent reads from assertion code free but copies the whole backing array on **every** captured line — fine for the hundreds of commands a focused test produces, quadratic if you point it at a load generator. Keep the observation window tight, or supply a narrower filter.

`MONITOR` is a debugging command with a real throughput cost — [Redis's own documentation](https://redis.io/docs/latest/commands/monitor/) measures a single `MONITOR` client reducing throughput by **more than 50 %**, and more clients cost more — so it belongs in targeted tests, on the node you are making a claim about, never switched on across a suite or used to measure performance. And the default line filter excludes replication traffic with the heuristic "the line contains `:6379]`", which is a pragmatic string match on the MONITOR client-address field, not a protocol-level distinction; supply your own `Predicate<String>` through the constructor if your topology makes that ambiguous. Also note the deprecation that tells the truth: `getAverageLatency()` is deprecated in favour of `getAverageCommandGap()`, because `MONITOR` timestamps tell you *when commands arrived*, not how long they took. Renaming that instead of quietly shipping a wrong number is the correct call.

### The inspector set

Four focused tools, each usable either shell-backed (`redis-cli` inside the container, no client dependency) or Lettuce-backed (typed, faster):

- **`SlowCommandDetector`** — typed `SLOWLOG GET` / `RESET`. The Lettuce backend decodes the raw wire shape (`[id, unix-seconds, duration-µs, [command, args…], client-addr, client-name]`) so you can assert "no command exceeded 10 ms under this fault" rather than eyeballing a log.
- **`ConnectionLeakTracker`** — `CLIENT LIST` diffing. Snapshot before, compare after: the connection-pool leak that only shows up after a reconnect storm, caught in CI.
- **`MemorySnapshotAnalyzer`** — `INFO memory` baseline versus current, for the write-then-delete pattern that should return to its starting footprint and sometimes doesn't.
- **`ReplicationConsistencyVerifier`** — writes N unique keys to the master, polls the replica until all arrive or the timeout expires, and reports matching/missing counts. `verifier.verify(50).assertFullConsistency()` is a one-line answer to "how far behind is the replica under 80 ms of injected latency".

### Which node is this connection actually on?

`ControlFacade.inspect(connection)` maps a live Lettuce connection back to the container serving it, in three tiers: auto-detection (extract the endpoint from the connection, match it against the known containers, resolve the role), an explicit container hint when auto-detection cannot see through your setup, and fully manual construction. It returns a `ConnectionInfo` carrying the role, the container and a health flag — so after a failover you can assert that your client *followed* the promotion instead of politely reconnecting to a demoted replica and reading stale data for the next four minutes. Tier 1 depends on Lettuce's connection `toString()` shape, which is why tiers 2 and 3 exist; when auto-detection fails it throws with the string it could not parse rather than guessing.

---

## 10. The recipes

**Cache eviction behaviour** — a bounded Redis, so the `allkeys-lru` policy your production config sets is actually exercised:

```java
@RedisStandalone(version = "7.4",
        args = {"--maxmemory", "32mb", "--maxmemory-policy", "allkeys-lru"})
class EvictionTest { }
```

**Several independent stores** — cache, sessions, rate limiter, started concurrently, isolated by construction:

```java
@RedisStandalone(id = "cache",        version = "7.4")
@RedisStandalone(id = "session",      version = "7.2")
@RedisStandalone(id = "rate-limiter")
class MultiStoreTest { }
```

**HA, with a Sentinel-aware client.** Lettuce, using the sentinel endpoints the cluster reports (and `mymaster`, per §6.4):

```java
@RedisSentinel(replicas = 2, sentinels = 3)
class SentinelClientTest {

    @Test
    void clientDiscoversMasterThroughSentinel(SentinelRedis cluster) {
        RedisURI.Builder uri = RedisURI.Builder
                .sentinel(cluster.sentinels().get(0).host(), cluster.sentinels().get(0).port(), "mymaster");
        cluster.sentinels().stream().skip(1).forEach(s -> uri.withSentinel(s.host(), s.port()));

        RedisClient client = RedisClient.create(uri.build());
        try (var conn = client.connect()) {
            assertThat(conn.sync().ping()).isEqualTo("PONG");
        } finally {
            client.shutdown();
        }
    }
}
```

Jedis, same idea:

```java
Set<String> nodes = cluster.sentinels().stream().map(Endpoint::toString).collect(toSet());
JedisSentinelPool pool = new JedisSentinelPool("mymaster", nodes);
```

**Spring Boot** — the extension publishes the first cluster's sentinel address as the `sentinel.nodes` system property, and clears it in `afterAll`; for standalone, resolve the instance in a `@DynamicPropertySource`:

```java
@DynamicPropertySource
static void redisProperties(DynamicPropertyRegistry registry) {
    StandaloneRedis redis = RedisStandalone.INSTANCE.get("cache");
    registry.add("spring.data.redis.host", redis::host);
    registry.add("spring.data.redis.port", redis::port);
}
```

**Mixed topology** — one HA cluster and two lightweight caches in a single class, eight containers, inside budget:

```java
@RedisSentinel(id = "ha-cluster", replicas = 2, sentinels = 3)
@RedisStandalone(id = "cache",   version = "7.4")
@RedisStandalone(id = "session", version = "7.2")
class MixedTopologyTest {

    @BeforeAll
    static void started(List<SentinelRedis> clusters, List<StandaloneRedis> standalones) {
        assertThat(clusters).hasSize(1);
        assertThat(standalones).hasSize(2);
    }
}
```

**The full stack, in one class** — a real HA topology, syscall-level faults prepared at process launch, and a named incident scoped to a single test method:

```java
@RedisSentinel(replicas = 2, sentinels = 3, enableConnectionChaos = true)
class RedisResilienceTest {

    @Test
    @IncidentChaosRedisNetworkFlap
    void clientRecoversFromResetStorm(SentinelCluster cluster) {
        await().atMost(Duration.ofSeconds(30)).untilAsserted(() -> {
            try (var conn = pool.getResource()) {
                assertThat(conn.ping()).isEqualTo("PONG");
            }
        });
    }
}
```

---

## 11. The design, named

Nothing in this module is novel construction; all of it is well-known patterns applied where their trade-offs happen to be right. Naming them is not decoration — it is what lets you predict the failure modes before you hit them, and it is the map for anyone extending the module to Postgres, Kafka or Cassandra, where the shape is identical and only the topology knowledge changes.

| Component | Pattern | Specification / source |
|---|---|---|
| `@RedisStandalone`, `@RedisSentinel` | Composed annotation + **Extension Object** | [JLS §9.6](https://docs.oracle.com/javase/specs/jls/se21/html/jls-9.html#jls-9.6) (annotation types); [JUnit 5 extension model](https://junit.org/junit5/docs/current/user-guide/#extensions) |
| Repeating instances → `@RedisStandalones` | **Containing annotation type** (compiler-synthesized) | [JLS §9.6.3](https://docs.oracle.com/javase/specs/jls/se21/html/jls-9.html#jls-9.6.3), [§9.7.5](https://docs.oracle.com/javase/specs/jls/se21/html/jls-9.html#jls-9.7.5) |
| Meta-annotation search (`@ChaosTest`) | **Registry** / single point of registration | Framework-level; the JLS deliberately does *not* provide it ([`@Inherited`](https://docs.oracle.com/javase/specs/jls/se21/html/jls-9.html#jls-9.6.4.3), JLS §9.6.4.3, is class-to-subclass only) |
| Test-method parameters | **Dependency injection** | [`ParameterResolver`](https://junit.org/junit5/docs/current/user-guide/#extensions-parameter-resolution) |
| `RedisStandalone.INSTANCE.get(id)` | **Service locator** over a thread-confined ambient context | [`ThreadLocal`](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/ThreadLocal.html); [Fowler, *Inversion of Control Containers*](https://martinfowler.com/articles/injection.html) |
| Container teardown | **Disposable / scope-bound resource** | [`ExtensionContext.Store.CloseableResource`](https://junit.org/junit5/docs/current/api/org.junit.jupiter.api/org/junit/jupiter/api/extension/ExtensionContext.Store.CloseableResource.html), plus Testcontainers' [Ryuk reaper](https://java.testcontainers.org/features/garbage_collector/) as an independent second path |
| `StartupResult`, `ClusterStartupResult` | **Sum type / ADT** with exhaustive matching | [JEP 409](https://openjdk.org/jeps/409) sealed types, [JEP 395](https://openjdk.org/jeps/395) records, [JEP 441](https://openjdk.org/jeps/441) pattern matching for `switch` |
| `StandaloneRedis`, `SentinelRedis`, `Endpoint` | **Value object** with constructor validation | [JEP 395](https://openjdk.org/jeps/395); compact canonical constructors |
| `Redis` (sealed over both topologies) | **Sum type** for exhaustive topology dispatch | [JEP 409](https://openjdk.org/jeps/409) |
| `StandaloneStartupOrchestrator`, `SentinelStartupOrchestrator` | **Scatter–gather** + **bulkhead** | `ExecutorService`, `Future.get(timeout)` |
| `StandaloneContainerInstanceFactory`, `SentinelClusterFactory`, `PackageInstallerPort` | **Ports & adapters** (hexagonal) / dependency inversion | Constructor-injected seams — the reason 90 %+ of the orchestration is unit-testable with no Docker daemon |
| `SentinelContainerFactory`, `SentinelNodeFactory` | **Abstract factory** + **builder** | — |
| `RedisPlugin` discovery | **Service provider interface** | [`ServiceLoader`](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/ServiceLoader.html) + [JAR file specification](https://docs.oracle.com/en/java/javase/21/docs/specs/jar/jar.html#service-provider) (`META-INF/services/…`) |
| `ControlFacade` | **Facade** over inspection, lifecycle, failover, role, network and connection chaos | — |
| `CompositeConnectionChaos` | **Composite** + **chain of responsibility** (syscall layer → proxy fallback) | — |
| `ControlFacade.connection()` | **Double-checked locking** over `volatile` | [JSR-133](https://www.cs.umd.edu/~pugh/java/memoryModel/) / [JLS §17.4](https://docs.oracle.com/javase/specs/jls/se21/html/jls-17.html#jls-17.4) — correct only *because* of the post-Java-5 memory model |
| `RoleResolver` | **Cache-aside** with explicit invalidation | `ConcurrentHashMap.computeIfAbsent` + `clearCache()` on topology change |
| `ResourceBudget` | **Guard clause** / fail-fast validator | Throws before the first Docker API call |
| `RedisCommandTracker` | **Observer** (passive) | [`MONITOR`](https://redis.io/docs/latest/commands/monitor/) over [RESP](https://redis.io/docs/latest/develop/reference/protocol-spec/) |
| `RedisCommandAssert` | **Fluent interface** / internal DSL | AssertJ-style chaining |
| `DisabledOnNonLinuxHost` | **Specification** object as an execution predicate | [`ExecutionCondition`](https://junit.org/junit5/docs/current/user-guide/#extensions-conditions) |
| `LibchaosTransport(...).prepare()` | **Two-phase construction** (configure-then-start) | Forced by [`ld.so(8)`](https://man7.org/linux/man-pages/man8/ld.so.8.html): `LD_PRELOAD` is honoured only at `execve` |
| Sentinel node discovery | **Publish–subscribe rendezvous** | [`__sentinel__:hello`](https://redis.io/docs/latest/operate/oss_and_stack/management/sentinel/#adding-or-removing-sentinels) |
| Sentinel leader election | **Quorum** (detection) + **majority vote** (authorization), versioned by **configuration epoch** | [Redis Sentinel: Quorum](https://redis.io/docs/latest/operate/oss_and_stack/management/sentinel/#quorum), [Configuration epochs](https://redis.io/docs/latest/operate/oss_and_stack/management/sentinel/#configuration-epochs) |
| `kill` vs `stop` in failover | Uncatchable-signal fault model | [`signal(7)`](https://man7.org/linux/man-pages/man7/signal.7.html) — `SIGKILL` cannot be caught or ignored |

---

## 12. Honest scope

What this module is not, and where its edges are:

- **Not Redis Cluster.** OSS Cluster mode — hash slots, `MOVED`/`ASK` redirection, resharding — is out of scope today. `RedisTopology.CLUSTER` exists in the enum; the provisioning does not.
- **Sentinel is Linux-host or in-container only.** §6.5. Enforced as a *skip*, with the reason printed.
- **Three `@RedisSentinel` attributes are not plumbed** — `quorum`, `version`, and `masterName` (the Sentinel config monitors `mymaster` and the nodes are `redis:7-alpine`). Defaults behave correctly; non-defaults are silently ignored. Documented here rather than discovered by you.
- **`--port` in `args` breaks port resolution.** The module resolves the host port via `getMappedPort(6379)`; move Redis's internal port and that lookup finds nothing. The constant carries this warning in its own Javadoc, which is the right place for it, but it belongs here too.
- **The memory budget is nominal**, not measured, and it is per test class, not per JVM or per runner.
- **Containers are ephemeral.** No persistence, no volume, no reuse. Data dies with the class — which is the correct default for tests, and means this is not the tool for testing AOF/RDB recovery across restarts.
- **`MONITOR` costs throughput**, and its capture buffer is copy-on-write. Targeted use only.
- **`SentinelCluster.getControl()` is lazier than it looks.** Its Javadoc says double-checked locking; the implementation is `AtomicReference.updateAndGet(…)`, whose contract explicitly permits the mapping function to be **re-applied** when a CAS loses under contention. Two threads racing the first call can therefore each build a `ControlFacade`, one of which is discarded. Harmless here — the facade is stateless at construction and the winner is published atomically — but it is not the idiom the Javadoc names, and it would matter if construction ever acquired a resource.
- **`CloseableResource` is deprecated as of JUnit 5.13.** The module targets 5.12.2, where the interface is still `@API(status = STABLE)` — verified in the jar — so nothing is wrong today. But 5.13 deprecated it: *"By default, `AutoCloseable` objects put into `ExtensionContext.Store` are now treated like instances of `CloseableResource` (which has been deprecated)."* A JUnit bump turns this into a two-inner-class change (`CloseableStoreMap` and `ThreadLocalCleanup` implement `AutoCloseable` instead) — worth knowing before a release note surprises you.
- **A 2 s `down-after-milliseconds` is tight** on a loaded laptop or a noisy runner. Fine for "did it fail over"; raise it for "it must not fail over".
- **Startup is not free.** A standalone container is a fraction of a second on a warm image cache; a six-container Sentinel cluster is seconds, dominated by replication convergence and the stabilisation window. Class-scoped provisioning amortises that across every method in the class — use it.

---

## 13. What this buys

Every layer of this stack exists to convert something unrehearsable into something a pull request can be red about. Layer 1 does it for syscalls: `EIO` on the third write to the WAL, `ECONNRESET` on 3 % of sends, a clock forty seconds fast. Layer 2 does it for the JVM: a checked exception thrown from a method that does not declare it, a return value nulled at 50 % probability.

This layer does it for **topology**. One master, two replicas, three Sentinels, a quorum, a private network, correct startup ordering, semantic wait strategies, a `SIGKILL` at the moment of your choosing, and a `Duration` you can assert against — expressed as one annotation on one class, torn down automatically, skipped honestly where it cannot run, and budget-checked before it can eat your CI runner.

The measured cost, on the slowest supported platform: eleven tests across two classes and three containers, sixteen seconds end to end including compilation. The Sentinel suite on the same machine: four tests, skipped with a printed reason, two seconds.

`SIGKILL` on a pod tells you *whether* your system dies. A real Sentinel cluster in a JUnit test tells you **what your client library actually does** while the master is being promoted — which connection it holds, which node it reads from, how many writes it lost, and how long it took to notice.

**Repo:** [github.com/macstab/chaos-testing](https://github.com/macstab/chaos-testing) · module `macstab-chaos-redis` · Apache 2.0 · built against the Java 25 toolchain · JUnit 5 · Testcontainers · Lettuce-aware, client-agnostic

---

## Read next, and watch

### 🎬 Watch the stack, bottom to top

Two episodes are out now — start with either:

- ▶️ **[The Macstab Chaos Stack — Testing the Vertical From Syscall to CI Pipeline](https://youtu.be/jxdcB84Q3bk)** *(00-0, the overview)* — how the three layers fit together, and why each one exists. Also [on Medium](https://medium.com/@schnapka.christian/the-macstab-chaos-stack-testing-the-vertical-from-syscall-to-ci-pipeline-03d96522a3a4).
- ▶️ **[You Tested the Catch Block. You Never Tested the Failure — 6 C99 Libraries That Fix It](https://youtu.be/B671lp4fKp8)** *(00-1, Layer 1)* — the `LD_PRELOAD` interposition underneath everything in this article: the `ECONNRESET` that `enableConnectionChaos` injects, the `EIO` that produces the `MISCONF` failure above, and the clock skew that puts a Sentinel into TILT. **If one thing here made you curious, make it this one** — it is the layer that makes the faults in this article real rather than simulated.

### 📖 Read next

- ⚙️ **Layer 2 (00-2):** the JVM bytecode agent — 62 JDK call sites, method-level effects, and checked exceptions the compiler forbids.
- 🧪 **Layer 3 (00-3):** the testing library — `@ChaosTest`, `ChaosSession`, `ActivationPolicy`, and deterministic, seeded fault sequences.
- ✍️ **The full series on Medium:** [medium.com/@macstab](https://medium.com/@macstab).

📺 **This episode (00-4) on video is in production** — the Sentinel cluster coming up, the master dying, and the promotion clock, on camera. Subscribe on **[Macstab Engineering](https://www.youtube.com/@MacstabEngineering)** to catch it.

---

## About the Author

**Christian Schnapka** is a principal+ backend engineer with 30 years of experience in Java, C++, cloud, and hardcore backend architecture. He has been the technical lead in nearly every team he has joined — from embedded systems to large-scale distributed platforms.

He built the Macstab Chaos Stack because the combination he needed — call-specific libc faults, JVM-internal effects, real container topologies, and test-scoped PR regression gates — was not available in one coherent workflow. Manual reproduction is possible but difficult to repeat, while infrastructure chaos validates a different and complementary set of resilience properties.

He publishes under **[Macstab](https://macstab.com)**, the company he works for.

Current work: authorization platforms (OAuth2, Spring Authorization Server) and energy management systems (Spring Boot, React). Principal embedded developer. Lives in Wentorf bei Hamburg, Germany.

Follow along:

- **Website:** [macstab.com](https://macstab.com)
- **YouTube:** [Macstab Engineering](https://www.youtube.com/@MacstabEngineering) — the chaos engineering series
- **GitHub:** [github.com/macstab](https://github.com/macstab)
- **Aether (security framework):** [*A NOP did not break my YARA rule — one encoding change did*](https://medium.com/@schnapka.christian/a-nop-did-not-break-my-yara-rule-one-encoding-change-did-f95c6ec3c26c) · [watch on YouTube](https://youtu.be/7TAAwTselJc)
- **LinkedIn:** [Christian Schnapka](https://www.linkedin.com/in/christian-schnapka/)
- **Twitter:** [Per!](https://twitter.com/nolem2008)
- **Security contact:** security@macstab.com and info@macstab.com

*If any of this fails on your system — open an issue. The examples are meant to run.*
