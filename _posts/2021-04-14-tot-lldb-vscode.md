---
layout: post
categories: post
author: Stefan Gränitz
date: 2021-04-14 15:57:01 +0200
image: https://github.com/vadimcn/vscode-lldb/raw/master/images/lldb.png
title:  "Get tip-of-tree LLDB to work in vscode"
description: "Build and integrate mainline LLDB in Visual Studio Code"
source: https://github.com/weliveindetail/blog/blob/main/_posts/2021-04-14-tot-lldb-vscode.md
comments: https://www.reddit.com/r/debugging/comments/mqrpvm/get_tipoftree_lldb_to_work_in_vscode/
---

<style>
code.language-platform-debian,
code.language-platform-macos,
code.language-vscode-settings-json,
code.language-vscode-launch-json {
  padding-top: 20px;
}

code.language-platform-debian::before {
  content: 'Debian';
}

code.language-platform-macos::before {
  content: 'macOS';
}

code.language-vscode-settings-json::before {
  content: '.vscode/settings.json';
}

code.language-vscode-launch-json::before {
  content: '.vscode/launch.json';
}

dd {
  padding-left: 30px;
  margin-bottom: 10px;
}
</style>

There are good reasons for building and integrating tools on our own where possible. Here is how I did it for LLDB and Visual Studio Code to profit from [my own bugfix](https://llvm.org/PR36209#c6){:target="_blank"}, before the next release got rolled out eventually.

<p style="text-align: right;">
  TL;DR &darr; <a href="#build-tip-of-tree-lldb">Build and integrate LLDB in vscode</a>
</p>

### Motivation

The LLVM project has a tight release cycle, publishing a new version every 6 months. Does this mean we will have our bugfixes rolled out in production quickly? Not quite. After the 6 months development cycle, the typical release and stabilization period takes up to 3 months and then it's on the integrators to bundle new versions in their tools. For the example of the [vscode-lldb](https://marketplace.visualstudio.com/items?itemName=vadimcn.vscode-lldb){:target="_blank"} plugin, a bug fixed in late January 2020 was available in production only in late October 2020:

2020, Jan 15:
: [11.0 release cycle starts with `release/10.x` branching from mainline](https://github.com/llvm/llvm-project/commit/0b5157db53a3bd1988d27820491bbf02cd1a1278){:target="_blank"}

2020, Jul 15:
: [`release/11.x` branches from mainline](https://github.com/llvm/llvm-project/commit/0e377e253c16d82a60e73ae21ca6b902e7a78775){:target="_blank"}

2020, Oct 12:
: [11.0 release officially available](https://lists.llvm.org/pipermail/llvm-announce/2020-October/000089.html){:target="_blank"}

2020, Oct 13:
: [vscode-lldb v1.6.0 bundles the new version](https://github.com/vadimcn/vscode-lldb/releases/tag/v1.6.0){:target="_blank"}

With only one day from release to production roll-out, Vadim Chugunov is an exemplary integrator certainly -- and a lucky one with very few dependencies. Alpine Linux [upgraded only on March 12, 2021](https://git.alpinelinux.org/aports/commit/?id=86c1654aee6f13ad63136b7e6aa84990dcca8477){:target="_blank"} (because they were waiting for 11.1). That's all is just the tip of the iceberg though. Toolchain roll-out is slow by definition and new releases are often not backported to existing distributions.

There is quite some [official documentation on how to build LLDB](https://lldb.llvm.org/resources/build.html){:target="_blank"}, but it suffers from bitrotting at times like all documentation does. Do we really need to [install `subversion`](https://github.com/llvm/llvm-project/blob/release/12.x/lldb/docs/resources/build.rst){:target="_blank"} for building Release 12? And which optional dependencies do we need for the Visual Studio Code integration?

### Build tip-of-tree LLDB

vscode-lldb uses [LLDB's Python interface](https://lldb.llvm.org/python_api.html){:target="_blank"}. Enabling it in LLDB requires [swig](http://www.swig.org/). As usual, we build with [CMake](https://llvm.org/docs/CMake.html){:target="_blank"} and [ninja](https://ninja-build.org/){:target="_blank"}:

```platform-debian
> sudo apt-get install build-essential swig python3-dev git cmake ninja-build
```

```platform-macos
> brew install swig git cmake ninja
```

Yes, the build will take a while:

```terminal
> cd /workspace
> git clone --depth=1 https://github.com/llvm/llvm-project
> mkdir llvm-build && cd llvm-build
> cmake -GNinja -DCMAKE_BUILD_TYPE=Release -DLLVM_TARGETS_TO_BUILD=host -DLLVM_ENABLE_PROJECTS="clang;lldb" -DLLDB_ENABLE_PYTHON=On ../llvm-project/llvm
...
-- clang project is enabled
-- lldb project is enabled
-- Enable Python scripting support in LLDB: TRUE
...
> ninja lldb
```

### Integrate tip-of-tree LLDB in Visual Studio Code

First of all, we need the [vscode-lldb](https://marketplace.visualstudio.com/items?itemName=vadimcn.vscode-lldb){:target="_blank"} plugin. Then, we open a new workspace or folder and select `Use Alternate Backend...` from the [Command Palette](https://code.visualstudio.com/docs/getstarted/userinterface#_command-palette){:target="_blank"}. We pass in the path to the just-built lldb executable (`/workspace/llvm-build/bin/lldb` in this example) and it will figure out the path to the `liblldb` dynamic library automatically. There should be a confirmation dialog popping up. And we get a new file entry in the [workspace configuration](https://github.com/vadimcn/vscode-lldb/blob/master/MANUAL.md#advanced){:target="_blank"}:

```vscode-settings-json
{
  "lldb.library": "/workspace/llvm-build/lib/liblldb.13.0.0git.dylib"
}
```

### Voilà!

Form now on the workspace will use the just-built LLDB for debugging. We can double-check with this [custom launch configuration](https://github.com/vadimcn/vscode-lldb/blob/master/MANUAL.md#custom-launch){:target="_blank"}:

```vscode-launch-json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "lldb",
      "request": "custom",
      "name": "Dump Version",
      "initCommands": [ "version", "quit" ],
    },
  ]
}
```

Running it should dump something like this in the [debug console](https://code.visualstudio.com/docs/editor/debugging#_debug-console-repl){:target="_blank"}:
```Output
Executing script: initCommands
lldb version 13.0.0 (https://github.com/llvm/llvm-project.git revision 792ee5be36926bca22291cc93595cf65d0cd6985)
  clang revision 792ee5be36926bca22291cc93595cf65d0cd6985
  llvm revision 792ee5be36926bca22291cc93595cf65d0cd6985
```
