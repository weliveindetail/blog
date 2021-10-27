---
layout: post
categories: post
author: Stefan Gränitz
date: 2021-10-27 13:30:00 +0200
image: https://weliveindetail.github.io/blog/res/cling-llvm13-orcv2-preview.png
preview: summary_large_image
title: "Porting Cling to LLVM 13 and ORCv2"
description: "The upstream version of the C++ interpreter is based on LLVM 9 and uses the now deprecated ORCv1 JIT libraries. Here are my notes and learnings from rebasing it to the latest release."
source: https://github.com/weliveindetail/blog/blob/main/_posts/2021-10-27-cling-llvm13-orcv2.md
comments: https://news.ycombinator.com/item?id=29011792
---

<style>
  #teaser-image {
    max-width: 500px;
  }
  .center {
    display: block;
    margin: 0 auto;
    width: 80vmin;
  }
  .baobab {
    display: block;
    margin: 0 auto;
    width: 80vmin;
    height: 80vmin;
    max-width: 500px;
    max-height: 500px;
    border: 0;
  }
  .flex-grid {
    display: flex;
    flex-direction: row;
    margin-top: -20px;
    margin-bottom: 20px;
  }
  .flex-grid .baobab {
    width: 40vmin;
    height: 50vmin;
    min-width: 250px;
    min-height: 300px;
  }
  @media only screen and (max-width: 500px) {
    .flex-grid {
      flex-direction: column;
    }
  }
</style>

