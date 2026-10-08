---
layout: post
title: "Ambient Notifications Should Be Quiet, Specific, and Reversible"
date: 2026-10-08 16:30:00 +0200
categories: home-automation systems wled
excerpt: "A bin-collection reminder needed to be visible in the kitchen without sending another alert, taking over the light, or leaving a stale colour behind."
---

A bin-collection reminder already existed in Home Assistant as binary sensors. The missing part was making it useful where the decision happens: in the kitchen, while someone is already there.

A phone notification was not the right answer. It interrupts one person, can be dismissed at the wrong moment, and is easy to miss when the relevant task happens later. Turning on a lamp or using an animated WLED effect was not right either. The kitchen light should remain a light, not become an alarm system.

The goal was specific:

- show the active collection category on the existing kitchen WLED strip;
- do not turn the strip on just to display the reminder;
- keep the normal wall-control and presence-lighting behaviour intact;
- support two collection categories on the same day;
- clear the display reliably when the reminder is no longer active.

## Problem

The WLED strip already had two useful behaviours:

1. a normal main light entity used by the room controls;
2. a short startup playlist when the light turns on.

The first attempt used small dedicated WLED segments for the collection status. It was technically correct, but not visible enough in a normally lit kitchen. A reminder that requires looking for it is not a reminder.

There was also a lifecycle problem. The collection sensors can change while the kitchen light is already on. Home Assistant can restart while a reminder is active. A simple presence automation cannot cover both cases without mixing two unrelated jobs: deciding when to turn the room light off, and deciding which status the display should show.

## Possible approaches

### Phone notification

This is the default Home Assistant solution: send a notification when the collection date is near.

It is useful for urgent or remote information. This status was neither. It only mattered when someone was already in the kitchen and about to act. Adding another phone interruption would solve delivery, but not timing or attention.

### Turn the WLED strip on automatically

The strip could light up whenever a collection sensor becomes active.

That makes the status impossible to miss, but it also lights an empty room and turns a passive reminder into an unsolicited action. The light may be off deliberately. The automation should not override that decision.

### A permanent small segment marker

A dedicated LED segment seems simple because it does not affect the rest of the strip.

In practice, the marker was too small to be useful. The strip had enough space for a clear visual signal, so hiding the information in a tiny segment was the wrong trade-off.

### One presence automation that does everything

It would be possible to evaluate collection sensors, start the playlist, render segments, reset colours, and manage the inactivity timer in one large automation.

That creates an avoidable coupling. A collection-status update while the kitchen is occupied could restart the inactivity timer. A Home Assistant restart would need special handling inside the same state machine. The result works until a harmless change in one concern breaks the other.

## Solution

The deployed solution uses the existing collection sensors as the source of truth and treats the WLED strip as a renderer.

When presence turns the kitchen light on, the automation starts the normal playlist. After a conservative delay, it calls a dedicated script that renders the current collection state.

When the light is already on, presence does not replay the playlist. It renders the current state only if a collection sensor is active.

```yaml
# Pseudocode: identifiers are deliberately generic.
when presence becomes active:
  if main_light is off:
    turn_on(main_light)
    select_playlist("Startup")
    wait_for_playlist_to_finish()
    render_collection_status()

  if main_light is on and collection_status_is_active:
    render_collection_status()
```

The collection-status automation is separate. It reacts when one of the binary sensors changes and when Home Assistant starts. It renders only when the main kitchen light is already on.

```yaml
when collection_status_changes or Home_Assistant_starts:
  wait_for_integrations_after_startup_if_needed()

  if main_light is on:
    render_collection_status()
```

This is the important boundary: collection status can update the indicator, but it cannot turn on the room light or interfere with the presence automation's inactivity timer.

### Rendering the status

The renderer first selects a WLED preset named `Segments`. That restores the known segment layout and clears any old colour state. It then waits briefly for the WLED entities to settle before applying a static colour to the two long middle segments.

One active category uses the same colour across both segments. A known double collection day uses one colour on each segment.

```yaml
sequence:
  - select_preset("Segments")
  - wait_for_segment_entities()
  - choose:
      - when: two_categories_are_active
        do: render_one_colour_per_middle_segment
      - when: one_category_is_active
        do: render_its_colour_on_both_middle_segments
```

This is deliberately static. The status is visible in normal room light, but it does not flash, animate indefinitely, or compete with people in the room.

WLED provides the segment and preset model; Home Assistant only selects the active layout and colours. [WLED segments](https://kno.wled.ge/features/segments/) and [presets](https://kno.wled.ge/features/presets/) are the relevant device-side features.

## Why this solution holds up

The solution has three useful properties.

First, it is contextual. The indicator appears when the kitchen light is already being used. It does not illuminate an empty room or add a phone alert.

Second, it keeps normal control paths intact. The main light entity remains the control surface for presence automation, wall controls, and manual use. The internal segment entities are not exposed as a second user interface.

Third, it is reversible. When the source sensors turn off, the renderer restores the stable `Segments` preset and does not apply a banner. The next time the light turns on, it starts with the normal playlist again.

There is no counter of notifications, no queue of old collection states, and no helper that tries to reconstruct what should be displayed after a restart. The current binary-sensor state is enough.

That is the rule I would use for similar Home Assistant indicators: make the source state authoritative, render it only in the right context, and explicitly define how the display returns to normal.
