---
layout: post
author: Stefan Gränitz
title:  "LLVM Release 9.0 changes"
date:   2019-07-18 16:57:01 +0200
categories: post
comments: true
---

Today LLVM release 9.0 branched, here's what changed in a hot-cold map. Click on the diagrams for the interactive view.

[![Changes in C++ files outside tests](https://weliveindetail.github.io/git-baobab/examples/llvm9-cpp-sources.png)](https://weliveindetail.github.io/git-baobab/examples/llvm9-cpp-sources.html)

The diagram shows the accumulated changes in C++ files outside tests since Release 8.0 branched on Jan 16, 2019. The size of an arc represents the amount of change in a file/directory relative to its sibling files/directories. The color of an arc indicates the amount of change relative to its line count today. The amount of change is the sum of insertions and deletions.

I generated the diagram with [git-baobab](https://github.com/weliveindetail/git-baobab), a little tool I played around with in spare time during the last months:
```
$ git clone https://github.com/llvm/llvm-project.git
$ cd llvm-project
$ git checkout release/9.x
$ git merge-base origin/release/8.x HEAD
7b5565418f4d6e113ba805dad40d471d23bca6f6
$ git baobab 7b5565418f4 --cpp -exclude "/(test|unittest|unittests)/"
Commits 7b5565418f4d..2cf681a11aea
Filter matches 11896 tracked files
4879337 lines today, 707834 lines changed
Export chart to /var/folders/2k/myk8kt8d4f52dzr19331wtxr0000gn/T/tmp1knfujcn.html
Show in browser? [Y/n] y
```

Here are two more examples from the ongoing release.

<table>
  <tr><td style="padding:20px;">
    <a href="https://weliveindetail.github.io/git-baobab/examples/llvm9-cmake.html">
      <img alt="Changes in CMake files" src="https://weliveindetail.github.io/git-baobab/examples/llvm9-cmake.png">
    </a>
  </td><td style="padding:20px;">
    <a href="https://weliveindetail.github.io/git-baobab/examples/llvm9-cpp-executionengine.html">
      <img alt="Changes in ExecutionEngine C++ files" src="https://weliveindetail.github.io/git-baobab/examples/llvm9-cpp-executionengine.png">
    </a>
  </td></tr>
</table>

The left diagram uses the `--cmake` flag, which includes `CMakeLists.txt` files as well as files with `.cmake` or `.in` extensions. Speciically for people working with the JIT components of LLVM, the right diagram shows what changed in ExecutionEngine C++ files.

