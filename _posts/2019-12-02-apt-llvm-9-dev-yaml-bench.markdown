---
layout: post
author: Stefan Gränitz
title:  "What's the matter with yaml-bench in the llvm-9-dev apt package?"
date:   2019-12-02 16:07:01 +0200
categories: post
---

The LLVM project publishes prebuilt [apt packages](http://apt.llvm.org/).
A client project can easily install a release build of version X via
`apt-get install llvm-X-dev` and call `find_package(LLVM)` from CMake.
This mechanism is broken in the prepackaged LLVM 9, which was released in
Sepember 2019. The CMake configuration [fails with an error message like this](
https://travis-ci.org/weliveindetail/apt-llvm-9-dev-repro/builds/619590445#L204):

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

The issue here is that `LLVMExports.cmake` references the `yaml-bench`
exectuable that is not part
[of the package](https://packages.debian.org/sid/llvm-9-dev) or
[any of its dependencies](https://salsa.debian.org/pkg-llvm-team/llvm-toolchain/blob/9/debian/control#L320).

### Workaround

It's very very unlikely you actually need `yaml-bench`. If you have the
necessary privileges, you may simpliy `touch /usr/lib/llvm-9/bin/yaml-bench`
to put in an empty dummy file. It's the
[workaround I choose in my repro](https://github.com/weliveindetail/apt-llvm-9-dev-repro/commit/86497a3b).

Otherwise install `libclang-common-9-dev`, which happens to
[provide `yaml-bench`](https://packages.debian.org/sid/amd64/libclang-common-9-dev/filelist)
together with [46MB of unrelated files](https://packages.debian.org/sid/libclang-common-9-dev#pdownload).
