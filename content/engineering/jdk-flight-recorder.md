+++
title = "JDK Flight Recorder - Your JVM's Black Box"
date = 2026-03-13
[taxonomies]
tags = ["java", "jvm", "observability"]
+++

For years, the go-to approach for JVM observability was stitching together metrics libraries, APM agents, and log aggregation and hoping the picture was complete enough to diagnose production issues. Then I actually started using JDK Flight Recorder, and I felt betrayed that nobody told me about it sooner.

JFR has been in the JDK since Java 11 (open-sourced from the old JRockit Mission Control). It's free, it's always available, and it captures data that external tools literally cannot see. If you're not using it, you're debugging with one hand tied behind your back.

## What JFR Captures

JFR records events from inside the JVM: GC pauses, thread states, memory allocation, class loading, JIT compilation, lock contention, file I/O, socket I/O, and more. It captures this with overhead so low (typically under 2%) that Oracle and the OpenJDK team recommend running it in production. Always.

Starting a recording is trivial:

```bash
# Start recording, dump to file on exit
java -XX:StartFlightRecording=filename=recording.jfr,dumponexit=true -jar myapp.jar

# Or with more control
java -XX:StartFlightRecording=name=prod,settings=profile,maxage=1h,maxsize=500m -jar myapp.jar
```

You can also start and stop recordings dynamically:

```bash
# Start a 60-second recording on a running JVM
jcmd <pid> JFR.start name=debug duration=60s filename=debug.jfr

# Dump an ongoing recording
jcmd <pid> JFR.dump name=prod filename=snapshot.jfr
```

## JFR Internals

JFR uses a circular buffer architecture. Events are written to thread-local buffers, then flushed to global buffers, then written to disk. Because the buffers are thread-local, recording events doesn't require synchronization - which is why the overhead is so low.

Events have timestamps, durations, and stack traces. The "settings" configuration (either `default` or `profile`) controls which events are recorded and at what threshold:

- `default`: low overhead, captures events with durations over 10ms. Good for continuous production monitoring.
- `profile`: more detail, lower thresholds (events over 1ms). Higher overhead (~2-3%). Good for debugging sessions.

You can create custom settings files to fine-tune what's captured. For example, to capture all I/O events regardless of duration:

```xml
<configuration>
  <event name="jdk.FileRead">
    <setting name="enabled">true</setting>
    <setting name="threshold">0 ms</setting>
    <setting name="stackTrace">true</setting>
  </event>
</configuration>
```

## Custom Events for REST API Monitoring

Out-of-the-box events are great, but custom events let you record application-specific data:

```java
@Name("com.myapp.HttpRequest")
@Label("HTTP Request")
@Category({"Application", "HTTP"})
public class HttpRequestEvent extends jdk.jfr.Event {
    @Label("Method")
    public String method;

    @Label("Path")
    public String path;

    @Label("Status Code")
    public int statusCode;

    @Label("Duration (ms)")
    public long durationMs;

    @Label("User ID")
    public String userId;
}
```

Then instrument your filter or interceptor:

```java
@Component
public class JfrRequestFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest request,
            HttpServletResponse response, FilterChain chain) throws Exception {
        HttpRequestEvent event = new HttpRequestEvent();
        event.begin();

        try {
            chain.doFilter(request, response);
        } finally {
            event.method = request.getMethod();
            event.path = request.getRequestURI();
            event.statusCode = response.getStatus();
            event.durationMs = event.getDuration().toMillis();
            event.userId = SecurityContextHolder.getContext()
                .getAuthentication().getName();
            event.commit();
        }
    }
}
```

Now every request shows up in JFR with method, path, status code, duration, and user. You can correlate this with GC events, thread states, and I/O events in the same recording. That's the power of JFR - a unified timeline of everything that happened, application-level and JVM-level, in one file.

## Thread Leak Detection

Thread leaks are insidious. Someone creates a thread pool or starts a daemon thread and forgets to shut it down. Over days, the thread count creeps up until you hit OS limits or run out of memory.

JFR records thread lifecycle events. The `jdk.ThreadStart` and `jdk.ThreadEnd` events, combined with periodic `jdk.ThreadDump` snapshots, give you a complete picture:

```bash
# Analyze thread events from a recording
jfr print --events jdk.ThreadStart recording.jfr | grep -c "Thread"
jfr print --events jdk.ThreadEnd recording.jfr | grep -c "Thread"
```

If starts significantly outnumber ends, you have a leak. The stack traces on `jdk.ThreadStart` events tell you exactly where the threads are being created.

