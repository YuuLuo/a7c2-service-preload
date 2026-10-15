# ILCE-7CM2 Service Mode Preload Research

[简体中文](README.zh-CN.md)

This repository traces the path from the ILCE-7CM2's USB service session to one-shot root code execution, observed on firmware 2.01. It maps the authentication exchange, Jiritsu path resolution, dynamic loader behavior and execution evidence.

## Execution chain

1. The camera starts as USB Mass Storage (`054C:0E8A`). The host opens interface 0, starts Senser, and completes a three-round challenge–response exchange.
2. The camera re-enumerates as `054C:0336`. A fresh Senser exchange authenticates this service identity. [The protocol notes](PROTOCOL.md) give the control request, frame structure, digest inputs, replies, and two-mode sequence.
3. Jiritsu file commands select paths containing `/tmp/jiritsu/../..`. On the investigated firmware, path resolution reaches outside the Jiritsu directory. A shared object is placed in temporary Work storage; after the preload-path write, a service read of `/etc/ld.so.preload` returns that object's path.
4. A Jiritsu DEBUG request starts a root child for a fixed system command. Its dynamic loader reads `/etc/ld.so.preload`, loads the shared object, and runs the object's initialization function before the command.
5. The worker writes a versioned result record and removes the preload entry. The host reads back both the record and the preload path, then verifies that USB returns to the normal identity.

```mermaid
flowchart LR
    A[0E8A: MSC] --> B[Senser authentication]
    B --> C[0336: service authentication]
    C --> D[Jiritsu path resolution]
    D --> E[Preload configuration]
    E --> F[DEBUG root child]
    F --> G[Shared-object initialization]
```

The trust-boundary failure is the combination of path resolution outside Jiritsu's intended directory and a privileged child consuming a service-controlled preload file. USB service access is the entry point; the loader is where file control becomes instruction execution. The result record provides stronger evidence than a successful USB transfer or mode switch alone.

## What this enables

The observed result is temporary code execution in the child process. It can be used to inspect internal state or access data allowed by that process. Firmware construction is a separate offline task. Installing a modified firmware image also needs an update acceptance path; preload execution does not permanently disable signature checks. The investigated update workflow restored its temporary gate afterward.

The observations here are specific to ILCE-7CM2 firmware 2.01. Other versions require their own checks of the USB session, Jiritsu path behavior, loader, and child privileges. [Sony-PMCA-RE's Senser source](https://github.com/ma1co/Sony-PMCA-RE/blob/master/pmca/usb/sony.py) is the public reference for the authentication framing.

Documentation © 2026 YuuLuo · [CC BY-SA 4.0](LICENSE).
