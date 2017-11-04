---
layout: post
author: Stefan Gränitz
title:  "Rich Recoverable Error Handling with llvm::Expected<T> — Part 2"
date:   2017-10-22 17:43:01 +0200
categories: post
comments: true
series: expected
---

There are good reasons for and against the use of C++ Exceptions. The lack of good alternatives, however, is often considered a strong argument _for_ them. Exception-free codebases just too easily retrogress to archaic error code passing. If your project doesn't go well with exceptions, it can be a terrible trade-off.

This post is the second in a series presenting the rich error handling implementation introduced to the LLVM libraries recently. In order to make it usable for third parties, I provide a stripped-down version:
[https://github.com/weliveindetail/llvm-expected](https://github.com/weliveindetail/llvm-expected).

### Alexandrescu's Expected&lt;T&gt;

[This well-known proposal](https://onedrive.live.com/?cid=F1B8FF18A2AEC5C5&id=F1B8FF18A2AEC5C5%211158&parId=root&o=OneUp), for which you can [find an implementation here](https://github.com/martinmoene/spike-expected/tree/master/alexandrescu), is a close relative to `llvm::Expected<T>`. The major difference is the interoperability with exceptions, which makes it a bad choice for the exception-free codebase. However, it gives the opportunity to make it simple and store the error payload as [`std::exception_ptr`](http://en.cppreference.com/w/cpp/error/exception_ptr). While Alexandrescu added support for `Expected<void>` explicitly, we would return `llvm::Error` in this case.

### Boost Outcome

[Boost Outcome](https://ned14.github.io/outcome/) looks like the current candidate for a future [`std::expected`](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/p0323r2.pdf). Besides also supporting interoperability with exceptions it seems very similar to `llvm::Expected<T>` at first appearance. But there is one fundamental difference: Boost Outcome's [`outcome::result<T,EC>`](https://ned14.github.io/outcome/tutorial/result/) does not type-erase its payload. Instead error types are exposed as template parameters. Naturally, the next idea is to represent multiple possible error types with variadic templates: `expected<Y, E1, ..., En>` — that's where I get nervous.

Using error types in function signatures may not seem like a big deal, but considering the real size of codebases as well as the impact on API versioning, it could make you skeptic. For me this looks like a renewal of exception specifications, just without the really stupid parts.

Sure you could somehow manage to write your own type-erasing wrapper for Boost Outcome but it comes with a need for tooling. And once you finished reading this post, I think you don't want to write that on your own anymore.

Anyway, before rolling out Boost Outcome throughout your codebase, I would recommend reading [The Trouble with Checked Exceptions](http://www.artima.com/intv/handcuffsP.html).


### Less Generalization, More Common Ground

LLVM's rich error handling implementation defines a common base class for the error payload and gets away without error types in the signatures of its wrappers `llvm::Error` and `llvm::Expected<T>`.

Substantiating this focal detail, rather then generalizing it, allows the library to provide seasoned tooling around its basic entities. I consider that a compelling benefit, because we don't write code for the sake of error handling. I think _simple_ and _compact_ are the most appreciated properties in this problem domain. That's where we will need helper functions.

A common base class for error types makes it easy to provide special-case implementations:

* `llvm::ErrorList` is handy in situations where one error leads to another or separate operations fail independently
* `llvm::StringError` represents general-purpose errors that can fit all information into a string and have no distinct recovery strategy
* `llvm::ECError` stores a simple `std::error_code` and does its part for interoperability

These types can now be considered by other frequently used helper functions to simplify your code.


### ErrorList

The following code shows the synergy of `llvm::ErrorList` with `llvm::handleAllErrors()`, a library function which acts like a `catch` clause selector for errors.

{% highlight cpp %}
using namespace llvm;
std::error_code EC = std::make_error_code(std::errc::invalid_argument);

Error firstErr = errorCodeToError(EC);
Error aftereffect = make_error<StringError>("msg", inconvertibleErrorCode());

// payload of bar is ErrorList of ECError and StringError
Error bar = joinErrors(firstErr, aftereffect);

handleAllErrors(
  std::move(bar),
  [](const ErrorList &err) {
    // never called
  },
  [](const StringError &err) {
    // called second
  },
  [](const ECError &err) {
    // called first
  }
);
{% endhighlight %}

The behavior of `llvm::handleAllErrors()` is obvious for regular errors: compare the type of the given error to the argument type of each handler top-down and invoke the first match. That's great and intuitive behavior, but it's not really what we want in case of `llvm::ErrorList`.

Without special handling, the above example would invoke the first handler and we had to use another `llvm::handleAllErrors()` inside to reach the actual errors. Too much code for error handling and most likely no practical use case will require this kind of behavior. As `llvm::handleAllErrors()` knows about the `llvm::ErrorList` special-case, it does the decomposition for us and dispatches the internal errors' payloads directly. For nested error lists, this results in a depth-first traversal.

Effect: simple and compact code plus no need to prepare the error handling code for multiple failures.


### Error Type Hierarchies

When everyone can define their own error types, it may get messy at some point, even if the visibility of definitions is well restricted. Type Hierarchies can help to establish a structure and they are — again — pretty simple.

{% highlight cpp %}
// Type hierarchy:      IOError
//                     /       \
//      FileNotFoundError     AccessDeniedError
//
class IOError : public ErrorInfo<IOError> { ... };
class FileNotFoundError : public ErrorInfo<FileNotFoundError, IOError> { ... };
class FileAccessDeniedError : public ErrorInfo<FileAccessDeniedError, IOError> { ... };

auto file = openFile(...);

handleAllErrors(
  file.takeError(),
  [](const FileNotFoundError &err) {
    // special handling for IO error FileNotFoundError
  },
  [](const IOError &err) {
    // handling for all other IO errors
  });
{% endhighlight %}

This code defines an error base class `IOError` and two specializations `FileNotFoundError` and `FileAccessDeniedError`. `openFile()` returns a `IOError`. Within `llvm::handleAllErrors` we can now filter precisely for with types we need distinct handling in this case. In the example that's the case for `FileNotFoundError`, while `FileAccessDeniedError` and all other `IOError`s end up in the second handler.

Like all type structures, error type hierarchies are powerful when designed right. Involved class names should in some way express their categories to clarify type-dependent control flow. Also note that `llvm::handleAllErrors()` doesn't currently invoke the handler that matches *best*, but instead the first one that matches *somehow*. If you swap the handlers in the example, all errors will go to the `IOError` handler!


### Conclusion

It's always great to use a library and get handy features that work out of the box. The reason LLVM's rich error handling can provide such features (and implement them in a straightforward and pragmatic way), is that it hardcodes a crucial detail of the system: a common base class for all user-defined error types.

Writing your own wrappers the same kind of tooling for Boost Outcome is not straightforward. I assume it goes through template hell at least once :) For an error handling approach I consider this a downside. I'd rather go for the simplest possible solution and write compact code that is intuitively understandable to the reader. This is where robustness originates.

<a style="float: left;" href="{{ site.baseurl }}{% post_url 2017-09-06-llvm-expected-basics %}">&lt; Prev: Motivation</a>
<a style="float: right;" href="{{ site.baseurl }}{% post_url 2017-10-28-llvm-expected-helpers %}">Next: All Helpers &gt;</a>
<br>
