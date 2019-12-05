---
layout: post
author: Stefan Gränitz
title:  "What's the matter with yaml-bench in the llvm-9-dev apt package?"
date:   2019-12-02 16:07:01 +0200
categories: post
comments: https://www.reddit.com/user/weliveindetail/comments/e5fwo3/whats_the_matter_with_yamlbench_in_the_llvm9dev/
---

The LLVM project publishes prebuilt [apt packages](http://apt.llvm.org/). Client projects can easily install a release build of version X via `apt-get install llvm-X-dev` and call `find_package(LLVM)` from CMake. The mechanism worked well so far, but it broke with LLVM 9, [released in Sepember 2019](http://lists.llvm.org/pipermail/llvm-dev/2019-September/135304.html), where the CMake configuration [fails with an error message like this](https://travis-ci.org/weliveindetail/apt-llvm-9-dev-repro/builds/619590445#L204):

```
CMake Error at /usr/lib/llvm-9/lib/cmake/llvm/LLVMExports.cmake:1323 (message):

  The imported target "yaml-bench" references the file

     "/usr/lib/llvm-9/bin/yaml-bench"

  but this file does not exist.  Possible reasons include:

  * The file was deleted, renamed, or moved to another location.

  * An install or uninstall procedure did not complete successfully.

  * The installation package was faulty and contained

     "/usr/lib/llvm-9/lib/cmake/llvm/LLVMExports.cmake"

  but not all the files it references.
```

The issue here is that `LLVMExports.cmake` references the `yaml-bench` exectuable, which is not [part of the package](https://packages.debian.org/sid/llvm-9-dev) or [any of its dependencies](https://salsa.debian.org/pkg-llvm-team/llvm-toolchain/blob/9/debian/control#L320).

### Workaround

It's very very unlikely you actually need `yaml-bench`. If you have the necessary privileges, you may simply `touch /usr/lib/llvm-9/bin/yaml-bench` to put in an empty dummy file. It's the [workaround I choose in my repro](https://github.com/weliveindetail/apt-llvm-9-dev-repro/commit/86497a3b).

Otherwise you can install `libclang-common-9-dev`, which happens to [provide `yaml-bench`](https://packages.debian.org/sid/amd64/libclang-common-9-dev/filelist). Though, please note that it comes with [46MB of unrelated content](https://packages.debian.org/sid/libclang-common-9-dev#pdownload).

### Background

In LLVM jargon `yaml-bench` is a utility, which means it's not supposed to be used directly by library clients, but instead it's required by LLVM itself, i.e. for benchmarking. In the apt packages we have currently, benchmarking isn't fitting well:

* [llvm-9](https://packages.debian.org/sid/llvm-9) provides toolchain executables and libraries. This is what users want to install.
* [llvm-9-dev](https://packages.debian.org/sid/llvm-9-dev) provides libraries and headers. Library clients would use this package to develop applications that use LLVM.
* [llvm-9-tools](https://packages.debian.org/sid/llvm-9-tools) provides tools for testing LLVM. If you want to run the LLVM tests yourself or you want to use its test infrastructure for your on project, you need this in addition to llvm-9-dev.

A simple solution for package managers would be to extend the purpose of the tools package to benchmarking and add `yaml-bench` there. But why does llvm-9-dev [depend on llvm-9-tools](https://salsa.debian.org/pkg-llvm-team/llvm-toolchain/blob/9/debian/control#L320) in the first place?

In order to allow client projects to reference the package content from their build scripts, [CMake offers a target export mechanism](https://cmake.org/cmake/help/latest/command/export.html). From the build-system's point of view, the package content is what is commonly called install-tree (as compared to source-tree and build-tree). It is populated from the stripped build output after the build has finished and tests succeeded. It appears reasonable that everything that exists in the install-tree should also be exported and thus be accessible to client projects. Up to version 8.0 LLVM's utility targets have not been exported, neither from the build-tree nor from the install-tree. During the 9.0 cycle, however, this became interesting for other sub-projects, i.e. LLDB. It resulted in [a change](https://reviews.llvm.org/D56606), which makes utility targets accessible from:

* the build-tree only, if the [`LLVM_BUILD_UTILS`](https://github.com/llvm/llvm-project/blob/e99a087fff6c/llvm/CMakeLists.txt#L508) flag is set (default `ON`)
* both, build-tree and install-tree, if the [`LLVM_INSTALL_UTILS`](https://github.com/llvm/llvm-project/blob/e99a087fff6c/llvm/CMakeLists.txt#L177) flag is set too (default `OFF`)

Apparently, `LLVM_INSTALL_UTILS` is set explicitly in the build that ends up in the llvm-9-dev apt package. Thus, the package does export `yaml-bench` and other utility targets, even though it doesn't povide them. This is a problem, because CMake checks that the referenced files do actually exist, when it runs the `find_package(LLVM)` command.

As I was involved with the change that made this issue visible, [I brought it up during the LLVM 9 release phase](https://llvm.org/PR43035) and explained the solution. Adding the dependency to llvm-9-tools was the workaround that was taken instead. Unfortunately, it didn't respect the special case for `yaml-bench`. I hope we can fix this for the 10.0 release in Q2 2020. It's up for [discussion on the mailing list](http://lists.llvm.org/pipermail/llvm-dev/2019-December/thread.html#137337) just now.
