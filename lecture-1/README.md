# Lecture 1

**Warning**: this lecture is quite sysadmin heavy. It shows you how Linux runs and how you can hook into the kernel. This is not stuff students should be able to reproduce, but they should be able to answer general level questions about it.

Such as: "Can you explain what giving a process a separate namespace in Linux means for the visibility of that process to the host system?".

## Lecture plan

| Lecture | Goals | Activities |
| ------- | ----- | ---------- |
| 1. | Learn how Linux runs processes and uses namespaces. Learn what a container actually is | Talk through processes. Show processes in the terminal. Talk through namespaces. Show namespaces in the terminal. Then show a Docker container and how it's little different |

## Let's play

### Introduction

This Docker tutorial is special in that we won't actually talk about Docker that much in the first lecture. There are two problems with learning Docker:

1. The Docker documentation is very good, but it assumes you are a knowledgeable sysadmin. You can't really use it unless you know how Linux works.
2. Every YouTube and Medium tutorial out there assumes you just want to use Docker to run the most standard JavaScript app in the world and does not actually teach you how to use Docker. The teachers in that video don't understand Docker all that more than you do, but they got it running and they want to show you how they did it. Doesn't matter that they can't explain why, just copy paste everything and it "works".

Our tutorial tries to bridge that gap by showing you just enough Linux stuff to make you understand what Docker does under the hood. Then it will teach you two Docker tools do setup your own environments: Dockerfile and Docker Compose. Hopefully you'll understand those files at that point, since you then understand what Docker is actually doing. The last lecture is about deployments. Containers are nice on your own laptop, but they can also be nice in production.

### Linux processes and namespaces

**Prerequisits**: have a Linux(y) OS. Either native, macOS or WSL is fine. Cygwin will not work. You cannot use Windows since Docker needs Linux. Installing Docker for Windows secretly installs a VM with Linux that contains Docker.

**Note to self**: this lecture is quite inspired by [LiveOverflow - How Docker Works](https://www.youtube.com/watch?v=-YnMr1lj4Z8) video.


#### Processes

Open up Task Manager / System Monitor / whatever macOS uses. What do you see?

This is what you see in Linux.

```shell
$ ps -aux
```
That's a lot right? Most of it doesn't even make sense at all. Let's look at it from a tree perspective.

```
$ pstree -p # -p (show PIDs)
```

This hopefully makes a bit more sense. You can probably spot my browser, code editor and terminal in here. Let's take a closer look.

All the way at the top is `systemd`. This is the first thing that is booted by your Linux installation. It acts as the initializer of your system. `systemd` in turns starts all kinds of other stuff: network services, authentication and a desktop environment.

(Btw, if you've ever read that Arch Linux is hard, that's because you need to create your own `systemd` config. All other Linux installations just have a good default one, such as Ubuntu that I use.)

This tree might look strange or unintuitive, but it's literally how Linux works. The BIOS starts your hardware and loads Systemd. Systemd in turn starts all the services you need to run your operating system. Once that's done, the operating system can take over and then you can use your computer as you're used to.

Every new item in this tree is a `process`. Processes can have subprocesses (i.e. my browser, editor, or Teams). We often call these `child processes` because they have a parent-child relationship. There is 1 parent for every child, and every parent can have many childs.

Now we can inspect the output of the following better:

```shell
$ ps -aux    # Process Snapshot -a (all processes) -u (list the user) -x (even if they don't have a terminal attached to them)
```

Let's take a close look at the top:

- **USER** = Which user is running the process. Note there are a lot of `root`s. That is the Linux system itself (systemd). You can compare it to Windows Administrator rights.
- **PID** = Process ID. Just a number for the process
- **START** = When did the process start?
- **TIME** = How long has the process been running?
- **COMMAND** = How was the process started?

Most interesting to us are `user`, `PID` and `command`. Remember, in Linux, everything is a process and everything process is invoked through a command. This command can be one you type in the terminal, but it can also be an executable file such as your browser.

Let's type a nice Linux command to find a certain process. Let's say I want to find my shell (the process that runs in your terminal).

```shell
$ ps -aux | grep zsh    # Find all processes and filter out the ones that do not contain the string "zsh".
```

Why does it show two processes? Right, everything is a process. The command we just ran, is a process as well. Let's run it again.

```shell
$ ps -aux | grep zsh    # Find all processes and filter out the ones that do not contain the string "zsh".
```

Notice that the grep command has a different PID, but the ZSH command does not. What's going on here? Exactly, the grep command quit as soon as it was done. But the zsh command will keep running until I close my terminal.

#### Namespaces

Let's take a look at the Linux documentation for namespaces: https://man7.org/linux/man-pages/man7/namespaces.7.html. By the way, maybe you understand now why we always say that Laravel has such good documentation.

But let's look specificially at the first paragraph of the description, the namespace types and the `unshare` function if you scroll down a bit. Let's read the first paragraph first. When you read "resource", think "my source code files" or "my running application".

> A namespace wraps a global system resource in an abstraction that makes it appear to the processes within the namespace that they have their own isolated instance of the global resource. Changes to to the global resource are visible to other processes that are members of the namespace, but are invisible to other processes. One use of namespaces is to implement containers.

Last sentence says all right. Okay, but really, who got anything of what was written? What does it mean?

> A namespace wraps a global system resource in an abstraction that makes it appear to the processes within the namespace that they have their own isolated instance of the global resource.

Files or applications in a namespace are being fooled by that namespace. These files think they have access to the global system, but in reality they're isolated of that global system.

> Changes to to the global resource are visible to other processes that are members of the namespace, but are invisible to other processes.

This is key. When I change a resource in a namespace, all the process in that namespace can see that change. But all other processes running outside the namespace, can not see that change. Maybe the name "container" is starting to make sense now.

> One use of namespaces is to implement containers.

Okay, duh. Why would the teacher show you all this sysadmin stuff if it didn't have anything to do with Docker.

Scroll down to `namespace types` and find `Network`, `PID` and `User`. What would these mean? Right, these flags show the different things you can isolate in a namespace.

Now scroll a bit further down until you see the `unshare` function.

> The unshare(2) system call moves the calling process to a new namespace. If the flags argument of the call specifies one or more of the CLONE_NEW* flags listed above, then new namespaces are created for each flag, and the calling process is made a member of those namespaces. (This system call also implements a number of features unrelated to namespaces.)

Let's walk through every sentence.

> The unshare(2) system call moves the calling process to a new namespace.

So when a process calls `unshare`, the process is moved into a new namespace.

> If the flags argument of the call specifies one or more of the CLONE_NEW* flags listed above, then new namespaces are created for each flag, and the calling process is made a member of those namespaces.

This speaks for itself I think? You can create new namespaces for every flag we just saw in Namespace Types.

> (This system call also implements a number of features unrelated to namespaces.)

Ok bye.

#### Putting it all together

Let's take a moment and put it all together.

Every command in Linux is a process. Every process has a parent and child. Except for Systemd, that only has children. Processes can see each other if they're in the same namespace. There are ways for processes to create new namespaces and isolate them from the other processes in that way.

If you've ever used Docker before, this will probably be a BIG BRAIN EXPLOSION moment like you've never had before. If you haven't used Docker before, this is probably bringing you to tears. Bear with me!

### Docker

**Prerequisits**: Docker :-)

