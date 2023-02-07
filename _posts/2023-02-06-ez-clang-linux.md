---
layout: post
categories: post
author: Stefan Gränitz
date: 2023-02-06 23:00:00 +0200
image: https://weliveindetail.github.io/blog/res/ez-clang-linux-banner.png
preview: summary_large_image
title: "Remote C++ live coding on Raspberry Pi with ez-clang-linux"
description: "ez-clang 0.0.6 can target Raspberry Pi Models 2 and 3 through TCP sockets. It has never been easier to play and experiment with the remote C++ REPL."
source: https://github.com/weliveindetail/blog/blob/main/_posts/2023-02-06-ez-clang-linux.md
ycombinator: https://news.ycombinator.com/item?id=34689861
---

<style>
  #banner-image {
    max-width: min(100%, 500px);
  }
  #large-image {
    max-width: min(100%, 800px);
  }
  .center {
    display: block;
    margin: 0 auto;
  }
</style>

![ez-clang-pycfg-banner](https://weliveindetail.github.io/blog/res/ez-clang-linux-banner.png){: #banner-image}{: .center}

[ez-clang](http://echtzeit.dev/ez-clang/){:target="_blank"} is a C++ REPL for bare-metal embedded devices. Serial connections to such devices can be hairy. Hosted remote devices are usually much easier to handle and might help with experimentation and development. In [release 0.0.6](https://github.com/echtzeit-dev/ez-clang/releases/tag/v0.0.6){:target="_blank"} ez-clang gained a reference implementation for Linux-based remotes, that can now be connected via the [pycfg layer](https://weliveindetail.github.io/blog/post/2023/02/03/ez-clang-pycfg.html).

In Python it's easy to set up a socket connection to another device via TCP. If we find a 32-bit ARM device that runs Linux, then we just have to port our remote endpoint there and connect it to the TCP socket. Well, no need to search for long, of course Raspberry Pi Models 2 and 3b are both a great fit. Model 2 features a Cortex-A7 (ARMv7a instruction set) and Model 3b a Cortex-A53 with ARMv8a. While ARMv8a is in-fact a 64-bit instruction set, it does provide user-space compatibility with ARMv7a. This is what we want for ez-clang! We install 32-bit Raspbian and are ready to go.

<script id="asciicast-3Mdujgd1EI4OL8FAnHEmU1h9G" src="https://asciinema.org/a/3Mdujgd1EI4OL8FAnHEmU1h9G.js" async></script>

The screencast shows how to SSH into a Raspberry Pi Model 3b and configure it as a ez-clang remote. The relevant steps are:

1. Checkout the [ez-clang-linux](https://github.com/echtzeit-dev/ez-clang-linux){:target="_blank"} repository
2. Configure with CMake and build the `ez-clang-linux-socket` target with Ninja. You can keep the default system toolchain or use your own one. There are no special requirements here.
3. Use the `ez-clang-linux-socket.service` file produced by CMake to configure Systemd so that the executable is auto-restarted infinitely. This is great because we don't need to implement session handling on our own. Each connection will automatically get its own process and on disconnect it simply shuts down.

On our host machine we can just start ez-clang now and connect to the respective IP and port. In my local network it looks like this when running with docker:
```
> docker run -it echtzeit/ez-clang:0.0.6 --connect=raspi32@192.168.1.105:10819
Welcome to ez-clang, your friendly remote C++ REPL. Type `.q` to exit.
Connected: TCP -> raspi32 (arm-none-eabi @ cortex-a53)
raspi32>.q
>
```

On the remote we can inspect the RPC traffic from this (minimal) session with `journalctl`:
```
> journalctl --user -u ez-clang-linux-socket.service --follow
systemd[629]: Started Permanent ez-clang-linux remote host service.
ez-clang-linux-socket[26884]: Listening on port 10819
ez-clang-linux-socket[26884]: Connected
ez-clang-linux-socket[26884]: Send ->
ez-clang-linux-socket[26884]:   6a 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
ez-clang-linux-socket[26884]: Send ->
ez-clang-linux-socket[26884]:   05 00 00 00 00 00 00 00 30 2e 30 2e 36 00 30 aa 00 00 00 00 00 00 80 00 00 00 00 00 00 01 00 00 00 00 00 00 00 15 00 00 00 00 00 00 00 5f 5f 65 7a 5f 63 6c 61 6e 67 5f 72 70 63 5f 6c 6f 6f 6b 75 70 74 26 01 00 00 00 00 00
ez-clang-linux-socket[26884]: Receive <-
ez-clang-linux-socket[26884]:   47 00 00 00 00 00 00 00 03 00 00 00 00 00 00 00 01 00 00 00 00 00 00 00 74 26 01 00 00 00 00 00
ez-clang-linux-socket[26884]: Receive <-
ez-clang-linux-socket[26884]:   01 00 00 00 00 00 00 00 17 00 00 00 00 00 00 00 5f 5f 65 7a 5f 63 6c 61 6e 67 5f 72 65 70 6f 72 74 5f 76 61 6c 75 65
ez-clang-linux-socket[26884]: Send ->
ez-clang-linux-socket[26884]:   31 00 00 00 00 00 00 00 02 00 00 00 00 00 00 00 01 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
ez-clang-linux-socket[26884]: Send ->
ez-clang-linux-socket[26884]:   00 01 00 00 00 00 00 00 00 c0 2b 01 00 00 00 00 00
ez-clang-linux-socket[26884]: Receive <-
ez-clang-linux-socket[26884]:   20 00 00 00 00 00 00 00 01 00 00 00 00 00 00 00 02 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
ez-clang-linux-socket[26884]: Send ->
ez-clang-linux-socket[26884]:   21 00 00 00 00 00 00 00 01 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
ez-clang-linux-socket[26884]: Send ->
ez-clang-linux-socket[26884]:   00
ez-clang-linux-socket[26884]: Receive <-
ez-clang-linux-socket[26884]:   20 00 00 00 00 00 00 00 01 00 00 00 00 00 00 00 03 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
ez-clang-linux-socket[26884]: Exiting
systemd[629]: ez-clang-linux-socket.service: Succeeded.
systemd[629]: ez-clang-linux-socket.service: Scheduled restart job, restart counter is at 1.
systemd[629]: Stopped Permanent ez-clang-linux remote host service.
systemd[629]: Started Permanent ez-clang-linux remote host service.
```

### Voilà!

With these few steps we can start and run C++ code on Raspberry Pi on the fly! While keeping in mind, of course, that the [ez-clang project](https://github.com/echtzeit-dev/ez-clang){:target="_blank"} is still in a very experimental state ;-)
