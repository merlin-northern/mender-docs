---
title: Overview
taxonomy:
    category: docs
---

The main purpose of the Mender updater client is to install software updates to the device it is running on.
However, it also includes functionality to work with a Mender server, such as authenticate, report inventory
and the outcome of deployments. The Mender client runs in user space on top of an embedded Linux operating system.
It can be run in standalone or managed mode, as described in [Modes of operation](../../02.Overview/01.Introduction/docs.md#client-modes-of-operation).
For a more general overview of where the Mender client fits in as part of the deployment process,
please see the [Architecture overview](../../02.Overview/01.Introduction/docs.md).

In order to enable the client to work in as many environments as possible, it is built to be generic and extensible,
while providing default setup and configuration that should work with most environments. Especially when running
the client in managed mode, i.e. connected to a Mender server, there are a number of settings and extension points that can be utilized.
