---
title: "QEMU and Spice agents on Windows VM"
date: '2019-03-19'
taga:
 - kvm
 - qemu
 - virtualization
categories:
 - blog
 - virtualization
author: jtorrex
---

# Description:

Basic configuraton of agents to improve the use of a VM under KVM

## [KVM]: qemu and spice agents to improve VMS on virt-manager

After deploy a Windows VM under KVM managed with virt-manager for testing purposes, I found that some of the components are not well integrated as on other alternatives software for virtualization, for example, the copypaste function between host and guest or the automatic resize of the guest screen.

Trying to improve this, and after search a little over internet, I found that maybe the only reason for this missfunction is that some agents are not installed properly on the guest.

The two components that I missed to install are::

* qemu-guest-agent
* spice-agent

The first what I did was to install the appropriate binaries on the VM, so I've downloaded from the next pages:

* virtio-drivers <https://docs.fedoraproject.org/en-US/quick-docs/creating-windows-virtual-machines-using-virtio-drivers/index.html>

On the Fedora page you can find an ISO with all the drivers available also the agents ready to install directly on your VM. To install it, you only need to mount the ISO directly in your guest and update the drivers or install it directly from the binary.

* spice-guest-tools <https://www.spice-space.org/download.html>

The spice-guest-tools are downloaded from the official page of the project. This agent basically provides a better integration with your VM and a better user experience and functionalities when you manage the guest through virsh or virt-manager.

Some of the functionalities that it provide are:

* QXL display device, for better video integration
* USB Redirection (useful if you need to manage hardware only supported on Windows, for example SUUNTO watches, etc).
* Screen resizing. This functionality adjust the size of the VM according to the size of the console window.

After install the agents, we need to add 2 channels on the XML definition of the machine, you can find this file on the defined qemu path for your system:

In my case:

```
/etc/libvirt/qemu
```

The code parts that we need to add are:

* A qemu agent channel

```
<channel type='unix'>
    <target type='virtio' name='org.qemu.guest_agent.0'/>
    <address type='virtio-serial' controller='0' bus='0' port='1'/>
</channel>tc/libvirt/qemu
```

* And a spice agent channel

```
<channel type='spicevmc'>
    <target type='virtio' name='com.redhat.spice.0'/>
    <address type='virtio-serial' controller='0' bus='0' port='2'/>
</channel>
```

Another part that could be interesting to configure is the video device to use QXL directly.

```
<video>
    <model type='qxl' ram='65536' vram='65536' vgamem='16384' heads='1' primary='yes'/>
    <address type='pci' domain='0x0000' bus='0x00' slot='0x01' function='0x0'/>
</video>
```

Once you have installed the agents and configured the two required channels + the video definition, you should to start the VM to check if it works, so you can benefit for the improved integration with your guest host.

This procedure it's highly recommendable to apply it in machines that you run if you are using KVM over virsh or virt-manager as your main virtualization platform.
