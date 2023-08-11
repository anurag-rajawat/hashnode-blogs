---
title: "Linux Jobs Management"
seoDescription: "Learn how to manage your Linux jobs effectively with this hands-on guide."
datePublished: Fri Aug 11 2023 04:47:10 GMT+0000 (Coordinated Universal Time)
cuid: cll63wkjl000k09mo9js6djl0
slug: linux-jobs-management
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1691728163437/5b90450c-9b9e-425e-9c33-e8f71d7e4bd1.jpeg
tags: linux, linux-for-beginners, linux-basics, linux-commands

---

## Introduction

In the [previous part](https://anurag-rajawat.hashnode.dev/linux-process-management), you learned about process management in Linux. Now, let's build on that knowledge and learn about Linux jobs.

## Jobs

A Linux [job](https://en.wikipedia.org/wiki/Job_control_(Unix)) is a group of related processes. It's like a team of employees working on a project. The parent process is the manager, and the child processes are the employees.

Jobs are important because they allow you to manage groups of processes as a single unit. This can be helpful for things like scheduling, resource allocation, and debugging.

In Linux, there are two types of jobs:

1. **Background Jobs**
    
2. **Foreground Jobs**
    

### Background Jobs

**Background jobs** run in the background, which means they are not displayed on the terminal screen and do not take up all of the CPU resources. This allows you to continue using your terminal while background jobs are running.

### Foreground Jobs

**Foreground jobs** run in the foreground, which means they are displayed on the terminal screen and take up all of the CPU resources. This means you cannot use your terminal while a foreground job is running.

### Use cases

Now that you know about foreground and background jobs, let's explore where and how to use them.

Imagine you're working on a long-running task, like a backup or a data processing job. You don't want to wait for the task to be completed before you can do anything else, so you want to run it in the background. But how do you do that? If you're using a GUI, you can just open a new terminal and start the task there. But what if you're working on a Linux server and you don't have a GUI?

That's where background jobs come in. Background jobs allow you to run a process in the background so that it can do its work without tying up your terminal. This means you can continue using your terminal to do other things.

## Jobs Management

**Let's See it in Action!**

### List all jobs

**To see all the jobs that are currently running** on your system, use the `jobs` command.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1691646469338/5403f2ef-63eb-447b-9f8e-4402e0b6fa33.png align="center")

As you can see, no jobs are running on my system yet. Let's try running a few commands and then check again to see if any jobs are running.

### Run a job in the background

**To run a process in the background** so that it doesn't tie up your terminal just append ampersand (`&`) at the end of a command. For example, to download a large file.

As you can see below the process is in the foreground and tied up with the terminal.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1691646729906/6b9c6d5a-73f7-4026-a86f-e8b26271a656.png align="center")

Run it in the background

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1691646781682/0cbc3d95-547b-4bb4-a431-4e1b8c00365e.png align="center")

If you have sharp eyes you've noticed that when you run a process in the background, it often outputs a number. That number is the process ID (PID) of the job.

Let's break down the output of the `jobs` command:

* `[1]+`: The job ID or job number. Each job has a unique ID, which is used to identify the job and is referred using a `%` character when you want to control it. The `+` sign indicates that the job is currently running.
    
* `Running`: The status of the job.
    
* `COMMAND`: The command that was used to create the job.
    

There is another way to do the same. First, execute the command, and then press `CTRL + Z` to suspend it. This will put the process in a **suspended** state, which means that it will stop running but **it will not be terminated**.

To continue running the process in the background, you need to use the `bg` command followed by the `% <job_id>` as an argument. As you can see below.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1691651674884/b4edf205-be2c-4a03-96d4-311f5ae9d3d1.png align="center")

Did you notice something? The job status changed from `Stopped` to `Running` !

### Bring a Job in the foreground

**To bring a job back in the foreground**, use the `fg` command followed by the `% <job_id>` as an argument. As you can see below.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1691651867250/c04680ae-efbb-4c96-af7f-44e54879091a.png align="center")

### Stop a background job

So you've created a background job, but now you need to stop it. How will you do that?

You can use the `kill` command to stop it. You can either use the job ID or the process ID (PID) of the job you want to stop as an argument to `kill` command. You can find the job ID or PID of a background job by using the `jobs` command followed by `-l` flag. As you can see below.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1691652318505/6f172bab-4065-41e8-9fc0-76bf19e87239.png align="center")

If you have only one background job running, you can stop it quickly and easily by using the `kill` command followed by the `%` character. The `%` character refers to the job number of the last background job. So, if you type `kill %`, it will stop the last background job that is running.

The `kill` command by default sends the **SIGTERM** signal to the process, which tells the process to exit gracefully. However, you can also use other signals with the `kill` command.

**NOTE: Use** `%` character **only when you use the job ID otherwise directly use PID.**

**TIP**:

* You can use the `bg` or `fg` command followed by the `%` character as an argument if you've only one job. If you're lazy just using `%` character you can bring your last background job in the foreground.
    
* You can also use the `bg` or `fg` command without any argument to bring your last job to the background or foreground.
    

## **Disown a job**

So you've created a long-running background job, and you don't want it to stop running if you close your terminal or your session gets closed for any reason. How do you do that?

You can use the `disown` command followed by the `% <job_id>` as an argument to do so. The `disown` command will detach a background job from your current shell session. This means that the job will no longer be listed in the `jobs` command output, and it will not be affected by signals sent to the shell and continue running even if you close your terminal or your session gets closed. As you can see below.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1691656084002/3bc2b4f0-498d-4e1e-8699-fb4653396ae3.png align="center")

After logging back into the system

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1691655466299/0737b2ad-4fce-4a5f-802d-e7d7b00c6f8d.png align="center")

You can see it is still running.

To stop it, you can use the `kill` command in the same way that you would use it to kill another process. Simply type `kill` followed by the process ID (PID) of the job. As you can see below.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1691655574105/626a6851-2e38-4e3e-8588-20de438f7854.png align="center")

As usual, you can get more information about any of these commands by using the `man` command to view the man pages.

**That's all for this part!**

I hope you found this article informative and helpful. If you have any feedback, please feel free to share it in the comments below. Thank you for reading!

**Stay tuned for the next part of the** [**Master Linux series**](https://anurag-rajawat.hashnode.dev/series/linux-the-practical-way)**!**