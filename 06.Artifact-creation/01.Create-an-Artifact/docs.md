---
title: Create an Artifact
taxonomy:
    category: docs
---

Mender uses [Artifacts](../../02.Overview/02.Artifact/docs.md) to package the
software updates for delivery to devices. As a user you manage the artifacts
with the help of `mender-artifact` command. You can get it either as a pre-built
executable from the [downloads section](../../08.Downloads)
or [build from sources](04.Artifacts/25.Modifying-a-Mender-Artifact/docs.md#compiling-mender-artifact).
The two basic usage scenarios of this utility reflect the two main update types
Mender supports: full filesystem update and application update. 

### Create a full filesystem update artifact

Assuming you already have a generated filesystem image in `rootfs.ext4` file,
you can use the following syntax to generate an artifact that will carry the whole
partition layout, and all data as the payload:

```bash
mender-artifact write rootfs-image -t beaglebone \
   -n release-1 \
   -f rootfs.ext4 \
   -o artifact.mender
```

where `artifact.mender` is the resulting artifact, and `rootfs.ext4`
is the filesystem image that you either generated with the help
of [Yocto](../../03.Devices/02.Yocto-project/docs.md), modified from existing
[Debian image](../../03.Devices/03.Debian-family/docs.md) with [mender-convert](https://github.com/mendersoftware/mender-convert),
or otherwise had prepared. The remaining flags: `-t`, `-n`, `-o` stand for the device
type compatible with the artifact (will be used to [match the deployments](../../02.Overview/04.Deployment/docs.md#Algorithm-for-selecting-the-Deployment-for-the-Device)),
name of the artifact (will be visible on the device in the UI once deployed),
and the path to the output file, respectively.

### Create an application update artifact

Creating an artifact takes different form in the case of [application updates](l).
Let's assume that you want bring a new `authorized_keys` file to `/home/pi/.ssh`
directories on your devices. With mender-artifact you can achieve this by running
the following commands:

```bash
echo /home/pi/.ssh > dest_dir # store the target directory
echo authorized_keys > filename # store the filename of the file we want to udpate
mender-artifact write module-image \ # write an application update module artifact
  -T single-file \ # type of the artifact is the nameof an existing update module
  --device-type raspberrypi4 \ # can be given multiple times, device type compatible with this artifact
  -o artifact.mender \ # output file with the artifact
  -n updated-authorized_keys-1.0 \ # name of the artifact
  -f dest_dir \ # the payload holding the file with the destination directory path
  -f filename \ # the payload holding the file with the destination filename
  -f authorized_keys # path to the actual file to store in the payload 
```

The resulting file `artifact.mender` holds the artifact providing a single file update.
In the above _single-file_ is both the name of the update module and the artifact
type. Please note also, that _dest_dir_ file stores the absolute path to the destination
directory on the device, while _filename_ file contents are the destination filename.
It is also important, that both _dest_dir_ and _filename_ are [built-in](https://github.com/mendersoftware/mender/blob/master/support/modules-artifact-gen/single-file-artifact-gen#L173)
into the single-file update module, and their names cannot be changed without 
modifications to the update module code.

#### Update modules generation script

You can see the above example in action in the [single file update module](https://hub.mender.io/t/single-file/486).
Where you can enjoy how simple it is to actually [write one](https://github.com/mendersoftware/mender/blob/master/support/modules/single-file),
and how powerful the mechanism is. We would like to note that most of the update modules
come with a script to simplify the arifact creation, but as [you can see](https://github.com/mendersoftware/mender/blob/master/support/modules-artifact-gen/single-file-artifact-gen)
they call the well-known `mender-arifact` utility.

#### Server side artifact generation

[Hosted Mender](https://hosted.mender.io) generates application update artifacts
automatically using the [single file](https://hub.mender.io/t/single-file/486)
update module. You can test it with uploading any file of your choice 
on the [releases](https://hosted.mender.io/ui/#/releases) page; the resulting artifact
will carry in the payload only the file you have uploaded, the destination
directory, and the filename, exactly in the same way as we saw above. On deployment
the Mender Client will automatically handle it and update the given file.

For more details on how to write update modules, and what exactly they are
refer to [this section](link-to-update-modules) and the [API specification](https://github.com/mendersoftware/mender/blob/2.2.0/Documentation/update-modules-v3-file-api.md).
