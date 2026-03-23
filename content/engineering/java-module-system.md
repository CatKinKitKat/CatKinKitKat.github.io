+++
title = "The Java Module System - JAR Hell, Meet Module Hell"
date = 2026-03-23
[taxonomies]
tags = ["java", "jpms", "architecture"]
+++

JPMS (Java Platform Module System) shipped with Java 9 in 2017. It was supposed to fix JAR Hell - the tangled mess of classpath conflicts, split packages, and accidental dependencies that every large Java project eventually stumbles into. Nine years later, I can tell you with confidence that JPMS solves real problems and creates new ones, and the honest truth about its adoption is more nuanced than either its advocates or detractors admit.

## What JPMS Is

At its core, JPMS adds a layer of encapsulation above packages. A module declares:

- What it exports (which packages are public).
- What it requires (which other modules it depends on).
- What services it provides or consumes.

```java
// module-info.java
module com.myapp.orders {
    requires com.myapp.common;
    requires spring.web;
    requires java.sql;

    exports com.myapp.orders.api;
    exports com.myapp.orders.model;

    // Internal packages are NOT exported - truly encapsulated
    // com.myapp.orders.internal is invisible to other modules
}
```

Before JPMS, any public class was accessible to any code on the classpath. Your carefully designed internal APIs? Anyone could use them. That `com.myapp.internal` package? Just a naming convention that Javadoc politely asked people to respect. Nobody respected it.

JPMS makes this enforcement real. If a package isn't exported, it's invisible at compile time and runtime. Reflection can't reach it either (unless you explicitly `opens` the package). This is genuine, enforced encapsulation for the first time in Java's history.

## The JAR Hell Problems It Solves

**Split packages:** Two JARs providing classes in the same package. The classloader picks one arbitrarily. JPMS detects this at startup and refuses to launch. Problem found in seconds instead of hours of debugging `ClassCastException: Foo cannot be cast to Foo`.

**Transitive dependency conflicts:** Your app needs library A and library B. Both depend on library C but different versions. On the classpath, one version wins (whichever comes first). With JPMS, the module system can detect the conflict explicitly.

**Accidental internal API usage:** Using `sun.misc.Unsafe` or internal Guava classes that were never meant to be public API. JPMS makes these genuinely inaccessible (with painful migration warnings for existing code, admittedly).

**Runtime missing dependencies:** On the classpath, you discover missing dependencies when the code path hits a `ClassNotFoundException`. JPMS validates all dependencies at startup - if a required module is missing, the application won't start. Fast failure is better than mysterious runtime errors.

## The Module Hell It Creates

And now for the problems.

**The `--add-opens` / `--add-exports` epidemic:**

```bash
java --add-opens java.base/java.lang=ALL-UNNAMED \
     --add-opens java.base/java.lang.reflect=ALL-UNNAMED \
     --add-opens java.base/sun.nio.ch=ALL-UNNAMED \
     -jar myapp.jar
```

If you've seen startup scripts like this, you've seen the practical cost of JPMS. Frameworks that use deep reflection (Spring, Hibernate, Jackson) need access to internals that modules lock down. The fix is `--add-opens`, which defeats the encapsulation JPMS provides. Many production JVMs run with a dozen `--add-opens` flags, effectively poking holes in the module boundaries.

**Library compatibility:** Many libraries still don't ship `module-info.java`. They end up on the "unnamed module" - a compatibility layer that works but doesn't participate in module encapsulation. Your carefully modularized application has a Swiss cheese boundary wherever it touches a non-modular library.

**Automatic modules:** For JARs without `module-info.java`, JPMS derives a module name from the JAR filename. This is fragile - rename the JAR and the module name changes. The `Automatic-Module-Name` manifest entry helps, but not all libraries set it.

**Build tool complexity:** Maven and Gradle handle JPMS, but the module path vs classpath distinction adds configuration complexity. Multi-module builds where some modules are JPMS modules and others aren't is a recipe for headaches.

```xml
<!-- Maven: telling the compiler about module paths -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <configuration>
        <release>21</release>
        <!-- Module path is auto-detected, but sometimes you need to override -->
    </configuration>
</plugin>
```

## Plug-in Architectures

Where JPMS genuinely shines is building plug-in systems. The `ServiceLoader` mechanism combined with modules provides a clean plug-in architecture:

```java
// API module
module com.myapp.plugin.api {
    exports com.myapp.plugin;
}

// In the API module
public interface Plugin {
    String name();
    void execute(PluginContext context);
}
```

```java
// A plugin implementation module
module com.myapp.plugin.csv {
    requires com.myapp.plugin.api;
    provides com.myapp.plugin.Plugin with com.myapp.plugin.csv.CsvPlugin;
}
```

