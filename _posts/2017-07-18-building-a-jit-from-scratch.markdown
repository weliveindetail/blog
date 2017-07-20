---
layout: post
author: Stefan Gränitz
title:  "Building a JIT from scratch"
date:   2017-07-18 12:13:01 +0200
categories: post
comments: true
--- 
Implementing a cross-platform native JIT has never been easier than today with LLVM. My GitHub project [JitFromScratch](https://github.com/weliveindetail/JitFromScratch) shows how to use the LLVM ORC libraries to compile the code for a simple function at runtime: 

{% highlight c++ %}
template <size_t sizeOfArray>
int *integerDistances(const int (&x)[sizeOfArray], int *y) {
  int items = arrayElements(x);
  int *results = customIntAllocator(items);

  for (int i = 0; i < items; i++) {
    results[i] = abs(x[i] - y[i]);
  }

  return results;
}
{% endhighlight %}

You can find out how to do this by following the [history of the jit-basics branch](https://github.com/weliveindetail/JitFromScratch/commits/jit-basics). This article guides you through the steps to build the project on different platforms. Please note that the initial build process takes at least 1 hour and requires at least 15GB disk space. 



## Linux Mint 18

{::options parse_block_html="true" /}
<div id="linux-mint-18-content" style="display: none;">
Mint comes with Clang 3.8 (full C++14) and the GNU ld system linker. Additionally we need the following packages:
<pre>
~/Develop $ sudo apt-get install git
~/Develop $ sudo apt-get install cmake
~/Develop $ sudo apt-get install ninja-build
</pre>

Checkout the LLVM source code and build the `release_40` branch in debug mode like this:
<pre>
/Develop $ mkdir llvm40
/Develop $ cd llvm40
/Develop/llvm40 $ <b>git clone https://github.com/llvm-mirror/llvm llvm</b>
/Develop/llvm40 $ cd llvm
/Develop/llvm40/llvm $ <b>git checkout -b release_40</b>
/Develop/llvm40/llvm $ cd ..
/Develop/llvm40 $ mkdir llvm40-debug
/Develop/llvm40 $ cd llvm40-debug
/Develop/llvm40/llvm40-debug $ echo $CC $CXX

/Develop/llvm40/llvm40-debug $ export CC=clang-3.8
/Develop/llvm40/llvm40-debug $ export CXX=clang++-3.8
/Develop/llvm40/llvm40-debug $ echo $CC $CXX
clang-3.8 clang++-3.8
/Develop/llvm40/llvm40-debug $ <b>cmake -G Ninja -DCMAKE_BUILD_TYPE=Debug -DLLVM_TARGETS_TO_BUILD=X86 -DLLVM_INCLUDE_EXAMPLES=OFF -DLLVM_INCLUDE_TESTS=OFF -DLLVM_INCLUDE_DOCS=OFF ../llvm</b>
/Develop/llvm40/llvm40-debug $ cmake --build .
</pre>

You can find more details on building LLVM from source [in the previous post]({{ site.baseurl }}{% post_url 2017-07-17-notes-setup %}). The build process will take time, but you can already prepare the next steps. Checkout the sources for the JitFromScratch project and get ready for generating build files. Once the LLVM build is done, you can run the `cmake` commands:
<pre>
~/Develop $ mkdir JitFromScratch
~/Develop $ cd JitFromScratch
~/Develop/JitFromScratch $ <b>git clone https://github.com/weliveindetail/JitFromScratch JitFromScratch</b>
~/Develop/JitFromScratch $ cd JitFromScratch
~/Develop/JitFromScratch/JitFromScratch $ <b>git checkout -b jit-basics</b>
~/Develop/JitFromScratch/JitFromScratch $ cd ..
~/Develop/JitFromScratch $ mkdir JitFromScratch-debug
~/Develop/JitFromScratch $ cd JitFromScratch-debug
~/Develop/JitFromScratch/JitFromScratch-debug $ export CC=clang-3.8
~/Develop/JitFromScratch/JitFromScratch-debug $ export CXX=clang++-3.8
~/Develop/JitFromScratch/JitFromScratch-debug $ <b>cmake -G Ninja -DCMAKE_BUILD_TYPE=Debug -DLLVM_DIR=~/Develop/llvm40/llvm40-debug/lib/cmake/llvm ../JitFromScratch</b>
~/Develop/JitFromScratch/JitFromScratch-debug $ cmake --build .
~/Develop/JitFromScratch/JitFromScratch-debug $ ./JitFromScratch
Integer Distances: 3, 0, 3
</pre>

To explore the code and debug JitFromScratch I'd recommend the cross-platform [QtCreator IDE](https://download.qt.io/official_releases/qtcreator/). As it works with CMake natively, you can just click *Open Project* and select `~/Develop/JitFromScratch/JitFromScratch/CmakeLists.txt`. QtCreator will find your Ninja build and import its settings.

<br>
<br>
</div>



## Mac OS X 10.12

{::options parse_block_html="true" /}
<div id="mac-os-x-1012-content" style="display: none;">
On Mac OS X you need to install Xcode and the command line tools (I currently use version 8.3). You can download both from [https://developer.apple.com/download/more/](https://developer.apple.com/download/more/) after logging in with your Apple ID. Alternatively they can also be installed via the App Store. Please note that the download will take long either way.

Additionally we need the following packages:
<pre>
mac:Develop user$ brew install git
mac:Develop user$ brew install cmake
</pre>

Checkout the LLVM source code and generate a Xcode project to build the `release_40` branch like this:
<pre>
mac:Develop user$ mkdir llvm40
mac:Develop user$ cd llvm40
mac:llvm40 user$ <b>git clone https://github.com/llvm-mirror/llvm llvm</b>
mac:llvm40 user$ cd llvm
mac:llvm user$ <b>git checkout -b release_40</b>
mac:llvm user$ cd ..
mac:llvm40 user$ mkdir llvm40-build
mac:llvm40 user$ cd llvm40-build
mac:llvm40-build user$ <b>cmake -G Xcode -DCMAKE_OSX_SYSROOT=macosx10.12 -DLLVM_TARGETS_TO_BUILD=X86 -DLLVM_INCLUDE_EXAMPLES=OFF -DLLVM_INCLUDE_TESTS=OFF -DLLVM_INCLUDE_DOCS=OFF ../llvm</b>
mac:llvm40-build user$ cmake --build .
</pre>

You can find more details on building LLVM from source [in the previous post]({{ site.baseurl }}{% post_url 2017-07-17-notes-setup %}). The build process will again take time, but you can already prepare the next steps. Checkout the sources for the JitFromScratch project and get ready for generating build files. Once the LLVM build is done, you can run the `cmake` commands:
<pre>
mac:Develop user$ mkdir JitFromScratch
mac:Develop user$ cd JitFromScratch
mac:JitFromScratch user$ <b>git clone https://github.com/weliveindetail/JitFromScratch JitFromScratch</b>
mac:JitFromScratch user$ cd JitFromScratch
mac:JitFromScratch user$ <b>git checkout -b jit-basics</b>
mac:JitFromScratch user$ cd ..
mac:JitFromScratch user$ mkdir JitFromScratch-build
mac:JitFromScratch user$ cd JitFromScratch-build
mac:JitFromScratch-build user$ <b>cmake -G Xcode -DCMAKE_OSX_SYSROOT=macosx10.12 -DLLVM_DIR=~/Develop/llvm40/llvm40-build/lib/cmake/llvm ../JitFromScratch</b>
mac:JitFromScratch-build user$ cmake --build .
mac:JitFromScratch-build user$ cd Debug
mac:Debug user$ ./JitFromScratch
Integer Distances: 3, 0, 3
</pre>

To explore the code and debug JitFromScratch open the generated `~/Develop/JitFromScratch/JitFromScratch-build/JitFromScratch.xcodeproj` with Xcode.

<br>
<br>
</div>



## Windows 10

{::options parse_block_html="true" /}
<div id="windows-10-content" style="display: none;">
On Windows you will need to download and install the following tools:
* Visual Studio 2017 (2015 should work too): Get the free community edition downloader from [https://www.visualstudio.com/Downloads](https://www.visualstudio.com/Downloads) and install the *Desktop development with C++* package
* Git: Any Git Client should be fine, e.g. the GPLv2 licensed [Git-for-Windows](https://github.com/git-for-windows/git/releases) or the commercial/community dual-licensed [SmartGit](http://www.syntevo.com/smartgit/download)
* CMake: Get the latest stable Windows installer from [https://cmake.org/download/#latest](https://cmake.org/download/#latest)

Now open a **new** command prompt that includes the Visual Studio command line utilities: click Start, type "x64 native tools command", open the *x64 Native Tools Command Prompt for VS 2017* and run these commands:
<pre>
C:\Develop>mkdir llvm40
C:\Develop>cd llvm40
C:\Develop\llvm40><b>git clone https://github.com/llvm-mirror/llvm llvm</b>
C:\Develop\llvm40\llvm>cd llvm
C:\Develop\llvm40\llvm><b>git checkout -b release_40</b>
C:\Develop\llvm40\llvm>cd ..
C:\Develop\llvm40>mkdir llvm40-build
C:\Develop\llvm40>cd llvm40-build
C:\Develop\llvm40\llvm40-build><b>cmake -G "Visual Studio 15 2017 Win64" -DLLVM_TARGETS_TO_BUILD=X86 -DLLVM_ENABLE_WARNINGS=OFF -DLLVM_INCLUDE_EXAMPLES=OFF -DLLVM_INCLUDE_TESTS=OFF -DLLVM_INCLUDE_DOCS=OFF ../llvm</b>
C:\Develop\llvm40\llvm40-build>cmake --build .
</pre>

You can find more details on building LLVM from source [in the previous post]({{ site.baseurl }}{% post_url 2017-07-17-notes-setup %}). The build process will take time, but you can already prepare the next steps. Checkout the sources for the JitFromScratch project and get ready for generating build files. Once the LLVM build is done, you can run the `cmake` commands:
<pre>
C:\Develop>mkdir JitFromScratch
C:\Develop>cd JitFromScratch
C:\Develop\JitFromScratch><b>git clone https://github.com/weliveindetail/JitFromScratch JitFromScratch</b>
C:\Develop\JitFromScratch\JitFromScratch>cd JitFromScratch
C:\Develop\JitFromScratch\JitFromScratch><b>git checkout -b jit-basics</b>
C:\Develop\JitFromScratch\JitFromScratch>cd ..
C:\Develop\JitFromScratch>mkdir JitFromScratch-build
C:\Develop\JitFromScratch>cd JitFromScratch-build
C:\Develop\JitFromScratch\JitFromScratch-build><b>cmake -G "Visual Studio 15 2017 Win64" -DLLVM_DIR="C:/Develop/llvm40/llvm40-build/lib/cmake/llvm" ..\JitFromScratch</b>
C:\Develop\JitFromScratch\JitFromScratch-build>cmake --build .
C:\Develop\JitFromScratch\JitFromScratch-build>cd Debug
C:\Develop\JitFromScratch\JitFromScratch-build\Debug>JitFromScratch
Integer Distances: 3, 0, 3
</pre>

To explore the code and debug JitFromScratch open the generated `C:\Develop\JitFromScratch\JitFromScratch-build\JitFromScratch.sln` with Visual Studio.

<br>
<br>
</div>



<script language="JavaScript">
$("#linux-mint-18").click(function() { $("#linux-mint-18-content").toggle(); });
$("#mac-os-x-1012").click(function() { $("#mac-os-x-1012-content").toggle(); });
$("#windows-10"   ).click(function() { $("#windows-10-content"   ).toggle(); });

$(function() {
  return $("h2").each(function(i, el) {
    var $el = $(el);
    if ($el.attr('id')) {
      return $el.prepend('<i class="fa fa-chevron-down" style="font-size: 0.8em; color: #333; padding: 5px;"></i>');
    }
  });
});
</script>
