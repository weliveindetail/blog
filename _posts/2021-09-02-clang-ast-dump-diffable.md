---
layout: post
categories: post
author: Stefan Gränitz
date: 2021-09-03 18:00:00 +0200
image: https://weliveindetail.github.io/blog/res/clang-ast-dump-diffable.png
title: "Diffing Clang AST dumps"
description: "Searching for differences in two given dumps of the Clang AST is a little tricky. A short Python script helps."
---

!["Inspect lit.local.cfg in vscode"](https://weliveindetail.github.io/blog/res/clang-ast-dump-diffable.png)

Clang makes it easy to dump the AST, but searching for differences in two given AST dumps is a little tricky. [A short Python script](https://github.com/weliveindetail/astpp) can fix most of it. Let's have a look at an example.

Here is the AST for a function with a plain C++11 lambda:
```
➜ clang++ -fsyntax-only -fno-color-diagnostics -Xclang -ast-dump plain.cpp 1>plain.ast
➜ cat plain.ast
TranslationUnitDecl 0x7f8916040008 <<invalid sloc>> <invalid sloc>
`-FunctionDecl 0x7f891607bc28 <plain.cpp:1:1, line:4:1> line:1:5 main 'int (int, char **)'
  |-ParmVarDecl 0x7f891607b9c0 <col:10, col:14> col:14 used argc 'int'
  |-ParmVarDecl 0x7f891607bb08 <col:20, col:31> col:26 argv 'char **':'char **'
  `-CompoundStmt 0x7f89160a9ba8 <col:34, line:4:1>
    |-DeclStmt 0x7f89160a9a20 <line:2:3, col:46>
    | `-VarDecl 0x7f891607bd88 <col:3, col:45> col:8 used lambda '(lambda at plain.cpp:2:17)':'(lambda at plain.cpp:2:17)' cinit
    |   `-ExprWithCleanups 0x7f89160a9a08 <col:17, col:45> '(lambda at plain.cpp:2:17)':'(lambda at plain.cpp:2:17)'
    |     `-CXXConstructExpr 0x7f89160a99d8 <col:17, col:45> '(lambda at plain.cpp:2:17)':'(lambda at plain.cpp:2:17)' 'void ((lambda at plain.cpp:2:17) &&) noexcept' elidable
    |       `-MaterializeTemporaryExpr 0x7f89160a9970 <col:17, col:45> '(lambda at plain.cpp:2:17)' xvalue
    |         `-LambdaExpr 0x7f891607c4c8 <col:17, col:45> '(lambda at plain.cpp:2:17)'
    |           |-CXXRecordDecl 0x7f891607bed0 <col:17> col:17 implicit class definition
    |           | |-DefinitionData lambda pass_in_registers empty standard_layout trivially_copyable can_const_default_init
    |           | | |-DefaultConstructor defaulted_is_constexpr
    |           | | |-CopyConstructor simple trivial has_const_param implicit_has_const_param
    |           | | |-MoveConstructor exists simple trivial
    |           | | |-CopyAssignment trivial has_const_param needs_implicit implicit_has_const_param
    |           | | |-MoveAssignment
    |           | | `-Destructor simple irrelevant trivial
    |           | |-CXXMethodDecl 0x7f891607c018 <col:28, col:45> col:17 used operator() 'int (int) const' inline
    |           | | |-ParmVarDecl 0x7f891607be08 <col:20, col:24> col:24 used argc 'int'
    |           | | `-CompoundStmt 0x7f891607c118 <col:30, col:45>
    |           | |   `-ReturnStmt 0x7f891607c108 <col:32, col:39>
    |           | |     `-ImplicitCastExpr 0x7f891607c0f0 <col:39> 'int' <LValueToRValue>
    |           | |       `-DeclRefExpr 0x7f891607c0d0 <col:39> 'int' lvalue ParmVar 0x7f891607be08 'argc' 'int'
    |           | |-CXXConversionDecl 0x7f891607c358 <col:17, col:45> col:17 implicit operator int (*)(int) 'int (*() const noexcept)(int)' inline
    |           | |-CXXMethodDecl 0x7f891607c408 <col:17, col:45> col:17 implicit __invoke 'int (int)' static inline
    |           | | `-ParmVarDecl 0x7f891607c2f0 <col:20, col:24> col:24 argc 'int'
    |           | |-CXXDestructorDecl 0x7f891607c4f0 <col:17> col:17 implicit referenced ~ 'void () noexcept' inline default trivial
    |           | |-CXXConstructorDecl 0x7f89160a9600 <col:17> col:17 implicit constexpr  'void (const (lambda at plain.cpp:2:17) &)' inline default trivial noexcept-unevaluated 0x7f89160a9600
    |           | | `-ParmVarDecl 0x7f89160a9730 <col:17> col:17 'const (lambda at plain.cpp:2:17) &'
    |           | `-CXXConstructorDecl 0x7f89160a97d0 <col:17> col:17 implicit used constexpr  'void ((lambda at plain.cpp:2:17) &&) noexcept' inline default trivial
    |           |   |-ParmVarDecl 0x7f89160a9900 <col:17> col:17 '(lambda at plain.cpp:2:17) &&'
    |           |   `-CompoundStmt 0x7f89160a99c8 <col:17>
    |           `-CompoundStmt 0x7f891607c118 <col:30, col:45>
    |             `-ReturnStmt 0x7f891607c108 <col:32, col:39>
    |               `-ImplicitCastExpr 0x7f891607c0f0 <col:39> 'int' <LValueToRValue>
    |                 `-DeclRefExpr 0x7f891607c0d0 <col:39> 'int' lvalue ParmVar 0x7f891607be08 'argc' 'int'
    `-ReturnStmt 0x7f89160a9b98 <line:3:3, col:21>
      `-CXXOperatorCallExpr 0x7f89160a9b58 <col:10, col:21> 'int' '()'
        |-ImplicitCastExpr 0x7f89160a9ad8 <col:16, col:21> 'int (*)(int) const' <FunctionToPointerDecay>
        | `-DeclRefExpr 0x7f89160a9a78 <col:16, col:21> 'int (int) const' lvalue CXXMethod 0x7f891607c018 'operator()' 'int (int) const'
        |-ImplicitCastExpr 0x7f89160a9b28 <col:10> 'const (lambda at plain.cpp:2:17)' lvalue <NoOp>
        | `-DeclRefExpr 0x7f89160a9a38 <col:10> '(lambda at plain.cpp:2:17)':'(lambda at plain.cpp:2:17)' lvalue Var 0x7f891607bd88 'lambda' '(lambda at plain.cpp:2:17)':'(lambda at plain.cpp:2:17)'
        `-ImplicitCastExpr 0x7f89160a9b40 <col:17> 'int' <LValueToRValue>
          `-DeclRefExpr 0x7f89160a9a58 <col:17> 'int' lvalue ParmVar 0x7f891607b9c0 'argc' 'int'

```

And the corresponding source code:
```cpp
int main(int argc, char *argv[]) {
  auto lambda = [](int argc) { return argc; };
  return lambda(argc);
}
```

Here is an equivalent implementation that uses a [C++14 generic lambda](https://isocpp.org/wiki/faq/cpp14-language#generic-lambdas) instead:
```cpp
int main(int argc, char *argv[]) {
  auto lambda = [](auto argc) { return argc; };
  return lambda(argc);
}
```

Now let's try to fingure out what parts of the AST changed:
```
➜ clang++ -fsyntax-only -fno-color-diagnostics -Xclang -ast-dump -std=c++14 generic.cpp 1>generic.ast
➜ diff -u plain.ast generic.ast | wc -l
     101
```

Well, for now the diff doesn't help much:
```diff
--- plain.ast	2021-09-03 11:50 +0200
+++ generic.ast	2021-09-03 11:50 +0200
@@ -1,46 +1,58 @@
-TranslationUnitDecl 0x7f9492040008 <<invalid sloc>> <invalid sloc>
-`-FunctionDecl 0x7f949207bc28 <plain.cpp:1:1, line:4:1> line:1:5 main 'int (int, char **)'
-  |-ParmVarDecl 0x7f949207b9c0 <col:10, col:14> col:14 used argc 'int'
-  |-ParmVarDecl 0x7f949207bb08 <col:20, col:31> col:26 argv 'char **':'char **'
-  `-CompoundStmt 0x7f94920a9d58 <col:34, line:4:1>
-    |-DeclStmt 0x7f94920a9b90 <line:2:3, col:46>
-    | `-VarDecl 0x7f949207bd88 <col:3, col:45> col:8 used lambda '(lambda at plain.cpp:2:17)':'(lambda at plain.cpp:2:17)' cinit
-    |   `-ExprWithCleanups 0x7f94920a9b78 <col:17, col:45> '(lambda at plain.cpp:2:17)':'(lambda at plain.cpp:2:17)'
-    |     `-CXXConstructExpr 0x7f94920a9b48 <col:17, col:45> '(lambda at plain.cpp:2:17)':'(lambda at plain.cpp:2:17)' 'void ((lambda at plain.cpp:2:17) &&) noexcept' elidable
-    |       `-MaterializeTemporaryExpr 0x7f94920a9ae0 <col:17, col:45> '(lambda at plain.cpp:2:17)' xvalue
-    |         `-LambdaExpr 0x7f949207c6d8 <col:17, col:45> '(lambda at plain.cpp:2:17)'
-    |           |-CXXRecordDecl 0x7f949207bef8 <col:17> col:17 implicit class definition
-    |           | |-DefinitionData lambda pass_in_registers empty standard_layout trivially_copyable can_const_default_init
+TranslationUnitDecl 0x7f8fde840008 <<invalid sloc>> <invalid sloc>
+`-FunctionDecl 0x7f8fde87bc28 <generic.cpp:1:1, line:4:1> line:1:5 main 'int (int, char **)'
+  |-ParmVarDecl 0x7f8fde87b9c0 <col:10, col:14> col:14 used argc 'int'
+  |-ParmVarDecl 0x7f8fde87bb08 <col:20, col:31> col:26 argv 'char **':'char **'
+  `-CompoundStmt 0x7f8fde88c228 <col:34, line:4:1>
+    |-DeclStmt 0x7f8fde88bc10 <line:2:3, col:47>
+    | `-VarDecl 0x7f8fde87bd88 <col:3, col:46> col:8 used lambda '(lambda at generic.cpp:2:17)':'(lambda at generic.cpp:2:17)' cinit
+    |   `-ExprWithCleanups 0x7f8fde88bbf8 <col:17, col:46> '(lambda at generic.cpp:2:17)':'(lambda at generic.cpp:2:17)'
+    |     `-CXXConstructExpr 0x7f8fde88bbc8 <col:17, col:46> '(lambda at generic.cpp:2:17)':'(lambda at generic.cpp:2:17)' 'void ((lambda at generic.cpp:2:17) &&) noexcept' elidable
+    |       `-MaterializeTemporaryExpr 0x7f8fde88bb60 <col:17, col:46> '(lambda at generic.cpp:2:17)' xvalue
+    |         `-LambdaExpr 0x7f8fde88b578 <col:17, col:46> '(lambda at generic.cpp:2:17)'
+    |           |-CXXRecordDecl 0x7f8fde87c048 <col:17> col:17 implicit class definition
+    |           | |-DefinitionData generic lambda pass_in_registers empty standard_layout trivially_copyable can_const_default_init
     |           | | |-DefaultConstructor defaulted_is_constexpr
     |           | | |-CopyConstructor simple trivial has_const_param implicit_has_const_param
     |           | | |-MoveConstructor exists simple trivial
 ...
```

### Challenges

For Clang it's efficient to identify AST nodes by their memory address. Almost all lines in our diff represent a AST node and thus contain an address. That's why they never match: addresses are not deterministic. Some are referred to in subsequent lines, but most of them are not. We can drop the solo ones and replace the others with a simple incremental ID. This eliminates the major source of conflicts, but for now it only reduces our diff by 9 lines:
```
➜ astpp plain.ast --process node-ids > plain1.ast
➜ astpp generic.ast --process node-ids > generic1.ast
➜ diff -u plain1.ast generic1.ast | wc -l
      92
```

Another source of conflicts that we are not really interested in are source locations: varying file names as well as line- and column-numbers. We can drop them altogether and make sure they don't leave artifacts like brackets or whitespace. It cuts our diff by an additional 27 lines:
```
➜ astpp plain.ast --process node-ids source-locs > plain2.ast
➜ astpp generic.ast --process node-ids source-locs > generic2.ast
➜ diff -u plain2.ast generic2.ast | wc -l
      65
```

Running the script without the explicit `--process` parameter runs both steps and additionally trims trailing whitespace.

### Voilà!

The result gives some good insight what has changed in the AST:

```diff
--- plain.out.ast	2021-09-03 11:50 +0200
+++ generic.out.ast	2021-09-03 11:50 +0200
@@ -10,36 +10,48 @@
     |       `-MaterializeTemporaryExpr '(lambda at  )' xvalue
     |         `-LambdaExpr '(lambda at  )'
     |           |-CXXRecordDecl implicit class definition
-    |           | |-DefinitionData lambda pass_in_registers empty standard_layout trivially_copyable can_const_default_init
+    |           | |-DefinitionData generic lambda pass_in_registers empty standard_layout trivially_copyable can_const_default_init
     |           | | |-DefaultConstructor defaulted_is_constexpr
     |           | | |-CopyConstructor simple trivial has_const_param implicit_has_const_param
     |           | | |-MoveConstructor exists simple trivial
     |           | | |-CopyAssignment trivial has_const_param needs_implicit implicit_has_const_param
     |           | | |-MoveAssignment
     |           | | `-Destructor simple irrelevant trivial
-    |           | |-CXXMethodDecl ID0003 used operator() 'int (int) const' inline
-    |           | | |-ParmVarDecl ID0004 used argc 'int'
-    |           | | `-CompoundStmt ID0005
-    |           | |   `-ReturnStmt ID0006
-    |           | |     `-ImplicitCastExpr ID0007 'int' <LValueToRValue>
-    |           | |       `-DeclRefExpr ID0008 'int' lvalue ParmVar ID0004 'argc' 'int'
-    |           | |-CXXConversionDecl implicit operator int (*)(int) 'int (*() const noexcept)(int)' inline
-    |           | |-CXXMethodDecl implicit __invoke 'int (int)' static inline
-    |           | | `-ParmVarDecl argc 'int'
+    |           | |-FunctionTemplateDecl operator()
+    |           | | |-TemplateTypeParmDecl ID0003 implicit class depth 0 index 0 argc:auto
+    |           | | |-CXXMethodDecl operator() 'auto (auto) const' inline
+    |           | | | |-ParmVarDecl ID0004 referenced argc 'auto'
+    |           | | | `-CompoundStmt ID0005
+    |           | | |   `-ReturnStmt ID0006
+    |           | | |     `-DeclRefExpr ID0007 'auto' lvalue ParmVar ID0004 'argc' 'auto'
+    |           | | `-CXXMethodDecl ID0008 used operator() 'int (int) const' inline
+    |           | |   |-TemplateArgument type 'int'
+    |           | |   | `-BuiltinType  'int'
+    |           | |   |-ParmVarDecl ID0009 used argc 'int':'int'
+    |           | |   `-CompoundStmt
+    |           | |     `-ReturnStmt
+    |           | |       `-ImplicitCastExpr 'int':'int' <LValueToRValue>
+    |           | |         `-DeclRefExpr 'int':'int' lvalue ParmVar ID0009 'argc' 'int':'int'
+    |           | |-FunctionTemplateDecl implicit operator auto (*)(type-parameter-0-0)
+    |           | | |-TemplateTypeParmDecl ID0003 implicit class depth 0 index 0 argc:auto
+    |           | | `-CXXConversionDecl implicit operator auto (*)(type-parameter-0-0) 'auto (*() const noexcept)(auto)' inline
+    |           | |-FunctionTemplateDecl implicit __invoke
+    |           | | |-TemplateTypeParmDecl ID0003 implicit class depth 0 index 0 argc:auto
+    |           | | `-CXXMethodDecl implicit __invoke 'auto (auto)' static inline
+    |           | |   `-ParmVarDecl argc 'auto'
     |           | |-CXXDestructorDecl implicit referenced ~ 'void () noexcept' inline default trivial
-    |           | |-CXXConstructorDecl ID0009 implicit constexpr  'void (const (lambda at  ) &)' inline default trivial noexcept-unevaluated ID0009
+    |           | |-CXXConstructorDecl ID0010 implicit constexpr  'void (const (lambda at  ) &)' inline default trivial noexcept-unevaluated ID0010
     |           | | `-ParmVarDecl 'const (lambda at  ) &'
     |           | `-CXXConstructorDecl implicit used constexpr  'void ((lambda at  ) &&) noexcept' inline default trivial
     |           |   |-ParmVarDecl '(lambda at  ) &&'
     |           |   `-CompoundStmt
     |           `-CompoundStmt ID0005
     |             `-ReturnStmt ID0006
-    |               `-ImplicitCastExpr ID0007 'int' <LValueToRValue>
-    |                 `-DeclRefExpr ID0008 'int' lvalue ParmVar ID0004 'argc' 'int'
+    |               `-DeclRefExpr ID0007 'auto' lvalue ParmVar ID0004 'argc' 'auto'
     `-ReturnStmt
       `-CXXOperatorCallExpr 'int':'int' '()'
         |-ImplicitCastExpr 'int (*)(int) const' <FunctionToPointerDecay>
-        | `-DeclRefExpr 'int (int) const' lvalue CXXMethod ID0003 'operator()' 'int (int) const'
+        | `-DeclRefExpr 'int (int) const' lvalue CXXMethod ID0008 'operator()' 'int (int) const'
         |-ImplicitCastExpr 'const (lambda at  )' lvalue <NoOp>
         | `-DeclRefExpr '(lambda at  )':'(lambda at  )' lvalue Var ID0002 'lambda' '(lambda at  )':'(lambda at  )'
         `-ImplicitCastExpr 'int' <LValueToRValue>
```