```java
// Host application discovers plugins at runtime
module com.myapp.host {
    requires com.myapp.plugin.api;
    uses com.myapp.plugin.Plugin;
}

// Discovery
ServiceLoader<Plugin> plugins = ServiceLoader.load(Plugin.class);
plugins.forEach(plugin -> {
    log.info("Loaded plugin: {}", plugin.name());
    plugin.execute(context);
});
```

Each plugin is a separate module with its own dependencies, isolated from other plugins. This is cleaner and more robust than the classpath-based ServiceLoader approach. If you're building an extensible application - an IDE, a build tool, a data processing pipeline with pluggable transformers - JPMS provides real value here.

## jlink: Custom Runtime Images

`jlink` creates a stripped-down JRE containing only the modules your application needs:

```bash
jlink --module-path $JAVA_HOME/jmods:target/modules \
      --add-modules com.myapp.main \
      --output custom-jre \
      --strip-debug \
      --no-man-pages \
      --compress=2
```

A full JRE is ~300MB. A jlinked runtime for a typical web service can be 40-80MB. Combined with a slim container base, your Docker image shrinks dramatically:

```dockerfile
FROM debian:bookworm-slim
COPY custom-jre /opt/jre
COPY myapp.jar /opt/app/myapp.jar
CMD ["/opt/jre/bin/java", "-jar", "/opt/app/myapp.jar"]
```

The catch: jlink only works if your application and all its dependencies are proper modules (or at least automatic modules). If anything is on the classpath as an unnamed module, jlink can't include it. This is the practical blocker for most applications - you need a fully modular dependency chain.

For applications that can achieve this (microservices with controlled dependencies, CLI tools, embedded systems), jlink is excellent. For Spring Boot apps with 150 transitive dependencies, half of which aren't modularized... good luck.

## The Honest Truth About JPMS Adoption

Here's what I see in practice across the projects and teams I've worked with:

**Most applications don't use JPMS.** They run on Java 21+ but treat modules as transparent - everything goes on the classpath (the "unnamed module"), and JPMS is invisible except for the occasional `--add-opens` flag.

**Libraries are slowly adopting.** Major libraries (Guava, Jackson, Netty) have added `module-info.java` or `Automatic-Module-Name`. But the long tail of smaller libraries hasn't.

**The JDK itself is modular.** `java.base`, `java.sql`, `java.net.http` - the JDK's own modules work well and benefit from JPMS encapsulation. This is perhaps JPMS's biggest success: it cleaned up the JDK's internal structure.

**Spring Boot doesn't require JPMS.** Spring Boot applications typically run on the classpath without `module-info.java`. Spring's own modules have `Automatic-Module-Name` for compatibility, but full JPMS modularization of a Spring Boot app is possible but not common.

## When I Use JPMS

I use module-info.java for:

- **Shared libraries** consumed by multiple services. The explicit exports and requires serve as enforced documentation of the API surface. Other teams can't accidentally depend on internal classes.
- **CLI tools** where jlink reduces the distribution size.
- **Plugin architectures** where module boundaries provide real isolation.

I don't bother with JPMS for:

- **Typical Spring Boot services.** The classpath works fine. The encapsulation benefits don't justify the migration effort for a service owned by one team.
- **Applications with many non-modular dependencies.** You'll spend more time fighting the module system than benefiting from it.
- **Prototypes and small projects.** The overhead isn't worth it.

## A Pragmatic Approach

If you want to try JPMS without committing fully:

1. Add `Automatic-Module-Name` to your library JARs' manifests. This reserves a module name without requiring a full `module-info.java`. Low effort, future-compatible.

```xml
<!-- In Maven -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-jar-plugin</artifactId>
    <configuration>
        <archive>
            <manifestEntries>
                <Automatic-Module-Name>com.myapp.common</Automatic-Module-Name>
            </manifestEntries>
        </archive>
    </configuration>
</plugin>
```

2. For shared libraries, add `module-info.java` with explicit exports. This gives downstream consumers clear API boundaries.

3. For applications, consider JPMS only if you have a specific need (jlink, plugin isolation, strict encapsulation).

JPMS is not the revolution Java 9 promised. It's a useful tool with a specific set of strengths that doesn't fit every use case. The Java ecosystem has voted with its feet - most projects use it implicitly (through JDK modules) without adopting it explicitly. And honestly, that's fine. Not every feature needs universal adoption to be valuable.

The JAR Hell it was designed to fix still exists on the classpath. It just turns out that most teams would rather deal with occasional classpath conflicts than commit to full modularization. I can't say they're wrong.
