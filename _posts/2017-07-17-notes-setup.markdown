---
layout:     post
author:     Stefan Gränitz
title:      "Notes on setting up projects with LLVM"
date:       2017-07-17 22:13:01 +0200
updated:    2017-08-03 23:05:00 +0200
categories: post
comments:   true
---

A few notes on building LLVM and how to build against it. There's different ways to do this and there's pros and cons with all of them. This is what I got used to.

### Build LLVM from source

Well, since the LLVM prebuilt binaries do not include headers anymore, you have to build LLVM from source yourself. Please find the [full documentation here](http://llvm.org/docs/CMake.html).

If you only use LLVM and stay with release versions until the next comes out, you should probably build and install LLVM as shown in the documentation. If you, however, want to hack forward in the sources or build against different versions, here's a few tweaks that make life easier.

### Use version-specific build directories

The repository is large and rebuilding takes time. Switching between versions in the sources is likely to trigger long rebuilds. I usually have the sources checked out for each major version and build directories for each configuration or purpose. Use [LLVM_DIR as shown below](#use-llvm_dir-to-specify-the-right-build) to select against which one to build your project.

<pre style="line-height: 1.125em;">
~/Develop
 ├── llvm40
 │   ├── clang
 │   ├── lldb
 │   ├── llvm
 │   ├── build-debug-clang-lldb
 │   ├── build-release-clang-lldb
 │   └── build-xcode-jitfromscratch
 └── llvm50
     ├── clang
     ├── compiler-rt
     ├── llvm
     └── build-ninja-debug-shared-asan
</pre>

### Use ENABLE_PROJECTS

Projects like LLDB and Clang are based on LLVM. By default they live 'in-tree' in the tools subdirectory and get built with LLVM automatically. However, this adds overhead e.g. with git versioning as you need submodules now for these projects. You can also check them out side-by-side and tell CMake which projects to include using the `LLVM_ENABLE_PROJECTS` option like this:

<div class="language-terminal highlighter-rouge">
<pre class="highlight">
<code><span class="w">$ </span><span class="nc">git</span><span class="kv"> clone https://github.com/llvm-mirror/llvm llvm
</span><span class="w">$ </span><span class="nc">cd</span><span class="kv"> llvm
</span><span class="w">$ </span><span class="nc">git</span><span class="kv"> checkout -b release_40
</span><span class="w">$ </span><span class="nc">cd</span><span class="kv"> ..
</span><span class="w">$ </span><span class="nc">git</span><span class="kv"> clone https://github.com/llvm-mirror/clang clang
</span><span class="w">$ </span><span class="nc">cd</span><span class="kv"> clang
</span><span class="w">$ </span><span class="nc">git</span><span class="kv"> checkout -b release_40
</span><span class="w">$ </span><span class="nc">cd</span><span class="kv"> ..
</span><span class="w">$ </span><span class="nc">mkdir</span><span class="kv"> build-debug-clang
</span><span class="w">$ </span><span class="nc">cd</span><span class="kv"> build-debug-clang
</span><span class="w">$ </span><span class="nc">cmake</span><span class="kv"> -DCMAKE_BUILD_TYPE=Debug <b>-DLLVM_ENABLE_PROJECTS=clang</b> ../llvm
</span></code></pre>
</div>

### Safe time with the right generator and additional options

You can easily exclude examples, tests and documentation. If you don't do cross-compilation you can use `LLVM_TARGETS_TO_BUILD` to prevent LLVM from building all platform backends. In a debug configuration you can use `LLVM_OPTIMIZED_TABLEGEN` to build the TableGen executable in release mode, resulting in a significant speedup for each subsequent source code change that triggers a rerun of TableGen. With the `BUILD_SHARED_LIBS` option each component is built as a shared library instead of a static library, which is the default (`OFF`). Very useful to reduce link times. Only use it in debug configurations.

Ninja is the recommended build system for LLVM, ideally in combination with ccache. You might end up with a command line like this:

```terminal
$ cmake -G Ninja -DCMAKE_CXX_COMPILER_LAUNCHER=ccache -DCMAKE_BUILD_TYPE=Debug -DBUILD_SHARED_LIBS=ON -DLLVM_OPTIMIZED_TABLEGEN=ON -DLLVM_TARGETS_TO_BUILD=host -DLLVM_INCLUDE_EXAMPLES=OFF -DLLVM_INCLUDE_TESTS=OFF -DLLVM_INCLUDE_DOCS=OFF ../llvm
```

For codebase exploration I need and IDE. On Linux I use QtCreator which can deal with CMake directly [since version 4.3](https://blog.qt.io/blog/2016/11/15/cmake-support-in-qt-creator-and-elsewhere/) and it finds my Ninja-ccache builds. I use `Xcode` generator on OSX and `Visual Studio 15 2017 Win64` on Windows.

### Use LLVM_DIR to specify the right build

`find_package(LLVM)` is obviously very useful, but in order to find the right **not installed** LLVM it needs the `LLVM_DIR` variable pointing to it's `LLVMConfig.cmake`. Ideally pass it in via the command line as recommended for the [JitFromScratch](https://github.com/weliveindetail/JitFromScratch) project:

```terminal
$ cmake -DCMAKE_BUILD_TYPE=Debug <b>-DLLVM_DIR=~/Develop/llvm40/build-debug-clang/lib/cmake/llvm</b> ../JitFromScratch
```

### Define the expected LLVM version

Interfaces change a lot and also dealing with multiple versions it can get confusing. Safe yourself and your users by defining which exact version you expect:

```cmake
find_package(LLVM 4.0 REQUIRED)
```

### Takeover LLVM_DEFINITIONS

It is important to build your own project with matching preprocessor definitions to prevent mysterious compiler errors when switching to another platform or build configuration. The LLVM package sets various useful variables like `LLVM_ENABLE_RTTI`, `LLVM_BUILD_MAIN_SRC_DIR`, `LLVM_BUILD_BINARY_DIR`, `LLVM_PACKAGE_VERSION` and also <b>a space-separated list of `LLVM_DEFINITIONS`</b>. In the old days of CMake they were applied like this:

```cmake
add_definitions(${LLVM_DEFINITIONS})
```

As CMake expansion rules changed, this doesn't work anymore with "modern" target-centric CMake. The <b>`target_compile_definitions` command expects a semicolon-separated list</b> or otherwise surrounds the expanded value with quotation marks. It can be an extermely annoying detail. Fortunately after a little research, I found this special CMake helper command, which transforms a space-separated list into a semicolon-separated one:

```cmake
separate_arguments(LLVM_DEFINITIONS)
target_compile_definitions(MyTarget PRIVATE ${LLVM_DEFINITIONS}) 
```
