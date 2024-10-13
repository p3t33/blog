+++
title = "lsp and cross compilation pitfalls "
date = 2024-10-13T22:11:32+03:00
tags = ["lsp", "cmake", "cross-compiling"]
draft = false
+++

## About {#about}

The aim of this tutorial is to go over the creation of the compilation database(compile_commands.json) for
cland lsp using cmake, and then more importantly expand on its limitations when it comes to cross compilation.


## Resources {#resources}

You can get all the source described bellow at the following [repository](https://github.com/p3t33/compilation_database).


## Creating the source code {#creating-the-source-code}


### Understanding the include search order by clang++/g++ {#understanding-the-include-search-order-by-clang-plus-plus-g-plus-plus}

standard headers such as iostream or pthread(posix) are located by the compiler at standard paths, other headers(foo.h)
need to be specified using the directory they are located in using -I directive.


#### #include "..." {#include-dot-dot-dot}

1.  current directory containing the source file where the #include directive appears.
2.  directories specified by -I option(using the -I compile flage).
3.  standard system directories, the default directories where the compiler looks for head files such as /user/include.


#### #include &lt;...&gt; {#include-dot-dot-dot}

1.  directories specified by -I option(using the -I compile flage).
2.  standard system directories, the default directories where the compiler looks for head files such as /user/include.

Note: current directory isn't being searched unless specified by -I.


### Considerations when creating the source {#considerations-when-creating-the-source}

The source included with "..." should not be in the same directory as the main.cpp file.
At least one standard library should be included using &lt;...&gt;.

I have added as a separate commit with pthread, as it is a POSIX library and not a cpp standard library and so it
is not linked implicitly and needs to be linked explicitly with -lpthread(compiling without cmake file). The relevance of this library
for our context is that the pthread.h is located at the standard path where compiler looks for headers and effects the compilation
using macros and ifdef statements. And so you don't need to use -I to specify it for the compiler, But in order to link it, unlike standard
cpp library you will need to explicitly link it using -lpthread).

What is important to understand is that with or without including pthread the compile_commands.json output will look the same.


### The source code {#the-source-code}

I have created a source directory and files:

```nil
compilation_database
├── app
│   └── main.cpp
└── lib
    ├── hello_world.cpp
    └── hello_world.h
```

hello_world.h:

```cpp
#include <iostream>
void print_hello_world(void);
```

hello_world.cpp:

```cpp
#include "hello_world.h"

void print_hello_world(void)
{
    std::cout << "Hello, World!" << std::endl;
}
```

main.cpp:

```cpp
#include "hello_world.h"

int main()
{
    print_hello_world();
    return 0;
}
```


### compiling the code {#compiling-the-code}

At this stage calngd will fail to find hello_world.h inside of main.cpp and you will get bunch of lsp erros.
The code will be compiled, form the root of the compilation_database:

```nil
clang++ app/main.cpp lib/hello_world.cpp -I./lib
```

At this point I have created my first commit


## Creating Cmake file and compiling the source {#creating-cmake-file-and-compiling-the-source}

To make compilation easier, but mostly to generate compile_commands.json I have created CmakeLists.txt at the root of the
compilation_databse.

```cmake
cmake_minimum_required(VERSION 3.0)
project(HelloWorldProject)

# Set the C++ standard to C++11 (you can adjust this if needed)
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED True)

# Enable compile_commands.json generation
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# Include directories
include_directories(${PROJECT_SOURCE_DIR}/lib)

# Add the executable target
add_executable(hello_world
    app/main.cpp
    lib/hello_world.cpp
)
```

To build the source:

```bash
mkdir buiid
cd build
#resovle thnigs that might be messing such as make
cmake ..
cmake --build .
```

- The first command generates the necessary build files(make,ninja...) and sets the build environment and project structure
- the --build commands uses the output of the first command and compiles the code.

You should get hello_world executable inside the build directory and be able to run it.

At this point I made my second commit.


## Generating compile_commands.json {#generating-compile-commands-dot-json}

Inside of build directory execute

```bash
cmake -DCMAKE_EXPORT_COMPILE_COMMANDS=ON ..
```

