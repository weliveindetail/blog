---
layout: post
categories: post
author: Stefan Gränitz
date: 2021-05-28 12:30:00 +0200
image: https://llvm.org/img/LLVMWyvernSmall.png
title:  "Installing latest stable LLVM in a bare Linux container with apt"
description: "This is a reminder to myself on installing LLVM in a fresh debian:buster-slim container"
source: https://github.com/weliveindetail/blog/blob/main/_posts/2021-05-28-debian-llvm-quick-install.md
---

### Installing latest stable LLVM in a bare Linux container with apt

I got used to develop in [debian:buster-slim](https://hub.docker.com/_/debian) containers as they have a minimal memory footprint and still allow me to get the all the tools easily via [apt](https://wiki.debian.org/Apt).

One of the most frequent tasks is installing the latest stable Clang and other LLVM tools. The [LLVM foundation provides](https://apt.llvm.org/) an apt package source and a very handy script to install them. Bare Linux containers, however, don't come with the required tools and so each time I set up a new environment, it bumps me to install them one after the other. This is a reminder to myself on what to install upfront:

```
> apt update
> apt install lsb-release wget software-properties-common gnupg2
> bash -c "$(wget -O - https://apt.llvm.org/llvm.sh)"
```

This installs clang, lldb, lld and clangd. It also adds the right apt.llvm.org repository to your apt package sources, so we can move on quickly and install more tools from the same release.
