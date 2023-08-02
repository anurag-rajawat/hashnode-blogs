---
title: "Linux Boot Process Overview"
seoTitle: "Everything You Need to Know About How Linux Boots"
seoDescription: "Learn the Linux boot process from powerup to login prompt in this guide."
datePublished: Tue Aug 01 2023 05:09:55 GMT+0000 (Coordinated Universal Time)
cuid: clkrubb2d000l09kvg0qv15qf
slug: linux-boot-process-overview
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1690788834043/7f8f61e8-0001-4025-9da6-edaf17eed601.jpeg
tags: linux, linux-boot-process

---

Have you ever wondered what happens when you press the power button of your computer to the login prompt?

In this post, I'll walk you through the entire boot process, from the moment you press the power button to the moment you see the login prompt. You'll learn about the different stages of the boot process, the role of the BIOS, and how the operating system is loaded into memory.

By the end of this post, you'll have a better understanding of how your computer works and what happens behind the scenes when you turn it on.

**I know you're thinking, "But I want to learn Linux!"**

I hear you. In this series, we're going to learn Linux practically. But before we can get started, we need to understand how computers work at a fundamental level. That's why this post is important. It's the foundation on which we'll build our understanding of Linux.

So buckle up, and let's dive in! 🚀

## Boot Process

When you press the power button of your computer, a series of events happen in the background to get you to the login prompt. First, let's understand what is **booting.**

**Booting** is the standard term for **starting up a computer**. The term comes from the word **bootstrapping**, which means to **start something from scratch**. The idea is that when a computer boots up, it has to load all of its essential software from scratch.

The boot process is a series of steps that a computer takes when it is turned on. The following are some of the broadly defined tasks that are performed during the boot process:

* **Finding and loading the bootstrapping code:** The bootstrapping code is a small program that is stored in the computer's [BIOS](https://en.wikipedia.org/wiki/BIOS) (Basic Input/Output System). The bootstrapping code is responsible for loading the operating system kernel into memory.
    
* **Finding and loading the operating system kernel:** The operating system kernel is the core of the operating system. It is responsible for managing the computer's hardware and providing services to other software.
    
* **Running startup scripts and system daemons:** Startup scripts and system daemons are programs that are run when the computer boots up. These programs perform tasks such as mounting file systems, starting network services, and loading device drivers.
    
* **Maintaining process hygiene and managing system state transitions:** Process hygiene refers to the process of managing and monitoring running processes. System state transitions refer to the changes that occur in the system's state as it boots up and runs.
    

Now let's understand in detail:

When a machine is powered on, the power supply sends electricity to the components of the computer, such as the motherboard, hard drive, and fan(s). The CPU is hardwired to execute the boot code stored in the system firmware (BIOS) which is read-only memory ([ROM](https://en.wikipedia.org/wiki/Read-only_memory)) that knows about all the devices that live on the motherboard.

> **Traditional PC firmware was called the** [**BIOS**](https://en.wikipedia.org/wiki/BIOS)**, but it has been supplanted by a more formalized and modern standard called** [**UEFI**](https://en.wikipedia.org/wiki/UEFI)**.**

During the boot process, the system firmware performs a power-on self-test ([POST](https://en.wikipedia.org/wiki/Power-on_self-test)) to check the hardware. If the POST is successful, the firmware looks for the next stage of the boot process. In most cases, this is the hard disk. If the hard disk is not found, the firmware will try to boot from a DVD drive or a USB drive.

The [bootloader](https://en.wikipedia.org/wiki/Bootloader) then identifies and loads the appropriate operating system kernel. If your system has multiple kernels, it will also provide a boot-time interface that allows you to select which one to load. In most Linux distributions version 2 of [GRUB](https://en.wikipedia.org/wiki/GNU_GRUB) is the default bootloader.

Once the kernel is loaded into memory, it begins to execute. A variety of initialization tasks are performed, such as loading drivers, checking and mounting filesystems, and starting system daemons. These procedures are managed by a series of shell scripts called **init scripts** or **unit files**. The init scripts or unit files are run by the [init](https://en.wikipedia.org/wiki/Init#SysV-style) or [systemd](https://en.wikipedia.org/wiki/Systemd) depending on your system. The majority of Linux distributions have adopted systemd.

Once the system is fully booted, the login prompt is displayed, where you can enter your username and password to log in.

The figure below shows the general overview of the steps involved in the Linux boot process.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1690866339322/f66b4490-4e5c-43ab-a977-4a21ff741456.png align="center")

The boot process is a complex and essential part of how computers work. It is responsible for getting the computer up and running so that you can use it.

The entire process from power up to login prompt can take anywhere from a few seconds to a few minutes, depending on the speed of your computer and the operating system you are using.

That's all about the Linux boot process. I hope you found this article informative and helpful. If you have any feedback, please feel free to share it in the comments below. Thank you for reading!

This is the first part of the [Master Linux](https://anurag-rajawat.hashnode.dev/series/linux-the-practical-way) series, where we will learn about the Linux boot process. Stay tuned for more!