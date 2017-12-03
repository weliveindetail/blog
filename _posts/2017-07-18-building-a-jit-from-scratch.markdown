---
layout: post
author: Stefan Gränitz
title:  "Building a JIT from scratch"
date:   2017-07-18 12:13:01 +0200
categories: post
comments: true
---

Implementing a cross-platform native JIT has never been easier than today with LLVM. My GitHub project [JitFromScratch](https://github.com/weliveindetail/JitFromScratch) shows how to use the LLVM ORC libraries to compile the code for a simple function at runtime:

```cpp
template <size_t sizeOfArray>
int *integerDistances(const int (&x)[sizeOfArray], int *y) {
  int items = arrayElements(x);
  int *results = customIntAllocator(items);

  for (int i = 0; i < items; i++) {
    results[i] = abs(x[i] - y[i]);
  }

  return results;
}
```

You can find out how to do this by following the [history of the jit-basics branch](https://github.com/weliveindetail/JitFromScratch/commits/jit-basics). This article guides you through the steps to build the project on different platforms. Please note that the initial build process takes at least 1 hour and requires at least 15GB disk space.



### Linux Mint 18

{::options parse_block_html="true" /}
<div id="linux-mint-18-content" style="display: none;">
Mint comes with Clang 3.8 (full C++14) and the GNU ld system linker. Additionally we need the following packages:
```terminal
$ sudo apt-get install git
$ sudo apt-get install cmake
$ sudo apt-get install ninja-build
```

Checkout the LLVM source code and build the `release_40` branch in debug mode:
```terminal
$ cd /path/to/llvm/home
$ git clone https://github.com/llvm-mirror/llvm
$ cd llvm
$ git checkout -b release_40
$ cd ..
$ mkdir llvm40-debug
$ cd llvm40-debug
$ export CC=clang-3.8
$ export CXX=clang++-3.8
$ cmake -G Ninja -DCMAKE_BUILD_TYPE=Debug -DBUILD_SHARED_LIBS=ON -DLLVM_TARGETS_TO_BUILD=host -DLLVM_INCLUDE_EXAMPLES=OFF -DLLVM_INCLUDE_TESTS=OFF -DLLVM_INCLUDE_DOCS=OFF ../llvm
$ cmake --build .
```

Please find more details on building LLVM [in the previous post]({{ site.baseurl }}{% post_url 2017-07-17-notes-setup %}). The build process will take time, but you can already prepare the next steps. Checkout the sources for the JitFromScratch project and get ready for generating build files:
```terminal
$ mkdir JitFromScratch
$ cd JitFromScratch
$ git clone https://github.com/weliveindetail/JitFromScratch
$ cd JitFromScratch
$ git checkout -b jit-basics
$ cd ..
$ mkdir JitFromScratch-debug
```

Once the LLVM build finished, you can configure cmake and build:
```terminal
$ cd JitFromScratch-build
$ cmake -G Ninja -DCMAKE_BUILD_TYPE=Debug -DLLVM_DIR=/path/to/llvm/home/llvm40-debug/lib/cmake/llvm ../JitFromScratch
$ cmake --build .
$ ./JitFromScratch
Integer Distances: 3, 0, 3
```

[QtCreator](https://download.qt.io/official_releases/qtcreator/) is great for codebase exploration. [Since version 4.3](https://blog.qt.io/blog/2016/11/15/cmake-support-in-qt-creator-and-elsewhere/) it works with CMake natively, so you can just click *Open Project* and select `JitFromScratch/CMakeLists.txt` and it should import the Ninja build.

<br>
<br>
</div>



### Mac OS X 10.12

{::options parse_block_html="true" /}
<div id="mac-os-x-1012-content" style="display: none;">
On Mac OS X 10.12 install Xcode and the command line tools (I currently use version 8.3). You can download both from [https://developer.apple.com/download/more/](https://developer.apple.com/download/more/) after logging in with an Apple ID.

Additionally you need:
```terminal
$ brew install git
$ brew install cmake
```

Checkout the LLVM source code and generate a Xcode project to build the `release_40` branch like this:
```terminal
$ cd /path/to/llvm/home
$ git clone https://github.com/llvm-mirror/llvm
$ cd llvm
$ git checkout -b release_40
$ cd ..
$ mkdir llvm40-build
$ cd llvm40-build
$ cmake -G Xcode -DCMAKE_OSX_SYSROOT=macosx10.12 -DBUILD_SHARED_LIBS=ON -DLLVM_TARGETS_TO_BUILD=host -DLLVM_INCLUDE_EXAMPLES=OFF -DLLVM_INCLUDE_TESTS=OFF -DLLVM_INCLUDE_DOCS=OFF ../llvm
$ cmake --build .
```

Please find more details on building LLVM [in the previous post]({{ site.baseurl }}{% post_url 2017-07-17-notes-setup %}). The build process will take time, but you can already prepare the next steps. Checkout the sources for the JitFromScratch project and get ready for generating build files:
```terminal
$ git clone https://github.com/weliveindetail/JitFromScratch JitFromScratch
$ cd JitFromScratch
$ git checkout -b jit-basics
$ cd ..
$ mkdir JitFromScratch-build
```

Once the LLVM build finished, you can configure cmake and build:
```terminal
$ cd JitFromScratch-build
$ cmake -G Xcode -DCMAKE_OSX_SYSROOT=macosx10.12 -DLLVM_DIR=/path/to/llvm/home/llvm40-build/lib/cmake/llvm ../JitFromScratch
$ cmake --build .
$ cd Debug
$ ./JitFromScratch
Integer Distances: 3, 0, 3
```

To explore the code and debug JitFromScratch, open the generated `~/Develop/JitFromScratch/JitFromScratch-build/JitFromScratch.xcodeproj` with Xcode.

<br>
<br>
</div>



### Windows 10

{::options parse_block_html="true" /}
<div id="windows-10-content" style="display: none;">
On Windows download and install the following tools:
* Visual Studio 2017 (2015 should work too): Get the free community edition downloader from [https://www.visualstudio.com/Downloads](https://www.visualstudio.com/Downloads) and install the *Desktop development with C++* package
* Git: Any Git Client should be fine, e.g. the GPLv2 licensed [Git-for-Windows](https://github.com/git-for-windows/git/releases) or the commercial/community dual-licensed [SmartGit](http://www.syntevo.com/smartgit/download)
* CMake: Get the latest stable Windows installer from [https://cmake.org/download/#latest](https://cmake.org/download/#latest)

Now open a **new** command prompt that includes the Visual Studio command line utilities: click Start, type "x64 native tools command", open the *x64 Native Tools Command Prompt* and run these commands:
```terminal
$ cd C:\path\to\llvm\home
$ mkdir llvm40
$ cd llvm40
$ git clone https://github.com/llvm-mirror/llvm
$ cd llvm
$ git checkout -b release_40
$ cd ..
$ mkdir llvm40-build
$ cd llvm40-build
$ cmake -G "Visual Studio 15 2017 Win64" -DBUILD_SHARED_LIBS=ON -DLLVM_TARGETS_TO_BUILD=host -DLLVM_ENABLE_WARNINGS=OFF -DLLVM_INCLUDE_EXAMPLES=OFF -DLLVM_INCLUDE_TESTS=OFF -DLLVM_INCLUDE_DOCS=OFF ../llvm
$ cmake --build .
```

Please find more details on building LLVM [in the previous post]({{ site.baseurl }}{% post_url 2017-07-17-notes-setup %}). The build process will take time, but you can already prepare the next steps. Checkout the sources for the JitFromScratch project and get ready for generating build files:
```terminal
$ git clone https://github.com/weliveindetail/JitFromScratch
$ cd JitFromScratch
$ git checkout -b jit-basics
$ cd ..
$ mkdir JitFromScratch-build
```

Once the LLVM build finished, you can configure cmake and build:
```terminal
> cd JitFromScratch-build
> cmake -G "Visual Studio 15 2017 Win64" -DLLVM_DIR="C:/path/to/llvm/home/llvm40-build/lib/cmake/llvm" ..\JitFromScratch
$ cmake --build .
$ cd Debug
$ JitFromScratch
Integer Distances: 3, 0, 3
```

To explore the code and debug JitFromScratch open the generated `C:\Develop\JitFromScratch\JitFromScratch-build\JitFromScratch.sln` with Visual Studio.

<br>
<br>
</div>



<script language="JavaScript">
$("#linux-mint-18").click(function() { $("#linux-mint-18-content").toggle(); });
$("#mac-os-x-1012").click(function() { $("#mac-os-x-1012-content").toggle(); });
$("#windows-10"   ).click(function() { $("#windows-10-content"   ).toggle(); });

$(function() {
  return $("h3").each(function(i, el) {
    var $el = $(el);
    if ($el.attr('id')) {
      return $el.prepend('<i class="fa fa-chevron-down" style="font-size: 0.8em; color: #333; padding: 5px;"></i>');
    }
  });
});
</script>
