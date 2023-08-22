---
title: "Redirection and Pipes in Linux"
seoDescription: "This guide will teach you everything you need to know about redirection and pipes in Linux"
datePublished: Tue Aug 22 2023 04:39:47 GMT+0000 (Coordinated Universal Time)
cuid: clllthfq5000w09lehf27fh52
slug: redirection-and-pipes-in-linux
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1692678317983/91dc154e-749f-4142-a8d8-648fc119114d.jpeg
tags: linux, linux-for-beginners, linux-basics, linux-commands

---

# Introduction

**Welcome everyone! 👋🏻**

In the [previous part](https://anurag-rajawat.hashnode.dev/hands-on-file-operations) of this [series](https://anurag-rajawat.hashnode.dev/series/linux-the-practical-way), you learned about file operations. In this part, you will learn about redirection and pipes, two powerful Linux essentials that can be used to improve your productivity and control the flow of data in Linux.

This part going to be long but I'm confident that by the end of this article, you'll have a clear understanding of redirection and pipes.

So without further ado, let's get started! 🚀

# File descriptors

In Linux, everything is a file, even the input and output streams. Each file is identified by a file descriptor, which is a non-negative integer. The bash shell reserves the first 3 file descriptors (0, 1, and 2) for special purposes:

* File descriptor 0 is the standard input (STDIN).
    
* File descriptor 1 is the standard output (STDOUT).
    
* File descriptor 2 is the standard error (STDERR).
    

Let's see this in more detail.

* **STDIN**
    

**STDIN** (standard input) is the stream that is used to read input from the user. By default, it is connected to the keyboard. However, you can use the input redirect operator (`<`) to redirect input from a file.

* **STDOUT**
    

**STDOUT** (standard output) is the stream that is used to write the output from a command. By default, it is connected to the terminal screen. The shell redirects the output from commands to STDOUT by default. However, you can redirect the output of a command to a file or another device using the output redirection operator (`>`).

* **STDERR**
    

**STDERR** (standard error) is the stream that is used to write an error message from a command. By default, STDERR is also connected to the terminal screen same as STDOUT. However, you can redirect the error using the `2>` redirect operator.

I know you're probably feeling confused right now. Don't worry, I'm going to explain everything in detail in the next section.

# Redirection

In the previous part, you used the `>` redirect operator to save the output of the `cat` command to a file. Let's take a closer look at what is redirection and how it works.

[Redirection](https://en.wikipedia.org/wiki/Redirection_(computing)) is the process of changing the destination of the output of a command. It can be used for both input and output. For example, you could use redirection to read the contents of a file into a command or, you can use redirection to save the output of a command to a file.

In Linux there are two main types of redirection operators:

* `>`: The greater-than operator redirects the standard output (STDOUT) of a command to a file or another command.
    
* `<`: The less-than operator redirects the standard input (STDIN) of a command from a file or another command.
    

## Redirect Standard Output

When I was learning about Linux and programming. I downloaded a few books on these topics and organized them into directories on my computer. I was excited to start reading them, but one day I accidentally deleted some system files. This caused all of my data to be deleted, including the books I had downloaded.

I didn't have a backup of the books, and I couldn't remember the names of all of them. I felt like I had lost a valuable resource.

However, I remembered something I had learned about Linux standard output redirection. So I ran the `tree` command to list all of the files in my directories, and I redirected the output to a file.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1692598633344/cc507951-0d7d-45c0-90e8-20316e46e57c.png align="center")

I saved that file to my Google Drive so that I could easily find it again if I needed to. I learned a valuable lesson that day: always make backups of your important files. I'm also grateful for the power of Linux standard output redirection.

There are two ways to redirect STDOUT:

* **Append (**`>>`)**:** This appends the output of the program to the end of a file. If the file does not exist, it is created.
    
* **Truncate (**`>`**):** This removes the file's content before writing the output of the program. If the file does not exist, it is created.
    

Let's see more examples.

* **Truncate:** As you can see below the output of `ls` command is successfully written to the `files` file. You can also use the `1>` redirection operator, it will work in the same way.
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1692599020072/3b0192c7-c850-4f81-a8ce-5b13944a01d9.png align="center")

* **Append:** As you can see below the output of `echo 'Append text'` command is successfully appended to the `files` file.
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1692678028938/b024001f-7ac9-4cf1-a78a-817e4d040968.png align="center")

*When redirecting standard output (STDOUT), it is important to be cautious about whether you want to append or truncate the output. If you are not sure which option to choose, it is always best to append. This will ensure that you do not lose any data.*

* Here the output of `ls` command is not saved, why?
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1692599203182/ce7153cb-1559-4db0-8a3b-00153f155f22.png align="center")

Because the `ls file-not-exist` command results in an error and we've not told the output redirect operator to redirect the error as well.

To redirect errors you've to use the STDERR descriptor that will be discussed in the ***redirect standard error*** section.

## Redirect Standard Input

In Linux, the standard input (stdin) is the source of input for a command. By default, the standard input is the keyboard. This means that when you type a command at the terminal, the command is reading its input from the keyboard.

However, you can redirect the standard input from a file or another command using the redirection operator `<`.

Let's see this in action.

* The following command reads the contents of the `files` file and passes them to the `cat` command:
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1692607901229/9c304076-7869-4bb2-ade2-f8b244ae3482.png align="center")

* Inline Input Redirection
    

**Inline input redirection** is a method of redirecting input from the command line instead of from a file. This can be useful when you want to provide input to a command that is not stored in a file.

