---
title: "Hands-On File Operations"
seoDescription: "Learn the essential commands for file operations on the command line in this hands-on guide."
datePublished: Fri Aug 18 2023 04:30:09 GMT+0000 (Coordinated Universal Time)
cuid: cllg3dnok001l09l1exow2hq6
slug: hands-on-file-operations
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1692331912540/5fc7f75e-3064-4c90-9c27-142025ce5814.jpeg
tags: linux, linux-for-beginners, linux-basics, linux-commands

---

## Introduction

**Welcome everyone! 👋🏻**

You learned about Linux file permissions and ownership in the [previous part](https://anurag-rajawat.hashnode.dev/linux-file-permissions-and-ownership) of this [series](https://anurag-rajawat.hashnode.dev/series/linux-the-practical-way). In this part, you'll learn how to work with files in the command line.

So without further ado, let's get started! 🚀

## **Viewing files**

There are several commands you can use to look inside files without opening them in a text editor. Here are a few of the most common commands:

* [cat](https://en.wikipedia.org/wiki/Cat_(Unix)) - The `cat` command is a versatile command that can be used to print the contents of a file to the terminal, or to concatenate the contents of multiple files into a single file. It will print the entire file to the terminal, regardless of its size. This can be useful for small files, but it can be overwhelming for large files.
    
* [less](https://en.wikipedia.org/wiki/Less_(Unix)) - The `less` command is a more powerful alternative to `cat` command, as it allows you to scroll through the contents of a file one page at a time, and to search for specific text patterns.
    
* [head](https://en.wikipedia.org/wiki/Head_(Unix)) -  The `head` command prints the first few lines of a file to the terminal. This can be useful for quickly viewing the contents of a large file, or for getting a sense of what a file is about. By default, it will print the first 10 lines of a file. However, you can specify a different number of lines to print.
    
* [tail](https://en.wikipedia.org/wiki/Tail_(Unix)) - The `tail` command prints the last few lines of a file to the terminal. This can be useful for seeing the most recent changes to a file or for troubleshooting problems. By default, it will print the last 10 lines of a file. However, you can specify a different number of lines to print.
    

**Let's see all of these in action!**

### Using `cat`

* Viewing the content of `kubernetes.yaml` file.
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1692176497414/3b3b05c7-0d29-48c3-9e8f-f9cde9984e39.png align="center")

As you can see above the whole content of the `kubernetes.yaml` file has been printed to the console, regardless of its size.

* Printing the content of multiple files.
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1692180577104/9c65a6b0-334f-4b7d-bd61-b531f6e88c57.png align="center")

* Concatenating content of multiple files.
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1692180801331/6e6261a2-6391-4508-8b87-e3d7adcc0d05.png align="center")

The first command, `cat greet.sh hello.sh > concatenated.sh`, will concatenate the contents of the files `greet.sh` and `hello.sh` and save the result to a new file called `concatenated.sh`. The `>` symbol is used to redirect the output of the `cat` command to the file `concatenated.sh`. You don't need to worry about the `>` symbol for now, we'll cover it in more detail in the next part.

You can do a lot more with the `cat` command check man pages.

### Using `less`

The `less` command allows you to view a text file one page at a time. To search for a word, type `/` followed by the word and press Enter. The word will be highlighted in the text. This can be seen in the below image.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1692290127112/b181e739-fc81-4eca-9830-dd8d48e2be29.gif align="center")

### Using `head`

* Viewing the first few lines of the `kubernetes.yaml` file.
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1692176661430/1354868b-8ec3-4a95-9a96-486667dc173a.png align="center")

As you can see, the command only printed the first 10 lines of the file. You can verify this by counting the lines yourself.

* To view lines other than the default 10 lines, use the `head` command with the `-n` option followed by the number of lines you want to view. For example, you can see below I asked `head` to print the first 5 and 12 lines of the `kubernetes.yaml` file respectively.
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1692176904452/a14237ee-74b1-42fe-8187-915fd9592bba.png align="center")

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1692177022671/d7b80c1b-866e-4863-89d9-0d1f01834cc8.png align="center")

