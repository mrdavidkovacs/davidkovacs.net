---
layout: post
title: "When a Sensor Is Not a Button"
date: 2026-10-01 09:00:00 +0200
categories: systems home-automation automation
excerpt: "A desk vibration sensor was wired like a click handler. Turning it into glare control required treating the event as evidence, with bounded state, explicit stop conditions, and restart recovery."
---

A desk vibration sensor was originally connected to a notification blueprint. A tap produced one notification; two taps produced another. In that role, the sensor behaved enough like a button that the distinction did not matter.

Then the sensor was repurposed to start an office glare-shading session.

That made the distinction important. A sensor event is evidence that something happened. It is not necessarily an imperative to perform an action every time it arrives.

A vibration can mean that someone is at the desk, adjusting equipment, or simply moving a chair. It is useful context for a shading decision, but it should not repeatedly command a cover or create an unbounded sequence of timers.

## The wrong model: event equals command

The first model was tempting:

1. The sensor reports vibration.
2. Start shading.
3. Stop shading later.

This works only if every event represents a deliberate request and if there is exactly one clean path to the later stop. Neither assumption holds for a sensor.

Vibration events can arrive several times during normal work. Sun position changes continuously. Home Assistant can restart between the start and end conditions. A global automation-disable switch may be turned off while a session is active.

Treating each event as a command turns ordinary sensor noise into repeated automation decisions.

## One bounded state

The useful state is not a counter, a queue, or one timer per event. It is one helper representing whether a glare-shading session is active.

A qualifying vibration marks that session active. Further vibration events converge on the same state rather than creating another timer or another independent workflow.

Conceptually, the automation has this shape:

```yaml
# Pseudocode: logical names are intentionally generic.
start when:
  - vibration is observed
  - automation is globally enabled
  - the sun is within the glare range

actions:
  - mark the shading session active
  - apply the glare-shading action
```

The helper is the boundary. It records the decision that has already been made, rather than preserving every event which contributed to it.

## Use the sun as a condition, not a clock

The start and end of glare are determined by the relationship between the sun and the window, not by a fixed time of day. A useful model therefore uses sun-vector thresholds.

A start threshold identifies the range in which direct light can become a problem at the desk. An end threshold identifies when the sun has moved far enough that the session should no longer apply.

The thresholds are deliberately separate. This creates hysteresis: the session starts in one defined range and ends only after the sun has clearly left the relevant range. Without that gap, small changes around one boundary can repeatedly start and stop the automation.

The end condition also waits for fifteen minutes before closing the session:

```yaml
end when:
  - shading session is active
  - the sun has passed the end threshold for 15 minutes

actions:
  - mark the shading session inactive
  - release the glare-shading action
```

The delay is not intended to make the system slower. It makes the end condition conservative around a discretely evaluated or imperfectly calibrated threshold. A session should end because the glare period has ended, not because the model observed one boundary value.

## The disable switch must remain authoritative

Automation needs a visible escape hatch. The global disable switch is checked before the automation starts a session. If it is switched off during an active session, the helper is cleared and the cover is released while manual control remains available.

This is more useful than trying to encode every exceptional situation into the glare model. It is a direct operational rule: when automation is disabled, the system does not infer that it should continue acting merely because an old sensor event or session state still exists.

## Restart is part of the behaviour

A restart can interrupt the interval between the end threshold and the delayed close. It can also happen while the helper says that a session is active.

The automation therefore includes a startup reconciliation path. After Home Assistant starts, it reads the bounded session state and evaluates the current sun-vector condition again. A session outside the relevant sun window is cleared. A session within the valid range remains active, reapplies the glare-shading action, and continues to watch for the normal end condition.

This avoids relying on a missed state transition before the restart. Recovery is based on current state and current environment, not on an event which may no longer be available.

## Result and limitation

The vibration sensor still contributes to the decision, but it no longer acts as a button. It is evidence that a shading session may be useful, subject to explicit environmental conditions and one bounded piece of state.

The resulting automation has a small failure model:

- repeated vibration converges on one session state;
- sun-vector thresholds define when the session is relevant;
- the fifteen-minute delay avoids ending on a transient condition;
- the global disable switch remains an immediate operational escape hatch;
- restart reconciliation restores the decision process from current state.

The thresholds still need calibration. Window direction, desk position, and seasonal sun paths are physical constraints, not values that can be chosen once in YAML and assumed correct forever. The important part is that adjusting those thresholds changes a bounded policy, rather than changing the meaning of every sensor event.
