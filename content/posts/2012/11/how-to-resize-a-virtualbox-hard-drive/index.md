---
title: "How To Resize a VirtualBox Hard Drive"
date: "2012-11-14"
categories: 
  - "linux"
  - "virtualbox"
---

Have you ever found that the virtual hard drive attached to one of your VirtualBox VMs is getting too small? Want to know how to make it beigger?

It's simple. The steps are as follows:

- Disconnect the drive from your VM. The VM should be shut down and not snapshotted for this step. You can either use the VMs storage settings to disconnect the hard drive, or go to `File->Virtual Media Manager`, choose the correct hard drive, and click the `release` button.
- In a command line session, type the following command - you do not have to be root:
    ```bash
    VBoxManage modifyhd /path/to/vdi/file.vdi --resize nnnn
    ```
    The size given, nnnn in the above, is the new size in Mb and must be _bigger_ than the present size. You will then see something like the following example where I resize a virtual hard disc to 12Gb (12288 Mb):
    ```bash
    norman@hubble $ VBoxManage modifyhd /data/VirtualBox/HardDisks/Antix.vdi --resize 12288
    0%...10%...20%...30%...40%...50%...60%...70%...80%...90%...100%
    ```
- Go back to the VM's storage settings, and re-attach the newly resized hard drive.
- Start the VM.
- Run the disc partitioner tool of your distro to extend an existing partition into the new free space, or to create a new partition. You may add the extra space to an existing logical volume if you are using LVM devices.

As mentioned above, you cannot reduce the size of a virtual disc. If you try, you will see the following:
```bash
norman@hubble $ VBoxManage modifyhd /data/VirtualBox/HardDisks/Antix.vdi --resize 6144
0%...
Progress state: VBOX\_E\_NOT\_SUPPORTED
VBoxManage: error: Resize hard disk operation for this format is not implemented yet!
```

That's all there is to it!