I once tracked down a thread leak to a third-party HTTP client that created a new `ScheduledExecutorService` per instance and never shut it down. Each instance leaked two threads. The service created a new client per request (bad design, but not my code). After 24 hours, we had 50,000 threads. JFR made this obvious in minutes - the `jdk.ThreadStart` events all had the same stack trace pointing to the client's constructor.

## JFR in GraalVM Native Images

GraalVM native images support a subset of JFR events since GraalVM 21.2. The coverage has been expanding steadily:

```bash
native-image --enable-monitoring=jfr -jar myapp.jar

./myapp -XX:StartFlightRecording=filename=native.jfr
```

Not all events are available. As of GraalVM 23+, you get: GC events, thread events, memory allocation, exceptions, and custom application events. Missing (or limited): JIT compilation events (there's no JIT in native images), some class-loading events, and some detailed GC internals.

For custom events, you need to register them at build time. If you're using Spring Boot with GraalVM, the Spring AOT processing handles most of this. For manual registration:

```java
@AutomaticFeature
public class JfrFeature implements Feature {
    @Override
    public void afterRegistration(AfterRegistrationAccess access) {
        FlightRecorder.register(HttpRequestEvent.class);
    }
}
```

The native image JFR recordings are compatible with the same analysis tools as regular JFR recordings. JDK Mission Control opens them just fine.

## The JFR File Format

JFR files (`.jfr`) use a compact binary format. Events are stored with metadata that describes the event types, their fields, and constant pools for common values (like thread names and stack traces). The format is designed for minimal write overhead - constant pool deduplication means recurring values (like method names in stack traces) are stored once and referenced by ID.

You can work with JFR files several ways:

```bash
# Print all events
jfr print recording.jfr

# Print specific event type
jfr print --events jdk.GCPausePhase recording.jfr

# JSON output
jfr print --json recording.jfr

# Summary of event types and counts
jfr summary recording.jfr
```

For deeper analysis, JDK Mission Control (JMC) provides a GUI. The automated analysis engine highlights issues like lock contention, hot methods, and excessive allocation. It's not pretty, but it's effective.

Programmatic access via the JFR API:

```java
try (RecordingFile file = new RecordingFile(Path.of("recording.jfr"))) {
    while (file.hasMoreEvents()) {
        RecordedEvent event = file.readEvent();
        if (event.getEventType().getName().equals("jdk.GCPause")) {
            Duration pause = event.getDuration("duration");
            System.out.println("GC pause: " + pause.toMillis() + "ms");
        }
    }
}
```

## JFR on Kubernetes

Running JFR in containers requires a bit of planning:

**Storage:** JFR recordings need somewhere to go. Use an `emptyDir` volume or persistent storage. Don't write to the container's writable layer - if the pod restarts, you lose the recording.

```yaml
spec:
  containers:
  - name: app
    volumeMounts:
    - name: jfr-data
      mountPath: /var/jfr
    env:
    - name: JAVA_OPTS
      value: >-
        -XX:StartFlightRecording=name=prod,settings=default,
        maxage=2h,maxsize=500m,filename=/var/jfr/recording.jfr,
        dumponexit=true
  volumes:
  - name: jfr-data
    emptyDir:
      sizeLimit: 1Gi
```

**Retrieval:** When you need a recording from a running pod:

```bash
# Trigger a dump
kubectl exec <pod> - jcmd 1 JFR.dump name=prod filename=/var/jfr/snapshot.jfr

# Copy it out
kubectl cp <pod>:/var/jfr/snapshot.jfr ./snapshot.jfr
```

**JFR streaming (Java 14+):** Instead of dumping files, you can stream events to an external system:

```java
try (var stream = new RecordingStream()) {
    stream.enable("jdk.GCPause").withThreshold(Duration.ofMillis(1));
    stream.onEvent("jdk.GCPause", event -> {
        meterRegistry.timer("jvm.gc.pause").record(event.getDuration());
    });
    stream.startAsync();
}
```

This bridges JFR events to your metrics system (Prometheus, Datadog, whatever). You get JVM-level detail in your existing dashboards without managing recording files.

## What I Run in Production

Every service I deploy gets JFR enabled. No exceptions. The configuration:

```bash
-XX:StartFlightRecording=name=continuous,settings=default,maxage=6h,maxsize=1g,dumponexit=true,filename=/var/jfr/continuous.jfr
```

Six hours of history, 1GB cap, auto-dump on shutdown. When something goes wrong, the recording is there waiting. I don't have to reproduce the issue or attach a profiler after the fact. The data is already captured.

JFR is the single most underused tool in the Java ecosystem. It's free, it's built-in, it has negligible overhead, and it gives you observability that no external agent can match. Use it.
