---
title: "Package Management in Linux"
seoDescription: "Learn how to manage applications using package managers on different Linux distributions with this hands-on guide."
datePublished: Tue Sep 05 2023 04:27:53 GMT+0000 (Coordinated Universal Time)
cuid: clm5t82bh000q08myejke3bys
slug: package-management-in-linux
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1693822399549/d37d17bc-4553-4715-ac4e-dcad4f44fea3.png
tags: linux, linux-for-beginners, linux-basics

---

## **Introduction**

**Welcome everyone! 👋🏻**

In the [**previous part**](https://anurag-rajawat.hashnode.dev/data-compression-in-linux) of this [**series**](https://anurag-rajawat.hashnode.dev/series/linux-the-practical-way), you learned about data compression. This part will teach you how to manage applications using package managers on Debian and Red Hat-based systems.

So without further ado, let's get started! 🚀

## Package Managers

In the early days of Unix and Linux, software was distributed as compressed archives. This was fine for developers, who could easily compile the source code and install the software. But for end users, it was a pain. They had to manually download the archive, extract the source code, compile it, and install it. And if the software depended on other libraries, they had to install those libraries too. This could be a daunting task, especially for users unfamiliar with Linux.

To simplify this process, [package managers](https://en.wikipedia.org/wiki/Package_manager) were developed. Package managers are tools that automate the installation, updation, and removal of software. They keep track of which packages are installed, and they can automatically download and install the dependencies that a package needs. This makes it much easier for end users to manage software on Linux.

### Debian based systems

Debian-based systems such as Ubuntu, Linux Mint, etc., use the `.deb` package format. This means that software for these systems is distributed in `.deb` files. The most common tool for managing `.deb` packages is **apt**. [APT](https://en.wikipedia.org/wiki/APT_(software)) is very mature and has been around for many years. This means that it is well-tested, reliable, very powerful and can do a lot of things.

APT uses a utility called **dpkg**. [Dpkg](https://en.wikipedia.org/wiki/Dpkg#References) is a lower-level utility that does the actual work of installing, removing, and managing packages. However, most of the time you won't need to use dpkg directly, as apt provides a higher-level interface that is easier to use.

Apt is a powerful package manager that can do a lot of things, including:

* Simplifying the download and removal of packages
    
* Automating the process of updating or upgrading the system
    
* Managing the dependencies between packages
    

Let's see it in action!

To use APT, you first need to update the APT cache. This ensures that APT has the latest information about available packages otherwise it might not work as you expect it to.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1693632853214/06cb5ce9-f042-4fb0-85aa-b3c290d3c3a1.png align="center")

* Install packages
    

Installing a package is very simple just use `apt install <package>` as a root user or with sudo privileges. When you install a package using APT it will automatically install any dependencies that the package needs.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1693824304281/697fe8e5-b005-42dc-94fd-87ae789ae0c5.png align="center")

You can use the `-y` option to automatically answer yes to all prompts.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1693824364593/94b8cba8-1879-4cb5-800a-1bd750d483c1.png align="center")

* Remove packages
    

Removing is also very simple just use `apt remove <package>`

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1693632909352/b57bf18c-bf0c-419c-b439-7323b32f3c72.png align="center")

Again it is waiting for our confirmation.

**One thing to note is that it will not remove any configuration files only remove packages.**

If you want to remove the configuration too you need to use `apt purge <package>`. This will remove the package as well as all of its configuration files except of user's home directory.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1693632940988/950256ca-9319-4abd-b22d-fdf65dafc006.png align="center")

As before you can use the `-y` option here as well.

* Search packages
    

Guess, which command we need to search a package?

If you said `apt search <package>` then congratulations you're right. Let's see this.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1693632980611/bb2c005d-d579-4eb1-a5fa-bf18ef56e2e2.png align="center")

* Upgrade packages
    

To upgrade just use `apt upgrade`

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1693633921453/f0c17918-70d3-4f0b-90e4-2ac9328548d0.png align="center")

As you can see above the `apt upgrade` command outputs a list of all the packages that will be upgraded or installed, as well as the size of the upgrade. This is a great way to see what changes are being made to your system and how much space they will take up.

I know you're wondering how APT knows about all the packages available for your Linux distribution, and where to install them from.

APT gets its information from two main sources:

* **Official repositories:** These are maintained by the developers of your Linux distribution, and contain thousands of packages that have been tested and certified to work with your system.
    
* **Third-party repositories:** These are maintained by other organizations, and may contain packages that are not available in the official repositories.
    

The official repository configuration file is `/etc/apt/sources.list`. This file tells APT where to download the package for your distribution.

### Red Hat-based systems

Red Hat-based systems, such as Fedora, CentOS, Rocky Linux, RHEL, etc. use the `.rpm` package format. The most common tool for managing packages in these systems is [**DNF**](https://en.wikipedia.org/wiki/DNF_(software)).

DNF is a newer and more powerful package manager than YUM, which was previously used in these systems. DNF is faster, more efficient, and has better handling of dependencies and conflicts. It also uses a lower-level utility called [RPM](https://en.wikipedia.org/wiki/RPM_Package_Manager), which is similar to the dpkg of Debian systems.

Let's see this in action!

* Install packages
    

Installing is pretty straightforward just like apt, `dnf install <package>`

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1693639031046/95822978-d1d7-43a8-8054-1476292a086b.png align="center")

Unlike `apt`, `dnf install` automatically updates its cache for you, so you don't need to run `dnf update` every time you install a new package.

You can use the `-y` option to automatically answer yes to all prompts.

* Remove packages
    

Removing is also easy, just use `dnf remove <package>`

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1693639159514/283bf5c1-3543-4c79-b999-c607cf2a1339.png align="center")

* Search packages
    

Guess the command.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1693639390325/33d345ea-8c4d-4635-ab25-80ea3d36ea32.png align="center")

* Listing all packages
    

To know which packages are installed on your system just use `dnf list` and it will list all the packages.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1693639492125/060f300c-4220-40a2-82a7-12c58a2d7dce.png align="center")

* Upgrade packages
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1693639679556/9c675ff1-c3c7-4aef-9b75-e843d34af89a.png align="center")

My system is up to date that's why it is not showing any upgrades but it may be different for you.

As usual, you can get more information about any of these tools by checking their man pages.

**That's all for this part!**

I hope you found this article informative and helpful. If you have any feedback, please share it in the comments below. Thank you for reading!

**Stay tuned for the next part of the** [**Master Linux series**](https://anurag-rajawat.hashnode.dev/series/linux-the-practical-way)**!**