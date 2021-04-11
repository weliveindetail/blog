---
layout: post
categories: post
author: Stefan Gränitz
date: 2021-03-29 13:57:01 +0200
image: https://llvm.org/img/DragonMedium.png
title:  "Remote native JIT compilation and debugging with LLVM"
description: "JIT compile and run a minimal program on a remote target connected via TCP. Inspect and modify the program state from the host machine just like any static executable running locally."
source: https://github.com/weliveindetail/blog/blob/main/_posts/2021-03-29-remote-compile-and-debug.md
comments: https://www.reddit.com/r/LLVM/comments/mfoizc/remote_native_jit_compilation_and_debugging_with/
---

**Any toolchain is only as powerful as its diagnostics.** I learned that early on when I worked on DSLs in the audio industry. It's amazing to see how runtime generated code can optimize another few percent out of your DSP code, but one central question is easily overlooked: How would I debug that?

### Overview

This post gives a short introduction to LLVM's latest tools for generating, executing and debugging runtime-compiled code. In an example we will run native JITed code on a remote target. In order to make it simple to set up and reproducible for everyone, the remote target will be a Docker container based on [Alpine Linux](https://hub.docker.com/r/amd64/alpine/){:target="_blank"}. All you need is an x86-64 host system running Linux or macOS with Docker and the regular dev tools installed (C++ toolchain, git, cmake, ninja).

