% title: The Clang AST
% type: post
% author: vortex
% description: An overview of Clang's AST
% date: 2024-11-5
% tags: llvm,clang,compilers

While exploring LLVM discourse and diving into the source code, I came across [this talk](https://www.youtube.com/watch?v=VqCkCDFLSsc) that showcases the power and versatility of Clang's AST. This blog is a collection of my notes aimed at demystifying the structure and intricacies od Clang's *huge* AST which is pivotal in the compilation of C/C++ and other languages.

Clang’s AST is a powerful abstraction that provides an in-depth representation of the source code. It is essential for tasks such as type checking, optimization, and code transformation.

At its core, Clang operates in multiple stages:
```text
[Front] -> [Middle] -> [Back]
   |         |          |
   |    AST  |          |
   +-------->|    IR    |
             +--------->|
[Front] -> [Middle] -> [Back]
```

# Why is it special
Clang’s AST isn't just another structure for holding the pieces of code. It’s a masterfully crafted system that optimizes for clarity and efficiency, providing several key advantages:

- **Precision in Source Locations**: Imagine being able to pinpoint the exact location of every variable, function, and statement in a program with absolute precision. Clang’s AST does just that, maintaining an exact map of each source code element’s position. This is like having a map that not only tells you where each landmark is but also provides detailed street names.

- **Complete Type Resolution**: C++ is a maze of types, where variables and functions can take on multiple forms. Clang’s AST navigates this maze with ease by fully resolving types, ensuring that even the most complex type declarations are handled correctly. Without this, parsing C++ would be like reading a book with no glossary for its cryptic language.

- **Optimized Memory Usage**: Despite its vastness—often stretching into the hundreds of thousands of lines—Clang's AST is designed to use memory efficiently. It's like building a city with a vast network of streets but only using the most efficient infrastructure to keep it running smoothly.

# ASTContext

At the heart of Clang’s AST lies the **ASTContext**, which serves as the central hub for storing and managing AST nodes. When parsing a source file, Clang is like a skilled architect who must decide how best to allocate space for each building block. To optimize memory usage, AST nodes don’t store all their data directly. Instead, they point to centralized resources, such as an **identifier table**, to reference common information. This is similar to how a vast library system might use a centralized catalog rather than storing every single book in each section.

The AST itself is anchored by the `TranslationUnitDecl`, which represents the entire source code and serves as the entry point into this complex structure. Think of it as the grand entrance to the compilation process, leading into the detailed map of functions, types, and statements that make up the program.

Now consider this example: 
```c
int main(int argc, char *argv[])
{
    int x = 0;
    x+=4;
    return x;
}
```

Now clang emits the AST of this code as the following:

```
TranslationUnitDecl 0x5a96088e97d8 <<invalid sloc>> <invalid sloc>
|-TypedefDecl 0x5a96088ea008 <<invalid sloc>> <invalid sloc> implicit __int128_t '__int128'
| `-BuiltinType 0x5a96088e9da0 '__int128'
|-TypedefDecl 0x5a96088ea078 <<invalid sloc>> <invalid sloc> implicit __uint128_t 'unsigned __int128'
| `-BuiltinType 0x5a96088e9dc0 'unsigned __int128'
|-TypedefDecl 0x5a96088ea380 <<invalid sloc>> <invalid sloc> implicit __NSConstantString 'struct __NSConstantString_tag'
| `-RecordType 0x5a96088ea150 'struct __NSConstantString_tag'
|   `-Record 0x5a96088ea0d0 '__NSConstantString_tag'
|-TypedefDecl 0x5a96088ea428 <<invalid sloc>> <invalid sloc> implicit __builtin_ms_va_list 'char *'
| `-PointerType 0x5a96088ea3e0 'char *'
|   `-BuiltinType 0x5a96088e9880 'char'
|-TypedefDecl 0x5a96088ea720 <<invalid sloc>> <invalid sloc> implicit referenced __builtin_va_list 'struct __va_list_tag[1]'
| `-ConstantArrayType 0x5a96088ea6c0 'struct __va_list_tag[1]' 1 
|   `-RecordType 0x5a96088ea500 'struct __va_list_tag'
|     `-Record 0x5a96088ea480 '__va_list_tag'
```

Each line here represents a specific building block: from the top-level `TranslationUnitDecl` to the types and declarations that Clang generates as it parses the source. These nodes are like the floorplans that define how each part of the program is connected.

# Core Classes

There are 3 main types of classes in clang's AST:

1. [Decl](https://clang.llvm.org/doxygen/classclang_1_1Decl.html): These are the blueprints of your program's components—variables, functions, and types. Each declaration represents a building block, defining what exists in your program.

2. [Stmt](https://clang.llvm.org/doxygen/classclang_1_1Stmt.html) : These are the actions—the bricks that, when laid down, execute tasks like assignments, loops, and conditionals. Without them, the program would be static, much like a house with no doors or windows.

3. [Types](https://clang.llvm.org/doxygen/classclang_1_1Type.html) : Types are the materials that make up the structure, defining how variables are constructed and manipulated. Whether it’s an integer or a custom struct, types lay the foundation for what can be done with the data.

These are the types that have identity - 2 of them can be compared.

> Unlike other ASTs there is no "base" class. There is no unified way to access all the nodes in the AST in favour of performance and efficiency as each node won't need to carry unnecessary fields or methods.

# Then how do we traverse these Classes?

Since the classes don't have a single common interface to interact with, the AST has "Glue" methods that help with connecting between different AST constructs.

## Types

Due to the nuances of C++ and C, types are *complicated* to represent in the AST.
For example:
```cpp
#include <vector>

struct MyStruct {
    int a;
    // qualified type
    const double b;
};

using FuncType = int (*)(const MyStruct &, std::vector<int> &);

// nested type
FuncType getFunctionPointer();
```

In the above case, the the variable has a qualifier(`const`) and datatype(`double`).
If we parse if normally, there would need to be separate representations stored for *each* variation which leads to an exponential growth, increasing redundancy.

For instance, take the example below:

```c
int x;
const int& y = x;
```
The types `const int&` and `int&` would need to be distinct types in the AST, even though they differ only by the qualifier const. So for every base type like `int` you'd need separate representations for `int`, `const int`, `int&`, `const int&` and so on.


In other words, this would lead to a combinatorial explosion in how many unique types you would need to store.

Hence the ClangAST has a type called the `QualType` that encapsulates the qualifier information into a different level thereby ridding the base classes of the burden of storing representations of qualifier information.


The AST representation:
```sh
$ clang -Xclang -ast-dump test.c
|       | `-QualType 0x595bb71d1241 'const MyStruct' const
|       |   `-ElaboratedType 0x595bb71d1240 'MyStruct' sugar
|       |     `-RecordType 0x595bb71d1040 'MyStruct'
|       |       `-CXXRecord 0x595bb71d0fa8 'MyStruct'
|       `-LValueReferenceType 0x595bb71d13d0 'int &'
|         `-BuiltinType 0x595bb7186ea0 'int'
```

## Locations

Locations in the AST are represented by an ID. The exact location is managed using the `SourceManager` which points to tokens.

```txt
             getQualiferLoc()    getPosition()
                       \              /
                        \            /
                         \          /
                    void MyClass::someMethod() {}
                    /                           \
                   /                             \
                  /                               \
             getLocStart()                         getLocEnd()
```

# RecursiveASTVisitor

Clang's [RecursiveASTVisitor](https://clang.llvm.org/doxygen/classclang_1_1RecursiveASTVisitor.html#details) is a powerful class that simplifies traversing and analyzing the AST of C/C++ code. This class is part of Clang’s libraries, which provide tools for static analysis, code generation, and refactoring. 

The visitor pattern  allows you to walk through the AST and perform operations on different nodes, such as function declarations, variable definitions, or control flow statements.

Using this, you can easily analyze or transform source code without manually parsing it. It's particularly valuable for creating custom linters, refactoring tools, or static analyzers

I personally never understand most things unless I implement them so here's an example of a simple Clang plugin that traverses a C++ source file and collects all function declarations using RecursiveASTVisitor.

```cpp
#include "clang/AST/ASTConsumer.h"
#include "clang/AST/RecursiveASTVisitor.h"
#include "clang/Frontend/FrontendPluginRegistry.h"
#include "clang/Frontend/CompilerInstance.h"


class FunctionVisitor : public clang::RecursiveASTVisitor<FunctionVisitor> {
public:
    explicit FunctionVisitor(clang::ASTContext *Context) : Context(Context) {}

    bool VisitFunctionDecl(clang::FunctionDecl *F) {
        if (F->hasBody()) {
            llvm::outs() << "Function Name: " << F->getNameInfo().getName().getAsString() << "\n";
            llvm::outs() << "Return Type: " << F->getReturnType().getAsString() << "\n";
            llvm::outs() << "Location: " << F->getLocation().printToString(Context->getSourceManager()) << "\n\n";
        }
        return true;
    }

private:
	clang::ASTContext *Context;
};

class FunctionASTConsumer : public clang::ASTConsumer {
public:
    explicit FunctionASTConsumer(clang::ASTContext *Context) : Visitor(Context) {}

    void HandleTranslationUnit(clang::ASTContext &Context) override {
        Visitor.TraverseDecl(Context.getTranslationUnitDecl());
    }

private:
    FunctionVisitor Visitor;
};

class FunctionPluginAction : public clang::PluginASTAction {
protected:
  std::unique_ptr<clang::ASTConsumer> CreateASTConsumer(clang::CompilerInstance &CI,
                                                 llvm::StringRef) override;

  bool ParseArgs(const clang::CompilerInstance &,
                 const std::vector<std::string> &) override {
    return true;
    }
};
std::unique_ptr<clang::ASTConsumer>
FunctionPluginAction::CreateASTConsumer(clang::CompilerInstance &CI,
                                        llvm::StringRef) {
  return std::make_unique<FunctionASTConsumer>(&CI.getASTContext());
}

static clang::FrontendPluginRegistry::Add<FunctionPluginAction>
    X("function-visitor", "Collect function declarations");


```
Quickly spin up a `CMakeLists.txt` that builds the plugin.

```sh
cmake_minimum_required(VERSION 3.10)
project(myPlugin LANGUAGES C CXX)

find_package(LLVM REQUIRED CONFIG)
include_directories(${LLVM_INCLUDE_DIRS})
add_definitions(${LLVM_DEFINITIONS})

add_library(myPlugin MODULE plugin.cpp)
target_link_libraries(myPlugin PRIVATE clangTooling clangBasic clangAST)
```

Lets make a test file that we can run through the plugin

```cpp
 test.cpp
int add(int a, int b) {
    return a + b;
}

double multiply(double x, double y) {
    return x * y;
}

void printHello() {
    // Empty function
}

```

Now use the built plugin with Clang's `-Xclang` option:

```sh
clang -Xclang -load -Xclang build/myPlugin.so -Xclang -plugin -Xclang function-visitor test.cpp
```

The plugin should output something like the following

```sh
Function Name: greet
Return Type: void
Location: test.cpp:3:6

Function Name: add
Return Type: int
Location: test.cpp:7:5

Function Name: main
Return Type: int
Location: test.cpp:11:5

```

# Conclusion
Clang’s AST is a remarkable cornerstone of modern compiler design, offering unmatched precision, efficiency, and flexibility. Whether you’re a researcher, developer, or enthusiast, diving into Clang’s AST opens up countless opportunities for code analysis and transformation. Its robust ecosystem, combined with tools like RecursiveASTVisitor, allows for deep exploration and manipulation of source code, empowering users to push the boundaries of what’s possible in compiler technology.


