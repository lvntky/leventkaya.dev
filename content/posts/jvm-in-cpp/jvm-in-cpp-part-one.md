---
title: "Creating a Java Virtual Machine in C++(Again) - part #1"
date: 2024-06-10T19:54:54+03:00
slug: 2024-06-10-jvm-in-cpp-part-one
type: posts
draft: false
categories:
  - development
tags:
  - jvm
  - java
  - cpp
  - compiler
---


![logo](/images/jvm-in-cpp/javalogo.gif)


> The first implementation of CVM (in C) : [CMV/archive](https://github.com/lvntky/CVM/tree/archive) <br>

> The WIP re-implementation of CVM a.k.a CVM++ (in C++) : [CVM/dev](https://github.com/lvntky/CVM/tree/dev) 

# The `CVM`++
About a year ago, I decided to write a jvm in C, and I did. It just wasn't quite what I wanted and it lacked many functions. Actually, I wrote a java debugger on a binary level rather than a jvm. A year later, I decided to look at this project again and give it the functionality it deserves. This time, I decided to step out of my comfort zone and do it in C++ instead of C (Also, I'm too old to deal with string manipulation in C anymore).

I believe this series will be quite long and will be useful for both Java and C++. Welcome to the first part.

## The Functional Requirements
Since I didn't want to make the same mistakes as last time, I wanted to start by writing the functional requirements of the jvm that I wanted to create. The software I will make throughout this series will have the following functional requirements.

- Class loading
  - Load classes from the file system
  - Resolve symbolic references.
- Bytecode verification
  - Verify bytecode integrity to prevent security vulnerabilities and ensure correct behavior.
  - Perform structural and behavioral checks on bytecode instructions.
- Runtime Data Areas
  - Implement runtime data areas such as heap, method area, program counter (PC), stack, and native method stacks.
  - Manage memory allocation and deallocation.
  - Handle thread-local storage for each thread executing on the JVM.
- Execution Engine
  - Interpret bytecode instructions or use just-in-time (JIT) compilation for performance optimization.
  - Support execution of Java bytecode instructions according to the Java Virtual Machine Specification.
  - Implement support for core instructions like load/store, arithmetic, flow control, method invocation, and exception handling.
  
Of course, a JVM that can be used for real production should have much more than these features. But my goal in this series is to make a working jvm, it does not need to be used in the real world.

## What is the ```.class``` file ?

A .class file is a binary file format used by the Java programming language to store compiled Java bytecode. When you compile a Java source file (.java), the Java compiler (javac) generates a .class file for each class defined in the source code.

**The structure of .class**

A .class file follows a specific structure defined by the Java Virtual Machine Specification. Here's a brief overview of its structure:

- Magic Number: The first 4 bytes of the file represent a magic number (0xCAFEBABE), which identifies it as a valid Java class file.
- Version Information: The next 4 bytes represent the minor and major version numbers of the Java compiler used to generate the file.
- Constant Pool: The constant pool contains various constant values used by the class (e.g., strings, numbers, class and method references). It starts with a 2-byte count of entries in the pool.
- Access Flags: 2 bytes representing access flags indicating the access permissions and properties of the class.
- This Class and Super Class: 2 bytes each representing indices into the constant pool, specifying the class and its direct superclass.
- Interfaces: 2 bytes representing the number of interfaces implemented by the class, followed by 2-byte indices into the constant pool for each interface.
- Fields: 2 bytes representing the number of fields defined in the class, followed by field definitions.
- Methods: 2 bytes representing the number of methods defined in the class, followed by method definitions.
- Attributes: Additional attributes, such as annotations, debugging information, and method code, are stored as attributes.

**The purpose of .class:**

`.class` files serve as the intermediary between Java source code and the Java Virtual Machine (JVM). They contain the compiled bytecode, which is platform-independent and can be executed by any JVM regardless of the underlying hardware and operating system. When you run a Java application, the JVM loads the appropriate .class files, interprets the bytecode, and executes the program instructions.

## A Humble Goal
![goal](/images/jvm-in-cpp/goal.gif)

My modest goal is this, I will write a very small Java program and my own JVM will run this program. For this, I created a structure like this:

```
├── CMakeLists.txt
├── CONTRIBUTING.md
├── Dockerfile
├── LICENSE
├── Makefile
├── PULL_REQUEST_TEMPLATE.md
├── README.md
├── cmake
│   ├── CVMConfig.cmake.in
│   ├── CompilerWarnings.cmake
│   ├── Conan.cmake
│   ├── Doxygen.cmake
│   ├── SourcesAndHeaders.cmake
│   ├── StandardSettings.cmake
│   ├── StaticAnalyzers.cmake
│   ├── Utils.cmake
│   ├── Vcpkg.cmake
│   └── version.hpp.in
├── codecov.yaml
├── docs
│   └── banner.jpg
├── include
│   └── cvm
│       ├── fmt_commons.hpp
│       └── tmp.hpp
├── sample
│   ├── Add.class
│   └── Add.java
├── src
│   ├── main.cpp
│   └── tmp.cpp
└── test
    ├── CMakeLists.txt
    └── src
        └── tmp_test.cpp
```

I thought it was a modern and very versatile project structure, so I started writing the Java code that I would run next.

```
  public class Add {
      public static int add(int a, int b) {
          return a + b;
      }
  }
```

As you can see, it is a very simple program, then I compiled this program with javac and created the .class file.

The hexdump of the .class file:

```
00000000: cafe babe 0000 0042 000f 0a00 0200 0307  .......B........
00000010: 0004 0c00 0500 0601 0010 6a61 7661 2f6c  ..........java/l
00000020: 616e 672f 4f62 6a65 6374 0100 063c 696e  ang/Object...<in
00000030: 6974 3e01 0003 2829 5607 0008 0100 0341  it>...()V......A
00000040: 6464 0100 0443 6f64 6501 000f 4c69 6e65  dd...Code...Line
00000050: 4e75 6d62 6572 5461 626c 6501 0003 6164  NumberTable...ad
00000060: 6401 0005 2849 4929 4901 000a 536f 7572  d...(II)I...Sour
00000070: 6365 4669 6c65 0100 0841 6464 2e6a 6176  ceFile...Add.jav
00000080: 6100 2100 0700 0200 0000 0000 0200 0100  a.!.............
00000090: 0500 0600 0100 0900 0000 1d00 0100 0100  ................
000000a0: 0000 052a b700 01b1 0000 0001 000a 0000  ...*............
000000b0: 0006 0001 0000 0001 0009 000b 000c 0001  ................
000000c0: 0009 0000 001c 0002 0002 0000 0004 1a1b  ................
000000d0: 60ac 0000 0001 000a 0000 0006 0001 0000  `...............
000000e0: 0003 0001 000d 0000 0002 000e            ............
```

We will be doing a detailed review of this hexdump in the series. But I would like to draw attention to the phrase cafe babe at the beginning. This is a magic number used to verify that what the JVM is reading is a .class file. Maybe I'll tell you his story sometime. Anyway, after this point, I had a .class file, now all I have to do is load this class file and then parse it. At this point, I'm taking advantage of [Oracle's JVM specification](https://docs.oracle.com/javase/specs/jvms/se7/html/).

## The ClassLoader
So here is my base class loader class:

![class loader](/images/jvm-in-cpp/class_loader.png)

I started the project with this simple loader. Basically, its function is to load the class file compiled with javac into memory and validate it.

- ```std::vector<uint8_t> loadClassFile(const std::string& filePath):``` Loads the .class file from the specified file path into memory.
- ```void parseClassFile(const std::vector<uint8_t>& buffer):``` Parses the contents of the .class file buffer.

Thanks to this class, I was able to import .class files and start processing them. We will examine the structure of these .class files in the next part of the series, until then, goodbye.

![class loader](/images/jvm-in-cpp/goodbye1.gif)