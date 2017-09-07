---
layout: post
author: Stefan Gränitz
title:  "LLVM 5.0 Release ORC API Changes"
date:   2017-08-23 15:08:01 +0200
categories: post
comments: true
--- 
This week's 5.0 release of the LLVM libraries is a good opportunity to see what changed on the ORC JIT API. This is a summary of the most important things I came across when porting the [JitFromScratch](https://github.com/weliveindetail/JitFromScratch) examples.

Please note that the `release_40` compatible branches are still available and just moved into the `llvm40/` nesting level. So you can diff for a specific 4.0 to 5.0 change set quite easily:
<pre>
$ git diff llvm40/jit-basics jit-basics
$ git diff llvm40/jit-basics jit-basics -- CMakeLists.txt
$ git diff llvm40/jit-debug/gdb-interface jit-debug/gdb-interface -- SimpleOrcJit.h
</pre>

### OrcLayerConcept::addModule

The most anticipated news for me is the simplification of the ORC layer API from module-sets to single modules. Please don't misunderstand, I liked the idea to commit a set of modules at once in order to achieve cross-module optimization! However, in practice I never saw it used anywhere and so it always seemed overcomplicate the API without actual use.

Now simple changes like `ModuleSetHandleT` becoming `ModuleHandleT` made a lot of things easier. In order to still make use of cross-module optimizations, you can link the your modules into one using [`llvm::Linker::linkModules`](http://llvm.org/doxygen/classllvm_1_1Linker.html#a72e11e8404db974fa400748b888ea49d) before the commit.

### Errorization

[Errorization](https://github.com/llvm-mirror/llvm/commit/a81793582b3c47869680d354a97d59c55779c349) is another interesting thing that happened not only on this API entry point, but [in a number of places](https://github.com/llvm-mirror/llvm/commit/c6bf9be16da829a7292b1aa7307c4f162b4c6f72). It incorporates the new [structured error handling](https://llvm.org/docs/ProgrammersManual.html#error-handling) techniques aiming to improve consistency when dealing with error cases.

{% highlight diff %}
$ git diff release_40 release_50 -- ./include/llvm/ExecutionEngine/Orc/IRCompileLayer.h
diff --git a/include/llvm/ExecutionEngine/Orc/IRCompileLayer.h b/include/llvm/ExecutionEngine/Orc/IRCompileLayer.h
index f16dd021ea5..fadd334bed0 100644
--- a/include/llvm/ExecutionEngine/Orc/IRCompileLayer.h
+++ b/include/llvm/ExecutionEngine/Orc/IRCompileLayer.h
   ...
-  /// @brief Compile each module in the given module set, then add the resulting
-  ///        set of objects to the base layer along with the memory manager and
-  ///        symbol resolver.
+  /// @brief Compile the module, and add the resulting object to the base layer
+  ///        along with the given memory manager and symbol resolver.
   ///
-  /// @return A handle for the added modules.
-  template <typename ModuleSetT, typename MemoryManagerPtrT,
-            typename SymbolResolverPtrT>
-  ModuleSetHandleT addModuleSet(ModuleSetT Ms,
-                                MemoryManagerPtrT MemMgr,
-                                SymbolResolverPtrT Resolver) {
   ...
-    ModuleSetHandleT H =
-      BaseLayer.addObjectSet(std::move(Objects), std::move(MemMgr),
-                             std::move(Resolver));
-
-    return H;
+  /// @return A handle for the added module.
+  Expected<ModuleHandleT>
+  addModule(std::shared_ptr<Module> M,
+            std::shared_ptr<JITSymbolResolver> Resolver) {
+    using CompileResult = decltype(Compile(*M));
+    auto Obj = std::make_shared<CompileResult>(Compile(*M));
+    return BaseLayer.addObject(std::move(Obj), std::move(Resolver));
   }
{% endhighlight %}

### Templates

The reduction of template code is another step towards simplification that I appreciate. As you can see in the new signature of the `addModule` function above, the `SymbolResolverPtrT` template parameter was replaced with a smart pointer to base class `JITSymbolResolver`. As symbol resolvers are commonly shared between modules, it uses a shared pointer. The overhead of additional write operations to memory (update reference count) is acceptable all over the ORC API — well the whole JIT compilation process is pretty expensive anyway.

One more example for this is the `RTDyldObjectLinkingLayer` (that btw. was renamed). It replaces template parameters for functors with `typedef`s to `std::function`. On the caller's side it actually makes no difference. While the template parameter was auto deduced in `release_40`, the same expression is now accepted as a regular typed function parameter:

{% highlight c++ %}
template <typename NotifyLoadedFtor = DoNothingOnNotifyLoaded>
class ObjectLinkingLayer : public ObjectLinkingLayerBase {
  ...
  ObjectLinkingLayer(
      NotifyLoadedFtor NotifyLoaded = NotifyLoadedFtor(),
      NotifyFinalizedFtor NotifyFinalized = NotifyFinalizedFtor())
{% endhighlight %}

This code turned into:

{% highlight c++ %}
class RTDyldObjectLinkingLayer : public RTDyldObjectLinkingLayerBase {
  ...
  /// @brief Functor for receiving object-loaded notifications.
  using NotifyLoadedFtor = std::function<void(ObjHandleT, const ObjectPtr &Obj,
                                              const LoadedObjectInfo &)>;
  ...
  RTDyldObjectLinkingLayer(
      MemoryManagerGetter GetMemMgr,
      NotifyLoadedFtor NotifyLoaded = NotifyLoadedFtor(),
      NotifyFinalizedFtor NotifyFinalized = NotifyFinalizedFtor())
{% endhighlight %}

### Component Dependencies

When building the JitFromScratch examples against `release_50` I was a little surprised about linker errors reporting undefined symbols. I hadn't really changed anything, so why was this happening? So far the only explaination for me is that some component dependencies were sorted out. In order to build `jit-basics` we now need to specify `executionengine`, `object`, `runtimedyld` and `support` explicitly. Previously their libraries were linked in automatically by using only `core`, `native` and `orcjit`.

{% highlight diff %}
diff --git a/CMakeLists.txt b/CMakeLists.txt
index 5ab1a01..2fbca43 100644
--- a/CMakeLists.txt
+++ b/CMakeLists.txt
...
 # JitFromScratch dependencies
 llvm_map_components_to_libnames(LLVM_LIBS
     core
+    executionengine
+    native
+    object
     orcjit
-    x86asmparser
-    x86codegen
+    runtimedyld
+    support
 )
{% endhighlight %}

Also using the ORC JIT API? Anything to add? I am happy to hear your opinion and answer questions!
