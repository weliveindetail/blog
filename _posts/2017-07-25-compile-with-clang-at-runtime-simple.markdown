---
layout: post
author: Stefan Gränitz
title:  "The simplest way to compile C++ with Clang at runtime"
date:   2017-07-25 15:08:01 +0200
categories: post
comments: true
---

JIT compilation is often about custom code generation or simple DSLs. But what if we have actual source code for a general programming language and want to compile it at runtime? We need a frontend that compiles it to LLVM IR first before we feed it into the JIT. Clang is a proper frontend for C++, but what's the simplest way to use it in this context?

It's a direct consequence from the [last post]({{ site.baseurl }}{% post_url 2017-07-19-debugging-clang %}): The Clang driver forks itself for each translation unit with a pre-processed set of arguments that invoke the `cc1` frontend tool, which does the actual compile job. We can do the same from our own project:

1. Save C++ source code to temporary file on disk
2. Invoke cc1 to compile source file to bitcode file
3. Stream back the bitcode file into a LLVM module
4. Feed the module into the JIT

### Add link libraries, include paths and definitions

This is obviously necessary in order to integrate Clang into our project. Please find a sample [CMakeLists.txt with the required additions here](https://github.com/weliveindetail/JitFromScratch/blob/jit-from-source/cpp-clang/CMakeLists.txt). If you're on Linux consider using the LLD linker to speed up your builds! The sample has an [option to use LLD](https://github.com/weliveindetail/JitFromScratch/blob/jit-from-source/cpp-clang/CMakeLists.txt#L55) that works with Clang.

### Invoke cc1 tool

The simplest way is a hack: The entry point to `cc1` is not the `main` function but `cc1_main` and it's implemented in a [separate translation unit](https://github.com/llvm-mirror/clang/blob/master/tools/driver/cc1_main.cpp). This means that we can simply include the cpp without a symbol conflict at link-time, as we do have our own `main` but no `cc1_main`.

A rough sketch could look like this:

```cpp
// Hack: cc1 lives in "tools" next to "include"
#include <../tools/driver/cc1_main.cpp>

// Simplified argument set
std::vector<const char *> argsX { "-emit-llvm", "-emit-llvm-bc", "-std=c++14", "-stdlib=libc++", "-resource-dir", "/path/to/llvm-clang-build/lib/clang/4.0.1/", "-o", "/tmp/bitcode.bc", "-x", "c++", "/tmp/source.cpp" };

if (int res = cc1_main(argsX, "", nullptr))
  exit(-1);

std::unique_ptr<llvm::Module> M = readModuleFromBitcodeFile("bitcode.bc");
jit->submitModule(std::move(M));
```

### Actual cc1 arguments

We can figure out which arguments to pass to cc1 to do the job, by invoking `clang` and prepending the command line with `-###` to run only the driver and print transformed arguments:

```terminal
$ clang -### -g -c -emit-llvm -o /tmp/bitcode.bc /tmp/source.cpp
```

Most of the arguments are defaults so you don't actually need to set them. This said, for any reasonable C++ code you will at least need to set `-resource-dir`.

### Set -resource-dir

Clang comes with a resource directory that contains vendor-specific include files like standard headers and intrinsics definitions. As the contents is also specific for each version of Clang, it needs to be shipped with the executable. In a developer environment, however, it can easily be inferred from the LLVM/Clang build while configuring CMake:

```cmake
target_compile_definitions(TargetName
  PRIVATE
    CUSTOM_CLANG_RESOURCE_DIR=
        ${LLVM_BUILD_BINARY_DIR}/lib/clang/${LLVM_PACKAGE_VERSION})
```

With a little preprocessor magic it can be inserted to the `argsX` vector like this:
```cpp
#define STRINGIFY_DETAIL(X) #X
#define STRINGIFY(X) STRINGIFY_DETAIL(X)

std::vector<const char *> argsX {
  "-emit-llvm", "-emit-llvm-bc", "-std=c++14", "-stdlib=libc++",
  "-resource-dir", STRINGIFY(CUSTOM_CLANG_RESOURCE_DIR),
  "-o", "/tmp/bitcode.bc", "-x", "c++", "/tmp/source.cpp"
};
```

That's it! Add a little infrastructure around it and go compile with Clang at runtime. Please find a complete implementation in my [JitFromScratch example project on GitHub](https://github.com/weliveindetail/JitFromScratch/tree/jit-from-source/cpp-clang).

Happy hacking!
