---
layout: post
title: "The Repository Wasn't Driving Anything"
date: 2026-10-05 17:30:00 +0200
categories: systems containers maintenance
excerpt: "A failed package update looked like a TPU driver problem. The USB accelerator was already working; a stale APT source was merely preventing maintenance."
---

A routine package update failed on one of my Docker hosts. The error named an old Coral Edge TPU APT repository, which made the diagnosis look obvious: a kernel update had happened, the TPU probably needed a replacement driver, and the repository needed to be fixed.

That was the wrong story.

The host used a USB Coral accelerator for Frigate. Frigate documents USB Corals as not requiring a host driver and configures their detector as `device: usb`.[1][2] This is different from a PCIe or M.2 Coral, where the host needs the Apex device and its driver.[1]

The first useful question was therefore not “which package replaces this repository?” It was “what is actually on the runtime path?”

## Separate the failure from the hardware

The accelerator was already visible to the host. More importantly, Frigate was healthy and its own logs confirmed that the TPU had been found. That proved the path that mattered:

```text
USB device → container device mapping → Frigate detector
```

The failing APT source was outside that path. It was only metadata that `apt update` tried to download while refreshing configured package indexes. APT reads every enabled source during that operation; one unavailable third-party source can therefore turn a normal maintenance run into a failure.[3]

The conclusion was specific to this deployment: the active Frigate container was using the USB device successfully, while the obsolete host repository was not providing a missing part of that running system.

## The smallest safe repair

The repair was deliberately boring:

1. remove the unreachable APT source;
2. run `apt update` again;
3. verify the detector from the workload that depends on it.

No replacement repository. No kernel-module installation. No attempt to make a package manager ignore an error.

The verification mattered more than the deletion. Seeing a USB device in `lsusb` only confirms that the host can see it. The relevant check is that Frigate starts, remains healthy, and reports its TPU detector as available.

## Why the distinction matters

An APT source, a device driver, a runtime library, and a container device mapping are different layers. They may once have been installed together, but that does not make them permanent dependencies of one another.

The failed update tempted me to repair the most visible thing: the repository URL. Checking the live data path first led to a smaller and safer fix. The TPU continued to work, and host updates stopped failing on software that was no longer needed.

This is a useful maintenance rule for self-hosted systems: when an update fails near some hardware, verify the hardware's active path before adding drivers or replacement repositories. The package error may be adjacent to the system you care about rather than part of it.

## References

1. [Frigate: Coral hardware guidance](https://docs.frigate.video/frigate/hardware/#google-coral-tpu)
2. [Frigate detector configuration](https://docs.frigate.video/configuration/object_detectors/)
3. [Debian: sources.list manual](https://manpages.debian.org/bookworm/apt/sources.list.5.en.html)
