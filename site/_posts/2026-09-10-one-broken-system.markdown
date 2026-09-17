---
layout: post
title: "When Twenty Low-Battery Alerts Mean One Broken System"
date: 2026-09-10 10:00:00 +0200
categories: systems diagnostics home-automation
excerpt: "A group of battery sensors became unknown at the same time. The batteries were not the problem; the Zigbee integration was."
---

A Home Assistant installation started reporting low-battery and unavailable states for several unrelated Zigbee devices at the same time. Replacing batteries would have been the obvious reaction. It would also have changed nothing.

The affected devices used different battery types and were located in different rooms. The common component was Zigbee2MQTT.

## Initial situation

The alerts had three different forms:

- the original battery sensor became `unavailable`;
- a derived percentage sensor became `unknown`;
- the low-battery binary sensor became `unknown` as well.

This matters because the last two states are not independent observations. They are calculated from the first one. Once the source sensor is unavailable, every template which depends on it can become unknown too.

The useful question was therefore not which batteries were low. It was which component supplied all of these battery values.

## Check the shared dependency first

Home Assistant exposes the Zigbee2MQTT bridge connection as an entity. When the bridge connection was checked, it was `off`. That explained why battery values from different devices disappeared at the same time.

The investigation was reduced to a short sequence:

1. Compare the timestamps of the affected entities.
2. Identify the integration shared by the devices.
3. Check the bridge or integration state.
4. Only inspect individual devices after the shared dependency is healthy again.

This order is useful for any integration which provides many entities. A broken bridge can produce dozens of broken sensors. The number of alerts does not tell us how many independent faults exist.

## Keep derived states honest

A low-battery template should not turn an unavailable source into a low-battery warning. It should only report a low battery when it has a valid value to evaluate.

A generic Home Assistant template can make that distinction explicit:

{% raw %}
```yaml
template:
  - binary_sensor:
      - name: "Example device battery low"
        state: >
          {% set battery = states('sensor.example_device_battery') %}
          {% if battery in ['unknown', 'unavailable'] %}
            false
          {% else %}
            {{ battery | float(101) < 20 }}
          {% endif %}
```
{% endraw %}

The `float(101)` default is deliberate. If a value is malformed, it evaluates above the threshold instead of silently becoming zero and creating another false alert.

This template does not hide the bridge outage. It only prevents the outage from being misrepresented as a collection of empty batteries.

## Alert on the source as well

The source failure needs its own alert. A bridge state can be monitored directly, separately from the individual battery sensors:

```yaml
trigger:
  - platform: state
    entity_id: binary_sensor.zigbee_bridge_connection
    to: "off"
```

The actual entity name depends on the integration, but the pattern remains the same: alert once for the unavailable data source, then treat derived values as unavailable rather than inventing a more specific diagnosis.

## Result and limitations

After the Zigbee2MQTT bridge was restored, the affected battery sensors recovered without replacing batteries. The immediate problem was one integration failure, not a fleet of devices.

The template pattern has a limitation. It prevents false low-battery alerts while a source is unavailable, but it does not tell us why the source failed. That remains the responsibility of the integration alert, logs, and the bridge health check.

Both alerts are useful: the bridge alert identifies the system to fix; the battery alert identifies a real battery to replace. They should not try to do each other's job.
