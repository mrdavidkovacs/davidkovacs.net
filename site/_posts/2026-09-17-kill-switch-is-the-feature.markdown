---
layout: post
title: "The Kill Switch Is the Feature, Not the VPN"
date: 2026-09-17 10:00:00 +0200
categories: systems reliability
excerpt: "A secure network path is only useful when its failure mode is safe. Design the negative case first, then prove it."
---

“Route it through a VPN” sounds like a networking task. Usually, it is really a failure-mode task.

A service that should use a private tunnel has one important requirement: when the tunnel disappears, it must not quietly use the ordinary network instead. The happy path is easy to demonstrate. The safety property lives in the unhappy one.

## Start with what must never happen

It is tempting to begin with a diagram: service, network, VPN gateway, perhaps a reverse proxy. That describes normal operation, but it does not answer the useful question:

> If the tunnel is down, where can this service still send traffic?

“Nowhere” is a far stronger answer than “it should prefer the tunnel.” Preference is policy. Structural containment is a guarantee.

This is the small but important difference between a configuration that usually works and a system that fails safely.

## Make the unsafe path impossible

The simplest useful design gives the service only one network route: a dedicated gateway that owns the tunnel. It does not receive direct access to the normal network.

That means a tunnel failure makes the service unavailable rather than exposed. Availability has been traded for a constraint, deliberately.

The pattern is not limited to networking:

- a backup job should fail rather than write unencrypted data to an unexpected location;
- an authentication flow should deny access rather than accept an unverified identity provider response;
- an automation should stop rather than guess when a safety sensor becomes unavailable.

A fallback is not automatically resilient. It can be an invisible way to discard the very boundary the system was meant to enforce.

## Test the negative case

A design like this deserves one boring test: remove the dependency and observe that the unsafe action does not occur.

For a networked service, that means checking both sides:

1. With the tunnel available, the service can reach its intended destination.
2. With the tunnel unavailable, the service cannot reach that destination through any other path.

The second check is the feature. Without it, the configuration only proves that the pleasant demonstration worked once.

## Keep the first version small

The first implementation does not need a control plane, dynamic routing, or a clever recovery script. A local service, one constrained network path, and a negative test are enough to establish the safety property.

More machinery may become worthwhile later—for observability, multiple services, or high availability. But it should earn its place by protecting the same invariant, not by making the container diagram more impressive.

A good kill switch feels almost disappointing in normal operation. That is rather the point. Its value appears only when something else has already gone wrong.
