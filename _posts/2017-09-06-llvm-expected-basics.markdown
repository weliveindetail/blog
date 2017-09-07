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

There are good reasons for and against the use of C++ Exceptions. The lack of good alternatives, however, is often considered a strong argument FOR them. Exception-free codebases just too easily retrogress to archaic error code passing. If your project doesn't go well with Exceptions, it can be a terrible trade-off. This is the first in a series of posts about a solution recently introduced to the LLVM libraries. In order to make it usable for third parties, I provide a stripped-down version: 
[https://github.com/weliveindetail/llvm-expected](https://github.com/weliveindetail/llvm-expected).

### Short Example

The first example uses the `llvm::GlobPattern` class for a search query with wildcards, but the given pattern causes an internal error. This code uses `llvm::Expected<T>`, catches the error and dumps it to `stderr`:

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

Usually `pattern` held a `llvm::GlobPattern` object, so `takeError()` would return `llvm::Error::success()` which evaluated to `false`. In our case, however, it's not in success state but holds an actual error. So `takeError()` returns the object derived from `llvm::Error` and `llvm::logAllUnhandledErrors()` will give us an okish error message:

<pre>
[Glob Error] invalid glob pattern: [a*.txt
</pre>

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

As `GlobPattern::create` now returns the error code, we obtain the resulting `pattern` through an out parameter. We're forced to create and default-initialize it before the call. Also note that we cannot pass the `fileName` input anymore by move, as it's used to dump the output in case of errors. Otherwise the code is pretty much the same. As we worked hard for the error message it's even quite readable:

<pre>
[Glob Error] invalid_argument: [a*.txt
</pre>



### Motivation

So what do we gain with `llvm::Expected<T>`? A little readablility? A move instead of a const-ref or copy? Save one stack allocation plus initialization? No that's just the sugar. What we really gain is flexibility! We'll see that in the following example.

Let's imagine our `simpleExample` function becomes prominent and other people want to use it too. So we decide to move it to a library. Can we dump to `stderr` straight away in our library? Well, we could provide an extra argument to pass an arbitrary stream to dump the error message, but maybe the user of the library has an entirely different approach for error handling (thinking about Clang diagnostics for example). Quite likely we'd end up with something like this for error codes:

{% highlight cpp %}
std::error_code simpleExample(bool &result, std::string *&errorFileName) {
  GlobPattern pattern;
  std::string fileName = "[a*.txt";

  if (std::error_code ec = GlobPattern::create(fileName, pattern)) {
    errorFileName = new std::string(fileName);
    return ec;
  }

  result = pattern.match("...");
  return std::error_code();
}

int main() {
  bool res;
  std::string *errorFileName = nullptr; // heap alloc in error case

  if (std::error_code ec = simpleExample(res, errorFileName)) {
    std::cerr << "[simpleExample Error] " << getErrorDescription(ec) << " ";
    std::cerr << *errorFileName << "\n";
    delete errorFileName;
    return 0;
  }

  // ... more code ...
  return 0;
}
{% endhighlight %}

Moving the error handling to `main`, we have to change the signature of `simpleExample` to return the error code and add out parameters for both, the actual result and the potential error details. Variables for out parameters have to be created and initialized apriori, which adds unnecessary overhead (actually we never need both). The above code tries to be "clever" here and takes a reference to a pointer to `errorFileName`. With that it saves a few cycles in the non-error case. The price for that micro-optimization is a new dependency between the two functions: they share an asymmetric new-delete, which might be considered a code smell. All in all it's a significant change to our code and function signatures. Readability suffers, unit tests need to be changed and we need to look very precisely not to introduce new edge cases.

**The underlying problem here is the limitation of error codes: They communicate enumerable errors very well, but cannot carry extra information.** In many real-world cases people just won't make the effort to return error details or they even ignore the error entirely. There is no mechanism in error codes to make sure errors are actually handled. Hence it's hard to estimate the robustness of codebases that rely on error codes.

Using `llvm::Expected<T>` avoids almost all of these issues:

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

We simply moved the error handler one level up. The only change on the function signature is the return type, from `bool` to `llvm::Expected<bool>`. In `simpleExample` we just forward errors and otherwise call the `match()` member function through an indirection. Summarizing the benefits:

* **No unnecessary overhead**: this code out-performs even the optimized error codes version
* **No code bloat**: we only added 2 lines of code, while the error code example requires 8 extra lines
* **No risk to drop errors**: if we didn't check `pattern` for errors in line 4, it would abort with an error message to tell us about it (always, not only in error cases):

<pre>
Expected&lt;T&gt; must be checked before access or destruction.
Unchecked Expected&lt;T&gt; contained error:
invalid glob pattern: [a*.txt
</pre>

There's a lot more details to share about this way of error handling. I hope you are curious for Part 2?

<a href="/blog/post/2017/09/07/llvm-expected-differentiation.html" style="float: right;">Differentiation &gt;</a>
<br>