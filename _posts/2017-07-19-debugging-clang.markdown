---
layout: post
author: Stefan Gränitz
title:  "Debugging Clang"
date:   2017-07-19 18:30:01 +0200
categories: post
comments: true
---

There's comprehensive high-level documentation for [LLVM](http://llvm.org/docs/) and [Clang](https://clang.llvm.org/docs/). However, the further down you dig the more difficult things get. [Doxygen](http://clang.llvm.org/doxygen/) helps a lot to figure what is available through an API and which entry points exists, but it naturally lacks information on how to compose things to achieve your goal. Sooner or later you will need to find out yourself.

<p style="text-align: right;">
  TL;DR &darr; <a href="#so-how-can-we-debug-the-clang-frontend">Debug the Clang frontend</a>
</p>

### See how Clang does it

This is probably the top one answer in the mailing lists or stackoverflow for questions on IR code generation or library usage. It's a simple and pragmatic solution considering the pace of change in the code base of LLVM and Clang. Any written low-level documentation will quickly be outdated anyway. People spend their time better by keeping the code of the reference implementation clean and readable.

So after you managed to [build Clang locally]({{ site.baseurl }}{% post_url 2017-07-17-notes-setup %}#use-enable_projects), you eventually find yourself crawling through tons of Clang sources to actually find out how Clang does it. Let's say you want to know how Clang mangles C++ function names. On OSX you would set a breakpoint in [`CXXNameMangler::mangle()`](https://github.com/llvm-mirror/clang/blob/master/lib/AST/ItaniumMangle.cpp#L641). Then build Clang in debug mode and watch it compiling a C++ file like this:

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

The issue was discussed in more detail on the cfe-dev mailing list some time ago:<br> [http://lists.llvm.org/pipermail/cfe-dev/2014-January/034870.html](http://lists.llvm.org/pipermail/cfe-dev/2014-January/034870.html).

### So how can we debug the Clang frontend?

If you use flags to run the driver you need to translate them from driver to frontend representation first. The `-###` flag is very useful here. Prepend it to your command line to run only the driver and print out transformed arguments:

<pre>
$ <b>clang -### -c example.cpp</b>
clang version 3.8.0 (tags/RELEASE_380/final)
Target: x86_64-apple-darwin16.6.0
Thread model: posix
InstalledDir: /usr/local/bin
 "/Users/staehff/Dev/3rdParty/llvm40-build-xcode/Debug/bin/clang" "-cc1" "-triple" "x86_64-apple-macosx10.12.0" "-Wdeprecated-objc-isa-usage" "-Werror=deprecated-objc-isa-usage" "-emit-obj" "-mrelax-all" "-disable-free" "-disable-llvm-verifier" "-discard-value-names" "-main-file-name" "example.cpp" "-mrelocation-model" "pic" "-pic-level" "2" "-mthread-model" "posix" "-mdisable-fp-elim" "-masm-verbose" "-munwind-tables" "-target-cpu" "penryn" "-target-linker-version" "302.1" "-dwarf-column-info" "-debugger-tuning=lldb" "-coverage-notes-file" "/Users/staehff/Dev/3rdParty/llvm50/build-xcode-llvm-clang/Debug/bin/example.gcno" "-resource-dir" "/Users/staehff/Dev/3rdParty/llvm40-build-xcode/Debug/bin/../lib/clang/4.0.1" "-stdlib=libc++" "-fdeprecated-macro" "-fdebug-compilation-dir" "/Users/staehff/Dev/3rdParty/llvm50/build-xcode-llvm-clang/Debug/bin" "-ferror-limit" "19" "-fmessage-length" "181" "-stack-protector" "1" "-fblocks" "-fobjc-runtime=macosx-10.12.0" "-fencode-extended-block-signature" "-fcxx-exceptions" "-fexceptions" "-fmax-type-align=16" "-fdiagnostics-show-option" "-fcolor-diagnostics" "-o" "example.o" "-x" "c++" "example.cpp"
</pre>

Take what you need for your use case and invoke the frontend directly as a single process with:
<pre>
$ <b>clang -cc1 -emit-obj -o ~/example.o -x c++ ~/example.cpp</b>
</pre>

If you do this in your IDE your breakpoint will now hit. You can see how name mangling is invoked and you can debug through the process. Happy hacking!

<pre class="highlight">
$ lldb -- clang
(lldb) target create "clang"
Current executable set to 'clang' (x86_64).
(lldb) b CXXNameMangler::mangle
Breakpoint 1: 2 locations.
(lldb) <b>r -cc1 -emit-obj example.cpp</b>
Process 27909 launched: 'clang' (x86_64)
Process 27909 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = breakpoint 1.1
    frame #0: 0x000000010a69802b libclangAST.dylib`(anonymous namespace)::CXXNameMangler::mangle(this=0x00007fff5fbf5758, D=0x000000011484e3e8) at ItaniumMangle.cpp:641
   638    // &lt;mangled-name&gt; ::= _Z &lt;encoding&gt;
   639    //                ::= &lt;data name&gt;
   640    //                ::= &lt;special-name&gt;
-&gt; 641    Out &lt;&lt; "_Z";
   642    if (const FunctionDecl *FD = dyn_cast&lt;FunctionDecl&gt;(D))
   643      mangleFunctionEncoding(FD);
   644    else if (const VarDecl *VD = dyn_cast&lt;VarDecl&gt;(D))
</pre>

Happy hacking!
