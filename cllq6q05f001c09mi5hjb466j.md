---
title: "Data Compression in Linux"
seoDescription: "A Hands-on guide to data compression in Linux, covering all the essential concepts and techniques."
datePublished: Fri Aug 25 2023 06:01:26 GMT+0000 (Coordinated Universal Time)
cuid: cllq6q05f001c09mi5hjb466j
slug: data-compression-in-linux
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1692943254278/76edb6d3-9206-4abc-a500-034c6057f2a4.jpeg
tags: linux, linux-for-beginners, linux-basics, linux-commands

---

## Introduction

**Hi there! 👋**

In the [previous part](https://anurag-rajawat.hashnode.dev/redirection-and-pipes-in-linux) of this [series](https://anurag-rajawat.hashnode.dev/series/linux-the-practical-way), you learned about redirection and pipes in Linux. In this part, you'll learn how to compress files and folders in Linux.

So without further ado, let's get started! 🚀

## What is data compression?

[Data compression](https://en.wikipedia.org/wiki/Data_compression) is the process of reducing the size of a file by removing redundant data. This can be done using a variety of algorithms, each with its own strengths and weaknesses.

## Why compress files?

There are many reasons why you might want to compress files. Here are a few:

* To save space on your hard drive or other storage device.
    
* To make it easier to transfer files over the network.
    
* To archive files for future use.
    

## Compression utilities

Linux has several file compression utilities, each with its own strengths and weaknesses. The most popular utilities are:

* [**gzip**](https://en.wikipedia.org/wiki/Gzip)**:** This is a lossless compression utility that uses the gzip compression algorithm. It is very efficient and can compress files with a high degree of compression.
    
* [**bzip2**](https://en.wikipedia.org/wiki/Bzip2)**:** This is another lossless compression utility that uses the bzip2 compression algorithm. It is slightly less efficient than gzip, but it can compress files even smaller.
    
* [**zip**](https://en.wikipedia.org/wiki/ZIP_(file_format))**:** This is a lossy compression utility that uses the zip compression algorithm. It is not as efficient as gzip or bzip2, but it can compress files even smaller.
    

The [tar](https://en.wikipedia.org/wiki/Tar_(computing)) command is the most popular archiving tool in Unix and Linux. It can be used to create, extract, and list archives. Archives can contain multiple files and folders, and they can be compressed using a variety of algorithms.

**Tar command syntax**

`tar [-options] <name of the tar archive> [files or directories which to add into archive]`

***NOTE: There is a difference between data compression and data archiving.***

> Data compression can be used as part of the data archiving process. By compressing the data before it is archived, you can reduce the amount of storage space that is needed. However, data compression is not the same as data archiving. Data compression is a technique for reducing the size of a file, while data archiving is a process for storing data for long-term preservation.

## Compression

Now you know what and why of data compression, let's see this in action!

* **To create an archive of a directory**, you can use the `-cf` option. The `-c` means "create" and the `-f` is used to specify the name of the archive file to create.
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1692862849886/ebdcb16c-16ea-44bf-b85c-7ebb221f1200.png align="center")

As you can see above, the command creates a compressed archive file called `data.tar` in the current working directory. The archive file contains all the content of the `data` directory.

Want to know which files and directories are added or extracted to or from the archive? Use the `-v` option, which stands for **verbose output**. See the example below:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1692862954930/e8b43eb7-a07a-4ac5-b8c2-170205f4f322.png align="center")

* **To create an archive with gzip compression**, tell tar to use gzip for data compression by using the `-z` option.
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1692863230377/b193e9f4-b89b-4079-8af3-d4f07edd45e5.png align="center")

You can use the `.tgz` or `.tar.gz` extension for a tar file. Both are the same. The `.tgz` extension is more common, but the `.tar.gz` extension is also accepted by most applications.

* **To create an archive with bzip2 compression**, tell tar to use bzip2 for data compression by using the `-j` option.
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1692863121483/0677301d-13b8-4558-a15c-3c7272cdf485.png align="center")

* To create an archive with [**xz**](https://en.wikipedia.org/wiki/XZ_Utils) compression, tell tar to use xz for data compression by using the `-J` option.
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1692863171097/2ee1adcd-8b86-4b58-881b-db9697966ae6.png align="center")

## Decompression

The tar command can automatically detect the compression type of an archive and decompress it. This means that you don't need to specify the `-z`, `-j`, or `-J` options to extract a tar file.

This is a relatively recent change to the tar command. In older versions of tar, you did need to specify the option to extract a tar file.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1692863867776/1bd93665-cfc6-4232-a2c2-65ed1c9cc4be.png align="center")

Decompressing gzip compressed tar file.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1692866243888/623a3806-cf7c-416e-97fc-66608f0fc678.png align="center")

**To decompress an archive to a specific destination,** use the `-C` option after the archive name.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1692863907814/2393cefe-22a3-4e58-8f93-bc16b0d794f8.png align="center")

## Listing content

**Can you list the content of an archive file without extracting it?**

Yes, you can! Let me show you how.

To list the content of an archive use the `-tf` option where `-t` is used to list the contents and `-f` is used to specify the file.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1692865161040/8fbe09a6-ce00-41c8-8aa6-dc6b6febf1e3.png align="center")

**That's all for this part!**

I hope you found this article informative and helpful. If you have any feedback, please share it in the comments below. Thank you for reading!

**Stay tuned for the next part of the** [**Master Linux series**](https://anurag-rajawat.hashnode.dev/series/linux-the-practical-way)**!**