+++
title = "Docker Images for Java: Stop Shipping 800MB of Nothing"
date = 2026-02-25
[taxonomies]
tags = ["docker", "java", "devops"]
+++

I once reviewed a Dockerfile that produced a 1.2GB image for a Spring Boot service that did almost nothing. The base image was `ubuntu:latest`, it installed a full JDK, included Maven, left the build artifacts in the image, and didn't clean up the apt cache. It was a masterpiece of everything you shouldn't do.

## The Baseline Problem

A naive Java Docker image:

```dockerfile
FROM openjdk:17
COPY target/app.jar /app.jar
CMD ["java", "-jar", "/app.jar"]
```

This pulls `openjdk:17`, which is based on a full Debian image. The result is ~470MB before your application even gets added. Your actual JAR is probably 30-60MB. The rest is an operating system you don't need.

## Step 1: Use a JRE, Not a JDK

You don't need a compiler in production. Switch to a JRE-based image:

```dockerfile
FROM eclipse-temurin:21-jre-alpine
```

Alpine-based JRE images are around 100MB. That's already a 4x reduction.

## Step 2: Multi-Stage Builds

Build in one stage, copy only the output to the final stage:

```dockerfile
# Build stage
FROM eclipse-temurin:21-jdk-alpine AS build
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN apk add --no-cache maven && mvn clean package -DskipTests

# Runtime stage
FROM eclipse-temurin:21-jre-alpine
COPY --from=build /app/target/*.jar /app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

The build stage has Maven, the JDK, all your dependencies, and all your source code. None of that makes it into the final image. Only the JAR gets copied.

## Step 3: Layer Your JARs

Spring Boot fat JARs bundle everything into one file. If you change one line of code, Docker invalidates the entire layer and re-pushes 60MB.

Spring Boot's layered JAR feature splits the JAR into layers that change at different frequencies:

```dockerfile
FROM eclipse-temurin:21-jre-alpine AS extract
WORKDIR /app
COPY target/*.jar app.jar
RUN java -Djarmode=layertools -jar app.jar extract

FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=extract /app/dependencies/ ./
COPY --from=extract /app/spring-boot-loader/ ./
COPY --from=extract /app/snapshot-dependencies/ ./
COPY --from=extract /app/application/ ./
ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```

Now when you change your code, Docker only rebuilds the `application` layer (~few KB). The dependencies layer (~30-50MB) stays cached.

## Step 4: JVM Flags for Containers

The JVM needs to know it's in a container. Modern JVMs handle this better than they used to, but you should still be explicit:

```dockerfile
ENTRYPOINT ["java", \
  "-XX:+UseContainerSupport", \
  "-XX:MaxRAMPercentage=75.0", \
  "-XX:InitialRAMPercentage=50.0", \
  "-jar", "/app.jar"]
```

`MaxRAMPercentage=75.0` tells the JVM to use up to 75% of the container's memory limit for the heap. The remaining 25% is for metaspace, thread stacks, native memory, and the OS. If you set this too high, the OOM killer visits and your container restarts with no helpful error message.

## Step 5: Don't Run as Root

This should be obvious but I still see it everywhere:

```dockerfile
RUN addgroup -S app && adduser -S app -G app
USER app
```

Two lines. No excuses.

## The Result

Starting from 1.2GB, our images are now under 200MB. Build times dropped because Docker layer caching actually works when your layers are structured correctly. Deployments are faster because there's less to push and pull.

The optimizations are all boring. Multi-stage builds, smaller base images, layered JARs, correct JVM flags. None of this is clever. But it's the difference between a deployment that takes 30 seconds and one that takes 5 minutes, multiplied by every deployment across every service.