**TIP:** You can specify the number of lines to print by using the `-<number_of_lines>` option, without the `-n` option.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1692177244418/a93308a4-ffe6-47f8-90f0-463bb697d545.png align="center")

### Using `tail`

As you know the `tail` command is the opposite of `head`, so the syntax for both commands is similar, but with the `tail` command, you can monitor a file as it grows. See the below examples.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1692178569752/19ca5feb-6b95-471f-b35d-48272818330f.png align="center")

* Viewing a specific number of lines.
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1692178621635/2aee0676-4a66-4374-a882-f396405042ea.png align="center")

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1692178674957/8f3791a8-f674-46e6-8500-e6ed4e902b92.png align="center")

* Monitoring a file using `tail -f` command, can be seen in the below image.
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1692288930328/12b1f2a0-8536-4578-9c75-3325f679e7f5.gif align="center")

I ran the `hello.sh` script in the background and saved its output to the `hello` file using the `>` operator. This allows me to show you the usage of the tail -f command without blocking the terminal.

### Using `bat` (Bonus)

[bat](https://github.com/sharkdp/bat) is described as "**A cat(1) clone with syntax highlighting and Git integration."**

The bat command is a modern replacement for the `cat` command. It has all of the features of `cat`, plus syntax highlighting and Git integration.

To install it on your system follow their instructions [here](https://github.com/sharkdp/bat#installation).

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1692283280645/a2a96f39-60b4-4d80-aef8-a8acebd25991.png align="center")

As you can see above the output of the `bat` command is looking much better. Try it yourself, you won't be disappointed.

## Searching for data

Have you ever had to search for a specific line, word, or piece of text in a large file? If so, you know how tedious and time-consuming it can be to read through the file line by line.

That's where the [grep](https://en.wikipedia.org/wiki/Grep) command comes in. It is a powerful command-line tool that can be used to search for a specific pattern of text in a file. It's a lifesaver when you need to find something quickly and easily.

**Let's see it in action!**

* Searching a word `Deployment` in a file `kubernetes.yaml` that has 85 lines.
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1692183000279/aec038d5-cb73-4eea-b230-5ea72dc89f9c.png align="center")

As you can see above, it's easy to search for a word in a file using the `grep` command. But did you know that by default, grep only matches the word if it's spelt exactly the same way, including capitalization? This is fine if you know the word exactly, but what if you're not sure if it's capitalized correctly?

That's where the `-i` option comes in. The `-i` option tells grep to ignore the case when matching words. So, if you're searching for the word "hello", `grep` will also match the words "Hello" and "HELLO". This can be a very helpful option if you're not sure how a word is capitalized.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1692183697650/fef26c76-e7a2-4f65-8208-35c0beea36f2.png align="center")

* Sometimes, you may want to know the line number of the word that matches. To do this, you can use the `-n` option. As you can see below.
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1692183826369/2cf5e697-8949-47ae-9336-577a25278d94.png align="center")

* To count the number of occurrences of a word, use the `-c` option.
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1692291523823/36e46fd9-f494-4488-9d19-13081cae0c3e.png align="center")

* To search for the exact word, you can use the `-w` option which stands for "word match." This option tells grep to only match the exact word that you specify, and not any other words that contain that same sequence of letters.
    

For example, I just want to search `image` in a file called `kubernetes.yaml` as you can see below.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1692185114596/5bb05db1-9624-429d-a895-b4371785747d.png align="center")

Let's see what happens if we try to search word `image` without using the `-w` option.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1692185189438/0afd5402-fae9-4da8-9028-ef8130b4766c.png align="center")

As you expect, this will match all words starting with `image`.

This is just the tip of the iceberg. Explore all of these commands on your own to see what they can do. You may be surprised at how much you can accomplish with them.

I didn't explain writing to files in this post because most people use a text editor like Vim, Nano, Emacs or a graphical editor to do this. I will discuss writing to files in a separate blog post.

I hope you found this article informative and helpful. If you have any feedback, please feel free to share it in the comments below. Thank you for reading!

**Stay tuned for the next part of the** [**Master Linux series**](https://anurag-rajawat.hashnode.dev/series/linux-the-practical-way)**!**