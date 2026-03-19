+++
title = "CRaC - Checkpoint/Restore and the JVM Startup Problem"
date = 2026-03-19
[taxonomies]
tags = ["java", "jvm", "kubernetes"]
+++

The JVM startup problem has been haunting Java developers since containers became the default deployment target. A Spring Boot service that takes 3-5 seconds to start is fine when you deploy once a day. It's a disaster when Kubernetes needs to scale from 2 to 20 pods in response to a traffic spike and each pod sits useless for 5 seconds while Spring initializes.

GraalVM native images solve this with AOT compilation. CRaC (Coordinated Restore at Checkpoint) takes a completely different approach: what if you just... saved the JVM's state after startup and restored it later?

## How CRaC Works

The concept is deceptively simple:

1. Start your application normally on the JVM.
2. Wait until it's fully warmed up - Spring context initialized, connection pools established, JIT compilations done.
3. Take a checkpoint: the JVM serializes its entire state (heap, threads, JIT-compiled code) to disk.
4. To start a new instance, restore from the checkpoint instead of starting from scratch.

The restored JVM picks up exactly where it left off, with all the warmup work already done. Startup time drops from seconds to milliseconds.

```bash
# Start with CRaC-enabled JDK
java -XX:CRaCCheckpointTo=/var/crac-checkpoint -jar myapp.jar

# Wait for full startup and warmup, then trigger checkpoint
jcmd <pid> JDK.checkpoint

# Later, restore from checkpoint
java -XX:CRaCRestoreFrom=/var/crac-checkpoint
```

The restore is fast. Really fast. I've measured 50-200ms for a full Spring Boot application restore, compared to 3-5 seconds for a cold start. And unlike native images, you get the full JIT-compiled performance because the checkpoint includes the JIT's work.

## Spring Boot Integration

Spring Framework 6.1+ and Spring Boot 3.2+ have built-in CRaC support. The framework manages the lifecycle - closing resources before checkpoint and reopening them after restore:

```java
@Component
public class MyResource implements Lifecycle {
    private Connection connection;

    @Override
    public void start() {
        // Called after restore
        this.connection = dataSource.getConnection();
    }

    @Override
    public void stop() {
        // Called before checkpoint
        this.connection.close();
    }
}
```

Spring automatically handles the common cases: closing database connections, stopping web servers, and releasing file handles before the checkpoint. After restore, it re-establishes everything.

For your own resources, implement `org.crac.Resource`:

```java
import org.crac.*;

@Component
public class CacheWarmer implements Resource {

    public CacheWarmer() {
        Core.getGlobalContext().register(this);
    }

    @Override
    public void beforeCheckpoint(Context<? extends Resource> context) {
        // Clean up before checkpoint
        // Close connections, flush buffers
    }

    @Override
    public void afterRestore(Context<? extends Resource> context) {
        // Reinitialize after restore
        // Reopen connections, refresh caches
        // Update timestamps, regenerate session IDs
    }
}
```

## The Gotchas

CRaC checkpoints capture *everything* in the JVM, which means you need to be careful about what's in that state:

**Secrets in memory:** If your application loaded database credentials or API keys before the checkpoint, those are baked into the checkpoint image. If you distribute that image, you're distributing your secrets. Solution: load secrets after restore, or use environment variables that are resolved at restore time.