To use inline input redirection, you use the double less-than sign (`<<`) followed by a delimiter. This can be seen in the below image.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1692608248865/ed3ef4f1-4c5c-44ae-b500-a0ca422f269c.png align="center")

The `EOF` character acts as a delimiter that indicates the end of the input. You can use any string value for the delimiter but it must be the same at the beginning and at the end of the data.

* Another example is redirecting standard input and standard output.
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1692608901790/057e2152-6b46-4456-bd1b-e945ef812534.png align="center")

The inline redirection in the command `cat << EOF >> demo-ns.yaml` is used to read the input from the keyboard and write it to the file `demo-ns.yaml`.

The `<<` operator is called a here-document delimiter. It tells the shell to read the input until it sees the delimiter `EOF`.

The text between the delimiters is the input for the `cat` command.

The `>>` operator is used to append the output of the `cat` command to the file `demo-ns.yaml.`

## Redirect Standard Error

Let's see an example that prints the error.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1692611818171/91832351-1d1d-4187-b6fe-b4072c61b302.png align="center")

There are a few ways to redirect error messages. Let's take a look at them.

### Redirect only STDERR

Let's see the same example again, but this time we'll use the error redirection operator (`2>`). This operator tells the command to redirect all error messages to a file instead of printing them to the terminal. This can be seen in the below image.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1692612014772/6c74cc30-425a-4eb4-980e-ee59a3cdd925.png align="center")

### Redirect both STDOUT & STDERR

If you've ever worked with scripts, whether directly or indirectly, you've probably seen that they often print their output to the terminal and redirect their errors to a separate file mostly `/dev/null`. This is also done using redirection.

By redirecting errors to a file, the script can keep the output clean and easy to read. The errors can then be reviewed later, if necessary. In some cases, the errors can also be discarded, if they are not important.

Let's see this in action!

* I wanted to concatenate the contents of the files `file1` and `file2` and redirect the standard output (STDOUT) to a file called `exists`. I also wanted to redirect any errors (STDERR) to a file called `not-exist`. This can be seen below.
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1692615012426/82e8daee-a2d7-494a-8b0e-5d3ce5715c56.png align="center")

The `>` operator redirects the STDOUT to the file `exists`. The `2>` operator redirects the STDERR to the file `not-exist`.

* In this example, we redirect both STDERR and STDOUT to the same `output` file.
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1692616007451/3315d0f2-3e9e-4b65-91bd-18046facd48e.png align="center")

The `2>&1` redirection operator will redirect both the standard output and the standard error of the `cat` command to the file `output`.

You can also use the `&>` operator to shorten the `2>&1` operator. The `&>` operator is a shorthand for "redirect both the standard output and the standard error to the same file".

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1692616733075/902cd9ba-6834-4931-ba89-09dfe5eb0fef.png align="center")

# Pipes

Imagine a pipeline. It's a long, narrow channel that carries water from one place to another. The water flows through the pipeline, and it can be used to do things like irrigate crops or generate electricity.

[Pipes](https://en.wikipedia.org/wiki/Redirection_(computing)#Piping) in Linux work in a similar way. They allow you to connect the output of one command to the input of another command. They can be used to chain together any number of commands. This makes them a very powerful tool for automating tasks and performing complex operations.

A pipe is represented by `|`

Let's see a couple of examples for a better understanding.

* Let's say you want to find the number of files in the current directory, you could use the following commands.
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1692617724081/a511cde3-0079-46cf-95bb-a07ee6c4b62f.png align="center")

The `ls` command lists the contents of a directory and the `wc` command counts the number of lines in a file. In this case, the `wc` command is counting the number of lines in the output of the `ls`command. The `-l` option tells the `wc` command to count the number of lines, rather than the number of words or characters.

* Let's say you want to know how much space is occupied by the `var` directory. You could use the following commands:
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1692618608294/29a38ade-4b80-40f4-b565-39a8ca75a235.png align="center")

The command `sudo du -h /var | tail -1` uses the `du` command to find the total size of the `/var` directory, and then uses the `tail` command to display the last line of output.

* Let's say you want to download `go1.21.0` tar file according to your system architecture using the command line. You could use the following commands:
    

```bash
$ curl -s https://go.dev/dl/ | grep -w "go1.21.0.linux-$(dpkg --print-architecture).tar.gz" | tail -1 | cut -d'"' -f6 | cut -d"/" -f3
```

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1692619389863/e38ef444-e11f-40a0-911d-5d038b3e197c.png align="center")

Let's break down this big daunting command.

* The `curl` command downloads the contents of [`https://go.dev/dl/`](https://go.dev/dl/%60) URL. The `-s` option tells the `curl` command to be silent, so it will not print any output to the console.
    
* The `grep` command will search for the word `go1.21.0.linux-$(dpkg --print-architecture).tar.gz` in the output of the curl command. The `dpkg --print-architecture` gives the architecture of the system.
    
* The `tail -1` command takes the last line of the output of the `grep` command.
    
* The `cut -d'"' -f6` command takes the sixth field from the output of the `tail` command. The fields are separated by double quotes.
    
* The `cut -d"/" -f3` command takes the third field from the output of the `cut -d'"' -f6` command. The fields are separated by forward slashes.
    

Imagine doing this without using pipes 🤯.

I hope by now you've gotten a better understanding of pipes and how the output of one command can be passed as the input to another command using pipes and how they can be used to chain commands together.

**That's all for this part!**

I hope you found this article informative and helpful. If you have any feedback, please share it in the comments below. Thank you for reading!

**Stay tuned for the next part of the** [**Master Linux series**](https://anurag-rajawat.hashnode.dev/series/linux-the-practical-way)**!**