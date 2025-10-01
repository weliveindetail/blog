---
layout: post
categories: post
author: Stefan Gränitz
date: 2025-09-30 12:00:00 +0200
image: https://weliveindetail.github.io/blog/res/2025-tpde-orc.png
preview: summary
title: "Using the TPDE Codegen Backend in LLVM ORC"
description: "TPDE is the perfect fit for a baseline JIT compiler, let's see how to wire it up in ORC JIT!"
source: https://github.com/weliveindetail/blog/main/_posts/2025-09-23-tpde-in-llvm-orc.md
ycombinator: https://news.ycombinator.com/item?id=45423994
---

<style>
  #banner-image {
    margin-bottom: 50px;
  }
  #large-image {
    max-width: min(100%, 230px);
  }
  .center {
    display: block;
    margin: 0 auto;
  }
</style>

![tpde-banner](https://weliveindetail.github.io/blog/res/2025-tpde-orc.png){: #large-image}{: .center}

[TPDE](https://arxiv.org/abs/2505.22610){:target="_blank"} is a single-pass compiler backend for LLVM that was [open-sourced earlier this year](https://discourse.llvm.org/t/tpde-llvm-10-20x-faster-llvm-o0-back-end/){:target="_blank"} by researchers at [TUM](https://db.in.tum.de){:target="_blank"}. The [comprehensive documentation](https://docs.tpde.org/tpde-llvm-main.html){:target="_blank"} walks you through integrating TPDE into custom builds of Clang and Flang. Currently, it supports [LLVM 19](https://github.com/tpde2/tpde/blob/c857798/llvm.ab51eccf88f5.patch){:target="_blank"} and [LLVM 20](https://github.com/tpde2/tpde/blob/c857798/llvm.616f2b685b06.patch){:target="_blank"} release versions.

### Integration in LLVM ORC JIT

TPDE's primary strength lies in delivering low-latency code generation while maintaining reasonable `-O0` code quality — making it an ideal choice for a baseline JIT compiler. LLVM's [On-Request Compilation (ORC)](https://llvm.org/docs/ORCv2.html){:target="_blank"} framework provides a set of libraries for building JIT compilers for LLVM IR. While ORC uses LLVM's built-in backends by default, its flexible architecture makes it straightforward to swap in TPDE instead!

Let's say we use the `LLJITBuilder` interface to instantiate an off-the-shelf JIT:

```cpp
ExitOnError ExitOnErr;
auto Builder = LLJITBuilder();
std::unique_ptr<LLJIT> JIT = ExitOnErr(Builder.create());
```

The builder offers several extension points to customize the JIT instance it creates. To integrate TPDE, we'll override the `CreateCompileFunction` member, which defines how LLVM IR gets compiled into machine code:
```cpp
Builder.CreateCompileFunction = [](JITTargetMachineBuilder JTMB)
    -> Expected<std::unique_ptr<IRCompileLayer::IRCompiler>> {
  return std::make_unique<TPDECompiler>(JTMB);
};
```

To use TPDE in this context, we need to wrap it in a class that's compatible with ORC's interface:
```cpp
class TPDECompiler : public IRCompileLayer::IRCompiler {
public:
  TPDECompiler(JITTargetMachineBuilder JTMB)
      : IRCompiler(irManglingOptionsFromTargetOptions(JTMB.getOptions())) {
    Compiler = tpde_llvm::LLVMCompiler::create(JTMB.getTargetTriple());
    assert(Compiler != nullptr && "Unknown architecture");
  }

  Expected<std::unique_ptr<MemoryBuffer>> operator()(Module &M) override;

private:
  std::unique_ptr<tpde_llvm::LLVMCompiler> Compiler;
  std::vector<std::unique_ptr<std::vector<uint8_t>>> Buffers;
};
```

In the constructor, we initialize TPDE with a target triple (like `x86_64-pc-linux-gnu`). TPDE currently works on ELF-based systems and supports both 64-bit Intel and ARM architectures (`x86_64` and `aarch64`). For now let's assume that's all we need. Now let's implement the actual wrapper code:
```cpp
Expected<std::unique_ptr<MemoryBuffer>> TPDECompiler::operator()(Module &M) {
  Buffers.push_back(std::make_unique<std::vector<uint8_t>>());
  std::vector<uint8_t> &B = *Buffers.back();

  if (!Compiler->compile_to_elf(M, B)) {
    std::string Msg;
    raw_string_ostream(Msg) << "TPDE failed to compile: " << M.getName();
    return createStringError(std::move(Msg), inconvertibleErrorCode());
  }

  StringRef BufferRef{reinterpret_cast<char *>(B.data()), B.size()};
  return MemoryBuffer::getMemBuffer(BufferRef, "", false);
}
```

Here's what's happening: we create a new buffer `B` to store the compiled binary code, then pass both the buffer and the module `M` to TPDE for compilation. If TPDE fails, we bail out with an error. On success, we wrap the result in a `MemoryBuffer` and return it. (Note: LLVM still uses `char` pointers for binary buffers and the [three-way definition of `char` in the C Standard](https://www.open-std.org/JTC1/SC22/WG14/www/docs/n1256.pdf){:target="_blank"} falls on our feet sometimes, but it's difficult to change in a mature codebase like LLVM.)

For the basic integration this is it! No need to patch LLVM — this works with official release versions. We can compile simple LLVM IR code already:
```llvm
> cat 01-basic.ll 
; ModuleID = 'test.ll'
source_filename = "test.ll"

define i32 @main() {
entry:
  %1 = call i32 @custom_entry()
  %2 = sub i32 %1, 123
  ret i32 %2
}

define i32 @custom_entry() {
entry:
  ret i32 123
}
```

I've created a [complete working demo on GitHub](https://github.com/weliveindetail/tpde-orc){:target="_blank"} that you can try out. The code in the repo handles a few more details that we'll explore shortly. Here's what the output looks like:
```
> ./tpde-orc 01-basic.ll 
Loaded module: 01-basic.ll
Executing main()
Program returned: 0

> ./tpde-orc 01-basic.ll --entrypoint custom_entry
Loaded module: 01-basic.ll
Executing custom_entry()
Program returned: 123
```

<a name="compare-llvm-perf"></a>
We can see an impressive 20x speedup with TPDE compared to built-in LLVM codegen for 100 repetitions with a large self-contained module that was generated with [csmith](https://github.com/csmith-project/csmith){:target="_blank"}:
```
> ./build/tpde-orc --par 1 tpde-orc/03-csmith-tpde.ll
...
Compile-time was: 329 ms

> ./build/tpde-orc --par 1 tpde-orc/03-csmith-tpde.ll --llvm
...
Compile-time was: 6796 ms
```

**Please note:** This is not a benchmark! It's just one example on one machine with one possible implementation. The implementation isn't tuned to the limit and the example might not be representative.

My original post showed slower compile-times on both sides. I addressed some feedback and the TPDE case got a lot faster, so I updated the numbers. It's worth mentioning that it's not really fair to compare against LLVM like that, because the off-the-shelf ORC JIT uses optimization level `-O2`. I changed that to `-O0` and TPDE speedup comes down to 12x:
```
> ./build/tpde-orc --par 1 tpde-orc/03-csmith-tpde.ll --llvm
...
Compile-time was: 4060 ms
```

### LLJITBuilder has a Catch

While `LLJITBuilder` is convenient, it comes with a minor trade-off. The interface incorporates standard LLVM components [including `TargetRegistry`](https://github.com/llvm/llvm-project/blob/release/20.x/llvm/lib/ExecutionEngine/Orc/JITTargetMachineBuilder.cpp#L42){:target="_blank"}, which is perfectly reasonable for most use cases. However, this creates a dependency we might not want: the built-in LLVM target backend must be initialized first via `InitializeNativeTarget()`. This means we still need to ship the LLVM backend, even though TPDE could theoretically replace it entirely.

If you want to avoid this dependency, you'll need to set up your ORC JIT manually. For inspiration on this approach, check out [how the `tpde-lli` tool implements it](https://github.com/tpde2/tpde/blob/c2bf98b592a8/tpde-llvm/tools/tpde-lli.cpp#L134-L160){:target="_blank"}. Before diving into that rabbit hole though, let's explore another important aspect!

### Implementing a LLVM Fallback

One reason why TPDE is so fast and compact is that it focuses on the most common use cases rather than [covering every edge case](https://docs.tpde.org/tpde-llvm-main.html#autotoc_md91){:target="_blank"} in the LLVM instruction set. The documentation provides this guideline:

> Code generated by Clang (-O0/-O1) will typically compile; -O2 and higher will typically fail due to unsupported vector operations.

When your code includes advanced features like vector operations or non-trivial floating-point types, TPDE won't be able to handle it. In these cases, we need a fallback to LLVM. Since this scenario is quite common in real-world applications, most tools will include both backends (and we can keep using `LLJITBuilder`). Implementing the fallback is straightforward using [ORC's CompileUtils](https://github.com/llvm/llvm-project/blob/release/20.x/llvm/include/llvm/ExecutionEngine/Orc/CompileUtils.h#L36){:target="_blank"}:

```diff
@@ -29,7 +29,8 @@ static cl::opt<std::string> EntryPoint("entrypoint",
 class TPDECompiler : public IRCompileLayer::IRCompiler {
 public:
   TPDECompiler(JITTargetMachineBuilder JTMB)
-      : IRCompiler(irManglingOptionsFromTargetOptions(JTMB.getOptions())) {
+      : IRCompiler(irManglingOptionsFromTargetOptions(JTMB.getOptions())),
+        JTMB(std::move(JTMB)) {
     Compiler = tpde_llvm::LLVMCompiler::create(JTMB.getTargetTriple());
     assert(Compiler != nullptr && "Unknown architecture");
   }
@@ -37,9 +38,9 @@ Expected<std::unique_ptr<MemoryBuffer>> TPDECompiler::operator()(Module &M) {
   std::vector<uint8_t> &B = *Buffers.back();
 
   if (!Compiler->compile_to_elf(M, B)) {
-    std::string Msg;
-    raw_string_ostream(Msg) << "TPDE failed to compile: " << M.getName();
-    return createStringError(std::move(Msg), inconvertibleErrorCode());
+    errs() << "Falling back to LLVM for module: " << M.getName() << "\n";
+    auto TM = ExitOnErr(JTMB.createTargetMachine());
+    return SimpleCompiler(*TM)(M);
   }
 
   StringRef BufferRef{reinterpret_cast<char *>(B.data()), B.size()};
@@ -50,6 +51,7 @@ public:
 private:
   std::unique_ptr<tpde_llvm::LLVMCompiler> Compiler;
   std::vector<std::unique_ptr<std::vector<uint8_t>>> Buffers;
+  JITTargetMachineBuilder JTMB;
 };
 
 int main(int argc, char *argv[]) {
```

Let's test our fallback mechanism with the following IR file that uses an unsupported type:
```llvm
@const_val = global bfloat 0xR4248

define i32 @main() {
entry:
  %c = load bfloat, ptr @const_val
  %i = fptosi bfloat %c to i32
  ret i32 %i
}
```

Here's what happens when we run it:
```terminal
> ./tpde-orc 02-bfloat.ll
Loaded module: 02-bfloat.ll
[2025-09-25 12:54:03.076] [error] unsupported type: bfloat
[2025-09-25 12:54:03.076] [error] Failed to compile function main
Falling back to LLVM for module: 02-bfloat.ll
Executing main()
Program returned: 50
```

In this implementation, we create a new `SimpleCompiler` instance for each fallback case. While this adds some overhead, it's acceptable since we're already on the slow path. The key assumption is that most code in your workload will successfully compile with TPDE — if that's not the case, then TPDE might not be the right choice in the first place. Interestingly, this approach has a valuable side-effect that becomes important in the next section: it's inherently thread-safe!

### Adding Concurrent Compilation Support

ORC JIT has built-in support for concurrent compilation. This is neat, but it requires attention when customizing the JIT. Our current setup uses a single `TPDECompiler` instance, but TPDE's `compile_to_elf()` method isn't thread-safe. Enabling concurrent compilation would cause multiple threads to call this method simultaneously, leading to failures.

How can we solve this? One option would be creating a new `tpde_llvm::LLVMCompiler` instance for each compilation job, but that adds an overhead of `O(#jobs)` — not ideal for our fast path. Essentially, we want to avoid calling into `compile_to_elf()` while there is another call in-flight on the same thread. We can achieve this easily by making the `TPDECompiler` instance thread-local, reducing the overhead to just `O(#threads)`:

```diff
@@ -32,7 +32,6 @@ public:
   TPDECompiler(JITTargetMachineBuilder JTMB)
       : IRCompiler(irManglingOptionsFromTargetOptions(JTMB.getOptions())),
         JTMB(std::move(JTMB)) {
-    Compiler = tpde_llvm::LLVMCompiler::create(JTMB.getTargetTriple());
     assert(Compiler != nullptr && "Unknown architecture");
   }
 
@@ -50,11 +49,14 @@ public:
   }
 
 private:
-  std::unique_ptr<tpde_llvm::LLVMCompiler> Compiler;
+  static thread_local std::unique_ptr<tpde_llvm::LLVMCompiler> Compiler;
   std::vector<std::unique_ptr<std::vector<uint8_t>>> Buffers;
   JITTargetMachineBuilder JTMB;
 };
 
+thread_local std::unique_ptr<tpde_llvm::LLVMCompiler> TPDECompiler::Compiler =
+    tpde_llvm::LLVMCompiler::create(Triple(LLVM_HOST_TRIPLE));
+
 int main(int argc, char *argv[]) {
   InitLLVM X(argc, argv);
   cl::ParseCommandLineOptions(argc, argv, "TPDE ORC JIT Compiler\n");
```

We also need to guard access to our underlying buffers:
```diff
@@ -35,15 +35,19 @@ public:
   }
 
   Expected<std::unique_ptr<MemoryBuffer>> operator()(Module &M) override {
-    Buffers.push_back(std::make_unique<std::vector<uint8_t>>());
-    std::vector<uint8_t> *B = *Buffers.back().get();
+    std::vector<uint8_t> *B;
+    {
+      std::lock_guard<std::mutex> Lock(BuffersAccess);
+      Buffers.push_back(std::make_unique<std::vector<uint8_t>>());
+      B = Buffers.back().get();
+    }
 
     if (!Compiler->compile_to_elf(M, *B)) {
       errs() << "Falling back to LLVM for module: " << M.getName() << "\n";
@@ -50,6 +54,7 @@ public:
 private:
   static thread_local std::unique_ptr<tpde_llvm::LLVMCompiler> Compiler;
   std::vector<std::unique_ptr<std::vector<uint8_t>>> Buffers;
+  std::mutex BuffersAccess;
   JITTargetMachineBuilder JTMB;
 };

```

With thread safety handled, we can now enable concurrent compilation:
```diff
@@ -27,6 +27,10 @@ static cl::opt<std::string> EntryPoint("entrypoint",
                                       cl::desc("Entry point function name"),
                                       cl::init("main"));

+static cl::opt<unsigned>
+    Threads("par", cl::desc("Compile csmith code on N threads concurrently"),
+            cl::init(0));
+
class TPDECompiler : public IRCompileLayer::IRCompiler {
public:
@@ -65,6 +65,8 @@ int main(int argc, char *argv[]) {
       -> Expected<std::unique_ptr<IRCompileLayer::IRCompiler>> {
     return std::make_unique<TPDECompiler>(JTMB);
   };
+  Builder.SupportConcurrentCompilation = true;
+  Builder.NumCompileThreads = Threads;
   std::unique_ptr<LLJIT> JIT = ExitOnErr(Builder.create());
 
   ThreadSafeModule TSM(std::move(Mod), std::move(Context));
```

### Exercising Concurrent Lookup

It needs a lot more support code to actually exercise concurrent compilation and do basic performance measurements. The [complete sample project on GitHub](https://github.com/weliveindetail/tpde-orc){:target="_blank"} has one possible implementation: after loading the input module, it creates 100 duplicates of it with different entry-point names and issues a single JIT lookup for all the entry-points at once. Here's a simplified version of how this works:
```cpp
SymbolMap SymMap;
SymbolLookupSet EntryPoints = addDuplicates(JIT, Mod);

outs() << "Compiling " << EntryPoints.size() << " modules on " << Threads
        << " threads in parallel\n";

using namespace std::chrono;
auto ES = JIT->getExecutionSession();
auto SO = makeJITDylibSearchOrder({JIT->getMainJITDylib()});
auto Start = steady_clock::now();
{
  // Lookup all entry-points at once to execise concurrent compilation
  SymMap = ExitOnErr(ES.lookup(SO, EntryPoints));
}
auto End = steady_clock::now();
auto Elapsed = duration_cast<milliseconds>(End - Start);

outs() << "Compile-time was: " << Elapsed.count() << " ms\n";
```

Compile-times for our csmith example drop from ~2200ms to just ~740ms when utilizing 8 threads in parallel:
```
> ./tpde-orc --par 8 tpde-orc/03-csmith-tpde.ll
Load module: tpde-orc/03-csmith-tpde.ll
Compiling 100 modules on 8 threads in parallel
...
Compile-time was: 737 ms
```

### Et voilà!

Let's wrap it up and appreciate the remarkable complexity that LLVM effortlessly handles in our little example. We parse a [well-defined, human-readable representation](https://llvm.org/docs/LangRef.html){:target="_blank"} of Turing-complete programs generated from various general-purpose languages like C++, Fortran, Rust, Swift, Julia, and Zig.

LLVM's composable JIT engine seamlessly manages these parsed modules, automatically resolving symbols and dependencies. It compiles machine code in the native object format on-demand for multiple platforms and CPU architectures, while giving us complete control over the optimization pipeline, code generator (like our TPDE integration) and many more components. The engine then links everything into an executable form — all in-memory and without external tools or platform-specific dynamic library tricks! It's really impressive that we can simply enable compilation on N threads in parallel and have it "just work" :-)

<a name="follow-ups"></a>
**Follow-up:** After revisiting the single-threaded compile-times, the multi-threading result is rather disappointing. Digging deeper, there seem to be two important reasons why this is so slow:

1. If we enable concurrent compilation, LLJIT activates a mechanism that makes sure the workload can actually be processed in parallel. This requires modules to live in distinct LLVM contexts and [LLJIT clones all modules](https://github.com/llvm/llvm-project/blob/release/20.x/llvm/lib/ExecutionEngine/Orc/LLJIT.cpp#L1040){:target="_blank"} before dispatch. This is very expensive and actually dominates compile-times. The mechanism wouldn't be necessary in our case here. As long as we keep using `LLJITBuilder` though, we cannot disable it, because the API gives no access to LLJIT's `InitHelperTransformLayer`, which is responsible for the cloning.

2. The `DynamicThreadPoolTaskDispatcher` has a catch: It only pools compile jobs, but [keeps spawning new threads for everything else](https://github.com/llvm/llvm-project/blob/87f0227cb60147a26a1eeb4fb06e3b505e9c7261/llvm/lib/ExecutionEngine/Orc/TaskDispatch.cpp#L67){:target="_blank"}, which is the majority of `Task`s it dispatches in our examples. Respectively, `O(#threads)` is larger than `O(#jobs)` so that the thread-local compiler setup even slows down our multi-threading case! That's a pity and it caused issues in real-world projects already. Right now, implementing a custom task dispatcher downstream seems to be the best solution. This is how JuliaLang fixed it: [julia#58950](https://github.com/JuliaLang/julia/pull/58950){:target="_blank"}. [Lang Hames](https://github.com/lhames){:target="_blank"}, the author of ORC, wants to address this issue in [his upcoming presentation](https://llvm.swoogo.com/2025devmtg/session/3366605/jit-loading-arbitrary-programs-%E2%80%94-powering-xcode-previews-with-llvm%E2%80%99s-jit){:target="_blank"} at the US LLVM Developers' Meeting in October.

3. Last but not least, my post inspired [Alexis Engelke](https://github.com/aengelke){:target="_blank"}, one of the TPDE authors, to [sketch on a TPDE layer for ORC upstream](https://github.com/tpde2/tpde/commit/29bcf1841c572fcdc75dd61bb3efff5bfb1c5ac6){:target="_blank"}. This is a pretty good idea indeed!
