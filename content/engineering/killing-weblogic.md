+++
title = "Killing WebLogic, One Service at a Time"
date = 2026-02-10
[taxonomies]
tags = ["java", "spring-boot", "migration", "legacy"]
+++

If you've ever inherited a WebLogic estate, you have my sympathy. If you're currently migrating one to Spring Boot... pull up a chair, because I've been living this for a while now.

## The Starting Point

Picture this: a cluster of WebLogic servers running services that were deployed sometime in the late 2000s. Some have documentation. Most don't. The ones that do have documentation that's wrong, which is arguably worse than no documentation at all. EAR files, EJBs, JNDI lookups, XML configuration files that stretch longer than some novels I've read.

These services are keeping critical systems running. You can't just turn them off. You can't rewrite everything at once. You have to do this surgically - one service at a time, with zero downtime, while the old and new systems run in parallel.

No pressure.

## The Approach

What's worked for me so far:

### 1. Map Everything First

Before touching any code, I spend time understanding what each service actually does. Not what the documentation says it does (if it exists), not what the class names suggest it does - what it *actually* does in production. That means reading logs, tracing message flows, and having uncomfortable conversations with people who built it ten years ago and have since moved on to different companies.

### 2. Strangler Fig Pattern

You don't replace a monolith in one go. You wrap it. New requests go to the new Spring Boot service. Old requests keep hitting WebLogic until you've verified parity. The strangler fig pattern isn't exciting, but it works and it lets you sleep at night.

### 3. EJB to REST (or Messaging)

Most of the inter-service communication in these old systems is EJB remoting or JMS queues through WebLogic's built-in broker. The migration path depends on the communication pattern:

- **Request/response** - becomes a REST endpoint or gRPC service. Spring Boot handles this trivially.
- **Fire-and-forget** - moves to Kafka or Solace. This is usually the cleanest migration because the producer and consumer are already decoupled. You're just swapping the transport.
- **The weird ones** - EJB timers, MDBs with complex selectors, anything that leans on WebLogic-specific features. These require actual thought and usually some refactoring.

### 4. Test Against Production Data

Unit tests are great. Integration tests are better. But nothing beats running the new service against a copy of production data and comparing outputs. I've caught more bugs this way than I'd like to admit - subtle differences in date parsing, character encoding edge cases, that one service that returns XML with a BOM for reasons nobody can explain.

### 5. Cut Over Gradually

Feature flags, traffic splitting, canary deployments - use whatever your infrastructure supports. The goal is to get real traffic hitting the new service while having a kill switch if something goes wrong. Because something *will* go wrong. It always does. The question is whether you can roll back in seconds or if you're scrambling at 2 AM.

## What I've Learned

Legacy migrations aren't technically glamorous. Nobody writes blog posts about them (well, I guess I am now). But there's something satisfying about taking a system that's been running on life support and giving it a proper foundation. Every service you migrate is one less thing keeping you on WebLogic, and one step closer to the day you can finally decommission that cluster.

That day hasn't come yet for me. But it's getting closer.
