---
layout: post
title: "When Twenty Low-Battery Alerts Mean One Broken System"
date: 2026-09-10 10:00:00 +0200
categories: systems diagnostics
excerpt: "A noisy alert storm is not evidence of twenty independent failures. Start with correlation, dependencies, and the source of truth."
---

A notification storm creates a powerful instinct: start clearing notifications.

Twenty devices report low batteries, so perhaps twenty batteries need replacing. It is a wonderfully concrete plan. It also happens to be a very efficient way to spend an afternoon fixing the wrong thing.

The useful question is not *how many alerts arrived?* It is *what changed together?*

## Alert volume is not independent evidence

A system can produce many symptoms from one failed dependency. That is true for a production service, a CI pipeline, and the small collection of sensors that quietly run a home.

When many similar alerts arrive at once, the number is often less interesting than their timing. Batteries do not normally coordinate their decline. If a group of devices changes state within minutes, I first assume a shared cause:

- a receiver or gateway stopped reporting useful data;
- a source integration began returning an unknown value;
- a deployment changed how state is interpreted; or
- a derived automation turned a missing signal into a warning.

This is not proof. It is simply the cheapest hypothesis to test before making twenty independent changes.

## Follow the dependency chain backwards

Most alerts are derived state. A dashboard may show a low battery because an automation decided that an old or missing reading should be treated as low. The message is useful in normal operation, but it is not the source of truth.

So the investigation should move backwards:

1. Look for the first unusual timestamp, not the loudest notification.
2. Group affected devices by the system that reports their data.
3. Check whether healthy devices share the same path.
4. Inspect the source state before changing any device.

This is deliberately boring. It is also much faster than replacing batteries, restarting everything, and hoping the alerts become embarrassed enough to stop.

## Separate the outage from the consequence

A good monitoring setup distinguishes between a source failure and the warnings created by that failure.

If telemetry is unavailable, a single, explicit message about the missing source is more valuable than a wall of downstream low-battery alerts. The former explains what to investigate. The latter makes the operator count problems that may not exist.

That distinction matters because automation tends to be very literal. If an expression says “missing value means battery low”, it will faithfully report low batteries while the actual issue is that it cannot see the batteries at all.

The fix may be as small as teaching the derived alert to recognise an unavailable source. More importantly, the monitoring should retain the original failure as a visible, actionable signal.

## Use correlation as a habit

The broader lesson is not specific to batteries or home automation. Repeated errors in an application, failed jobs in CI, or a fleet of offline devices all invite the same mistake: treating every notification as a separate ticket.

Before acting, ask three questions:

- Did these symptoms start together?
- What dependency do they share?
- Which signal is closest to the real source?

Sometimes the answer really is twenty batteries. But correlation is cheap, and it protects the most valuable operational resource: attention.

The goal is not fewer alerts at any cost. It is alerts that preserve the story of the failure well enough for someone to fix the right system first.
