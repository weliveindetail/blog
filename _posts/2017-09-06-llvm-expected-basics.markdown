---
layout: post
author: Stefan Gränitz
title:  "Rich Polymorphic Error Handling with llvm::Expected<T> — Part 1"
date:   2017-09-06 18:33:01 +0200
categories: post
comments: true
series: expected
part: first
--- 

There are good reasons for and against the use of C++ Exceptions. The lack of good alternatives, however, is often considered a strong argument _for_ them. Exception-free codebases just too easily retrogress to archaic error code passing. If your project doesn't go well with Exceptions, it can be a terrible trade-off.

This is the first in a series of posts about a solution recently introduced to the LLVM libraries. In order to make it usable for third parties, I provide a stripped-down version: 
[https://github.com/weliveindetail/llvm-expected](https://github.com/weliveindetail/llvm-expected).

### Example llvm::Expected&lt;T&gt;

In a first example we will use the `llvm::GlobPattern` class for a search query with wildcards and make it fail. Handling this failure with `llvm::Expected<T>` and dumping the error info to `stderr`, looks like this:

{% highlight cpp %}
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
    // ... more code ...
  return 0;
}
{% endhighlight %}

In success case `pattern` holds a `llvm::GlobPattern` object, so `takeError()` returns `llvm::Error::success()` which evaluates to `false` and execution continues with the invocation of `GlobPattern::match()`. Our example, however, provokes the error case, because `[a*.txt` is no valid pattern. Instead it causes an internal error. Hence `takeError()` returns an `llvm::Error` object, which evaluats to `true` and (before execution escapes the function) `llvm::logAllUnhandledErrors()` will give us an okish error message:

<pre>
[Glob Error] invalid glob pattern: [a*.txt
</pre>

### Example std::error_code

Doing the same with error codes looks like this:

{% highlight cpp %}
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
    // ... more code ...
  return 0;
}
{% endhighlight %}

As `GlobPattern::create` now returns the error code, we obtain the resulting `pattern` through an out parameter. Note that this choice has a background: we don't want to use the out parameter for the error code as we would need to `clear()` it in success case explicitly to make sure
* it has no uninitialized memory and 
* it does not accidentally carry a previously assigned error code.

Also note that we have no choice than to pass `pattern` in and out by reference. We cannot do "better" and use a reference to pointer to `GlobPattern` here, as it required a heap allocation in success case, which is far too expensive. Hence we're forced to create and default-initialize `pattern` before the call. 

As a last side effect, we cannot pass `fileName` by move anymore, as it may be used for the error dump:

<pre>
[Glob Error] invalid_argument: [a*.txt
</pre>

### Motivation

So what do we gain with `llvm::Expected<T>`? A little readablility? A move instead of a const-ref or copy? Save one stack allocation plus initialization? No that's just the sugar. What we really gain is flexibility! We'll see that in the following example.

Let's imagine our `simpleExample` function becomes prominent and other people want to use it too. So we decide to move it to a library. Can we dump to `stderr` straight away in our library? Well, we could provide an extra argument to pass in an arbitrary stream to receive the error message, but maybe the user of the library has an entirely different approach for error handling. Quite likely we'd end up with something like this for error codes:

{% highlight cpp %}
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

  // ... more code ...
  return 0;
}
{% endhighlight %}

Moving the error handling to `main`, we have to change the signature of `simpleExample` to return the error code and add out parameters for both, the actual result and the potential error details. Variables for out parameters have to be created and initialized apriori, which adds unnecessary overhead (actually we never need both).

Now we have the option to be smart: we define `errorFileName` as `unique_ptr` to `std::string` and pass this by reference. Thus, in success case, we pay only for the initialization of the pointer and not the string itself! Instead we trade it for a heap allocation in error case. It makes the interface slightly more complicated, but for C++ it's a manual optimization that's perfectly reasonable.

All in all, shifting the error handling by one level in the call stack, is a significant change to our code and function signatures. Readability suffers, unit tests need to be changed and we need to look very precisely in order to keep the best possible performance and to avoid introducing new edge cases! It's a bunch of details to consider for a rather primitive change.

**The underlying problem here is the limitation of error codes: They communicate enumerable errors very well, but cannot carry extra information.** In my experience people do hesitate to make the effort and return error details. In the majority of cases you get a magic number that can be looked up in some table and figuring out the details is your task for the rest of the day. But things can get worse with error codes, when people start dropping errors "for now" and handling them "later".

**Well, we all know what "later" means and that's the next problem with error codes: There is no mechanism to make sure they are actually handled**. In case you use bools, ints and nullptrs to indicate error situations, establishing a consistent use of `std::error_code` (or any other enumeration) throughout your code base is a good first step, because it gives you a way to search for your failpoints! Nevertheless it still involves manual inspection of each and every occurrance, which makes it harder than necessary to estimate the robustness your codebase.

Using `llvm::Expected<T>` makes the task surprisingly simple. The only change on the function signature is the return type, from `bool` to `llvm::Expected<bool>`. In `simpleExample` we just forward errors and otherwise call the `match()` member function through an indirection:

{% highlight cpp %}
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

  // ... more code ...
  return 0;
}
{% endhighlight %}

Summarizing the benefits:

* **No manual optimization**: efficiency of the solution doesn't require "smartness"
* **No unnecessary overhead**: this code out-performs even the optimized error codes version
* **No code bloat**: we only added 2 lines of code, while the error code example requires 8 extra lines
* **No risk to drop errors**: destroying or accessing an unchecked instance will terminate the program in debug mode

### Entities

So going that way, what entities are we supposed to deal with? Instead of a primitive code, we want to hand around a user-defined structure that carries arbitrary information. For the sake of simplicity and other benefits we will see later, it's a class derived from `llvm::ErrorInfoBase`. The [LLVM Programmers Manual](http://llvm.org/docs/ProgrammersManual.html) gives a good example:

{% highlight cpp %}
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
{% endhighlight %}

As we don't want to pass around polymorphic objects directly, the library gives us two lightweight wrappers: 

* `llvm::Error` for functions that otherwise return `void`
* `llvm::Expected<T>` for functions that otherwise return `T`

These wrappers type-erase all error details (just like `std::function` type-erases the details of a function instance). Accessing these information will be cumbersome and also less performant, which is ok as it only happens in error cases. Additionally they implement a very natural behavior for errors:

* No duplicates: Similarly to `std::unique_ptr` they don't permit copy but only move.
* No lost instances: In debug mode, they make sure to be checked for failtures, before they are destroyed or values are accessed.

If we hadn't checked `pattern` for errors in line 4 of the example in the Motivation section, we would get the following message (always, not only in error cases):

<pre>
Expected&lt;T&gt; must be checked before access or destruction.
Unchecked Expected&lt;T&gt; contained error:
invalid glob pattern: [a*.txt
</pre>

There's a lot more details to share about this way of error handling. I hope you are curious for Part 2?

<a href="/blog/post/2017/09/07/llvm-expected-differentiation.html" style="float: right;">Differentiation &gt;</a>
<br>
