# Lithic C++ Programming Style and Design Guide

### What this document is:
  * Best practices for creating modular and reusable C++ architecture across projects and platforms.
  * An outline of some common C++ design antipatterns and how to avoid them.
  * Rules for consistent C++ style and formatting.
 
### What this document is *NOT*:
  * A fully comprehensive guide to using the C++ language well. The [CppCoreGuidelines](https://github.com/isocpp/CppCoreGuidelines/blob/master/CppCoreGuidelines.md) already do this better than I ever could. This is meant to be a condensed supplement with a more focused scope, not a replacement for more comprehensive usage guidelines. 
  * A criticism of those who choose to do things differently. 
    * These are my opinions, and I understand that not everyone shares them. However, I would appreciate if you are considering opening an issue or a pull request to tell me that I'm wrong about something, that it be limited to a correction of objective facts and not a disagreement of style or implementation. 
  * A tutorial on how to program in C++; you are assumed to have at least a fundamental working knowledge of the language.
    * *"There will be no hand-holding or silly wand-waving in this class."*
    
## Preface
This is not a catch-all document. C++ is a complex beast, and certain use cases require bending the rules a bit. However, unless you are writing code for extremely low level systems with very limited resources (in which case you should probably be using C anyway), or you really have to squeeze out every last drop of performance from the CPU for your application, most of this document should hold true. Even in those cases, a lot of it is probably decent advice anyway.

I mentioned the [CppCoreGuidelines](https://github.com/isocpp/CppCoreGuidelines/blob/master/CppCoreGuidelines.md) previously. You can consider this document a child of that one. If I don't address something specifically, whatever it says there is probably a good rule of thumb. Some things may be repeated here because I think they are important and worth emphasizing. Others may contradict and override what is written there. If that happens, I will do my best to explain the justification for the discrepancy.

## General Rules
1. **Use C++11** - It's supported pretty much everywhere. It is not unreasonable to ask someone to upgrade to a system that supports this version of the standard if they don't have it.
2. **Keep calm and trust your compiler** - Make your C++ as readable as possible. Let the compiler optimize the verbosity out of your code. 
3. **STL is your friend** - Using the built-in data structures from the standard library dramatically decreases the overhead required for someone else to use your code. Don't reimplement basic data structures.
4. **Don't be afraid to use classes** - Object-oriented design is a powerful tool in your belt. If you aren't comfortable with object-oriented design, get comfortable with it. If you aren't going to use classes in C++, write in C instead.

## Macros
Stop abusing the C preprocessor. Aside from blocking off platform specific implementations, if you're using any macros besides `#include` and `#pragma`, knock it off.

### Use `#pragma once` at the top of your header files instead of include guards. 
I know there's probably some push-back on this one.

> *"But `#pragma once` isn't an official part of the standard!"*

Maybe not, but it's supported on every compiler that matters. 

> *"They didn't make it part of the standard because some projects distribute their source tree and some implementations of it don't recognize that the same file retrieved on different paths is the same file."*

It sounds like you have much bigger problems than I'm capable of addressing here. Fix your source tree, then come back.

There is a rationale here; I don't just have a blind hatred for legacy programmers. When you define a macro, you define it everywhere and effectively pray that it doesn't collide with a macro declared somewhere else. This is fine if you are working on one project in a vacuum that doesn't use anyone else's code. However, as you depend on more and more code that uses macros, the global namespace becomes more and more polluted, and it becomes increasingly more likely that you have a collision somewhere. One of the things that this style guide tries to address is reusability and modularity of C++ code, and macros just don't lend themselves to this goal.

### Don't define constants as macros.
`const` works just fine.

### Don't use macros to define inline functions.

> *"But function calls are expensive. I'm optimizing!"*

Trust the compiler. It knows pushing and popping the stack and modifying the control flow of the program are expensive operations. It also knows how to optimize simple functions into inline calls to avoid this. You can even use the `inline` keyword if you feel like you need to nudge the compiler in the right direction. 

Macro functions create unreadable output from the compiler when something goes wrong inside of them, and it is very easy to unwittingly create unexpected behavior due to the obfuscated nature of the source code.

```c++
#define min(X,Y) X < Y ? X : Y
int ohNo = min(0, functionWithSideEffect());
```

### Don't use macros for flow control.

There is one unavoidable exception to this rule, and that is when you inevitably need to do something different on one platform vs. another. Even then, you should try to minimize the number of blocks that you have to create, and always wrap your platform specific behavior in a function; don't interrupt the body of your program.


:white_check_mark: **DO THIS**
```c++
#ifdef _WIN32
    #include <windows.h>
    void platformSpecificFunction() {
        windowsFunction();
    }
#else
    #include <unistd.h>
    void platformSpecificFunction() {
        posixFunction();
    }
#endif

...

void myApp() {
    platformSpecificFunction();
}
```

:x: **NOT THIS**
```c++
#ifdef _WIN32
    #include <windows.h>
#else
    #include <unistd.h>
#endif

...

void myApp() {
#ifdef _WIN32
    windowsFunction();
#else
    posixFunction();
#endif
}
```

> *"What about conditional dependencies?"*

Do **NOT** use macros to conditionally include/exclude your dependencies. 

> *"What are you on about? Loads of popular projects do that. I think some of yours even do that."*

Most of what is in this document should read as pretty common sense to practiced C++ developers. This is probably one of the only pieces of advice that is a radical deparature from established C++ practices. I assure you, I'm not off my rocker; let me explain.

This leads into a discussion of a much larger issue in the C++ community. This issue is that pretty much everyone builds in static mode instead of shared mode. Why do they do this? I would say in most cases because it's the default build setting for a lot of platforms, and many developers don't know any better. Some more experienced developers might talk about DLL hell. If you already understand the distinction, you can probably skip past the next section in which I outline the differences between the two.

#### Static vs. Shared
A static binary is a self-contained bundle of executable code. It resolves all of its external dependencies at compile time and copies all the relevant binary sections into its own bundle. In this way, a static binary has no dependencies to be resolved at runtime. When people say that static binaries "just work", this is what they mean. You can build a static binary and use it portably and consistently in any environment on any machine with the same platform and architecture.

> *"Sounds great, what's the problem?"*

The problem is code reuse. Every static library that is built on a dependency duplicates the code from that dependency. When many libraries have the same sets of dependencies, this can be very wasteful. For this reason, commonly reused platform dependencies like the standard libraries are usually implicitly linked in shared mode by your compiler.

> *"So we should build everything in shared mode?"*

Well, not quite. Static linking trades increased runtime memory usage and a little more work at compile time to make execution (slightly) faster. Shared linking has a (slightly) slower execution because it has to do the linking at runtime. Really, the reason to prefer static over shared is that there is nothing more frustrating than having your operating system not be able to find a compatible shared library at runtime when you are positive that you have the correct one in your path. Or even more frustrating, that you have it, and it recognizes the library, but the versions don't match up, or a function definition changed slightly. This is affectionately known as DLL hell. 

> *"Stop waffling and just tell me which one is better."*

That's the thing, neither one is strictly better than the other. They both have advantages and disadvantages, and you have to weigh those pros and cons for your particular application. However, in this case I think I can say that shared libraries are a better solution to conditional dependencies than using flow control macros.

#### Using dynamic linking for conditional dependency management
The first step is that you need to stop thinking of this optional code as a conditional dependency. Think of it as a plugin instead. Presumably the reason you wanted to do this in the first place was that you wanted to integrate some code that may or may not be present on a target platform. I think that some developers get caught in the trap of *"Well what if someone wanted to build it 'X' way."* You cannot think like that. You have to have **one build to rule them all.** It might seem fine while it's just one dependency. You have two builds, one with one without. But as you add dependencies, the number of build combinations that you have to test and maintain quickly becomes unmanageable. 

Build the optional code as a shared binary, and link it dynamically at runtime from your main binary. Use its success or failure to load to prevent undefined pathways from executing.

> *"But some of our developers don't want to have to have **all** of the dependencies installed in order to build."*

Either streamline setting up your build environment, or get better developers. Actually, probably do both.

## Project Structure
These are very much suggestions rather than real rules, but having a consistent project structure will make your life easier and make it easier for those unfamiliar with the codebase to find things in your source tree.
#### Example:
```
include
 - projectName
   - packageName
     - PackageClass.h
   - ProjectClass.h
src
 - packageName
   - PackageClass.cpp
 - ProjectClass.cpp
test
 - include
   - projectName
     - TestProjectClass.h
 - src
   - TestProjectClass.cpp
README.md
LICENSE.md
```
#### Notes:
* Don't clutter up your root project directory with source code.
  * All source code should be contained in their own directories. The root directory should be for administrative purposes only: `README.md`, `LICENSE.md`, `.gitignore`, etc. These things are not source code and do not belong in the same directory as source code.
* Headers have a parent directory with the same name as the project name, why?
  * When other projects link to yours, they will need to include your header files. Including that top level
    directory makes the includes look like this: 
    ```cpp
    #include "projectName/ProjectClass.h"
    ```
   * That means that you can name your class something simple without needing to worry about namespace collision. It helps avoid overly long file names like `OrganizationNamespacePackageClass.h`.
* Each class gets its own header file (and source file if necessary) with the same name as the class. Make your declarations and implementations easy to find.
  * You may have helper classes within a class that are very small. These don't need to get their own files as long as they are totally encapsulated by the other class, and they stay small. Use your judgement. If they start getting big, consider splitting them out into their own file for readability.
  * Functions should follow this same rule where possible. I say this because you really shouldn't have huge collections of loose functions. Try to avoid kitchen sinks like `ProjectHelpers.h`. There is probably a better place to put that function so that it is in a context more in line with its usage. More importantly, if you have a huge collection of utility functions that have no specific application to your project, you should consider making that into a separate project and then using it.
  
### Stop writing header-only libraries
Header-only libraries have gotten tremendously popular in modern C++. I suspect this is mostly just due to ease of building. Most header-only libraries say something in their README along the lines of "Just `#include <thisHeaderOnlyLibrary.h>` and you're ready to go".

#### Everything wrong with header-only libraries
1. Maintenance
  * Some (not all) of these libraries tend to be gigantic several thousand line header files with little to no distribution of the source tree. This is a nightmare for merging when more than one person works on the project, and it's also just a nightmare for readability.
    * Some libraries do have a distribution of source files that they pack into a single production header release. That still doesn't help with point 2.
2. The C pre-processor
  * When you `#include` something, the C pre-processor copies and pastes the contents of the file in place of the include. That means that wherever this header-only library is used uniquely, its entire several thousand line body is being copied and spliced into the body of your source code. This can slow down your build pretty dramatically depending on how many header-only libraries you use and how big they are.
3. Separation of concerns
  * Having your implementation separated from your declaration makes it easier to read and understand the API of your library or application. 
  
> *"Yeah... I'm not going to stop writing or using header only libraries. A little slower compilation is nothing compared to the headache of trying to shoehorn someone else's build process into mine."*

I get it. Reusability of code in C++ has historically been a huge problem. Header-only libraries have been a big step forward in reusable, modular code. I understand choosing to use them over the nightmare that has been C++ build systems. I hope to convince you that there is a better way.

## Build Systems
Love CMake or hate it, it's really the only game in town for getting consistent builds that work everywhere.

> *"Make is good enough and isn't nearly as bloated or hacky."*

I generally agree, but have fun writing C++ that doesn't work on Windows. Or worse yet, force your Windows developers to use MinGW and `gcc` instead of `MSVC` as you add more hacky additions and shims to your makefile in order to get your build to work on all of the platforms you need to support.

CMake isn't so bad. I think a lot of people get scared off when they see enormous, convoluted `CMakeLists.txt` files and they try to find answers to their questions and three people are all saying different things. To be honest, this is pretty much entirely CMake's fault. They have no real tutorial or examples, just an automatically generated documentation set which isn't super clear on actual usage. 

### How I learned to stop worrying and love the bomb
Keep your CMakeLists short and simple.

```cmake
cmake_required_version(3.1.0)

set(PROJECT_NAME myProject)
set(CMAKE_CXX_STANDARD 11)
set(${PROJECT_NAME}_INCLUDE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/include)

project(${PROJECT_NAME})
file(GLOB LIB_HEADERS "${${PROJECT_NAME}_INCLUDE_DIR}/${PROJECT_NAME}/*.h")
file(GLOB LIB_SOURCES "${CMAKE_CURRENT_SOURCE_DIR}/src/*.cpp")
add_library(${PROJECT_NAME} ${LIB_HEADERS} ${LIB_SOURCES})
```

