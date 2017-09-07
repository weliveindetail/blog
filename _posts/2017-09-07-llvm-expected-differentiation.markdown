---
layout: post
author: Stefan Gränitz
title:  "Rich Polymorphic Error Handling with llvm::Expected<T> — Part 2"
date:   2017-09-07 17:43:01 +0200
categories: post
comments: true
series: expected
--- 

There are good reasons for and against the use of C++ Exceptions. The lack of good alternatives, however, is often considered a strong argument FOR them. Exception-free codebases just too easily retrogress to archaic error code passing. If your project doesn't go well with exceptions, it can be a terrible trade-off. This is the second post in a series that presents a solution recently introduced to the LLVM libraries. In order to make it usable for third parties, I provide a stripped-down version:
[https://github.com/weliveindetail/llvm-expected](https://github.com/weliveindetail/llvm-expected).

### Alexandrescu's proposed Expected&lt;T&gt;

[This well-known proposal](https://onedrive.live.com/?cid=F1B8FF18A2AEC5C5&id=F1B8FF18A2AEC5C5%211158&parId=root&o=OneUp), for which you can [find an implementation here](https://github.com/martinmoene/spike-expected/tree/master/alexandrescu), is probably the closest relative to `llvm::Expected<T>` with the major difference that it holds errors of type `std::exception_ptr`. This is great if you need interoperability with exceptions. For the exception-free codebase, however, it may pull in unnecessary trouble as exceptions are implementation-dependent in many regards — citing the [C++11 Standard](http://en.cppreference.com/w/cpp/error/exception_ptr):

{% highlight cpp %}
typedef /*unspecified*/ exception_ptr;
{% endhighlight %}

Additionally Alexandrescu added a version that supports `Expected<void>`. This is not supported by `llvm::Expected<T>`, instead one would return a `llvm::Error` in this case, which can either be an error instance or `llvm::ErrorSuccess`.

### Other implementations

Other version like [expected-lite](https://github.com/martinmoene/expected-lite), [Boost.Outcome](https://ned14.github.io/outcome/) and [std::experimental::expected](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/p0323r2.pdf) go one step further. They propose the error type to be a template parameter: `expected<T,E>`. When it comes to the generality-requirements of the standard library this makes total sense, but it also adds complexity and limits the common ground helper routines can be built on. For illustration let's have a look at the following [example from the documentation](https://llvm.org/docs/ProgrammersManual.html#recoverable-errors):

{% highlight cpp %}
auto result = processFormattedFile(...);

handleAllErrors(
  result.takeError(),
  [](const BadFileFormat &BFF) {
    report("Unable to process " + BFF.Path + ": bad format");
  },
  [](const FileNotFound &FNF) {
    report("File not found " + FNF.Path);
  });
{% endhighlight %}

The `handleAllErrors` function takes an error as its first argument, followed by a variadic list of “handlers”, each of which must be a callable type (a function, lambda, or class with a call operator) with one argument. The `handleAllErrors` function will visit each handler in the sequence and check its argument type against the dynamic type of the error, running the first handler that matches. This is the same decision process that is used to decide which catch clause to run for a C++ exception.

This is possible so easily and without runtime type information, because there is a common base class for all errors. Although we want to have explicit code paths for errors, they should not dominate our code and destroy readability of what actually matters. That's why a good toolset around our error handling primitives is important. Part 3 of this series will investigate the tools that make life easier when working with `llvm::Expected<T>`.

<a style="float: left;" href="/blog/post/2017/09/06/llvm-expected-basics.html">&lt; Motivation</a>
<br>
