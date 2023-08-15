---
title: "Linux File Permissions and Ownership"
seoDescription: "Learn how to manage permission and ownership of your Linux files effectively with this hands-on guide."
datePublished: Tue Aug 15 2023 04:30:10 GMT+0000 (Coordinated Universal Time)
cuid: cllbt242v000208kwhiyw7r9o
slug: linux-file-permissions-and-ownership
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1692073670926/bfc4d300-d24c-4a11-a7eb-885c2033a281.png
tags: linux, linux-for-beginners, linux-basics, linux-commands

---

## Introduction

**Welcome everyone! 👋🏻**

In the [**previous part**](https://anurag-rajawat.hashnode.dev/linux-jobs-management) of this [**series**](https://anurag-rajawat.hashnode.dev/series/linux-the-practical-way), you learned about Linux job management. In this part, you'll learn about Linux files, file permissions and ownerships.

So without further ado, let's get started! 🚀

## File types

In Linux/Unix systems everything is a file and it can be one of the following seven types defined by most of the filesystems.

* Regular files
    
* Directories
    
* Symbolic links
    
* Character device files
    
* Block device files
    
* Sockets
    
* FIFO (named pipes)
    

*Please visit* [*Unix file types*](https://en.wikipedia.org/wiki/Unix_file_types) *for more info about these file types.*

To check the type of file you can use the `file` command. As you can see below.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1691902953348/f0791490-d3f3-4b8b-bea5-ae4500acdc14.png align="center")

In the [second part](https://anurag-rajawat.hashnode.dev/linux-command-line#heading-listing-files-and-directories), you learned that the `ls -l` command shows more information about files and directories in a long list format. I promised to revisit this command in more detail and now is the time to do so.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1691903496511/2d1b7007-7eb4-4cf1-9e42-3aa4177495ab.png align="center")

The first column of the `ls -l` command output shows the permissions of each file, the second column shows the owner of the file, and the third column shows the group to which the file belongs.

## File Permissions

The permissions of a file are represented in the format `drwxrwxrwx`. The first character represents the type of file. The next nine characters are the three sets of permission bits that control who can read, write, or execute the content of the file.

* The first set of permission bits represents the permissions for the **user** (the owner of the file).
    
* The second set of permission bits represents the permissions for the **group** that the file belongs to.
    
* The third set of permission bits represents the permissions for **other users**.
    

The order of the bits within each set is:

* **Read (**`r`**)**: The user can read the contents of the file.
    
* **Write (**`w`**)**: The user can write to the file.
    
* **Execute (**`x`**)**: The user can execute the file (if it is a program).
    

A dash (`-`) in a permission bit position indicates that the permission is denied for that user or group.

Let's decode the file permission of `hello.sh` which is `-rwxrwxr-x`.

* The first character `-` represents that it is a regular file.
    
* The first set `rwx` represents that it is readable, writable, and executable by the owner.
    
* The second set `rwx` represents that it is readable, writable, and executable by the group it belongs to.
    
* The next set `r-x` represents that is only readable and executable by other users.
    

To get a better understanding of file permissions, I encourage you to decode the permissions for other files on your own.

### Default File Permissions

I know you may have a doubt about where these file permissions come from.

Imagine you are the owner of a new house. You want to ensure that your family and friends have access to your house, but you also want to ensure that it is secure from intruders. You set the default permissions for your doors and windows so that only you and your family can open them.

The `umask` command in Linux works in a similar way. It sets the default permissions for any file or directory you create. The default umask value varies depending on the Linux distribution. On most distributions, the default umask value for root is `0022`, and the default umask value for non-root users is `0002`. The first zero in the umask value represents the special bits, and the remaining three values represent the umask value in the octal form. The special bits control the setuid, setgid, and sticky bits, which are advanced permissions that are not used often.

Umask is a powerful tool that deserves its own article. I'm not going into the details here, you can [check here](https://en.wikipedia.org/wiki/Umask) for more details.

### Change File Permissions

Now that you know about file permissions and the default permissions, let's learn how to change them depending on your needs.

***Only the owner of the filer or root user can change a file's permission.***

You can change the file permissions using the `chmod` command. The syntax of `chmod` command `chmod [options] permissions file...`

* **Options** are optional modifiers that can be used to change the behaviour of the `chmod` command. For example, the `-R` option can be used to change the permissions of all files and directories recursively.
    
* **Permissions** are a set of three letters or octal values that represent the read, write, and execute permissions for the owner, group, and others.
    
* **Files** are the files or directories whose permissions you want to change.
    

The following table shows the possible combinations of permissions and their meanings:

| **PERMISSIONS** | REPRESENTATION | OCTAL |
| --- | --- | --- |
| No permissions | `---` | 0 |
| Execute only | `--x` | 1 |
| Write only | `-w-` | 2 |
| Write and execute | `-wx` | 3 |
| Read-only | `r--` | 4 |
| Read and execute | `r-x` | 5 |
| Read and write | `rw-` | 6 |
| Read, write, and execute | `rwx` | 7 |

There is no need to memorize the table, you can simply remember that `r` = 4, `w` = 2, and `x` = 1, and then you can derive the octal value for any combination of permissions.

* `rwx` = `r` = 4, `w` = 2, `x` = 1 =&gt; 4 + 2 + 1 = 7
    
* `rw-` = `r` = 4, `w` = 2 =&gt; 4 + 2 + 0 = 6
    
* `-wx` = `w` = 2, `x` = 1 =&gt; 0 + 2 + 1 = 3
    

I hope this helps you to understand how to derive the octal value for file permissions.

Let's see it in action.

* **Setting executable permission.**
    

I have a script `greet.sh` but as you can see below when I tried to execute it failed to execute. Do you know why?

Because its execution bit is not set.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1691912875683/fc49d1d2-5784-45c0-af1c-7fd5b10be8cf.png align="center")

Now change it so that I can execute it. As you can see below in the first set of permissions, the execution bit is set.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1691913072504/1b76015f-8689-4d24-a479-04e28fd477bf.png align="center")

