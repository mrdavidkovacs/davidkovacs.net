---
layout: post
title: "Ambient Notifications Should Be Quiet, Specific, and Reversible"
date: 2026-10-08 16:30:00 +0200
categories: home-automation systems wled
excerpt: "A household reminder does not need another alert. A segmented WLED indicator can carry one useful status without taking over the room or leaving stale state behind."
---

A household reminder can be technically correct and still be unpleasant.

A phone notification interrupts whoever receives it. A spoken announcement interrupts everyone. A flashing light is excellent at being noticed, which is usually the problem. For a recurring status that matters only when someone is already in the relevant room, the better interface is often an ambient one.

I wanted a reminder visible on an existing WLED strip while the room lighting was already in use. It had to be noticeable in normal room light, support more than one status at once, and disappear reliably when the source status ended. It also had to leave ordinary lighting controls alone.

That last requirement rules out a surprising number of simple automations.

## Start with the decision, not the device

The useful question was not "how can this LED strip notify me?" It was "what decision should the reminder support?"

The answer was small: when someone enters the room and an existing household reminder is active, show which category needs attention. Do not wake the room up to announce it. Do not replay an animation every time motion is detected. Do not require anyone to understand several internal WLED segment entities to turn on a normal light.

The status already existed as Home Assistant binary sensors. They define the reminder window and become the only source of truth for the display. The WLED automation only renders their current state.

This makes the display disposable. If the source state is off, there is nothing for the strip to remember.

## Preserve normal lighting first

The strip already had a normal startup playlist and a main light entity used by the room controls. The reminder must not replace either.

The presence automation therefore has two paths:

1. If the main light is off, turn on the normal main entity, start its usual playlist, wait for it to finish, then render the reminder.
2. If the main light is already on, render the reminder only when a source sensor is active.

The important condition is what the status automation does *not* do: it never turns on the strip by itself.

```yaml
# Pseudocode: semantic names only.
when reminder_status_changes:
  if main_light is on:
    render_reminder()
```

This keeps the reminder contextual. A status can become active in the afternoon without turning on a visible signal in an empty room. The next ordinary use of the room lighting makes it available.

It also keeps the normal control path simple. The main light remains the thing that people and wall controls operate. Segments remain an implementation detail.

## Render one stable banner

The first version used small, dedicated segments. It was logically correct but too subtle in a real room. A status indicator that cannot be noticed without inspecting it is merely decorative telemetry.

The better compromise was to use two long middle segments as a static banner. One active reminder gives both segments the same colour. A known double case gives each segment its own colour. The display has enough visual weight to be visible, but it is still a quiet part of the existing light rather than a new light show.

Before applying colours, the automation selects a stable WLED preset that restores the intended segment layout:

```yaml
sequence:
  - action: select.select_option
    target:
      entity_id: select.wled_indicator_preset
    data:
      option: Segments
  - delay:
      seconds: 2
  - choose:
      - conditions: "two source statuses are active"
        sequence: "render one colour on each middle segment"
      - conditions: "one source status is active"
        sequence: "render its colour on both middle segments"
```

The preset reset is not incidental. It clears any previous banner state before applying the next one. Without that explicit reset, a prior double-status display can leak into a later single-status display. Household automations acquire folklore quickly when old state is allowed to survive without an owner.

WLED's segment model is well suited to this: the strip can retain its ordinary layout while Home Assistant changes only the two display areas. [WLED segments](https://kno.wled.ge/features/segments/) and [presets](https://kno.wled.ge/features/presets/) provide the device-side structure; Home Assistant supplies the current decision.

## Keep detection and rendering separate

A tempting design is to place every condition inside the presence automation. That works until a source status changes while the room is already occupied, or Home Assistant restarts while the indicator should still be visible.

The deployed design has a separate status synchronisation automation. It reacts to source-sensor changes and to Home Assistant startup. On startup it waits briefly for integrations to become ready, then renders only if the main light is already on and at least one source status is active.

This separation avoids a more subtle failure mode: a status update should not cancel the presence automation's inactivity wait. The presence automation owns the question "when should normal room lighting turn off?" The status automation owns the question "what should the current indicator show?"

Those are different lifecycles. Giving them separate owners avoids one innocent state update restarting an unrelated timer.

## Reset behaviour is part of the feature

A reminder is not complete when it turns on. It is complete when it stops being true.

The source binary sensors own that boundary. When their reminder window ends, they turn off. The status automation sees the change, restores the stable segment preset, and no longer applies a banner. Future presence events start the normal lighting path again.

There is no counter of notifications sent, no queue of pending colours, and no long-lived helper trying to reconstruct history after a restart. The current source state is enough.

That is the useful constraint for ambient notifications:

- show only information that is currently actionable;
- use the lowest-attention channel that works in context;
- preserve the normal control path;
- make stale state impossible rather than hoping it gets cleaned up later.

The strip is still a light. The notification is only a small, reversible layer on top of it.
