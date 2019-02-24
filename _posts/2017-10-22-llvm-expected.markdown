---
layout: post
author: Stefan Gränitz
title:  "Rich Recoverable Error Handling with llvm::Expected<T>"
date:   2017-10-22 17:43:01 +0200
categories: post
comments: true
---

There are good reasons for and against the use of C++ Exceptions. The lack of good alternatives, however, is often considered a strong argument _for_ them. Exception-free codebases just too easily retrogress to archaic error code passing. If your project doesn't go well with Exceptions, it can be a terrible trade-off.

This post is the first in a series presenting the rich error handling implementation introduced to the LLVM libraries recently. In order to make it usable for third parties, I provide a stripped-down version:
[https://github.com/weliveindetail/llvm-expected](https://github.com/weliveindetail/llvm-expected).

### Example llvm::Expected&lt;T&gt;

In a first example we will use the `llvm::GlobPattern` class for a search query with wildcards and make it fail. Handling this failure with `llvm::Expected<T>` and dumping the error info to `stderr`, looks like this:

```cpp
bool simpleExample() {
  std::string fileName = "[a*.txt";
  Expected<GlobPattern> pattern = GlobPattern::create(std::move(fileName));

  if (auto err = pattern.takeError()) {
    logAllUnhandledErrors(std::move(err), std::cerr, "[Glob Error] ");
    return false;
  }

  return pattern->match("...");
}

int main() {
  if (simpleExample())
    // success! more code here
  return 0;
}
```

In success case `pattern` holds a `llvm::GlobPattern` object, so `takeError()` returns `llvm::Error::success()` which evaluates to `false` and execution continues with the invocation of `GlobPattern::match()`.

Our example, however, provokes the error case: `[a*.txt` is no valid pattern and causes an internal error. Hence `takeError()` returns an `llvm::Error` object, which evaluats to `true` and execution enters the `if` branch. Before we return `false` from here, `llvm::logAllUnhandledErrors()` will give us an okish error message:

```output
[Glob Error] invalid glob pattern: [a*.txt
```

### Example std::error_code

Doing the same with error codes looks like this:

```cpp
bool simpleExample() {
  GlobPattern pattern;
  std::string fileName = "[a*.txt"

  if (std::error_code ec = GlobPattern::create(fileName, pattern)) {
    std::cerr << "[Glob Error] " << getErrorDescription(ec) << ": ";
    std::cerr << fileName << "\n";
    return false;
  }

  return pattern.match("...");
}

int main() {
  if (simpleExample())
    // success! more code here
  return 0;
}
```

As `GlobPattern::create` now returns a `std::error_code`, we obtain the resulting `pattern` through an out parameter. Note that this choice has a background: we don't want to use the out parameter for the error code as we would need to `clear()` it in success case explicitly to make sure
* it has no uninitialized memory and
* it does not accidentally carry a previously assigned error code.

Also note that we have no choice but to pass `pattern` in and out by reference. We cannot do "better" and use a reference to pointer to `GlobPattern` here, as it required a heap allocation in success case, which is far too expensive. We're forced to create and default-initialize `pattern` before the call.

As a last side effect, we cannot pass `fileName` by move anymore, as it may be used for the error dump:

```output
[Glob Error] invalid_argument: [a*.txt
```

### Motivation

So what do we gain with `llvm::Expected<T>`? A little readablility? A move instead of a const-ref or copy? Save one stack allocation plus initialization? No that's just the sugar. What we really gain is flexibility! We'll see that in the following example.

Let's imagine our `simpleExample` function becomes prominent and other people want to use it too. So we decide to move it to a library. Can we dump to `stderr` straight away in our library? Well, we could provide an extra argument to pass in an arbitrary stream to receive the error message, but maybe the user of the library has an entirely different approach for error handling. Quite likely we'd end up with something like this for error codes:

```cpp
std::error_code simpleExample(bool &result,
                              std::unique_ptr<std::string> &errorFileName) {
  GlobPattern pattern;
  std::string fileName = "[a*.txt";

  if (std::error_code ec = GlobPattern::create(fileName, pattern)) {
    errorFileName = std::make_unique<std::string>(fileName);
    return ec;
  }

  result = pattern.match("...");
  return std::error_code();
}

int main() {
  bool res;
  std::unique_ptr<std::string> errorFileName = nullptr; // heap alloc in error case

  if (std::error_code ec = simpleExample(res, errorFileName)) {
    std::cerr << "[simpleExample Error] " << getErrorDescription(ec) << " ";
    std::cerr << *errorFileName << "\n";
    return 0;
  }

  // success! more code here
  return 0;
}
```

Moving the error handling to `main`, we have to change the signature of `simpleExample` to return the error code and add out parameters for both, the actual result and the potential error details. Variables for out parameters have to be created and initialized apriori, which again adds unnecessary overhead (actually we never need both).

Well at least we are a little smart here: we define `errorFileName` as `std::unique_ptr` to `std::string` and pass this by reference. The interface gets slightly more complicated, but for C++ it's a manual optimization that's perfectly reasonable. In success case, we now pay only for the initialization of the pointer and not the string itself! It's a common pattern to optimize the success path at the expense of the error path. In case of errors performance is not our concern and we don't care about an extra heap allocation. Friendly reminder: never use error handling for regular control flow!

All in all, shifting the error handling by one level in the call stack, is a significant change to our code and function signatures. Readability suffers, unit tests need to be changed and we need to look very precisely in order to keep the best possible performance and to avoid introducing new edge cases! It's a bunch of details to consider for a rather primitive change.

**The underlying problem here is the limitation of error codes: They communicate enumerable errors very well, but cannot carry extra information.** In my experience people do hesitate to make the effort and return error details. In the majority of cases you get a magic number that can be looked up in some table and figuring out the details is your task for the rest of the day. But things can get worse with error codes, when people start dropping errors "for now" and handling them "later".

**Well, we all know what "later" means and that's the next problem with error codes: There is no mechanism to make sure they are actually handled**. In case you use bools, ints and nullptrs to indicate error situations, establishing a consistent use of `std::error_code` (or any other enumeration) throughout your codebase is a good first step, because it gives you a way to search for your failpoints! Nevertheless it still involves manual inspection of each and every occurrance, which makes it harder than necessary to estimate the robustness your codebase.

Using `llvm::Expected<T>` makes the task surprisingly simple. The only change on the function signature is the return type, from `bool` to `llvm::Expected<bool>`. In `simpleExample` we just forward errors and otherwise call the `match()` member function through an indirection:

```cpp
Expected<bool> simpleExample() {
  std::string fileName = "[a*.txt";
  Expected<GlobPattern> pattern = GlobPattern::create(std::move(fileName));
  if (!pattern)
    return pattern.takeError();

  return pattern->match("...");
}

int main() {
  Expected<bool> res = simpleExample();
  if (auto err = res.takeError()) {
    logAllUnhandledErrors(std::move(err), errs(), "[simpleExample Error] ");
    return 0;
  }

  // success! more code here
  return 0;
}
```

That's it! Compared to the changes in the error codes version, this was trivial! No impact on readability. Only 2 lines of new code. Minimal changes on the function signature, so unit test fixes should be fairly easy.

Most importantly though for C++ programmers: we keep the best possible performance without any smartness! No new edge cases! Yey!

### Entities

So going that way, what entities are we supposed to deal with? Instead of a primitive code, we want to hand around a user-defined structure that carries arbitrary information. For the sake of simplicity and other benefits we will see later, it's a class derived from `llvm::ErrorInfoBase`. The [LLVM Programmers Manual](http://llvm.org/docs/ProgrammersManual.html) gives a good example:

```cpp
class BadFileFormat : public ErrorInfo<BadFileFormat> {
public:
  static char ID;
  std::string Path;

  BadFileFormat(StringRef Path) : Path(Path.str()) {}

  void log(raw_ostream &OS) const override {
    OS << Path << " is malformed";
  }

  std::error_code convertToErrorCode() const override {
    return make_error_code(object_error::parse_failed);
  }
};

char BadFileFormat::ID; // This should be declared in the C++ file.
```

As we don't want to pass around polymorphic objects directly, the library gives us two lightweight wrappers:

* `llvm::Error` for functions that otherwise return `void`
* `llvm::Expected<T>` for functions that otherwise return `T`

These wrappers type-erase all error details (just like `std::function` type-erases the details of a function instance). Accessing these information will be cumbersome and also less performant. That's ok, it only happens in error cases. Additionally these wrappers implement a very natural behavior for errors:

* No duplicates: Similarly to `std::unique_ptr` they don't permit copy but only move.
* No lost instances: In debug mode, they make sure to be checked for failtures, before they are destroyed or values are accessed.

If we hadn't checked `pattern` for errors in line 4 of the example in the Motivation section, we would get the following message (always, not only in error cases):

```output
Expected<T> must be checked before access or destruction.
Unchecked Expected<T> contained error:
invalid glob pattern: [a*.txt
```


### Alexandrescu's Expected&lt;T&gt;

[This well-known proposal](https://onedrive.live.com/?cid=F1B8FF18A2AEC5C5&id=F1B8FF18A2AEC5C5%211158&parId=root&o=OneUp), for which you can [find an implementation here](https://github.com/martinmoene/spike-expected/tree/master/alexandrescu), is a close relative to `llvm::Expected<T>`. The major difference is the interoperability with exceptions, which makes it a bad choice for the exception-free codebase. However, it gives the opportunity to make it simple and store the error payload as [`std::exception_ptr`](http://en.cppreference.com/w/cpp/error/exception_ptr). While Alexandrescu added support for `Expected<void>` explicitly, we would return `llvm::Error` in this case.

### Boost Outcome

[Boost Outcome](https://ned14.github.io/outcome/) looks like the current candidate for a future [`std::expected`](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/p0323r2.pdf). Besides also supporting interoperability with exceptions it seems very similar to `llvm::Expected<T>` at first appearance. But there is one fundamental difference: Boost Outcome's [`outcome::result<T,EC>`](https://ned14.github.io/outcome/tutorial/result/) does not type-erase its payload. Instead error types are exposed as template parameters. Naturally, the next idea is to represent multiple possible error types with variadic templates: `expected<Y, E1, ..., En>` — that's where I get nervous.

Using error types in function signatures may not seem like a big deal, but considering the real size of codebases as well as the impact on API versioning, it could make you skeptic. For me this looks like a renewal of exception specifications, just without the really stupid parts. I recommend reading [The Trouble with Checked Exceptions](http://www.artima.com/intv/handcuffsP.html).

Well, if that's the only issue then let's write a type-erasing wrapper for Boost Outcome! In a first naive attempt, I tried to use `llvm::ErrorInfoBase` for the payload type in `outcome::result`:

```cpp
template <class T>
class expected {
  ...
  outcome::result<T, llvm::ErrorInfoBase> wrappee;
};
```

The experiment ended abruptly. Apparently abstract base classes are not what Boost Outcome considers a valid payload:
```output
outcome/detail/result_storage.hpp:162:5: error: static_assert failed "The type S must be void or default constructible"
```

Using a default constructible base class works, but there is still [some confusing discussions online](https://www.reddit.com/r/cpp/comments/5qzxdy/reasons_why_expectedt_e_should_always_use/) that suggest not to use Boost Outcome with arbitrary polymorphic types. Anyway, even if you manage to write a type-erased wrapper you still need to implement all the tooling around it youself.


### Less Generalization, More Common Ground

LLVM's rich error handling offers straightforward and pragmatic implementations for its wrappers `llvm::Error` and `llvm::Expected<T>` and at the same time it gets away with without error types in their signatures. The key to simplicity is a common base class for the error payload.

Substantiating this focal detail, rather then generalizing it, allows the library to provide seasoned tooling around its basic entities. I consider that a compelling benefit, because we don't write code for the sake of error handling. I think _simple_ and _compact_ are the most appreciated properties in this problem domain.

That's where we need library support. First of all we have a number of useful special-case implementations for errors:

* `llvm::ErrorList` is handy in situations where one error leads to another or separate operations fail independently
* `llvm::StringError` represents general-purpose errors that can fit all information into a string and have no distinct recovery strategy
* `llvm::ECError` stores a simple `std::error_code` and does its part for interoperability

Library functions now benefit from their knowledge about the existance of these special cases. The goal is to tune the interactions between frequently used entities to simplify the user's code. That's what makes a well-coordinated library.


### ErrorList

The following code shows the synergy of `llvm::ErrorList` with `llvm::handleAllErrors()`, a library function which acts like a `catch` clause selector for errors.

```cpp
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
```

The behavior of `llvm::handleAllErrors()` is obvious for regular errors: compare the type of the given error to the argument type of each handler top-down and invoke the first match. That's great and intuitive behavior, but it's not really what we want in case of `llvm::ErrorList`.

Without special handling, the above example would invoke the first handler and we had to use another `llvm::handleAllErrors()` inside to reach the actual errors. Too much code for error handling and most likely no practical use case will require this kind of behavior. As `llvm::handleAllErrors()` knows about the `llvm::ErrorList` special-case, it does the decomposition for us and dispatches the internal errors' payloads directly. For nested error lists, this results in a depth-first traversal.

Effect: simple and compact code plus no need to prepare the error handling code for multiple failures.


### Error Type Hierarchies

When everyone can define their own error types, it may get messy at some point, even if the visibility of definitions is well restricted. Type Hierarchies can help to establish a structure and they are — again — pretty simple.

```cpp
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
```

This code defines an error base class `IOError` and two specializations `FileNotFoundError` and `FileAccessDeniedError`. `openFile()` returns a `IOError`. Within `llvm::handleAllErrors` we can now filter precisely for with types we need distinct handling in this case. In the example that's the case for `FileNotFoundError`, while `FileAccessDeniedError` and all other `IOError`s end up in the second handler.

Like all type structures, error type hierarchies are powerful when designed right. Involved class names should in some way express their categories to clarify type-dependent control flow. Also note that `llvm::handleAllErrors()` doesn't currently invoke the handler that matches *best*, but instead the first one that matches *somehow*. If you swap the handlers in the example, all errors will go to the `IOError` handler!


### Conclusion

It's always great to use a library and get handy features that work out of the box. The reason LLVM's rich error handling can provide such features easily, is that it hardcodes a crucial detail of the system: a common base class for all user-defined error types.

Writing your own wrappers and tooling for Boost Outcome is not straightforward. For an error handling approach I consider this a downside. I'd rather go for the simplest possible solution and write compact code that is intuitively understandable to the reader. This is where robustness originates.


### References

* LLVM error handling docs<br> [http://llvm.org/docs/ProgrammersManual.html#error-handling](http://llvm.org/docs/ProgrammersManual.html#error-handling)

* `llvm::Expected<T>` as a C++17 Header-Only Library<br> [https://github.com/weliveindetail/llvm-expected](https://github.com/weliveindetail/llvm-expected)