When I tried again to execute it it worked. See the below image.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1691913271173/559a1b7b-513d-457a-bdd0-f45a3d140536.png align="center")

* **Removing permission**
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1691913529294/df94b287-e595-4222-8838-4aa534eda104.png align="center")

I don't think I need to explain anything here.

I leave other combinations to try as an exercise for you.

## File Ownership

Now that you know about file permissions, let's talk about file ownership and group membership.

When you use the `ls -l` command to list files in long format, the second and third columns show the owner and group of each file. The owner is the user who created the file, and the group is a collection of users who have similar permissions to the file.

You can use the `chown` command to change the owner of a file and the `chgrp` command to change the group of a file. By default, only the root user or a user with sudo privileges can change the owner of a file. This is done for security reasons, to prevent unauthorized users from changing the ownership of files that they do not own.

Let's see it in action!

### Change owner

To change the owner of a file use `chown` command followed by the owner you want to change and the file name. As you can see below I changed the owner from `anurag` to `robot`.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1691943916771/f36ed061-80b2-4b2b-b643-448cfa6df458.png align="center")

Let's try to execute it.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1691944038211/ad34298c-a42f-490c-a1d1-183c46647b6e.png align="center")

Since we changed the owner of the file and the execution bit for the group was not set. This means that we are no longer the owner of the file, and the group still does not have permission to execute it. Therefore, we get a permission denied error when we try to execute the file.

Let's read it and see what happens.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1691944490058/a6993a5a-949f-4b02-94a5-3884ef9a779f.png align="center")

We can read it, why?

Because the group has permission to read and write but not to execute. So let's change permission for the group and see what happens.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1691944715678/ae5bc932-d978-4756-82c6-9be9f7e546e3.png align="center")

We failed to change its permission, I think you know why.

Because we are no longer the owner of the file, only the owner or root user can change permission. So let's try again with sudo privileges.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1691944865382/0387e9d2-34f2-4a71-9c3a-e826248f5d35.png align="center")

As we expected it worked.

Now try again to execute it.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1691944938470/526105d2-4a6f-4dd3-acc2-c953e1aa222b.png align="center")

This time it is successfully executed.

If you want to change owner as well as group you can use the `chown user:group files...` command. As you can see below.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1691993105870/85711900-c572-451a-8d34-a009bca22fcb.png align="center")

### Change Group

To change the group ownership of a file, use the `chgrp` command followed by the new group name and the file name. As you can see below the group changed from `anurag` to `devs`.

*If you don't know how to create a group, use the following command. We will cover how to create a group in an upcoming blog post.*

```bash
$ sudo groupadd <group_name>
```

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1691991652629/4a052ecf-e836-4e0e-a7a8-bbe5c556df28.png align="center")

Now try to execute `greet.sh` script

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1691991777320/8aa18e65-babf-4b57-b8d4-d5750ec05c35.png align="center")

We got a `Permission denied` error, do you know why?

Because the user who is executing the script does not have the necessary permissions to access the file. The user is neither the owner of the file nor a member of the `devs` group. So you need to add the current user to the `devs` group and reboot the system.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1691992030837/a06630f9-2526-4df2-bbaa-3443c9372bc3.png align="center")

Let's try again to execute the script and see what happens,

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1691992402288/22de26df-653f-48b6-9e3b-4a4310f908cd.png align="center")

As we expected it worked.

**Here's another example:** To change the group of all files and subdirectories in a directory, you can use the `chgrp` command with the `-R` option. This will recursively change the group ownership of all files and directories in the specified directory.

I have a directory called `scripts` that contains some bash scripts. The `scripts` directory belongs to the `anurag` group.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1691992736114/ffb892c4-1252-49df-a697-2db085b9e001.png align="center")

As you can see below, I changed the group ownership of all files and subdirectories in the `scripts` directory from `anurag` to `devs` group.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1691992806609/95c3bada-b659-4ac8-8426-05d33e96eeda.png align="center")

**That's all for this part!**

I hope you found this article informative and helpful. If you have any feedback, please feel free to share it in the comments below. Thank you for reading!

**Stay tuned for the next part of the** [**Master Linux series**](https://anurag-rajawat.hashnode.dev/series/linux-the-practical-way)**!**