**Stale connections:** TCP connections in the checkpoint are dead after restore (the remote end doesn't know about the restore). Spring handles database connections, but custom TCP connections need explicit handling.

**Time-sensitive state:** The checkpoint has a timestamp. If your code caches `Instant.now()` during startup, the restored instance thinks it's the checkpoint time. Anything time-dependent needs to be refreshed after restore.

**PID changes:** The restored process gets a new PID. Code that caches the PID (rare, but it happens) will have stale values.

**File descriptors:** Open files and sockets from before the checkpoint are invalid after restore. CRaC coordinates with the OS to handle this, but custom native code (JNI) that holds file descriptors needs explicit management.

## Kubernetes Startup Acceleration

The Kubernetes use case is where CRaC shines. The workflow:

1. Build your application as a container image with CRaC support.
2. In your CI/CD pipeline, start the container, warm it up, take a checkpoint.
3. Store the checkpoint as a new container layer (or in a volume).
4. In production, pods restore from the checkpoint instead of cold-starting.

```dockerfile
# Stage 1: Create checkpoint
FROM azul/zulu-openjdk:21-crac as checkpoint
COPY myapp.jar /app/myapp.jar
RUN java -XX:CRaCCheckpointTo=/app/checkpoint -jar /app/myapp.jar &\
    sleep 30 && \
    jcmd $(pgrep java) JDK.checkpoint

# Stage 2: Restore image
FROM azul/zulu-openjdk:21-crac
COPY --from=checkpoint /app/checkpoint /app/checkpoint
COPY myapp.jar /app/myapp.jar
CMD ["java", "-XX:CRaCRestoreFrom=/app/checkpoint"]
```

Pod readiness drops from seconds to milliseconds. This changes the economics of Kubernetes autoscaling - you can scale aggressively because new pods are ready almost instantly.

**In-place pod resize (Kubernetes 1.27+):** Combined with CRaC, you can adjust pod resources without restart. But even when restart is needed, CRaC makes it nearly instant. The combination means your scaling strategy can be more reactive without the startup time tax.

## Project Leyden

CRaC is a checkpoint/restore approach. Project Leyden is OpenJDK's broader initiative to "shift and constrain" - moving work from runtime to earlier phases (build time, first run, training runs).

Leyden's goals overlap with CRaC but go further. The roadmap includes:

- **Premain AOT compilation:** Compile code ahead of time, but keep the JIT for further optimization at runtime. Best of both worlds.
- **Training runs:** Run the application once in a "training" mode that captures profiles and pre-initializes state. Subsequent starts use the training data.
- **Condensers:** Framework-aware optimization passes that can eliminate dead code based on the specific application configuration.

As of 2026, Leyden is still in development, but early results are promising - Spring Boot startup times under 500ms with full JIT performance. When Leyden matures, it may subsume CRaC's use case with a more integrated approach. But CRaC is available *now*, and "now" matters when you have scaling problems today.

## AppCDS: The Quick Win

Application Class Data Sharing (AppCDS) is a simpler, lower-risk startup optimization that's been in the JDK since Java 10. It pre-processes class metadata into a shared archive that's memory-mapped at startup:

```bash
# Step 1: Generate class list during a training run
java -Xshare:off -XX:DumpLoadedClassList=classes.lst -jar myapp.jar

# Step 2: Create the archive
java -Xshare:dump -XX:SharedClassListFile=classes.lst \
     -XX:SharedArchiveFile=app-cds.jsa -jar myapp.jar

# Step 3: Use the archive
java -Xshare:on -XX:SharedArchiveFile=app-cds.jsa -jar myapp.jar
```

Spring Boot 3.3+ simplifies this with a built-in command:

```bash
java -Dspring.context.checkpoint=onRefresh -jar myapp.jar
```

AppCDS typically saves 10-30% of startup time. Not as dramatic as CRaC, but it requires no code changes and has no gotchas about secrets or stale connections. It's the boring, safe option.

## The Numbers

Here's what I've measured on a real Spring Boot 3.2 service (REST API, PostgreSQL, Kafka consumer):

| Approach | Startup Time | Peak Throughput | Memory |
|---|---|---|---|
| Cold start (JVM) | 4.2 seconds | 100% (baseline) | 320 MB |
| AppCDS | 3.1 seconds | 100% | 310 MB |
| CRaC restore | 0.15 seconds | 100% | 320 MB |
| GraalVM native | 0.08 seconds | 78% | 85 MB |

CRaC gives you native-image-class startup speed while maintaining full JVM throughput. The trade-off is operational complexity (managing checkpoint images) versus native image's trade-off (build time and reflection pain).

## What I Use

For long-running services on Kubernetes, CRaC is my current preference over native images. The reasoning:

1. Full JIT performance - no throughput regression.
2. Standard debugging tools still work (JFR, jcmd, heap dumps).
3. Build times stay fast - the checkpoint step adds 30 seconds to CI, not 10 minutes.
4. Third-party library compatibility is a non-issue - if it runs on the JVM, it checkpoints.

The downsides are real: the checkpoint image contains heap state (including potentially sensitive data), and you need CRaC-aware JDK distributions (Azul Zulu, Liberica). But for my use cases, these are manageable.

For Lambda functions and CLI tools, I still use native images. For everything else, CRaC plus AppCDS gets me where I need to be.

The JVM startup problem isn't one problem with one solution. It's a spectrum of trade-offs. Know your options, measure your specific workload, and pick the one that hurts the least.
