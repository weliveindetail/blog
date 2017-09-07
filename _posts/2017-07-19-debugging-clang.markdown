---
layout: post
author: Stefan Gränitz
title:  "Debugging Clang"
date:   2017-07-19 18:30:01 +0200
categories: post
comments: true
--- 
The high-level documentation for [LLVM](http://llvm.org/docs/) and [Clang](https://clang.llvm.org/docs/) is pretty good and gives a lot of useful resources. However, when it comes to bare metal and you want to know how to use any specific API, things can get increasingly difficult.

Doxygen ([LLVM](http://llvm.org/doxygen/), [Clang](http://clang.llvm.org/doxygen/)) helps a lot to figure what is available through an API and which entry points exists, but it naturally lacks information on how to compose things to achieve your goal.

### See how Clang does it

...is probably the top one answer for questions in the mailing lists or stackoverflow. In fact it's not only a pragmatic solution, but IMHO also quite an effective one because:
* **Clang is the reference implementation** for LLVM-based compilers
* facing the pace of change in the code base of LLVM and Clang, any written low-level documentation will quickly be outdated anyway

So eventually you will find yourself crawling through tons of Clang sources to actually find out how Clang does it (if you don't know how to build Clang side-by-side with LLVM then [read it here]({{ site.baseurl }}{% post_url 2017-07-17-notes-setup %}#use-enable_projects)). Let's say you want to know how Clang mangles C++ function names. If you are on OSX you will probably set a breakpoint in [`CXXNameMangler::mangle()`](https://github.com/llvm-mirror/clang/blob/master/lib/AST/ItaniumMangle.cpp#L641). Then you will build Clang in debug mode and watch it compiling a C++ file like this:

{% highlight c++ %}
extern int returnValue(int a, char **b);

int main(int argc, char **argv) {
  return returnValue(argc, argv);
}
{% endhighlight %}

If you did that the first time, you may be surprised that your breakpoint never hits. Before you start searching for mistakes: You didn't do anything wrong but **just encountered a common pitfall**.

### Driver vs. Frontend

In Clang driver (clang/clang-cl) and frontend (cc1) are separate entities. The driver mainly manages scheduling for compile jobs and transforms command line arguments from GCC- or MSVC-compatible ones to an independent internal representation. Then for each job it forks itself with the new set of arguments that invoke the frontend directly. AFAIK the reasons for this behavior are mostly historical. However, there are a few benefits with it:
* in each fork memory deallocation can be skipped as the system cleans it up with the process
* if the forked clang crashes, the parent process can generate a preprocessed source to serve as a test case

The issue was discussed in more detail on the [cfe-dev] mailing list some time ago:<br> [http://lists.llvm.org/pipermail/cfe-dev/2014-January/034870.html](http://lists.llvm.org/pipermail/cfe-dev/2014-January/034870.html).

### So how can we debug the Clang frontend?

First we can manually process the flags that the Clang driver would use to invoke the frontend. Prepend your command line with `-###` to run only the driver and print transformed arguments:
<pre>
$ <b>clang -### -c -o ~/example.o ~/example.cpp</b>
clang version 3.8.0 (tags/RELEASE_380/final)
Target: x86_64-apple-darwin16.6.0
Thread model: posix
InstalledDir: /usr/local/bin
 "/usr/local/bin/clang" "-cc1" "-triple" "x86_64-apple-macosx10.12.0" "-Wdeprecated-objc-isa-usage" "-Werror=deprecated-objc-isa-usage" "-emit-obj" "-mrelax-all" "-disable-free" "-disable-llvm-verifier" "-main-file-name" "example.cpp" "-mrelocation-model" "pic" "-pic-level" "2" "-mthread-model" "posix" "-mdisable-fp-elim" "-masm-verbose" "-munwind-tables" "-target-cpu" "core2" "-target-linker-version" "242" "-dwarf-column-info" "-debugger-tuning=lldb" "-coverage-file" "/Users/user/example.o" "-resource-dir" "/usr/local/bin/../lib/clang/3.8.0" "-stdlib=libc++" "-fdeprecated-macro" "-fdebug-compilation-dir" "/Users/user/Dev/Personal/JitFromScratch" "-ferror-limit" "19" "-fmessage-length" "149" "-stack-protector" "1" "-fblocks" "-fobjc-runtime=macosx-10.12.0" "-fencode-extended-block-signature" "-fcxx-exceptions" "-fexceptions" "-fmax-type-align=16" "-fdiagnostics-show-option" "-fcolor-diagnostics" "-o" "/Users/user/example.o" "-x" "c++" "/Users/user/example.cpp"
</pre>

Then you invoke the frontend directly. For this example actually only a few arguments are required:
<pre>
$ <b>clang -cc1 -emit-obj -o ~/example.o -x c++ ~/example.cpp</b>
$ nm ~/example.o
                 U __Z11returnValueiPPc
0000000000000000 T _main
</pre>

If you do this in your IDE your breakpoint will now hit. You can see how name mangling is invoked and you can debug through the process. Happy hacking!
