---
layout: post
categories: post
author: Stefan Gränitz
date: 2021-05-19 14:00:00 +0200
image: https://llvm.org/img/LLVMWyvernSmall.png
title:  "Remote cross-JIT a Mongoose HTTP Server"
description: "The combination of cross-compilation and JIT execution enables rapid edit-compile-test cycles while keeping resource intensive tasks on the local host. The remote host can be a low-resource device that only runs the self-contained executor on a minimal Linux."
source: https://github.com/weliveindetail/blog/blob/main/_posts/2021-05-19-remote-cross-jit-mongoose.md
comments: https://www.reddit.com/r/embeddedlinux/comments/ng4vks/remote_crossjit_a_mongoose_http_server/
---

<style>
  .center-image {
    display: flex;
    justify-content: center;
    margin-bottom: 25px;
  }
  .center-image img {
    margin-right: auto;
    margin-left: auto;
    max-width: 100%;
    box-shadow: 0 0 15px #ddd;
  }
  #command-line-args {
    margin-top: -20px;
    cursor: pointer;
  }
  #command-line-args-arrow {
    font-size: 0.8em;
    padding: 5px;
  }
  #command-line-args-detail {
    display: none;
    padding: 0px 20px 20px;
  }
  dd {
    padding-left: 30px;
    margin-bottom: 10px;
  }
  code.language-executor-docker {
    padding-top: 20px;
  }
  code.language-executor-docker::before {
    content: '/path/to/demo/executor-docker/Dockerfile';
  }
</style>

