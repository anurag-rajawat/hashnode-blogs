---
title: "Linux Command Line"
seoTitle: "The Linux Command Line: Your Secret Weapon for Getting Things Done"
seoDescription: "Master the Linux command line and learn how to use it to perform everyday tasks."
datePublished: Fri Aug 04 2023 04:30:09 GMT+0000 (Coordinated Universal Time)
cuid: clkw37q9o000o09kyheyp269v
slug: linux-command-line
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1691046445460/73b19956-3866-424e-9466-76c9e17b614c.png
tags: linux, linux-for-beginners, linux-basics, linux-commands

---

## Introduction

**Welcome everyone! 👋🏻**

In the [first part](https://anurag-rajawat.hashnode.dev/linux-boot-process-overview) of this [series](https://anurag-rajawat.hashnode.dev/series/linux-the-practical-way), you learned about the boot process in Linux. In this part, you'll learn how to use the Linux command line and some basic commands.

The Linux [command line](https://en.wikipedia.org/wiki/Command-line_interface) is a powerful tool that allows you to control your system and perform tasks quickly and efficiently. It is used by system administrators, developers, and anyone else who needs to manage a Linux system.

**To follow this blog, you'll need a working Linux environment. If you don't have one, you can install Linux by searching for "how to install Linux" on Google.**

So what are you waiting for? Let's get started! 🚀

**The command line has been around since the early days of Linux.** Back then, it was the only way to interact with the operating system. Today, most Linux distributions come with a graphical user interface (GUI), but there are still many reasons why you should learn about the command line.

**Imagine you're working with Linux servers.** There's no graphical user interface (GUI), so you can't use your mouse to interact with the system. You need to use the command line interface (CLI) to do everything. If you don't know how to use the CLI, you're going to be in a lot of trouble. You won't be able to do anything, and you'll be completely lost.

That's why it's important to learn the CLI. It's a powerful tool that can help you automate tasks, troubleshoot problems, and gain a deeper understanding of how Linux works. Whether you're a system administrator, developer, or just a curious user, learning the CLI is a valuable skill.

## Starting a shell

The [shell](https://en.wikipedia.org/wiki/Shell_(computing)) is a program that provides a user with a way to interact with the Linux system via the command line. It is a regular program that runs whenever a user logs in to a terminal. In most of the Linux distributions [**bash**](https://en.wikipedia.org/wiki/Bash_(Unix_shell)) is the default shell.

**To get started, open your terminal emulator.** If you're using a graphical desktop environment (GUI), you can open it by searching for it in your application launcher. If you're using Linux without a GUI, you'll already have a terminal emulator open.

Below is a screenshot of a Linux bash shell using a terminal.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1690977420274/a697d6b1-3a43-43f1-a118-a21e321c043a.png align="center")

The prompt `anurag@server1:~$` indicates the current user name (**anurag**), the system name (**server1**), the current working directory `~` (**home**), and `$` indicates that the shell is waiting for a command. The prompt may be different in different distributions, but the idea remains the same.

## Shell Commands

Now that you have a basic understanding of the shell, let's take a look at commands. With shell commands, you can do anything from navigating directories to running programs to managing your system.

### Listing Files and Directories

* **To list the files and directories** in your current directory, run the `ls` command in your terminal.
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1690981969776/09b67e3f-4133-4d6b-a1ba-064d618700ae.png align="center")

In my case I've only one file in your case it may be different. By default `ls` doesn't list hidden files, so to list those files use the next command,

* **To list files and directories including hidden ones**, run the `ls -a` command.
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1690982321986/edbee62d-e96e-40a9-9c92-bc66bd4ad0ca.png align="center")
    

this time it lists all files and directories.

* **To list files and directories with more information**, run the `ls -l` or `ls -al` command.
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1690982912438/c515c06b-e4ea-4eb0-8213-d16f0e11c6ed.png align="center")
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1690982942354/65405955-b4ad-4291-89ac-88c56629e6be.png align="center")
    

Let's break down the above information:

The long listing format lists each file and subdirectory on a single line, with additional useful information. The first line shows the total number of blocks contained within the directory. The following lines show information about each file (or directory), including:

* **File type:** A character that indicates the type of file or directory, such as `d` for directory, `-` for file, `l` for linked file, `c` for character device, or `b` for block device.
    
* **Permissions:** The permissions for the file or directory.
    
* **Owner:** The owner of the file or directory.
    
* **Group:** The group that owns the file or directory.
    
* **Size:** The size of the file or directory in bytes.
    
* **Date and time:** The date and time the file or directory was last modified.
    
* **Filename:** The name of the file or directory.
    

Don't worry if you didn't understand all of that. We'll revisit this with more detail in upcoming blogs.

* Sometimes you only want information for a specific file or a few files. To do this, you can filter the output of the `ls` command on the command line. You can also use [regular expressions](https://en.wikipedia.org/wiki/Regular_expression) for more complex use cases.
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1690984670454/0bfa8610-8e19-490a-b788-3bcf4b24279a.png align="center")
    

### File Handling

