---
layout: post
title:  "Notes on setting up projects with LLVM"
date:   2017-07-17 22:13:01 +0200
categories: post
--- 
A few notes on building LLVM and how to build against it. There's different ways to do this and there's pros and cons with all of them. This is what I got used to.

### Build LLVM from source

Well, since the LLVM prebuilt binaries do not include headers anymore, you have to build LLVM from source yourself. Please find the [full documentation here](http://llvm.org/docs/CMake.html).

If you only use LLVM and stay with release versions until the next comes out, you should probably build and install LLVM as shown in the documentation. If you, however, want to hack forward in the sources or build against different versions, here's a few tweaks that make life easier.

### Use version-specific build directories

The repository is large and building takes time. Switching between versions is likely to trigger long rebuilds. I usually have the sources checked out for each major version and a build directories for different configurations. To build your project against one of these, use LLVM_DIR as shown later.

<pre style="line-height: 1.125em;">
~/Develop
 ├── llvm40
 │   ├── clang
 │   ├── lldb
 │   ├── llvm
 │   ├── llvm40-debug-clang-lldb
 │   └── llvm40-release-clang-lldb
 └── llvm50
     ├── llvm
     └── llvm50-xcode
</pre>

### Use ENABLE_PROJECTS

Projects like LLDB and Clang are based on LLVM. By default they live 'in-tree' in the tools subdirectory and get built with LLVM automatically. However, this introduces problems e.g. with git as you need submodules now for these projects. This adds unnecessary complexity. You can also check them out side-by-side and tell CMake which projects to include using the `ENABLE_PROJECTS` option like this:

<pre>
~/Develop/llvm40 $ git clone https://github.com/llvm-mirror/llvm llvm
~/Develop/llvm40 $ cd llvm
~/Develop/llvm40/llvm $ git checkout -b release_40
~/Develop/llvm40/llvm $ cd ..
~/Develop/llvm40 $ git clone https://github.com/llvm-mirror/clang clang
~/Develop/llvm40 $ cd clang
~/Develop/llvm40/clang $ git checkout -b release_40
~/Develop/llvm40/clang $ cd ..
~/Develop/llvm40 $ mkdir llvm40-debug-clang
~/Develop/llvm40 $ cd llvm40-debug-clang
~/Develop/llvm40/llvm40-debug-clang $ cmake -DCMAKE_BUILD_TYPE=Debug <b>-DENABLE_PROJECTS=clang</b> ../llvm
</pre>

### Safe time with the right generator and additional options

To save time during a build you can easily disable building examples, tests and documentation. If you don't do cross-compilation you should also avoid building LLVM for all target platforms with the `LLVM_TARGETS_TO_BUILD` flag.

<pre>
~/Develop/llvm/llvm40-debug $ cmake <b>-G Ninja</b> -DCMAKE_BUILD_TYPE=Debug <b>-DLLVM_TARGETS_TO_BUILD=X86 -DLLVM_INCLUDE_EXAMPLES=OFF -DLLVM_INCLUDE_TESTS=OFF -DLLVM_INCLUDE_DOCS=OFF</b> ../llvm40
</pre>

On Mac I typically use the Xcode generator as it's also useful to explore the code base. Same on Windows with "Visual Studio 15 2017 Win64". On Linux I like to use QtCreator for code exploration, which can deal with CMake directly and figures my Ninja build.

### Use LLVM_DIR to specify the right build

`find_package(LLVM)` is obviously very useful, but in order to find the right **not installed** LLVM it needs the LLVM_DIR variable pointing to it's `LLVMConfig.cmake`. Ideally pass it as a command line option. For the JitFromScratch project it looks like this:

<pre>
~/Develop/JitFromScratch/JitFromScratch-debug $ cmake -DCMAKE_BUILD_TYPE=Debug <b>-DLLVM_DIR=~/Develop/llvm40/llvm40-debug-clang/lib/cmake/llvm</b> ../JitFromScratch
</pre>

### Define the expected LLVM version

Interfaces change a lot and also dealing with multiple versions can get confusing. Safe your users and yourself by defining which version is expected:

<pre>
find_package(LLVM 4.0 REQUIRED)
</pre>