[In a recent post](https://weliveindetail.github.io/blog/post/2021/03/29/remote-compile-and-debug.html){:target="_blank"} I explained how to JIT compile and run a minimal C program on a remote target connected via TCP. We then inspected and modified the program state from the host machine with [LLDB](https://lldb.llvm.org){:target="_blank"} as if it was a static executable running locally:

<p class="center-image">
  <span style="max-width: 398px;">
    <img src="https://weliveindetail.github.io/blog/res/scheme-remote-jit-debug-c.svg"
         alt="scheme remote jit debug c code">
  </span>
</p>

Today we will cross-compile a simple HTTP server based on [Mongoose](https://github.com/cesanta/mongoose){:target="_blank"} and run it in the same way. The approach enables rapid edit-compile-test cycles while keeping resource intensive compilation and linking tasks on the local host. The remote host can be a low-resource device that only runs the self-contained executor on a minimal Linux.

### Build and run locally

[Mongoose](https://github.com/cesanta/mongoose){:target="_blank"} is an established embedded web server and networking library. Let's check out the sources and run the [http-server example](https://github.com/cesanta/mongoose/blob/master/examples/http-server/main.c){:target="_blank"} locally for illustration. In the end we won't need the local build anymore so you could as well skip the `make` step:
```terminal
> cd /path/to/demo
> git clone https://github.com/cesanta/mongoose
> git -C mongoose checkout 912dd518bf986e04
> cd mongoose/examples/http-server
> apt-get install libmbedtls-dev
> make
cc ../../mongoose.c main.c -I../.. -I../.. -W -Wall -DMG_ENABLE_IPV6=1 -DMG_ENABLE_LINES=1 -DMG_ENABLE_DIRECTORY_LISTING=1 -DMG_ENABLE_SSI=1  -o example
./example
2021-03-30 12:18:05  I sock.c:484:mg_listen      1 accepting on http://localhost:8000
2021-03-30 12:18:05  I main.c:78:main            Starting Mongoose v7.3, serving [.]
```

Navigating to [http://localhost:8000](http://localhost:8000){:target="_blank"} should bring up a directory index like this in the browser:

<p class="center-image">
  <span style="max-width: 398px;">
    <img src="https://weliveindetail.github.io/blog/res/mongoose-local.png"
         alt="local mongoose http-server browser">
  </span>
</p>

### Build a matching version of Clang

In order to run the server in [ORC](https://llvm.org/docs/ORCv2.html){:target="_blank"}, we have to compile the sources to LLVM bitcode first. Bitcode is the serialized binary form of LLVM's internal program representaiton. It is not stable and not forward compatible. This means we should exchange bitcode only between tools that use one and the same LLVM version. Hence, in a first step we build a Clang compiler that matches the version of our JIT.

Note: Not every new LLVM version introduces breaking changes. If you have a recent Clang installed, chances are that it works well and you can fast-forward to the [next section](#set-up-the-remote-executor).

Let's check out the [LLVM mono-repo](https://github.com/llvm/llvm-project){:target="_blank"} and start the build. If you followed [my recent post](https://weliveindetail.github.io/blog/post/2021/03/29/remote-compile-and-debug.html#build-the-demo-jit), you can simply reconfigure your existing LLVM build. Once again we try to avoid surprises by taking a [specific commit](https://github.com/llvm/llvm-project/commit/7b9df09e2050b8b2ab941fde7437fb2a67632cd6){:target="_blank"} that is sufficient and functional for this demo:
```terminal
> cd /path/to/demo
> git clone https://github.com/llvm/llvm-project
> git -C llvm-project checkout 7b9df09e2050b8b2
> mkdir llvm-build && cd llvm-build
> cmake -GNinja -DCMAKE_BUILD_TYPE=Release -DLLVM_TARGETS_TO_BUILD=host -DLLVM_ENABLE_PROJECTS=clang -DLLVM_BUILD_EXAMPLES=On ../llvm-project/llvm
> ninja clang
```

The CMake command should dump a `clang project is enabled` status line. As usual we build with [ninja](https://ninja-build.org/){:target="_blank"} and it will take a while. In the meantime we can take care of the other preparations.

### Set up the remote executor

The executor is the remote endpoint for our JIT and it gets connected via TCP. We keep it simple and use an Alpine Linux Docker container in this demo. Mongoose depends on [libc](https://en.wikipedia.org/wiki/C_standard_library){:target="_blank"} and the [`mbedtls`](https://github.com/ARMmbed/mbedtls){:target="_blank"} library. In principle we could provide them as a JIT modules as well, but it's a topic for a whole other post. For now they will be provided from the remote executor:

```executor-docker
FROM weliveindetail/llvm-jit-remote
RUN apk add --no-cache mbedtls && \
    ln -s $(ls -v /usr/lib/libmbedtls.so* | tail -1) /usr/lib/libmbedtls.so
```

We use the 8.83MB Alpine Linux image [llvm-jit-remote](https://hub.docker.com/r/weliveindetail/llvm-jit-remote){:target="_blank"} as a base. It's even smaller than [the one we used in the previous post](https://weliveindetail.github.io/blog/post/2021/03/29/remote-compile-and-debug.html#run-the-executor) because it comes without the debug server. We install [mbedtls](https://pkgs.alpinelinux.org/package/edge/main/x86/mbedtls){:target="_blank"} on-top and not `mbedtls-dev`, because the executor doesn't need any headers or static build artifacts. Eventually, the JIT driver on our local host will ask the remote executor process to load the dynamic library before running our code.

Installed dynamic library files have an [SO number](https://unix.stackexchange.com/questions/475/how-do-so-shared-object-numbers-work){:target="_blank"}  suffix that encodes the required package version to prevent conflicts. As I don't want to hardcode the demo for a specific package version, I added the extra `ln` command that creates a plain `libmbedtls.so` symlink to the file with the highest available SO number. This is fine for now. A solid solution for getting pre-installed library versions right is once again a topic for another post.

[`musl` libc](https://musl.libc.org/){:target="_blank"} is an inherent part of Alpine, so we don't have to install it. The executor binary links it dynamically and automatically exposes its symbols to the JITed code. We can build and run the container in a separate terminal like this:

```terminal
> docker build -t llvm-jit-remote:mongoose /path/to/demo/executor-docker
> docker run --rm -p 9000:9000 -p 8000:8000 -it llvm-jit-remote:mongoose
Listening at 0.0.0.0:9000
```

### Cross-compile bitcode for Alpine Linux

We have to pre-compile the Mongoose C sources to LLVM bitcode in order to feed it into the JIT. The pre-compilation is cross-platform, because our remote executor runs on a different operating system. Thus, Clang will need the target platform headers for our dependencies: Alpine's [`musl` libc](https://pkgs.alpinelinux.org/package/edge/main/x86/musl-dev){:target="_blank"} system headers and those from [APK](https://wiki.alpinelinux.org/wiki/Alpine_Linux_package_management){:target="_blank"}'s [`mbedtls-dev`](https://pkgs.alpinelinux.org/package/edge/main/x86/mbedtls-dev){:target="_blank"} package. Let's run a Docker container for that, where we install the dependencies and mount the necessary paths back to the host system. This is especially easy in our case since the container runs on the host architecture and we really only need the headers:
```terminal
> cd /path/to/demo
> mkdir x86_64-alpine-linux-musl
> docker run -v x86_64-alpine-linux-musl/usr:/usr -t alpine apk add --no-cache musl-dev mbedtls-dev
> find x86_64-alpine-linux-musl/usr/include | wc -l
     304
```

There is one detail we have to adjust in the [http-server example](https://github.com/cesanta/mongoose/blob/1b8624f135edfe5d3a2024156c00971f98b5e356/examples/http-server/main.c#L9){:target="_blank"} before we pre-compile it to bitcode. Because it's meant as a quick demo, it's hardcoded to be reachable only from the local network namespace. Once we run it in a Docker container and want it to be reachable from the host, it must listen for external connections as well. Thus, we have to change the network address from `localhost` to `0.0.0.0` in the source code:
```diff
--- a/path/to/demo/mongoose/examples/http-server/main.c
+++ b/path/to/demo/mongoose/examples/http-server/main.c
@@ -6,7 +6,7 @@
 static const char *s_debug_level = "2";
 static const char *s_root_dir = ".";
-static const char *s_listening_address = "http://localhost:8000";
+static const char *s_listening_address = "http://0.0.0.0:8000";
 static const char *s_enable_hexdump = "no";
 static const char *s_ssi_pattern = "#.shtml";
```

Now is the time when we actually need the Clang executable that we [started to build in the beginning](#build-a-matching-version-of-clang). Let's invoke it once for each source file:
```terminal
> cd /path/to/demo
> mkdir -p build-mongoose/x86_64-alpine-linux-musl
> llvm-build/bin/clang --target=x86_64-alpine-linux-musl --sysroot=x86_64-alpine-linux-musl -iquote mongoose -DMG_ENABLE_DIRECTORY_LISTING=1 -v -g -S -emit-llvm -o build-mongoose/x86_64-alpine-linux-musl/mongoose.ll mongoose/mongoose.c
> llvm-build/bin/clang --target=x86_64-alpine-linux-musl --sysroot=x86_64-alpine-linux-musl -iquote mongoose -DMG_ENABLE_DIRECTORY_LISTING=1 -v -g -S -emit-llvm -o build-mongoose/x86_64-alpine-linux-musl/http-server.ll mongoose/examples/http-server/main.c
```

<p id="command-line-args">
  <i id="command-line-args-arrow" class="fa fa-arrow-right"></i>
  Command-line arguments in detail
</p>

<script>
  $("#command-line-args").click(function() {
    $("#command-line-args-detail").toggle();

    var arrowElem = $("#command-line-args-arrow");
    var [add, rem] = arrowElem.hasClass("fa-arrow-right") ? [ "down", "right" ]
                                                          : [ "right", "down" ];
    arrowElem.removeClass("fa-arrow-" + rem);
    arrowElem.addClass("fa-arrow-" + add);
  });
</script>

{::options parse_block_html="true" /}
<div id="command-line-args-detail">
`--target=<triple>`
: Compile for the platform specified in the triple: `x86_64-alpine-linux-musl` for our Alpine Linux container.

`--sysroot=<path>`
: Set the root directory for system header and library search paths. Convenient, even though we only pre-compile and don't need any libraries. We pass the mount point from the previous section here and get `x86_64-alpine-linux-musl/usr/include` as the search path for system headers.

`-iquote <path>`
: Add private include path for quoted `#include` directives. In our case it's the Mongoose source directory.

`-emit-llvm`
: Emit bitcode. We also add `-S` to obtain the human-readable assembly form known as [LLVM IR](https://llvm.org/docs/LangRef.html){:target="_blank"} instead of the equivalent binary form. It's slower to parse but easier to read.

`-v`
: Run in verbose mode, i.e. dump the actual include search paths; useful to track down include errors.

`-g`
: Generate debug infomation.

`-DMG_ENABLE_DIRECTORY_LISTING=1`
: Mongoose preprocessor switch to enable code that populates file listings for a directory.
</div>

This gives us the two bitcode files `mongoose.ll` and `http-server.ll` in `build-mongoose/x86_64-alpine-linux-musl`. Have a look at the assembly and see how it relates to the original source. It's a [SSA representation](https://en.wikipedia.org/wiki/Static_single_assignment_form){:target="_blank"} of the individual compile units, that is specific for our target platform and LLVM version. Since we requested debug output, Clang didn't run any optimizations and the amount of bitcode is quite large. No linking has happened yet.

Having mounted the target system root inside our working directory caused all file entries in debug info tags to refer to the same directory. It comes handy when setting up a source-map for debugging:
```terminal
> cat build-mongoose/x86_64-alpine-linux-musl/mongoose.ll | grep DIFile
!3 = !DIFile(filename: "mongoose/mongoose.c", directory: "/path/to/demo")
!6 = !DIFile(filename: "mongoose/mongoose.h", directory: "/path/to/demo")
!46 = !DIFile(filename: "x86_64-alpine-linux-musl/usr/include/bits/alltypes.h", directory: "/path/to/demo")
!1369 = !DIFile(filename: "x86_64-alpine-linux-musl/usr/include/sys/socket.h", directory: "/path/to/demo")
!1379 = !DIFile(filename: "x86_64-alpine-linux-musl/usr/include/netinet/in.h", directory: "/path/to/demo")
!1825 = !DIFile(filename: "x86_64-alpine-linux-musl/usr/include/time.h", directory: "/path/to/demo")
!3600 = !DIFile(filename: "x86_64-alpine-linux-musl/usr/include/bits/stat.h", directory: "/path/to/demo")
!7901 = !DIFile(filename: "x86_64-alpine-linux-musl/usr/include/sys/select.h", directory: "/path/to/demo")
!8830 = !DIFile(filename: "x86_64-alpine-linux-musl/usr/include/ctype.h", directory: "/path/to/demo")
```

### Remote cross-JIT Mongoose to Alpine Linux

Now that we started our remote executor and cross-compiled the bitcode, we can build the demo JIT in a new terminal and use it to run the Mongoose server in the remote container:

```terminal
> cd /path/to/demo
> ninja -C llvm-build LLJITWithRemoteDebugging
> llvm-build/bin/LLJITWithRemoteDebugging --connect localhost:9000 --dlopen /usr/lib/libmbedtls.so build-mongoose/x86_64-alpine-linux-musl/mongoose.ll build-mongoose/x86_64-alpine-linux-musl/http-server.ll
Parsed input IR code from: build-mongoose/x86_64-alpine-linux-musl/mongoose.ll
Parsed input IR code from: build-mongoose/x86_64-alpine-linux-musl/http-server.ll
Connected to executor at localhost:9000
Established TargetProcessControl connection to the executor
Initialized LLJIT for remote executor
Running: main()
```

The terminal running the Docker container should now dump the server's stdout:
```
Connection established. Running OrcRPCTPCServer...
2021-05-19 10:01:49  I mongoose.c:3074:mg_listen 1 accepting on http://0.0.0.0:8000
2021-05-19 10:01:49  I main.c:67:main            Starting Mongoose v7.3, serving [.]
```

<div id="voila"></div>
Let's open a browser and [view the HTML page](http://localhost:8000/usr){:target="_blank"} provided from our cross-JITed Mongoose server:

<p class="center-image">
  <span style="max-width: 388px;">
    <img src="https://weliveindetail.github.io/blog/res/mongoose-remote.png"
         alt="remote mongoose http-server browser">
  </span>
</p>

Great, this looks as expected! Interestingly, if we navigate to the root folder we get an unexpected result: `Not found [/]`. Clicking on one of the linked directories shows a similar issue. [When running the server locally](#build-mongoose-locally) on my machine, I didn't observe any such misbehavior. Looks like an excellent opportunity for another post to demonstrate remote debugging in a real-world program!
