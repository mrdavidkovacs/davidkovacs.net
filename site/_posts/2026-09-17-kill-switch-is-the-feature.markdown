---
layout: post
title: "The Kill Switch Is the Feature, Not the VPN"
date: 2026-09-17 10:00:00 +0200
categories: systems reliability
excerpt: "A service which must use a VPN needs one important property: it must not use the ordinary network when the tunnel is unavailable."
---

A service which should use a VPN must be able to reach the internet through the VPN. This is the obvious requirement. There is a second requirement which is at least as important: the service must not use the ordinary network when the VPN connection is unavailable.

The VPN connection itself is therefore only a part of the solution. The more important part is the failure case.

## Initial situation

The starting point was a small self-hosted service which should only communicate through a VPN connection. The service is useful only if this restriction is reliable. A normal network fallback would make the setup look operational while violating its actual purpose.

The following requirement was defined:

> The service must be unavailable when the VPN connection is unavailable.

This is a deliberate trade-off. Availability is reduced in one failure case in order to retain the intended network boundary.

## Possible approaches

There are several ways to connect a service to a VPN. The two relevant approaches were:

1. Give the service access to the normal network and configure the VPN as its preferred route.
2. Give the service access only to a dedicated VPN gateway.

The first approach is easier to set up. It also requires trusting the routing configuration in every failure case. If the VPN client stops or the routing table changes, the service may still find a route through the ordinary network.

The second approach is more restrictive. The service has one network path and the gateway is responsible for the VPN connection. If the connection is down, the service has no usable route. This behaviour is preferable because it is observable and safe.

Therefore, the second approach was selected.

## Testing the failure case

Testing only the successful VPN connection is not sufficient. It proves that the happy path works but says nothing about the actual requirement.

The setup was tested in two states:

- With the VPN connection established, the service can reach its intended destination.
- With the VPN connection unavailable, the service cannot reach the destination through another path.

The second test is the relevant one. It verifies that the network boundary still exists when the dependency fails.

## Result and limitations

The resulting setup is intentionally small: one service and one dedicated network path. It does not need a proxy, dynamic routing, or a custom recovery mechanism for this use case.

This approach is useful whenever a fallback would be unsafe. The same principle applies to backups, authentication, and automation: a fallback is only useful if it preserves the original constraint.

The limitation is clear as well. If availability during a VPN outage becomes a requirement, the solution needs another VPN gateway or a different network design. Until then, a service which stops working is preferable to one which works on the wrong network.
