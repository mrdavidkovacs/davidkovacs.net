---
layout: post
title: "The Kill Switch Is the Feature, Not the VPN"
date: 2026-09-17 10:00:00 +0200
categories: systems reliability docker networking
excerpt: "A Docker stack using Gluetun and two application services needs one important property: the services must have no route when the VPN is unavailable."
---

The stack consists of three Docker containers: Gluetun provides the VPN connection while two application services use it. The requirement was simple: neither application service must use the normal internet connection of the Docker host if the VPN connection fails.

Establishing a VPN connection is not enough to meet this requirement. The relevant question is what happens after the VPN connection is lost.

## Initial situation

Both application services need outbound internet access, a shared data directory, and web interfaces reachable from the local network. They do not need their own network identity.

Giving each container its own Docker network and configuring the VPN as the preferred route would be easy. It would also leave every container with an independent route through the Docker host. A restart, DNS error, or routing mistake could therefore turn a VPN failure into ordinary internet traffic.

The requirement was defined as follows:

> If Gluetun is not connected, service1 and service2 must not be able to reach the internet.

## Shared network namespace

Docker provides a fitting mechanism for this case. Instead of giving service1 and service2 their own network configuration, both containers share Gluetun's network namespace:

```yaml
services:
  gluetun:
    image: qmcgaw/gluetun

  service1:
    network_mode: "service:gluetun"

  service2:
    network_mode: "service:gluetun"
```

This is more restrictive than connecting all three services to the same Docker network. `network_mode: "service:gluetun"` means that service1 and service2 do not receive a separate network interface, IP address, default route, or published ports. They use the interfaces and routing table of the Gluetun container.

The ports for the two web interfaces are therefore published on Gluetun, not on the application services. This looks slightly unusual in a Compose file but makes the network boundary explicit: Gluetun is the only container with external connectivity.

## Why the kill switch belongs to Gluetun

Gluetun manages the VPN connection and its firewall rules. When the tunnel is established, traffic from the shared namespace can leave through the VPN. When the tunnel is unavailable, the firewall blocks outbound traffic instead of allowing the normal Docker route.

The resulting behaviour is intentionally asymmetric:

- When the VPN is available, both application services work normally.
- When the VPN is unavailable, both application services lose outbound connectivity.

The second state is the important one. A failed request is visible and can be retried. Traffic which continues through the wrong network connection is harder to notice and defeats the purpose of the stack.

## Testing the failure case

The setup was verified in two states. First, the VPN connection was established and the clients could reach the internet through the shared Gluetun namespace. Then the VPN connection was interrupted. The clients were no longer able to reach the same destination.

This test is more useful than checking the VPN IP address once. It verifies that the clients do not have an independent fallback route.

## Result and limitations

The final setup needs no custom routing scripts, proxy, or additional Docker network. Gluetun is the only container which handles external networking; service1 and service2 only share its namespace and the shared data directory.

The trade-off is deliberate: a VPN outage also stops the application services. If the system later needs to remain available during an outage, it needs a second VPN gateway or a different network design. Until then, a stopped service is the correct failure mode.
