---
title: Create a Delta update Artifact
taxonomy:
    category: docs
---

Imagine that you have thousands of devices, and you deploy a new full filesystem
update to all of them. Furthermore, let's assume that you did not change a lot
of things in the new release, i.e.: looking at the bytes of the two images the difference
between them constitutes only a couple of percent of total image size. Passing
a thousand images to a thousand devices must take considerable time, while
we know that the [delta](../../02.Overview/15.Taxonomy/docs.md) between them
is relatively small. To address this issue we developed
[the binary delta update Artifacts](../../02.Overview/06.Delta-update/docs.md) that
pass only the difference between the two images.

Let's assume that you already have the images ready. One way of creating them
is with the help of [Yocto](https://hub.mender.io/t/robust-delta-update-rootfs/1144),
or [mender-convert](../../03.Devices/03.Debian-family/docs.md), but
it does not really matter how you got them; as long as there is a read-only root
filesystem support, you can use the following method to generate an artifact
that carries only the binary delta of the two images:

```bash
./mender-binary-delta-generator \
  -o v2.0-deltafrom-v1.0.mender \ # output filename holding the delta
  release-v.1.0.mender release-v.2.0.mender # the two images to calculate binary delta between
```

In the above we have assumed that the current working directory contains at least:

``` bash
$ ls -1
mender-binary-delta-generator
release-v.1.0.mender
release-v.2.0.mender
```
with the `mender-binary-delta-generator` coming from mender-binary-delta which
you need to [download](https://hub.mender.io/t/robust-delta-update-rootfs/1144).

You can now use `v2.0-deltafrom-v1.0.mender` with Mender, and the Client will 
automatically detect its type and handle it appropriately.

THe above approach acn same considerable time and storage, but it requires
the read-only root filesystem support. Please refer to the [Mender Hub](https://hub.mender.io/t/robust-delta-update-rootfs/1144)
for more information on how to incorporate the binary Delta update Artifacts into
your build.
