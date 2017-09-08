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

[This well-known proposal](https://onedrive.live.com/?cid=F1B8FF18A2AEC5C5&id=F1B8FF18A2AEC5C5%211158&parId=root&o=OneUp), for which you can [find an implementation here](https://github.com/martinmoene/spike-expected/tree/master/alexandrescu), is probably the closest relative to `llvm::Expected<T>` with the major difference that it stores errors of type `std::exception_ptr`. This is great if you need interoperability with exceptions. For the exception-free codebase, however, it's suboptimal. It won't compile with `-fno-exceptions` as it uses `throw` internally. It may also pull in unnecessary trouble as exceptions are implementation-dependent in many regards — citing the [C++11 Standard](http://en.cppreference.com/w/cpp/error/exception_ptr):

{% highlight cpp %}
typedef /*unspecified*/ exception_ptr;
{% endhighlight %}

Instead of exceptions `llvm::Expected<T>` interoperates very well with error codes. Additionally Alexandrescu added a version that supports `Expected<void>`. This is not supported by `llvm::Expected<T>`, instead one would return a `llvm::Error` in this case, which can either be an error instance or `llvm::ErrorSuccess`.

### Other implementations

Other version like [expected-lite](https://github.com/martinmoene/expected-lite), [Boost.Outcome](https://ned14.github.io/outcome/) and [std::experimental::expected](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/p0323r2.pdf) all go one step further. They propose the error type to be a template parameter `expected<T,E>`. Especially the idea to represent multiple possible errors as "variadic" `expected<Y, E1, ..., En>` via the type system causes me headache. Do I really want to adjust all my upstream function signatures, just because there's a new feature somewhere downstream that introduces a new source of errors? I think this goes in a wrong direction just as static exception specifications did. Error types is not something I want to check statically with the type system.

Another reason not to use freely templated error types is that it limits the common ground helper routines can be built on. For illustration let's have a look at the following [example from the LLVM documentation](https://llvm.org/docs/ProgrammersManual.html#recoverable-errors):

{% highlight cpp %}
auto result = processFormattedFile(...);

llvm::handleAllErrors(
  result.takeError(),
  [](const llvm::BadFileFormat &BFF) {
    report("Unable to process " + BFF.Path + ": bad format");
  },
  [](const llvm::FileNotFound &FNF) {
    report("File not found " + FNF.Path);
  });
{% endhighlight %}

The `handleAllErrors` function takes an error as its first argument, followed by a variadic list of “handlers”, each of which must be a callable type (a function, lambda, or class with a call operator) with one argument. The `handleAllErrors` function will visit each handler in the sequence and check its argument type against the dynamic type of the error, running the first handler that matches. This is the same decision process that is used to decide which catch clause to run for a C++ exception.

Although we want to have explicit code paths for errors, they should not dominate our code and destroy readability of what actually matters. That's why a good toolset around our error handling primitives is important. 
The tooling around `llvm::Expected<T>` can be used easily and they work without runtime type information, exactly because there is a common base class for all errors. It keeps things simple. Part 3 of this series will investigate these tools in more detail.

<a style="float: left;" href="/blog/post/2017/09/06/llvm-expected-basics.html">&lt; Motivation</a>
<br>
