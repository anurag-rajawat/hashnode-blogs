---
title: "Linux Process Management"
seoDescription: "Master the Linux command line with this guide to Linux commands."
datePublished: Tue Aug 08 2023 04:29:59 GMT+0000 (Coordinated Universal Time)
cuid: cll1syx6k000009jpdgkag8ta
slug: linux-process-management
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1691468865373/462ef790-8372-4045-955f-d64449c64cb2.png
tags: linux, linux-for-beginners, linux-basics, linux-commands

---

## Introduction

**Welcome everyone! 👋🏻**

In the [previous part](https://anurag-rajawat.hashnode.dev/linux-command-line) of this [series](https://anurag-rajawat.hashnode.dev/series/linux-the-practical-way), you learned about some basic commands and got a feel for the CLI. In this part, you'll learn about Linux process management.

So without further ado, let's get started! 🚀

## Man Pages

Most Linux distributions include an online manual for looking up information on shell commands. These manuals are called [man pages](https://en.wikipedia.org/wiki/Man_page) because they are read using the `man` command. Man pages typically include complete details of a program's options, helpful examples, and related commands.

You may wonder why you need man pages when you can just use Google to find information about Linux commands. You can but man pages are concise descriptions of individual commands, file formats, or library routines. They are often more accurate and up-to-date than information found on Google.

To access a man page, simply type `man` followed by the name of the command you want information about. For example, to see the man page for the `ls` command, you would type `man ls`.

I encourage you to explore the man pages for the commands you use regularly. You'll be surprised at how much you can learn about them.

## Process Management

A [process](https://en.wikipedia.org/wiki/Process_(computing)) is a running instance of a program. The `ps` command is a great way to see all the processes that are running on your system. The GNU `ps` command that’s used in Linux systems supports three different types of command line parameters:

* Unix-style parameters, which are preceded by a dash.
    
* BSD-style parameters, which are not preceded by a dash.
    
* GNU long parameters, which are preceded by a double dash.
    

That’s a lot of parameters! The key to using the `ps` command is not to memorize all the available parameters, only those you find most useful.

### Monitor Process

* **To list all processes of a user**, use the `ps` command with the `-fu` flag followed by the name of the user.
    
    * The `-f` flag tells `ps` to output the full process information.
        
    * The `-u` flag tells `ps` to only list the processes for the specified user.
        
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1691467296657/1d54cfae-8bc3-4e66-8858-3a8a9cdfa9a7.png align="center")
    
    Let's break down the output:
    
    * **UID:** The user ID of the process owner.
        
    * **PID:** The process ID of the process.
        
    * **PPID:** The parent process ID of the process.
        
    * **C:** The processor utilization for scheduling. This column shows the percentage of time in the schedule spent on certain processes.
        
    * **STIME:** The time at which the process started.
        
    * **CMD:** The command used to start the process.
        

**To get an overview of all processes running on the system**, use `ps` command with `aux` argument.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1691339293392/c66aaee4-bb55-4e9d-8379-27abe5f99b48.png align="center")

The output of the ps command provides a lot of information about each process. Let's break it down:

* **USER:** The user who launched the process.
    
* **PID:** The process ID of the process.
    
* **%CPU:** The percentage of CPU used by the process.
    
* **%MEM:** The percentage of memory used by the process.
    
* [**VSZ**](https://en.wikipedia.org/wiki/Virtual_memory)**:** The virtual memory that the process can access.
    
* [**RSS**](https://en.wikipedia.org/wiki/Resident_set_size)**:** The Resident Set Size (RSS) shows how much memory a process is using in RAM.
    
* **TTY:** The control terminal ID.
    
* **STAT:** The current status of the process.
    
* **START:** The time at which the process started.
    
* **TIME:** The CPU time used by the process.
    
* **COMMAND:** The command used to start the process.
    

For more information on what the ps command can do, use the manual.

### Interactive Monitoring

The `ps` command can display information only for a specific point of time, it is somewhat limited. If you want real-time updates, interactive summaries, and resource usage you can use `top` command.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1691341000076/2267a4fb-ebc1-47b4-91a6-320d0f4ad216.png align="center")

When you start the top command, it will sort the processes by their CPU usage by default. You can change the sort order by using one of several interactive commands while the top is running. The display updates every 1-2 seconds, depending on the system. The first few lines show summary information, which is a good place to start when analyzing the health of the system. The next section shows a detailed list of the currently running processes. The following columns are different from the output of the ps command:

* **PR** - The priority of the process.
    
* **NI -** The nice value of the process.
    
* **VIRT -** The virtual memory that the process can access.
    
* **RES -** The amount of physical memory the process is using.
    
* **SHR -** The amount of memory the process is sharing with other processes.
    
* **S -** The status of the process.
    
* **TIME+ -** The CPU time used by the process.
    

There are some other tools available like [htop](https://htop.dev), [glances](https://github.com/nicolargo/glances) which offer more features and a nice interface than the top.

### Stop Process

Processes in Linux communicate with each other using [signals](https://en.wikipedia.org/wiki/Signal_(IPC)). Signals are predefined messages that processes can choose to ignore or act on. Administrators and terminal drivers can send signals to processes to kill, interrupt, or suspend them.

To stop a hung process, you can send it a signal. If the process ignores the signal, you can forcefully stop it. Below are common signals that you should be familiar with:

| Signal | Name | Description |
| --- | --- | --- |
| 1 | HUP | Hangup |
| 2 | INT | Interrupt |
| 3 | QUIT | Quit |
| 9 | KILL | Kill unconditionally |
| 15 | TERM | Terminate gracefully |
| 17 | STOP | Stop unconditionally, but do not terminate |
| 19 | CONT | Resume after stop |

Many different signals can be sent to processes. To see a list of all the signals, use the `man signal` command.

To send a signal to a process, use the `kill` command. The syntax is: `kill -[SIGNAL] PID` where `SIGNAL` is the number or name of the signal to send, and `PID` is the process ID of the process you want to send the signal.

**You must either be the owner of the process or the root user to send a process signal.**

Let's see some examples:

* **Terminating a process** - In this example, there is a script that prints some text, sleeps for 5 seconds, and then continues. To terminate it, use the `kill` command followed by the signal name or number and followed by the process ID, as shown in the screenshot below.
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1691404902414/749fdd3e-9336-4453-9145-b92aa0a897c8.png align="center")
    
    ***Don't worry about scripting, it'll be covered soon. Stay tuned.***
    
* **Killing a process -** In this example, the process refused to terminate gracefully so it will be forcefully terminated using the `KILL` signal as shown in the screenshot below. Use the `KILL` signal only as a last resort, after a polite request has failed and the process is still running. The process may not be able to catch the `KILL` signal.
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1691406219846/9bd89417-4aba-474d-b32b-39a9d17a27de.png align="center")
    
    Let's see one more example to kill a process that is not owned by you.
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1691406791314/0782493b-856c-4b3c-a577-b236c3f8e902.png align="center")
    
    I think you know why we get this error. If not let me tell you why that happened, ***"You can only kill processes that you own. To kill other processes, you must be the root user."*** As you can see in the below screenshot.
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1691407236574/0db4b4d0-65ea-4f34-8e7a-8898537e9dbc.png align="center")
    

If you're not familiar with root or sudo, don't worry, we'll cover them in more detail in upcoming blogs. Stay tuned!

I leave other signals for you to experiment with.

There is a special way to kill processes by their names using `killall` command. This command will send the `KILL` signal to all processes and it cannot be ignored by the processes. This can be useful in two cases:

1. You know the name of the process that you want to kill, but you don't know its process ID.
    
2. You want to kill a group of processes that are all hung or unresponsive.
    

For example, if you have a web server that is not responding, you could use the `killall` command to kill all processes. This would force the webserver to stop. As you can see in the below screenshots.

All `apache2` webserver processes.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1691409390180/fee958b6-02ca-4d0d-be90-ea84ee829055.png align="center")

Killing all of them

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1691409534598/5c0570e2-6b5c-46ed-a78d-ce2ee32c615e.png align="center")

I don't think I need to explain anything here.

As usual, you can get more information about any of these commands by using the `man` command to view the man pages.

**That's all for this part!**

I hope you found this article informative and helpful. If you have any feedback, please feel free to share it in the comments below. Thank you for reading!

**Stay tuned for the next part of the** [**Master Linux series**](https://anurag-rajawat.hashnode.dev/series/linux-the-practical-way)**!**