![cling demo on llvm13 branch](https://weliveindetail.github.io/blog/res/cling-llvm13-orcv2-teaser.png){: #teaser-image}{: .center}

[Cling](https://github.com/root-project/cling){:target="_blank"} is a Clang-based C++ interpreter developed by [CERN](https://root.cern/cling/){:target="_blank"} as part of the high-energy physics data analysis project [ROOT](https://root.cern/){:target="_blank"}. It built against LLVM 5 since 2015 and was [ported to LLVM 9](https://lists.llvm.org/pipermail/llvm-dev/2020-July/143257.html){:target="_blank"} last year. With this latest official version it still uses LLVM's [now deprecated ORCv1 JIT libraries](https://reviews.llvm.org/D64609){:target="_blank"}. [ORCv2](https://llvm.org/docs/ORCv2.html){:target="_blank"} evolved in parallel to its predecessor since LLVM 7 and introduced a complete redesign of the JIT API. It's one of the breaking changes in LLVM that cling hasn't caught up with yet.

Updating an external dependency can be a major effort for any project. LLVM bears a high risk here, because the C++ APIs don't offer any guarantees for stability across release versions. Furthermore, cling maintains it's own set of downstream modifications on [LLVM's release/9.x](https://github.com/llvm/llvm-project/commits/release/9.x){:target="_blank"} branch. [Here](https://weliveindetail.github.io/blog/res/cling-llvm09-baobab.html){:target="_blank"} is an interactive visualization:

<iframe src="https://weliveindetail.github.io/blog/res/cling-llvm09-baobab.html" title="git-baobab: cling downstream changes LLVM 9" class="baobab">
  <a href="https://weliveindetail.github.io/blog/res/cling-llvm09-baobab.html"
     target="_blank" class="baobab">
    <img src="https://weliveindetail.github.io/blog/res/cling-llvm09-baobab.png"
         alt="git-baobab: cling downstream changes LLVM 9">
  </a>
</iframe>

### Strategy

Cling's downstream patches for LLVM won't apply anymore if the code was modified upstream. We will get merge conflicts when rebasing it. The majority of conflicts are plain mix-and-match exercises: Find the colliding commit(s) and adopt the change in the downstream patch. However, there will be non-obvious side effects from time to time. We'd be more confident in our conflict resolutions, if we could run a smoke test that exercised the modified code. Unfortunately, Cling won't compile until we finished rebasing onto the new version **and** also fixed Cling's usage of the new API!

Eventually, we want to bring Cling's downstream patches to LLVM's most recent release branch. But the distance from release 9 is huge. Upstream LLVM is subject to change, permanently. There's 73k commits in between the two releases:
```
➜ git clone https://github.com/weliveindetail/llvm-project
➜ cd llvm-project
➜ git log --oneline $(git merge-base release/9.x release/13.x)..release/13.x | wc -l
73303
```

Trying to do this in a single step bears the risk of ending up with runtime failures, that will be hard to nail down to a specific patch. Instead, it's common practice to go from one release version to the next. In each iteration we will follow a simple plan:

1. **Rebase Cling's downstream patches to the next release.** Solve each conflict on a best effort basis and make sure the affected LLVM libraries compile.
2. **Build Cling against the new LLVM libraries.** Each compiler error indicates an API change. For now we fix one after the other in the Cling code. Maybe we have to add further downstream patches in LLVM as well.
3. **Run a smoke test** once Cling compiles and links. Each bug that didn't exist in the previous version, was likely introduced by us. Fix it and amend the changes to the respective commit(s).
4. **Sort out the Cling API fixes** into meaningful commits and track them on a new branch. Don't skip this! A clean history is key for future iterations.

### Rebase to the next LLVM release version

In our first iteration we go from [release/9.x](https://github.com/llvm/llvm-project/commits/release/9.x){:target="_blank"} to [release/10.x](https://github.com/llvm/llvm-project/commits/release/10.x){:target="_blank"}:
```
➜ cd llvm-project
➜ git checkout cling-09
➜ git fetch origin release/9.x release/10.x
➜ git log --oneline release/9.x..HEAD
497d28c58a51 Allow interfaces to operate on in-memory buffers ...
...
5c50e7430981 Enable unicode output on terminals.
➜ git checkout -b cling-10
➜ git rebase release/10.x
```

This is mostly straightforward, [here](https://github.com/weliveindetail/llvm-project/commits/cling-10){:target="_blank"} is the rebased set of commits. There is [one additional patch](https://github.com/weliveindetail/llvm-project/commit/792e5b9169d005730178ee11c03d14cf7f7f0104){:target="_blank"} that fixes the [shared libraries](https://llvm.org/docs/CMake.html#llvm-related-variables){:target="_blank"} build in upstream LLVM (`libLLVMExtensions`  didn't link against `libLLVMSupport` and thus missed definitions for `LLVM_ENABLE_ABI_BREAKING_CHECKS`). Quick hacks like this can be useful, because **we can't fix the world while rebasing**. Make sure to mark commits as such!

Now that our downstream LLVM does compile, we can fix Cling's usage of the API. This is a **search-intensive task**. An editor or IDE with reference search and efficient code navigation makes a big difference. I've been using vscode with the [clangd plugin](https://marketplace.visualstudio.com/items?itemName=llvm-vs-code-extensions.vscode-clangd){:target="_blank"}. It's good for plain text and regex search. For search in CMake files I use [my own version of `ag` (The Silver Searcher)](https://github.com/weliveindetail/the_silver_searcher/commit/e50ba4de24415f421f3fd945dcadfe14bafcda19){:target="_blank"}.

[The number of changes](https://github.com/weliveindetail/cling/commits/cling-10){:target="_blank"} is fairly small. It contains [another hack](https://github.com/weliveindetail/cling/commit/db5be6e732fa02096843dd35a9bf59c398e4df61){:target="_blank"} which is useful to **suppress all compiler warnings** in Cling. Adopting API changes means that we compile and scan through the output over and over again. [Any warning that pops up](https://weliveindetail.github.io/blog/post/2021/10/20/vscode-hide-warning-markers.html) on the way will cause additional fuzz and slow down the process. We can deal with them later.

Once we tested the intermediate state and sorted out changes, we can start the next iteration.

### Skipping release/12.x

Sometimes it's worth skipping a release. In case of Cling LLVM 12 is a good candidate, because it had [removed the ORCv1 JIT libraries](https://lists.llvm.org/pipermail/llvm-dev/2020-September/144885.html){:target="_blank"} (for good reasons) while the ORCv2 API was still quite unstable: [The public ORCv2 API](https://github.com/llvm/llvm-project/llvm/include/llvm/ExecutionEngine/Orc){:target="_blank"} (left/top) saw [4.7K insertions/deletions](https://weliveindetail.github.io/blog/res/llvm13-orcv2-api.html?path=llvm-project/llvm/include/llvm/ExecutionEngine){:target="_blank"} during that time! For comparison, it had only [2K insertions/deletions](https://weliveindetail.github.io/blog/res/llvm11-orcv2-api.html?path=llvm-project/llvm/include/llvm/ExecutionEngine){:target="_blank"} in the LLVM 11 cycle (right/bottom).

<div class="flex-grid">
<iframe src="https://weliveindetail.github.io/blog/res/llvm13-orcv2-api.html?path=llvm-project/llvm/include/llvm/ExecutionEngine" title="git-baobab: ORCv2 API changes in LLVM 13" class="baobab">
  <a href="https://weliveindetail.github.io/blog/res/llvm13-orcv2-api.html?path=llvm-project/llvm/include/llvm/ExecutionEngine"
     target="_blank" class="baobab">
    <img src="https://weliveindetail.github.io/blog/res/llvm13-orcv2-api.html.png"
         alt="git-baobab: ORCv2 API changes in LLVM 13">
  </a>
</iframe>

<iframe src="https://weliveindetail.github.io/blog/res/llvm11-orcv2-api.html?path=llvm-project/llvm/include/llvm/ExecutionEngine" title="git-baobab: ORCv2 API changes in LLVM 11" class="baobab">
  <a href="https://weliveindetail.github.io/blog/res/llvm11-orcv2-api.html?path=llvm-project/llvm/include/llvm/ExecutionEngine"
     target="_blank" class="baobab">
    <img src="https://weliveindetail.github.io/blog/res/llvm11-orcv2-api.html.png"
         alt="git-baobab: ORCv2 API changes in LLVM 11">
  </a>
</iframe>
</div>

There was a moderate risk, that reimplementing [Cling's IncrementalJIT](https://github.com/weliveindetail/cling/blob/cling-09/lib/Interpreter/IncrementalJIT.cpp){:target="_blank"} for LLVM 12 would cause a large refactor in the next iteration. For me this justified a bigger rebase step from 11 directly to 13.

In the end, it didn't matter much, because the basic [ORCv2 replacement](https://github.com/weliveindetail/cling/blob/cling-13/lib/Interpreter/IncrementalJIT_ORCv2.cpp){:target="_blank"} uses the API on a pretty high level. Looking back, an **alternative strategy** might have been better: Reimplement the IncrementalJIT based on LLVM 11 (where ORCv1 was still present) and then move on to the next versions as usual.

### IncrementalJIT ORCv2 replacement

The rebase plan works very well until Cling's downstream patches [arrive on release/13.x](https://github.com/weliveindetail/llvm-project/commits/cling-13){:target="_blank"}. Step 2 of this last iteration is more difficult. In order to get Cling to build against the new LLVM libraries, we would need to reimplement the entire IncrementalJIT against the ORCv2 API.

This ORCv2 replacement is a big effort and we're better off with a version that's incomplete but executable. For that we first [adopt all non-JIT API changes](https://github.com/weliveindetail/cling/compare/baa8416fb30cde7371c412672eb7bcf8fc250111..eb078ba782dc9e2e7a7556525ce9c2f47f38d9af){:target="_blank"} and [comment out the old IncrementalJIT](https://github.com/weliveindetail/cling/commit/40647a9fa77f6136c513f69df6e3bbbdb7cdbb2f){:target="_blank"} until Cling compiles and links again. Next we [setup an ORCv2 replacement class](https://github.com/weliveindetail/cling/commit/e107d757734a035b512a52ba02f499e8de2f0e08){:target="_blank"} with stub functions that fit the existing interface. Note that we have to keep including the old `IncrementalJIT.h`, because  other components in Cling are using some of its type definitions.

From here on we can run Cling in a debugger again. We can put breakpoints in our stubs and run commands in Cling until they hit. This is a very **convenient way to reimplement the existing semantics**, because we can inspect the relevant states at runtime!

In the constructor we create a [basic greedy `llvm::orc::LLJIT`](https://github.com/weliveindetail/cling/commit/5917f160ce452e2bbd7651ce5ccdf6c55ce4bc89){:target="_blank"} instance. The [`addModule()` function](https://github.com/weliveindetail/cling/commit/21d5a188df1a7fe9dc15dedc4c7a5c47cb45eb11){:target="_blank"} wraps an incoming `llvm::Module` in a `llvm::orc::ThreadSafeModule` and passes it on to the `LLJIT`. The existing interface is not prepared to propagate LLVM's [rich recoverable errors](https://weliveindetail.github.io/blog/post/2017/10/22/llvm-expected.html){:target="_blank"}. If we get an error, we log it to `stderr` and stick to the existing interface semantics. The [`getSymbolAddress()` function](https://github.com/weliveindetail/cling/commit/2ace5803d4f385968db7b5a010783682117e7b9c){:target="_blank"} does a simple lookup in the JIT, while [`lookupSymbol()`](https://github.com/weliveindetail/cling/commit/b2ac058af82fc7c83ccedb8dd0d88d8c2dfd52bd){:target="_blank"} appears to be used only to inject symbols for existing function addresses (like `__cxa_atexit` and `__dso_handle`). It runs a lookup beforehand to make sure the symbol doesn't exist and it records the injected symbols in a map.

Cling has its own little runtime library. It's used when we assign existing values or print them out. In order to get it to work, we have to [implement host process lookup](https://github.com/weliveindetail/cling/commit/a606044670c3813184ffd62e29d19ab788fdc655){:target="_blank"}. With that we can evaluate simple expressions already!

### Basic transaction rollback

Cling's `.undo` command rolls back the JITed program to the state of the previous expression. The implementation in [`cling::TransactionUnloader`](https://github.com/weliveindetail/cling/blob/cling-13/lib/Interpreter/TransactionUnloader.h){:target="_blank"} makes some assumptions about the way `IncrementalJIT` works. In particular, it expects to find the transaction's `llvm::Module` in a map of "non-pending" modules. I guess this design has historic reasons: With `ORCv1` modules could only be unloaded once the JIT pipeline had finished processing them. `ORCv2` removed that restriction and allows any module to be unloaded, independent of its materialization state. Because fixing `TransactionUnloader` is not our goal, we hack the system and [report any module as non-pending immediately](https://github.com/weliveindetail/cling/commit/bfeac9e7f752c0e1105ed24f4d703333e7d66d2f){:target="_blank"}.

Now we can add resource tracking to our `LLJIT` and [implement `removeModule()` for basic code unloading](https://github.com/weliveindetail/cling/commit/5a554cc883f3f269aeba2ef81b0eb2d937920b49){:target="_blank"}. The patch adds a callback to our `IRCompileLayer`, which transfers back ownership of processed modules to `IncrementalJIT`, so we can hand it out to the caller of `removeModule()`. But there is a final hurdle: We must extract the `llvm::Module` from the received `llvm::orc::ThreadSafeModule` and LLVM's upstream API doesn't allow this (for good reasons). We'd like to keep it simple for now, so [we add a new downstream patch](https://github.com/weliveindetail/llvm-project/commit/949bfdb1381bec244280e6317515392cc9defe24){:target="_blank"} that does it.

### Voilà!

This way we got Cling to work with LLVM 13 and ORCv2. What is left are a few non-functional cleanup steps, like [removing ORCv1 types from function signatures](https://github.com/weliveindetail/cling/commit/987486d2a26396bd4bee8b1df07f8b067e3dfb67){:target="_blank"} and [removing the now unused ORCv1 IncrementalJIT](https://github.com/weliveindetail/cling/commit/93881402b425c7a144e89f27712f6bf18ad5fe9d){:target="_blank"}. Here is the final set of [changes in Cling](https://github.com/weliveindetail/cling/commits/cling-13) and in [Cling's LLVM version](https://github.com/weliveindetail/llvm-project/commits/cling-13){:target="_blank"}.

The resulting Cling is a best-effort version. I didn't go and investigate test-suite failures. It's good enough to produce [the initial screenshot](#teaser-image) on macOS, but it doesn't support the full feature-set yet. For now I didn't look at other platforms. Linux will be a low-hanging fruit, Windows is a different story. There was [one API change](https://github.com/weliveindetail/cling/commit/8d8fe60a4ce45281e2a9320d08646009638591bc){:target="_blank"} where I didn't find a solution quickly, so I left a note and skipped it. It causes a number of issues, like failing `.undo` commands in non-trivial cases. I am sure there are more problems that I didn't spot, but from here on we can iterate.

Thanks for reading! Here is how to build our new version of Cling in debug mode:
```
➜ git clone https://github.com/weliveindetail/llvm-project
➜ git -C llvm-project checkout cling-13
➜ git clone https://github.com/weliveindetail/cling llvm-project/llvm/tools/cling
➜ git -C llvm-project/llvm/tools/cling checkout cling-13
➜ ln -s ../../clang llvm-project/llvm/tools/clang
➜ mkdir build
➜ cmake -S "llvm-project/llvm" -B "build" -GNinja -DBUILD_SHARED_LIBS=On -DLLVM_ENABLE_PROJECTS=clang -DLLVM_EXTERNAL_PROJECTS=cling -DLLVM_TARGETS_TO_BUILD="ARM;NVPTX;X86"
➜ ninja -C build cling
➜ build/bin/cling --version
1.0~dev
LLVM (http://llvm.org/):
  LLVM version 13.0.0
  DEBUG build with assertions.
  Default target: x86_64-apple-darwin20.6.0
  Host CPU: skylake
```