At this point you should have compile_commands.json inside of your build directory and if you move it into the root of
compilation_database directory and restart lsp, it will start working and you will be able to jump to definitions(among
other things).


### Anatomy of compile_commands.json {#anatomy-of-compile-commands-dot-json}

If you look at the content of the generated file you will see that it has multiple entries, one for each compilation unit.

```json
[
{
  "directory": "/home/kmedrish/projects/compilation_database/build",
  "command": "/run/current-system/sw/bin/c++  -I/home/kmedrish/projects/compilation_database/lib -std=gnu++11 -o CMakeFiles/hell_world.dir/app/main.cpp.o -c /home/kmedrish/projects/compilation_database/app/main.cpp",
  "file": "/home/kmedrish/projects/compilation_database/app/main.cpp",
  "output": "CMakeFiles/hell_world.dir/app/main.cpp.o"
},
{
  "directory": "/home/kmedrish/projects/compilation_database/build",
  "command": "/run/current-system/sw/bin/c++  -I/home/kmedrish/projects/compilation_database/lib -std=gnu++11 -o CMakeFiles/hell_world.dir/lib/hello_world.cpp.o -c /home/kmedrish/projects/compilation_database/lib/hello_world.cpp",
  "file": "/home/kmedrish/projects/compilation_database/lib/hello_world.cpp",
  "output": "CMakeFiles/hell_world.dir/lib/hello_world.cpp.o"
}
]
```

- **directory**: specifies the directory where the compilation unit is being compiled(PWD). This is important because the compilation
  command might contain a relative path that will be resolved based on the specified directory. LSP uses it to
  resolve relative paths in the command.
- **command**: this is the exact compilation command that will be used to create a single object file. LSP uses it to provide
   information about the compilation unit, including the use of flags that will effect the code(ifdef). And paths to
   included headers form.
   - compiler(E.g c++): this is important as it allows LSP to resolve compiler specific flags and also system libraries paths that are
     specific to the compiler.
- **file**: specifies the single source file being compiled, LSP matches the file being edited with the corresponding entry in the
  compile_commands.json. LSP needs this because although the command itself has a single file with its full path and it is
  the same string as the one in "command", the file filed is much easier to parse.
- **output**; optional, and is used to specify the path to the object file. LSP typically doesn't require the output.

By putting it all together, when you open a file in your editor, LSP looks at its name(including full path) and uses it as a key
inside of compile_commands.json. Once it has found it, it parses the string in its command, it uses the paths to look for headers that
are being included(-I), the flags to enable(E.g, -std=gnu++11 and ifdef macros and uses the directory field to resolve relative paths.  The compiler that
is used in the command is also important as it not only effects pre defined macros and compiler specific features but also
the default system library headers paths all of which are taken into account by LSP.


## How all of this relates to cross compilation {#how-all-of-this-relates-to-cross-compilation}

By looking at compile_commands.json a few things become very clear.
- As previously mentioned standard libraries(iostream, pthread) are not reflected in the file. while user headers reflected using -I.
- All paths are absolute, meaning the if you change the name of the directory you compile in your compile_commands.json will become
  invalid until you generate a new one.
- A lot is derived from the compiler you are using, specifically the paths to system libraries.

So where does this leaves lsp?

- local compilation, with cross compiler installed on host system. Just open your editor and everything should work out of the box(not tested as this isn't my
development environment).

- local compilation with cross compiler inside of a docker, both absolute file paths and compiler paths are invalid.
- remote development with cross compiler installed on the build server. If you connect to the remote source via VSCode Remote Development
  you should be fine.


The bottom line is that you need to make sure that the path you access your code with LSP is the same path used for
building it and that the compiler that is used to compile your source exists in your system at the exact same path it is
stated in compile_commands.jsonn. Otherwise you will need to edit your compile_commands.json and make it true to your working
environment.

So although there is no "one size fits all" solutions, as it all depends on your specific cross compile environment. A simple fix
would be to edit the compile_commands.json(using something like sed), updating the paths, providing valid compiler, removing
--sysroot commands and cross compilation flags.




