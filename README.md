# How to Set Up an OverlayFS inside a KVM in Firecracker

This guide describes how to set up an instance of `firecracker` to use an OverlayFS. This allows multiple different instances of `firecracker` to share
a read-only rootfs and write their own files inside the OverlayFS.

This guide uses parts of the image builder tool of [firecracker-containerd](https://github.com/firecracker-microvm/firecracker-containerd).

It is assumed that you already have a running instance of `firecracker`. If not, refer to Firecrackers own guide on [getting-started](https://github.com/firecracker-microvm/firecracker/blob/main/docs/getting-started.md).

## Step 1: Prepare a Read-Only rootfs

First, we need to create a rootfs that is read-only and contains a few extra folders for mounting the OverlayFS during startup. There are two ways of doing this. Either start from scratch
and create a new rootfs, or copy everything over from an existing writable rootfs. Here, we will use the second method. To create a new rootfs,
refer to the section _Creating a rootfs Image_ of Firecrackers [rootfs-and-kernel-setup](https://github.com/firecracker-microvm/firecracker/blob/main/docs/rootfs-and-kernel-setup.md),
and use the following steps to create a SquashFS out of it.

We will use SquashFS as the filesystem for our read-only rootfs, since it is compressed, only takes up little space, and is already a read-only filesystem. In order to create a SquashFS, you will need to install `squashfs-tools` for your distribution.

1. Mount the existing rootfs.
```bash
mkdir /tmp/my-rootfs
sudo mount rootfs.ext4 /tmp/my-rootfs
```

2. Create necessary folders for mounting the OverlayFS.
```bash
sudo mkdir -p /tmp/my-rootfs/overlay/root /tmp/my-rootfs/overlay/work /tmp/my-rootfs/mnt /tmp/my-rootfs/rom
```

3. Copy over the `overlay-init` script (adapted from [overlay-init of firecracker-containerd](https://github.com/firecracker-microvm/firecracker-containerd/blob/main/tools/image-builder/files_debootstrap/sbin/overlay-init)).
```bash
sudo cp overlay-init /tmp/my-rootfs/sbin/overlay-init
```

4. Create a SquashFS out of the old rootfs.
```bash
sudo mksquashfs /tmp/my-rootfs rootfs.img -noappend
```

5. Unmount the old rootfs.
```bash
sudo umount /tmp/my-rootfs
```

Now we have successfully prepared the rootfs.

## Step 2 (Optional): Create an EXT4 Image as an Overlay

To allow instances of `firecracker` to save persistent files that are available after a reboot, we need to create an EXT4 image to use as an overlay. If data does not need to be
available again after a reboot, you can skip this step, as it is possible to use a tmpfs as an overlay instead.

1. Create the image file. We will use a size of 1 GiB (1024 MiB), but this can be increased.
```bash
dd if=/dev/zero of=overlay.ext4 conv=sparse bs=1M count=1024
```
The file will be created as a sparse file, so that it only uses as much disk space, as it currently needs. The file size may still be reported as 1 GiB (the files _apparent size_).
Its actual size can be checked with the following command (which should be 0 right now).
```bash
du -h overlay.ext4
```
`du` can also be used to report the apparant size of a file.
```bash
du -h --apparent-size overlay.ext4
```

2. Create an EXT4 file system on the image file.
```bash
mkfs.ext4 overlay.ext4
```

Done! The overlay is ready now.

## Step 3: Configure the rootfs and Kernel Parameters

The last step is to mount the new rootfs as read-only add new kernel parameters, so that during the init process the OverlayFS is activated.

Inside your config file, the root device should look like this.
```json
{
  "drive_id": "rootfs",
  "path_on_host": "rootfs.img",
  "is_root_device": true,
  "partuuid": null,
  "is_read_only": true,
  "cache_type": "Unsafe",
  "rate_limiter": null
}
```
And the kernel configuration should look like this.
```json
"boot-source": {
  "kernel_image_path": "vmlinux",
  "boot_args": "console=ttyS0 reboot=k panic=1 pci=off overlay_root=ram init=/sbin/overlay-init",
  "initrd_path": null
}
```
It is important that the rootfs is read-only and that the kernel parameters add `init=/sbin/overlay-init` and `overlay_root=ram`. This initiates an OverlayFS using a tmpfs.
To use an EXT4 image as the overlay instead of a tmpfs, we need to add that image as a drive and replace `overlay_root=ram` with `overlay_root=vdX`, where X is a placeholder for
the devices letter. An example of this can be found in `vm_config.json`.