Okay now let's actually use Docker. We will come back to processes and namespaces, just wait a bit.

Open two terminals inside a folder where you can mess around freely. When there, in one of the terminals, type the following:

```shell
$ docker run -it --rm -u 1000:1000 -v $PWD:/shared -w /shared ubuntu bash

# Windows
$ docker run -t --rm -u 1000:1000 -v ${PWD}:/shared -w /shared ubuntu bash
```

Let's break down that command first:

- `docker run`: Start a Docker container
- `-it`: Start it in interactive mode. This is necessary for this lecture, because the Bash command at the end will exit immediately when running. That will also shut down the container. Interactive mode keeps bash running until we manually exit the container.
- `--rm`: Remove the container when it's done running
- `u 1000:1000`: Run the container with User ID 1000 and Group ID 1000.
- `-v $PWD:/shared`: Synchronize (mount) the current working directory on the host system to the /shared folder in the container.
- `w /shared`: when we enter the container, open the terminal in the /shared folder instead of /.
- `ubuntu`: run the latest Ubuntu container
- `bash`: inside that container, run the bash command. Bash is the shell you use by default in Linux.

Now that we have two terminals, let's create a file in the Docker terminal.

```shell
$ touch file
```

You can see the file and its info with:

```shell
$ ls -alh
```

Also run this command in the regular terminal; the one not running Docker. Note that you can see the file in both terminals, but that something is going on with the user info. Inside the container, you are "I have no name!" and outside the container you are [your username].

Now let's go full magic. Run the following command in the container:

```shell
$ watch ps -aux # watch runs the ps -aux commands every 2 seconds
```

And this one outside the container:

```shell
$ ps -aux
```

In the container, you can see the watch command runs with a fixed PID for USER 1000. Outside the container, you need to look closely but if you can find the watch command... See that it runs with a very different PID, but it does run for USER 1000.

Think back on the namespaces and processes. Can you try to explain what is going on here?

Exactly! The container **is** a namespace in which all processes have a new PID. But since we told the container to run with a certain USER ID, the host system uses that same USER ID as well. The name is just different since Linux gets its usernames by ID from `/etc/passwd`.

Think back on our summary from earlier:

> Every command in Linux is a process. Every process has a parent and child. Except for Systemd, that only has children. Processes can see each other if they're in the same namespace. There are ways for processes to create new namespaces and isolate them from the other processes in that way.

This is literally what Docker does. It's just an easier way to create namespaces that isolate certain processes from accessing the host machine. What processes you want to isolate, is entirely up to you. That's where next lecture comes in with Dockerfiles and Docker images.

## Homework

1. Watch https://www.youtube.com/watch?v=-YnMr1lj4Z8
2. Install Docker Desktop for your system. If you run Linux natively and you know what you're doing, feel free to install Docker Engine for a lot more performant, though more unstable, experience.
