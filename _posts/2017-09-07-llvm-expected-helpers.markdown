---
layout: post
author: Stefan Gränitz
title:  "Rich Polymorphic Error Handling with llvm::Expected<T> — Part 3"
date:   2017-09-07 19:33:01 +0200
categories: post
comments: true
series: expected
--- 

There are good reasons for and against the use of C++ Exceptions. The lack of good alternatives, however, is often considered a strong argument _for_ them. Exception-free codebases just too easily retrogress to archaic error code passing. If your project doesn't go well with Exceptions, it can be a terrible trade-off.

This post is the third in a series presenting the rich error handling implementation introduced to the LLVM libraries recently. In order to make it usable for third parties, I provide a stripped-down version:
[https://github.com/weliveindetail/llvm-expected](https://github.com/weliveindetail/llvm-expected).

This is a summary of all things that come with the library ready to use!


### llvm::handleErrors()

Similarly to the `llvm::handleAllErrors` function we saw earlier, `llvm::handleErrors` makes it easy to handle errors by type. The difference is that catching the error is not mandatory here. If the type does not match any of the handlers in the list, the error is returned. Idiomatic use looks like this:

{% highlight cpp %}
auto result = processFormattedFile(...);

if (auto err = llvm::handleErrors(
      result.takeError(),
      [](const llvm::BadFileFormat &BFF) {
        report("Unable to process " + BFF.Path + ": bad format");
      },
      [](const llvm::FileNotFound &FNF) {
        report("File not found " + FNF.Path);
      }))
  return err;
{% endhighlight %}


### llvm::StringError

The go-to type for general-purpose errors that can fit all information into a string and have no distinct recovery strategy.


### llvm::ErrorList and llvm::joinErrors()

Joining two errors into one entity results in a `llvm::ErrorList`. Other library functions are prepared for it: `llvm::handleErrors()` will pass the inner errors to its handlers.


### llvm::ECError, errorCodeToError() and errorToErrorCode()

`llvm::ECError` has a primitive payload of type `std::error_code`. If you heard of [exploding return codes](https://groups.google.com/forum/#!msg/comp.lang.c++.moderated/BkZqPfoq3ys/H_PMR8Sat4oJ), it's pretty much this. Useful in situations where one part of the codebase uses rich error handling, while another one still works on `std::error_code`.

While `llvm::errorCodeToError()` returns `llvm::ECError`s exclusively, `llvm::errorToErrorCode()` can deal with arbitrary error types. This is because all classes derived from `llvm::ErrorInfoBase` must implement the `convertToErrorCode()` method, so per se they are all convertible to `std::error_code`.

{% highlight cpp %}
llvm::Error mayFail() {
  std::error_code ec = oldCode();
  return llvm::errorCodeToError(ec);
}
{% endhighlight %}


### llvm::inconvertibleErrorCode()

Unsurprisingly this returns an invalid error code. Useful for implementing the `convertToErrorCode()` method for classes that are not meant to be convertible to `std::error_code`.


### llvm::make_error<T>()

Instantiation helper, creating a `llvm::Error` with the specified payload.

{% highlight cpp %}
auto EC = std::make_error_code(std::errc::executable_format_error);
llvm::make_error<llvm::StringError>("Bad executable", EC);
{% endhighlight %}


### llvm::ExitOnError

Very useful for small command line tools that just exit properly with an error message.

{% highlight cpp %}
ExitOnError ExitOnErr;

void someFunction() {
  ...
  if (...)
    ExitOnErr(someFallibleFunction());
  ...
}

int main(int argc, char *argv[]) {
  ExitOnErr.setBanner(std::string(argv[0]) + " error:");
  ExitOnErr.setExitCodeMapper(
    [](const Error &Err) {
      if (Err.isA<BadFileFormat>())
        return 2;
      return 1;
    });

  ...
}
{% endhighlight %}


### llvm::cantFail()

Sound and compact helper for fallible functions that certainly won't fail despite returning a `llvm::Expected<T>`. Use with care, if you are wrong and it does fail the program terminates. A reasonable use case can be found e.g. in the [LLVM OrcJIT tutorials](https://github.com/llvm-mirror/llvm/blob/master/examples/Kaleidoscope/BuildingAJIT/Chapter1/KaleidoscopeJIT.h#L78):

{% highlight cpp %}
ModuleHandle addModule(std::unique_ptr<Module> M) {
  ...
  // Add the set to the JIT with the resolver we created above and a newly
  // created SectionMemoryManager.
  return cantFail(CompileLayer.addModule(std::move(M),
                                         std::move(Resolver)));
}
{% endhighlight %}

All OrcJIT layers implement the `addModule`/`removeModule` concept. These calls can fail in principle. However, the layers used in this specific case don't generate errors at all, so it's safe to use the helper here.

<a style="float: left;" href="/blog/post/2017/09/07/llvm-expected-differentiation.html">&lt; Differentiation</a>
<br>