Over the last years LLVM grew [ORC](https://llvm.org/docs/ORCv2.html){:target="_blank"}, a library for building JIT compilers that run bitcode. ORC makes it easy to model special-purpose compilers as a stacked collection of layers with attached utilities. With [JITLink](https://llvm.org/docs/JITLink.html){:target="_blank"} LLVM 9 introduced an ORC-specific extensible JIT linker implementation. One of the features it provides is a [target-process control class](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/ExecutionEngine/Orc/TargetProcessControl.h){:target="_blank"} that allows us to run code on various targets without major changes on our JIT stack. We can run bitcode:

* directly in the host process
* in a child process connected via pipes
* in another process on the same machine connected via pipes
* on a remote machine connected via TCP

### Build the demo JIT

The ORC and JITLink libraries are in active development. It's usually worth checking out the latest state of the [LLVM development branch](https://github.com/llvm/llvm-project/tree/main){:target="_blank"}. For the purpose of this demo, however, I choose a commit that I know is sufficient and functional:
```terminal1
> cd /path/to/demo
> git clone https://github.com/llvm/llvm-project
> git -C llvm-project checkout 7b9df09e2050b8b2
```

Debug builds of LLVM are huge. If you only want to follow the demo then make a release build. Also, our target container runs on the host machine, so we only need a code generator for the host architecture. Enabling only one target backend can save us a lot of build time. Last but not least, we tell the build system to include the LLVM example projects, because the demo uses the [LLJITWithRemoteDebugging example](https://github.com/llvm/llvm-project/blob/7b9df09e2050b8b2/llvm/examples/OrcV2Examples/LLJITWithRemoteDebugging/LLJITWithRemoteDebugging.cpp){:target="_blank"}:
```terminal1
> mkdir /path/to/demo/build
> cd /path/to/demo/build
> cmake -GNinja -DCMAKE_BUILD_TYPE=Release -DLLVM_TARGETS_TO_BUILD=host -DLLVM_BUILD_EXAMPLES=On ../llvm-project/llvm
> ninja LLJITWithRemoteDebugging
```

The build takes some time. When it's done we can run the [LLVM integrated tester](https://llvm.org/docs/CommandGuide/lit.html){:target="_blank"} to make sure it's working. That's optional and it requires a few more binaries to be present, but this should be fast:
```terminal1
> cd build
> ninja FileCheck llvm-jitlink-executor llvm-config count not
> bin/llvm-lit -vv --filter=lljit-with-remote-debugging test
```

It runs [this simple C program](https://github.com/llvm/llvm-project/blob/7b9df09e2050b8b2/llvm/test/Examples/OrcV2Examples/Inputs/argc_sub1.c){:target="_blank"} in a child process and validates its output:
```c
int sub1(int x) { return x - 1; }
int main(int argc, char **argv) { return sub1(argc); }
```

### Run the executor

There is a [ready to use Docker container here](https://hub.docker.com/r/weliveindetail/llvm-jit-remote-debug){:target="_blank"} that comes with the tools we need for this demo. Docker will fetch it for us once we run this in a second terminal:
```terminal2
> docker run --rm -p 9000:9000 -it weliveindetail/llvm-jit-remote-debug
```

The executor is now listening for a TCP connection at `localhost:9000`. Optionally, we can test it by running our demo JIT against it:
```terminal1
> cd /path/to/demo/build
> cp /path/to/demo/llvm-project/llvm/test/Examples/OrcV2Examples/Inputs/argc_sub1_elf.ll .
> bin/LLJITWithRemoteDebugging --connect=localhost:9000 argc_sub1_elf.ll --args 2nd 3rd 4th
Parsed input IR code from: argc_sub1_elf.ll
Found out-of-process executor: /path/to/demo/build/bin/llvm-jitlink-executor
Established TargetProcessControl connection to the executor
Initialized LLJIT for remote executor
Running: main("2nd", "3rd", "4th")
Exit code: 3
```

### Prepare the executor for debugging

In addition to the executor itself, the Docker container runs an [lldb-server](https://lldb.llvm.org/man/lldb-server.html){:target="_blank"} that we can connect to from our host. Before we can start debugging inside the container, we have to restart it in privileged mode and with additional ports forwarded:
```terminal2
> docker run --rm --privileged --security-opt seccomp=unconfined --cap-add=SYS_PTRACE --security-opt apparmor=unconfined -p 9000-9002:9000-9002 -it weliveindetail/llvm-jit-remote-debug
+ /workspace/lldb-server platform --server --listen 0.0.0.0:9001 --gdbserver-port 9002
+ /workspace/llvm-jitlink-executor listen=localhost:9000
Listening at localhost:9000
```

Next, we run LLDB in a third terminal on our host and connect it to the remote lldb-server. Note that we need [LLDB 12](https://github.com/llvm/llvm-project/releases/tag/llvmorg-12.0.0-rc3){:target="_blank"} or higher for JITed code debugging:
```terminal3
> lldb
(lldb) platform select remote-linux
  Platform: remote-linux
 Connected: no
(lldb) platform connect connect://localhost:9001
  Platform: remote-linux
    Triple: x86_64-unknown-linux-gnu
OS Version: 4.19.121 (4.19.121-linuxkit)
  Hostname: ec2c06925bea
 Connected: yes
WorkingDir: /
    Kernel: #1 SMP Thu Jan 21 15:36:34 UTC 2021
(lldb) process attach --name /workspace/llvm-jitlink-executor
Process 65535 stopped
* thread #1, name = 'llvm-jitlink-ex', stop reason = signal SIGSTOP
    frame #0: 0x00007f3048abd352 ld-musl-x86_64.so.1
->  0x7f3048abd352: ret
    0x7f3048abd353: jmp    0x7f3048aba5e2
    0x7f3048abd358: push   r12
    0x7f3048abd35a: mov    eax, 0x2
```

LLDB stopped the executor process and we can prepare it for debugging our [minimal example C program](https://github.com/llvm/llvm-project/blob/7b9df09e2050b8b2/llvm/test/Examples/OrcV2Examples/Inputs/argc_sub1.c){:target="_blank"}. First we tell it where to find the source code on our host machine. Then we set a breakpoint on the `sub1` function and resume the process. The executor itself is still waiting for a connection at this point and it has no idea what code we are going to run. We expect the breakpoint not to resolve to an existing function and thus remain pending for now:
```terminal3
(lldb) settings set target.source-map Inputs/ /path/to/demo/llvm-project/llvm/test/Examples/OrcV2Examples/Inputs/
(lldb) b sub1
Breakpoint 1: no locations (pending).
WARNING:  Unable to resolve breakpoint to any actual locations.
(lldb) c
Process 65535 resuming
```

### Debug JITed code in the executor

Now that both executor and debugger are ready, we run the minimal example program again in the demo JIT:
```terminal1
> cd /path/to/demo/build
> cp /path/to/demo/llvm-project/llvm/test/Examples/OrcV2Examples/Inputs/argc_sub1_elf.ll .
> bin/LLJITWithRemoteDebugging --connect=localhost:9000 argc_sub1_elf.ll
Parsed input IR code from: argc_sub1_elf.ll
Found out-of-process executor: /path/to/demo/build/bin/llvm-jitlink-executor
Established TargetProcessControl connection to the executor
Initialized LLJIT for remote executor
Running: main()
```

At this point the executor should hit the breakpoint on `sub1` that we set in LLDB earlier:
```terminal3
(lldb)  JITLoaderGDB::JITDebugBreakpointHit hit JIT breakpoint
 JITLoaderGDB::ReadJITDescriptorImpl registering JIT entry at 0x106b34000
1 location added to breakpoint 1
Process 65535 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = breakpoint 1.1
    frame #0: JIT(0x106b34000)`sub1(x=1) at argc_sub1.c:1:28
-> 1   	int sub1(int x) { return x - 1; }
   2   	int main(int argc, char **argv) { return sub1(argc); }
```

We should be able to inspect and modify the program state as if it was compiled statically. We can test that by setting the value of the parameter `x` to `42` and continue execution:
```terminal3
(lldb) p x
(int) $0 = 1
(lldb) expr x = 42
(int) $1 = 42
(lldb) c
```

The demo JIT now resumes and prints:
```terminal1
Exit code: 41
```

### Voila!

We successfully ran a JIT that compiled a minimal program to native code, sent it via a TCP connection to a remote target and executed it there. From our host machine we were able to inspect and modify the program state just like any static executable running locally.

This was a first quick walkthrough from a user perspective. There is a lot more going on under the hood that future posts will shine a light on.