* **To create a file**, run the `touch` command followed by the name of the file you want to create. The `touch` command will create a new file with the specified name and assign your username as the file owner. It is important to note that the `touch` command does not create a file with any content. It simply creates an empty file.
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1690986357279/ac6db562-79d7-4565-85ca-2fd5f7a11db8.png align="center")
    
    As you can see above, the file was created and its size is 0 bytes.
    
* **To delete a file,** you can run the `rm` command followed by the name of the file you want to remove. It is important to remember that there is no recycle bin in the terminal, so all actions are permanent.
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1690986686466/0757e318-13d1-40e9-8310-c90a2714d7f7.png align="center")
    
* **To copy a file**, you can run the `cp` command followed by the **source** and **destination** name.
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1690988116289/2717cf22-4ff0-4e3a-8717-16478cff4bbe.png align="center")
    
* **To rename or move a file,** you can run the `mv` command followed by the **source** and **destination** name.
    
    **Rename a file:**
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1690988142645/014a5928-781e-4d84-9fdb-c7fbc3d0cf8e.png align="center")
    
    **Move a file to another location:**
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1690988218178/557509b5-c4d4-40b6-9c07-8995c6d02a46.png align="center")
    
* You will learn two commands **to view the content of a file**: `cat` and `less`. Later in this series, you will learn more advanced commands to do the same.
    
    **Using the** `cat` **command:**
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1690990300274/51b0981f-4c86-4d2e-a4db-5bd22b65ae5f.png align="center")
    
    **Using the** `less` **command:**
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1690990390779/36df22d8-423d-4066-b7ca-d2804eee7776.png align="center")
    
    **The** `cat` command displays all the data inside a text file, while `less` provides several handy features for scrolling through a text file, both forward and backwards, as well as some advanced searching capabilities.
    
* **To view the type of a file, you can run the** `file` command followed by the file's name. This will help you avoid trying to view the content of a binary file, which may appear as gibberish on your monitor and may even freeze your terminal emulator.
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1690991080396/723376cf-ac92-4952-be79-8b1877c1abcf.png align="center")
    
    As you can see above, the output of the `file` command for a JSON, ASCII, and binary file is different.
    

> **You can use the** `-i` option with the `mv`, and`cp` commands. This will prompt you before the command attempts to overwrite any pre-existing files.

### **Na**vigating the Filesystem

If you're new to Linux, you may be confused by how it references files and directories. Linux uses **forward slashes** (`/`) to separate directories, and it does not use drive letters in pathnames. This is unlike Windows, which uses **backslashes** (`\`) to separate directories and drive letters to identify different drives.

* To find out your **current working directory**, you can run the `pwd` command.
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1690991713047/b704684c-2252-4a6f-bc29-aafb717ff57c.png align="center")
    
* **To change directories**, run the `cd` command followed by the directory name.
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1690994857359/b995c7fa-44b2-460f-b09d-2005126a0814.png align="center")
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1690994864471/bda00088-04d2-4a83-b992-d4a52ab043c2.png align="center")
    
    Running the `cd` command without any arguments will take you to your home directory.
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1690994918639/4c900a8c-0a5d-4d7a-b13d-f94e76e74cf2.png align="center")
    
    Running the `cd` command with a dash (`-`) as an argument will take you to your previous working directory.
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1690994975484/9ffed642-bbc7-4e85-a898-ad81e6c951e5.png align="center")
    

*Stay tuned for upcoming blogs for advanced navigation.*

### Managing Directories

* **To create a new directory**, run `mkdir` command followed by the directory name, you want to create.
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1690995259294/db30326d-24e7-4556-a5e2-8361c0dc575d.png align="center")
    
    The long listing shows a `d` character next to the name of the `new-directory`. This indicates that `new-directory` is a **directory.**
    
* **To create directories and subdirectories in bulk,** you can use the `mkdir` command with the `-p` argument. This will create the directories and subdirectories in a single step.
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1690995721376/0339c6c8-defe-44b2-b3c5-6f86fe79884f.png align="center")
    
    **The** `-R` argument to the `ls` command lists directories recursively. This means that it lists all of the directories and files in the current directory, as well as all of the subdirectories of those directories. However, there is a better way to traverse directories. I will show you how to do this in an upcoming blog on advanced commands.
    
* **To delete a directory**, you can use the `rm -r` command. This will recursively delete the directory, including all of the subdirectories and files it contains. It is a **permanent operation**, so be sure to back up any files that you want to keep before using this command.
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1691048375746/fef34d35-94e1-43d8-a77c-6318840fb6b5.png align="center")
    
* To copy a directory, you can use the `cp -r` command to copy a directory, followed by the **source** and **destination** name.
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1691048918753/c697ae55-fd1c-485b-b942-65bc331a77c2.png align="center")
    
* **To rename or move a directory** to a different location, use the `mv` command the same way you would use it to rename or move a file.
    

***Tip: Use the tab key to autocomplete commands, which can help you avoid typos.***

That's all for this part. I hope you found this article informative and helpful. If you have any feedback, please feel free to share it in the comments below. Thank you for reading!

This is the second part of the [Master Linux series](https://anurag-rajawat.hashnode.dev/series/linux-the-practical-way), where you've learned about the Linux CLI and some basic commands. In the upcoming blogs, you will learn about advanced commands. Stay tuned!