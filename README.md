llvm-tutor
=========
[![Build Status](https://travis-ci.org/banach-space/llvm-tutor.svg?branch=master)](https://travis-ci.org/banach-space/llvm-tutor)
[![Build Status](https://github.com/banach-space/llvm-tutor/workflows/x86-Ubuntu/badge.svg?branch=master)](https://github.com/banach-space/llvm-tutor/actions?query=workflow%3Ax86-Ubuntu+branch%3Amaster)
[![Build Status](https://github.com/banach-space/llvm-tutor/workflows/x86-Darwin/badge.svg?branch=master)](https://github.com/banach-space/llvm-tutor/actions?query=workflow%3Ax86-Darwin+branch%3Amaster)


Example LLVM passes - based on **LLVM 9**

**llvm-tutor** is a collection of self-contained reference LLVM passes. It's a
tutorial that targets novice and aspiring LLVM developers. Key features:
  * **Complete** - includes `CMake` build scripts, LIT tests and CI set-up
  * **Out of source** - builds against a binary LLVM installation (no need to
    build LLVM from sources)
  * **Modern** - based on the latest version of LLVM (and updated with every release)

### Overview
LLVM implements a very rich, powerful and popular API. However, like many
complex technologies, it can be quite daunting and overwhelming to learn and
master. The goal of this LLVM tutorial is to showcase that LLVM can in fact be
easy and fun to work with. This is demonstrated through a range self-contained,
testable LLVM passes, which are implemented using idiomatic LLVM.

This document explains how to set-up your environment, build and run the
examples, and go about debugging. It contains a high-level overview of the
implemented examples and contains some background information on writing LLVM
passes. The source files, apart from the code itself, contain comments that
will guide you through the implementation. All examples are complemented with
[LIT](https://llvm.org/docs/TestingGuide.html) tests and reference [input
files](https://github.com/banach-space/llvm-tutor/blob/master/inputs).

### Table of Contents
* [HelloWorld](#helloworld)
* [Development Environment](#development-environment)
* [Building & Testing](#building--testing)
* [Overview of the Passes](#overview-of-the-passes)
* [Debugging](#debugging)
* [About Pass Managers in LLVM](#about-pass-managers-in-llvm)
* [References](#references)


HelloWorld
==========
The **HelloWorld** pass from
[HelloWorld.cpp](https://github.com/banach-space/llvm-tutor/blob/master/HelloWorld/HelloWorld.cpp)
is a self-contained *reference example*. The corresponding
[CMakeLists.txt](https://github.com/banach-space/llvm-tutor/blob/master/HelloWorld/CMakeLists.txt)
implements the minimum set-up for an out-of-source pass.

For every function defined in the input module, **HelloWord** prints its name
and the number of arguments that it takes. You can build it like this:
```bash
export LLVM_DIR=<installation/dir/of/llvm/9>
mkdir build
cd build
cmake -DLT_LLVM_INSTALL_DIR=$LLVM_DIR <source/dir/llvm/tutor>/HelloWorld/
make
```
Before you can test it, you need to prepare an input file:
```bash
# Generate an LLVM test file
$LLVM_DIR/bin/clang -S -emit-llvm <source/dir/llvm/tutor/>inputs/input_for_hello.c -o input_for_hello.ll
```
Finally, run **HelloWorld** with [**opt**](http://llvm.org/docs/CommandGuide/opt.html):
```bash
# Run the pass
$LLVM_DIR/bin/opt -load-pass-plugin libHelloWorld.dylib -passes=hello-world -disable-output input_for_hello.ll
# Expected output
(llvm-tutor) Hello from: foo
(llvm-tutor)   number of arguments: 1
(llvm-tutor) Hello from: bar
(llvm-tutor)   number of arguments: 2
(llvm-tutor) Hello from: fez
(llvm-tutor)   number of arguments: 3
(llvm-tutor) Hello from: main
(llvm-tutor)   number of arguments: 2
```
The **HelloWorld** pass doesn't modify the input module. The `-disable-output`
flag is used to prevent **opt** from printing the output bitcode file.

Development Environment
=======================
## Platform Support And Requirements
This project has been tested on **Linux 18.04** and **Mac OS X 10.14.4**. In
order to build **llvm-tutor** you will need:
  * LLVM 9
  * C++ compiler that supports C++14
  * CMake 3.4.3 or higher

In order to run the passes, you will need:
  * **clang-9** (to generate input LLVM files)
  * **opt** (to run the passes)

There are additional requirements for tests (these will be satisfied by
installing LLVM 9):
  * [**lit**](https://llvm.org/docs/CommandGuide/lit.html) (aka **llvm-lit**,
    LLVM tool for executing the tests)
  * [**FileCheck**](https://llvm.org/docs/CommandGuide/lit.html) (LIT
    requirement, it's used to check whether tests generate the expected output)

## Installing LLVM 9 on Mac OS X
On Darwin you can install LLVM 9 with [Homebrew](https://brew.sh/):
```bash
brew install llvm@9
```
This will install all the required header files, libraries and tools in
`/usr/local/opt/llvm/`.

## Installing LLVM 9 on Ubuntu
On Ubuntu Bionic, you can [install modern
LLVM](https://blog.kowalczyk.info/article/k/how-to-install-latest-clang-6.0-on-ubuntu-16.04-xenial-wsl.html)
from the official [repository](http://apt.llvm.org/):
```bash
wget -O - https://apt.llvm.org/llvm-snapshot.gpg.key | sudo apt-key add -
sudo apt-add-repository "deb http://apt.llvm.org/bionic/ llvm-toolchain-bionic-9 main"
sudo apt-get update
sudo apt-get install -y llvm-9 llvm-9-dev clang-9 llvm-9-tools
```
This will install all the required header files, libraries and tools in
`/usr/lib/llvm-9/`.

## Building LLVM 9 From Sources
Building from sources can be slow and tricky to debug. It is not necessary, but
might be your preferred way of obtaining LLVM 9. The following steps will work
on Linux and Mac OS X:
```bash
git clone https://github.com/llvm/llvm-project.git
cd llvm-project
git checkout release/9.x
mkdir build
cd build
cmake -DCMAKE_BUILD_TYPE=Release -DLLVM_TARGETS_TO_BUILD=X86 <llvm-project/root/dir>/llvm/
cmake --build .
```
For more details read the [official
documentation](https://llvm.org/docs/CMake.html).

Building & Testing
===================
You can build **llvm-tutor** (and all the provided passes) as follows:
```bash
cd <build/dir>
cmake -DLT_LLVM_INSTALL_DIR=<installation/dir/of/llvm/9> <source/dir/llvm/tutor>
make
```

The `LT_LLVM_INSTALL_DIR` variable should be set to the root of either the
installation or build directory of LLVM 9. It is used to locate the
corresponding `LLVMConfig.cmake` script that is used to set the include and
library paths.

In order to run the tests, you need to install **llvm-lit** (aka **lit**). It's
not bundled with LLVM 9 packages, but you can install it with **pip**:
```bash
# Install lit - note that this installs lit globally
pip install lit
```
Running the tests is as simple as:
```bash
$ lit <build_dir>/test
```
Voilà! You should see all tests passing.

Overview of The Passes
======================
   * [**HelloWorld**](#helloworld) - prints the name and the number of
     arguments for each function encounterd in the input module
   * [**OpcodeCounter**](#opcodecounter) - prints the summary of the LLVM IR
     opcodes encountered in the input function
   * [**InjectFuncCall**](#injectfunccall) - instruments
     the input module by inserting calls to `printf`
   * [**StaticCallCounter**](#staticcallcounter) - counts
     direct function calls at compile-time
   * [**DynamicCallCounter**](#dynamiccallcounter) - counts
     direct function calls at run-time
   * [**MBASub**](#mbasub) - code transformation for integer `sub`
     instructions
   * [**MBAAdd**](#mbaadd) - code transformation for 8-bit integer `add`
     instructions
   * [**RIV**](#riv) - finds reachable integer values
     for each basic block
   * [**DuplicateBB**](#duplicatebb) - duplicates basic
     blocks, requires **RIV** analysis results

Once you've [built](#build-instructions) this project, you can experiment with
every pass separately. All passes work with LLVM files. You can generate one
like this:
```bash
export LLVM_DIR=<installation/dir/of/llvm/9>
# Textual form
$LLVM_DIR/bin/clang  -emit-llvm input.c -S -o out.ll
# Binary/bit-code form
$LLVM_DIR/bin/clang  -emit-llvm input.c -o out.bc
```
It doesn't matter whether you choose the binary (without `-S`) or textual
form (with `-S`), but obviously the latter is more human-friendly. All passes,
except for [**HelloWorld**](#helloworld), are described below.

## OpcodeCounter
**OpcodeCounter** prints a summary of the
[LLVM IR opcodes](https://github.com/llvm/llvm-project/blob/release/9.x/llvm/lib/IR/Instruction.cpp#L292)
encountered in every function in the input module. This pass is slightly more
complicated than **HelloWorld** and it can be [run
automatically](#run-opcodecounter-automatically). Let's use our tried and
tested method first.

#### Run the pass
We will use
[input_for_cc.c](https://github.com/banach-space/llvm-tutor/blob/master/inputs/input_for_cc.c)
to test **OpcodeCounter**:
```bash
export LLVM_DIR=<installation/dir/of/llvm/9>
# Generate an LLVM file to analyze
$LLVM_DIR/bin/clang  -emit-llvm -c <source_dir>/inputs/input_for_cc.c -o input_for_cc.bc
# Run the pass through opt
$LLVM_DIR/bin/opt -load <build_dir>/lib/libOpcodeCounter.dylib -legacy-opcode-counter input_for_cc.bc
```
For `main` **OpcodeCounter**, prints the following summary (note that when running the pass,
a summary for other functions defined in `input_for_cc.bc` is also printed):
```
=================================================
LLVM-TUTOR: OpcodeCounter results for `main`
=================================================
OPCODE               #N TIMES USED
-------------------------------------------------
load                 2
br                   4
icmp                 1
add                  1
ret                  1
alloca               2
store                4
call                 4
-------------------------------------------------
```

#### Run OpcodeCounter Automatically
**NOTE:** Currently this only works when building LLVM from sources. More
information is available
[here](https://github.com/banach-space/llvm-tutor/blob/master/lib/OpcodeCounter.cpp#L119).

You can configure **llvm-tutor** so that **OpcodeCounter** is run automatically
at any optimisation level (i.e. `-O{0|1|2|3|s}`). This is achieved through
auto-registration with the existing pipelines, which is enabled with the
`LT_OPT_PIPELINE_REG` CMake variable. Simply add `-DLT_OPT_PIPELINE_REG=On`
when [building](#building--testing) **llvm-tutor**.

Now you can run **OpcodeCounter** by specifying an optimisation level. Note
that you still have to specify the plugin file to be loaded:
```bash
$LLVM_DIR/bin/opt -load <build_dir>/lib/libOpcodeCounter.dylib -O1 input_for_cc.bc
```
This registration is implemented in
[OpcodeCounter.cpp](https://github.com/banach-space/llvm-tutor/blob/master/lib/OpcodeCounter.cpp#L123).
Note that for this to work I used the Legacy Pass Manager (the plugin file was
specified with `-load` rather than `-load-pass-plugin`).
[Here](#about-pass-managers-in-llvm) you can read more about pass managers in
LLVM.

## InjectFuncCall
This pass is a _HelloWorld_ example for _code instrumentation_. For every function
defined in the input module, **InjectFuncCall** will add (_inject_) the following
call to [`printf`](https://en.cppreference.com/w/cpp/io/c/fprintf):
```C
printf("(llvm-tutor) Hello from: %s\n(llvm-tutor)   number of arguments: %d\n", FuncName, FuncNumArgs)
```
This call is added at the beginning of each function (i.e. before any other
instruction). `FuncName` is the name of the function and `FuncNumArgs` is the
number of arguments that the function takes.

#### Run the pass
We will use
[input_for_hello.c](https://github.com/banach-space/llvm-tutor/blob/master/inputs/input_for_hello.c)
to test **InjectFuncCall**:

```bash
export LLVM_DIR=<installation/dir/of/llvm/9>
# Generate an LLVM file to analyze
$LLVM_DIR/bin/clang  -emit-llvm -c <source_dir>/inputs/input_for_hello.c -o input_for_hello.bc
# Run the pass through opt
$LLVM_DIR/bin/opt -load <build_dir>/lib/libInjectFuncCall.dylib -legacy-inject-func-call input_for_hello.bc -o instrumented.bin
```
This generates `instrumented.bin`, which is the instrumented version of
`input_for_hello.bc`. In order to verify that **InjectFuncCall** worked as
expected, you can either check the output file (and verify that it contains
extra calls to `printf`) or run it:
```
$LLVM_DIR/bin/lli instrumented.bin
(llvm-tutor) Hello from: main
(llvm-tutor)   number of arguments: 2
(llvm-tutor) Hello from: foo
(llvm-tutor)   number of arguments: 1
(llvm-tutor) Hello from: bar
(llvm-tutor)   number of arguments: 2
(llvm-tutor) Hello from: foo
(llvm-tutor)   number of arguments: 1
(llvm-tutor) Hello from: fez
(llvm-tutor)   number of arguments: 3
(llvm-tutor) Hello from: bar
(llvm-tutor)   number of arguments: 2
(llvm-tutor) Hello from: foo
(llvm-tutor)   number of arguments: 1
```

#### InjectFuncCall vs HelloWorld
You might have noticed that **InjectFuncCall** is somewhat similar to
[**HelloWorld**](#helloworld). In both cases the pass visits all functions,
prints their names and the number of arguments. The difference between the two
passes becomes quite apparent when you compare the output generated for the same
input file, e.g. `input_for_hello.c`. The number of times `Hello from` is
printed is either:
* once per every function call in the case of **InjectFuncCall**, or
* once per function definition in the case of **HelloWorld**.

This makes perfect sense and hints how different the two passes are. Whether to
print `Hello from` is determined at either:
* run-time for **InjectFuncCall**, or
* compile-time for **HelloWorld**.

Also, note that in the case of **InjectFuncCall** we had to first run the pass
with **opt** and then execute the instrumented IR module in order to see the
output.  For **HelloWorld** it was sufficient to run run the pass with **opt**.

## StaticCallCounter
The **StaticCallCounter** pass counts the number of _compile-time_ (i.e. visible
during the compilation) function calls in the input LLVM module. If a function
is called within a loop, that will always be counted as one function call, no
matter how many times the loop iterates. Only direct function calls are counted.

#### Run the pass
We will use
[input_for_cc.c](https://github.com/banach-space/llvm-tutor/blob/master/inputs/input_for_cc.c)
to test **StaticCallCounter**:
```bash
export LLVM_DIR=<installation/dir/of/llvm/9>
# Generate an LLVM file to analyze
$LLVM_DIR/bin/clang  -emit-llvm -c <source_dir>/inputs/input_for_cc.c -o input_for_cc.bc
# Run the pass through opt
$LLVM_DIR/bin/opt -load <build_dir>/lib/libStaticCallCounter.dylib -legacy-static-cc -analyze input_for_cc.bc
```
You will see the following output:
```
=================================================
LLVM-TUTOR: static analysis results
=================================================
NAME                 #N DIRECT CALLS
-------------------------------------------------
bar                  2
fez                  1
foo                  3
```

#### Run the pass through `static`
`static` is an LLVM based tool implemented in
[StaticMain.cpp](https://github.com/banach-space/llvm-tutor/blob/master/tools/StaticMain.cpp).
It is a command line wrapper that allows you to run **StaticCallCounter** without
the need for **opt**:
```bash
<build_dir>/bin/static input_for_cc.bc
```
It is an example of a relatively basic static analysis tool. Its implementation
demonstrates how basic pass management in LLVM works.

## DynamicCallCounter
The **DynamicCallCounter** pass counts the number of _run-time_ (i.e.
encountered during the execution) function calls. It does so by inserting
call-counting instructions that are executed every time a function is called.
Only calls to functions that are _defined_ in the input module are counted.
This pass builds on top of ideas presented in
[**InjectFuncCall**](#injectfunccall). You may want to experiment with that
example first.

#### Run the pass
We will use
[input_for_cc.c](https://github.com/banach-space/llvm-tutor/blob/master/inputs/input_for_cc.c)
to test **DynamicCallCounter**:
```bash
export LLVM_DIR=<installation/dir/of/llvm/9>
# Generate an LLVM file to analyze
$LLVM_DIR/bin/clang  -emit-llvm -c <source_dir>/inputs/input_for_cc.c -o input_for_cc.bc
# Instrument the input file
$LLVM_DIR/bin/opt -load <build_dir>/lib/libDynamicCallCounter.dylib -legacy-dynamic-cc input_for_cc.bc -o instrumented_bin
```
This generates `instrumented.bin`, which is the instrumented version of
`input_for_cc.bc`. In order to verify that **DynamicCallCounter** worked as
expected, you can either check the output file (and verify that it contains
new call-counting instructions) or run it:
```bash
# Run the instrumented binary
$LLVM_DIR/bin/lli ./instrumented_bin
```
You will see the following output:
```
=================================================
LLVM-TUTOR: dynamic analysis results
=================================================
NAME                 #N DIRECT CALLS
-------------------------------------------------
foo                  13
bar                  2
fez                  1
main                 1
```

#### DynamicCallCounter vs StaticCallCounter
The number of function calls reported by **DynamicCallCounter** and
**StaticCallCounter** are different, but both results are correct. They
correspond to _run-time_ and _compile-time_ function calls respectively. Note
also that for **StaticCallCounter** it was sufficient to run the pass through
**opt** to have the summary printed. For **DynamicCallCounter** we had to _run
the instrumented binary_ to see the output. This is similar to what we observed
when comparing [HelloWorld and InjectFuncCall](#injectfunccall-vs-helloworld).

## Mixed Boolean Arithmetic Transformations
These passes implement [mixed
boolean arithmetic](https://tel.archives-ouvertes.fr/tel-01623849/document)
transformations. Similar transformation are often used in code obfuscation (you
may also know them from [Hacker's
Delight](https://www.amazon.co.uk/Hackers-Delight-Henry-S-Warren/dp/0201914654))
and are a great illustration of what and how LLVM passes can be used for.

### MBASub
The **MBASub** pass implements this rather basic expression:
```
a - b == (a + ~b) + 1
```
Basically, it replaces all instances of integer `sub` according to the above
formula. The corresponding LIT tests verify that both the formula  and that the
implementation are correct.

#### Run the pass
We will use
[input_for_mba_sub.c](https://github.com/banach-space/llvm-tutor/blob/master/inputs/input_for_mba_sub.c)
to test **MBASub**:
```bash
export LLVM_DIR=<installation/dir/of/llvm/9>
$LLVM_DIR/bin/clang -emit-llvm -S inputs/input_for_mba_sub.c -o input_for_sub.ll
$LLVM_DIR/bin/opt -load <build_dir>/lib/libMBASub.so -legacy-mba-sub input_for_sub.ll -o out.ll
```

### MBAAdd
The **MBAAdd** pass implements a slightly more involved formula that is only
valid for 8 bit integers:
```
a + b == (((a ^ b) + 2 * (a & b)) * 39 + 23) * 151 + 111
```
Similarly to `MBASub`, it replaces all instances of integer `add` according to
the above identity, but only for 8-bit integers. The LIT tests verify that both
the formula and the implementation are correct.

#### Run the pass
We will use
[input_for_add.c](https://github.com/banach-space/llvm-tutor/blob/master/inputs/input_for_mba.c)
to test **MBAAdd**:
```bash
export LLVM_DIR=<installation/dir/of/llvm/9>
$LLVM_DIR/bin/clang -O1 -emit-llvm -S inputs/input_for_mba.c -o input_for_mba.ll
$LLVM_DIR/bin/opt -load <build_dir>/lib/libMBAAdd.so -legacy-mba-add input_for_mba.ll -o out.ll
```
You can also specify the level of _obfuscation_ on a scale of `0.0` to `1.0`, with
`0` corresponding to no obfuscation and `1` meaning that all `add` instructions
are to be replaced with `(((a ^ b) + 2 * (a & b)) * 39 + 23) * 151 + 111`, e.g.:
```bash
$LLVM_DIR/bin/opt -load <build_dir>/lib/libMBAAdd.so -legacy-mba-add -mba-ratio=0.3 inputs/input_for_mba.c -o out.ll
```

## RIV
**RIV** is an analysis pass that for each [basic
block](http://llvm.org/docs/ProgrammersManual.html#the-basicblock-class) BB in
the input function computes the set reachable integer values, i.e. the integer
values that are visible (i.e. can be used) in BB. Since the pass operates on
the LLVM IR representation of the input file, it takes into account all values
that have [integer type](https://llvm.org/docs/LangRef.html#integer-type) in
the [LLVM IR](https://llvm.org/docs/LangRef.html) sense. In particular, since
at the LLVM IR level booleans are represented as 1-bit wide integers (i.e.
`i1`), you will notice that booleans are also included in the result.

This pass demonstrates how to request results from other analysis passes in
LLVM. In particular, it relies on the [Dominator
Tree](https://en.wikipedia.org/wiki/Dominator_(graph_theory)) analysis pass
from LLVM, which is is used to obtain the dominance tree for the basic blocks
in the input function.

#### Run the pass
We will use
[input_for_riv.c](https://github.com/banach-space/llvm-tutor/blob/master/inputs/input_for_riv.c)
to test **RIV**:
```bash
export LLVM_DIR=<installation/dir/of/llvm/9>
$LLVM_DIR/bin/clang -emit-llvm -S -O1 inputs/input_for_riv.c -o input_for_riv.ll
$LLVM_DIR/bin/opt -load <build_dir>/lib/libRIV.so -legacy-riv inputs/input_for_riv.ll
```
You will see the following output:
```
=================================================
LLVM-TUTOR: RIV analysis results
=================================================
BB id      Reachable Ineger Values
-------------------------------------------------
BB %entry
             i32 %a
             i32 %b
             i32 %c
BB %if.then
               %add = add nsw i32 %a, 123
               %cmp = icmp sgt i32 %a, 0
             i32 %a
             i32 %b
             i32 %c
BB %if.end8
               %add = add nsw i32 %a, 123
               %cmp = icmp sgt i32 %a, 0
             i32 %a
             i32 %b
             i32 %c
BB %if.then2
               %mul = mul nsw i32 %b, %a
               %div = sdiv i32 %b, %c
               %cmp1 = icmp eq i32 %mul, %div
               %add = add nsw i32 %a, 123
               %cmp = icmp sgt i32 %a, 0
             i32 %a
             i32 %b
             i32 %c
BB %if.else
               %mul = mul nsw i32 %b, %a
               %div = sdiv i32 %b, %c
               %cmp1 = icmp eq i32 %mul, %div
               %add = add nsw i32 %a, 123
               %cmp = icmp sgt i32 %a, 0
             i32 %a
             i32 %b
             i32 %c
```

## DuplicateBB
This pass will duplicate all basic blocks in a module, with the exception of
basic blocks for which there are no reachable integer values (identified through
the **RIV** pass). An example of such a basic block is the entry block in a
function that:
* takes no arguments and
* is embedded in a module that defines no global values.

Basic blocks are duplicated by inserting an `if-then-else` construct and
cloning all the instructions (with the exception of [PHI
nodes](https://en.wikipedia.org/wiki/Static_single_assignment_form)) into the
new blocks.

#### Run the pass
This pass depends on the **RIV** pass, hence you need to load it too in order
for **DuplicateBB** to work. We will use
[input_for_duplicate_bb.c](https://github.com/banach-space/llvm-tutor/blob/master/inputs/input_for_duplicate_bb.c)
to test it:
```bash
export LLVM_DIR=<installation/dir/of/llvm/9>
$LLVM_DIR/bin/opt -load <build_dir>/lib/libRIV.so -load <build_dir>/lib/libDuplicateBB.so -riv inputs/input_for_duplicate_bb.c
```

Debugging
==========
Before running a debugger, you may want to analyze the output from
[LLVM_DEBUG](http://llvm.org/docs/ProgrammersManual.html#the-llvm-debug-macro-and-debug-option)
and
[STATISTIC](http://llvm.org/docs/ProgrammersManual.html#the-statistic-class-stats-option)
macros. For example, for **MBAAdd**:
```bash
export LLVM_DIR=<installation/dir/of/llvm/9>
$LLVM_DIR/bin/clang -emit-llvm -S -O1 inputs/input_for_mba.c -o input_for_mba.ll
$LLVM_DIR/bin/opt -load-pass-plugin <build_dir>/lib/libMBAAdd.dylib -passes=mba-add input_for_mba.ll -debug-only=mba-add -stats -o out.ll
```
Note the `-debug-only=mba-add` and `-stats` flags in the command line - that's
what enables the following output:
```bash
  %12 = add i8 %1, %0 ->   <badref> = add i8 111, %11
  %20 = add i8 %12, %2 ->   <badref> = add i8 111, %19
  %28 = add i8 %20, %3 ->   <badref> = add i8 111, %27
===-------------------------------------------------------------------------===
                          ... Statistics Collected ...
===-------------------------------------------------------------------------===

3 mba-add - The # of substituted instructions
```
As you can see, you get a nice summary from **MBAAdd**. In many cases this will
be sufficient to understand what might be going wrong.

For tricker issues just use a debugger. Below I demonstrate how to debug
[**MBAAdd**](#mbaadd). More specifically, how to set up a breakpoint on entry
to `MBAAdd::run`. Hopefully that will be sufficient for you to start.

## Mac OS X
The default debugger on OS X is [LLDB](http://lldb.llvm.org). You will
normally use it like this:
```bash
export LLVM_DIR=<installation/dir/of/llvm/9>
$LLVM_DIR/bin/clang -emit-llvm -S -O1 inputs/input_for_mba.c -o input_for_mba.ll
lldb -- $LLVM_DIR/bin/opt -load-pass-plugin <build_dir>/lib/libMBAAdd.dylib -passes=mba-add input_for_mba.ll -o out.ll
(lldb) breakpoint set --name MBAAdd::run
(lldb) process launch
```
or, equivalently, by using LLDBs aliases:
```bash
export LLVM_DIR=<installation/dir/of/llvm/9>
$LLVM_DIR/bin/clang -emit-llvm -S -O1 inputs/input_for_mba.c -o input_for_mba.ll
lldb -- $LLVM_DIR/bin/opt -load-pass-plugin <build_dir>/lib/libMBAAdd.dylib -passes=mba-add input_for_mba.ll -o out.ll
(lldb) b MBAAdd::run
(lldb) r
```
At this point, LLDB should break at the entry to `MBAAdd::run`.

## Ubuntu
On most Linux systems, [GDB](https://www.gnu.org/software/gdb/) is the most
popular debugger. A typical session will look like this:
```bash
export LLVM_DIR=<installation/dir/of/llvm/9>
$LLVM_DIR/bin/clang -emit-llvm -S -O1 inputs/input_for_mba.c -o input_for_mba.ll
gdb --args $LLVM_DIR/bin/opt -load-pass-plugin <build_dir>/lib/libMBAAdd.so -passes=mba-add input_for_mba.ll -o out.ll
(gdb) b MBAAdd.cpp:MBAAdd::run
(gdb) r
```
At this point, GDB should break at the entry to `MBAAdd::run`.

About Pass Managers in LLVM
===========================
LLVM is a quite complex project (to put it mildly) and passes lay at its
center - this is true for any [multi-pass
compiler](https://en.wikipedia.org/wiki/Multi-pass_compiler<Paste>). In order
to manage the passes, a compiler needs a pass manager. LLVM currently enjoys
not one, but two pass managers. This is important because depending on which
pass manager you decide to use, the implementation of your pass (and in
particular how you _register_ it) will look slightly differently.

## Overview of Pass Managers in LLVM
As I mentioned earlier, there are two pass managers in LLVM:
* _Legacy Pass Manager_ which currently is the default pass manager
	* It is implemented in the _legacy_ namespace
	* It is very well [documented](http://llvm.org/docs/WritingAnLLVMPass.html)
		(more specifically, writing and registering a pass withing the Legacy PM is
		very well documented)
* _New Pass Manager_ aka [_Pass Manager_](https://github.com/llvm-mirror/llvm/blob/ff8c1be17aa3ba7bacb1ef7dcdbecf05d5ab4eb7/include/llvm/IR/PassManager.h#L458) (that's how it's referred to in the code base)
	* I understand that it is [soon to become](http://lists.llvm.org/pipermail/llvm-dev/2019-August/134326.html) the default pass manager in LLVM
	* The source code is very throughly commented, but there is no official documentation. Min-Yih Hsu kindly wrote
		this great [blog series](https://medium.com/@mshockwave/writing-llvm-pass-in-2018-preface-6b90fa67ae82) that you can refer to instead.

If you are not sure which pass manager to use, it is probably best to make sure
that your passes are compatible with both. Fortunately, once you have an
implementation that works with one of them, it's relatively straightforward to
extend it so that it works with the other one as well.

## New vs Legacy PM When Running Opt
**MBAAdd** implements interface for both pass managers. This is how you will
use it with the legacy pass manager:
```bash
$LLVM_DIR/bin/opt -load <build_dir>/lib/libMBAAdd.so -legacy-mba-add input_for_mba.ll -o out.ll
```

And this is how you run it with the new pass manager:
```bash
$LLVM_DIR/bin/opt -load-pass-plugin <build_dir>/lib/libMBAAdd.so -passes=mba-add input_for_mba.ll -o out.ll
```
There are two differences:
* the way you load your plugin: `-load` vs `-load-pass-plugin`
* the way you specify which pass/plugin to run: `-legacy-mba-add` vs
  `-passes=mba-add`

These differences stem from the fact that in the case of Legacy Pass Manager you
register a new command line option for **opt**, whereas  New Pass Manager
simply requires you to define a pass pipeline (with `-passes=`).

Acknowledgments
===============
This is first and foremost a community effort. This project wouldn't be
possible without the amazing LLVM [online
documentation](http://llvm.org/docs/), the plethora of great comments in the
source code, and the llvm-dev mailing list. Thank you!

It goes without saying that there's plenty of great presentations on YouTube,
blog posts and GitHub projects that cover similar subjects. I've learnt a great
deal from them - thank you all for sharing! There's one presentation/tutorial
that has been particularly important in my journey as an aspiring LLVM
developer and that helped to _democratise_ out-of-source pass development:
* "Building, Testing and Debugging a Simple out-of-tree LLVM Pass" Serge
  Guelton, Adrien Guinet
  ([slides](https://llvm.org/devmtg/2015-10/slides/GueltonGuinet-BuildingTestingDebuggingASimpleOutOfTreePass.pdf),
  [video](https://www.youtube.com/watch?v=BnlG-owSVTk&index=8&list=PL_R5A0lGi1AA4Lv2bBFSwhgDaHvvpVU21))

Adrien and Serge came up with some great, illustrative and self-contained
examples that are great for learning and tutoring LLVM pass development. You'll
notice that there are similar transformation and analysis passes available in
this project. The implementations available here reflect what **I** (aka
banach-space) found most challenging while studying them.

I also want to thank Min-Yih Hsu for his [blog
series](https://medium.com/@mshockwave/writing-llvm-pass-in-2018-preface-6b90fa67ae82)
_"Writing LLVM Pass in 2018"_. It was invaluable in understanding how the new
pass manager works and how to use it. Last, but not least I am very grateful to
[Nick Sunmer](https://www.cs.sfu.ca/~wsumner/index.html) (e.g.
[llvm-demo](https://github.com/nsumner/llvm-demo)) and [Mike
Shah](http://www.mshah.io) (see Mike's Fosdem 2018
[talk](http://www.mshah.io/fosdem18.html)) for sharing their knowledge online.
I have learnt a great deal from it, thank you! I always look-up to those of us
brave and bright enough to work in academia - thank you for driving the
education and research forward!

References
==========
Below is a list of LLVM resources available outside the official online
documentation that I have found very helpful. Where possible, the items are sorted by
date.

* **LLVM IR**
  *  _”LLVM IR Tutorial-Phis,GEPs and other things, oh my!”_, V.Bridgers, F.
Piovezan, EuroLLVM 2019, ([slides](https://llvm.org/devmtg/2019-04/slides/Tutorial-Bridgers-LLVM_IR_tutorial.pdf),
  [video](https://www.youtube.com/watch?v=m8G_S5LwlTo&feature=youtu.be))
  * _"Mapping High Level Constructs to LLVM IR"_, M. Rodler ([link](https://mapping-high-level-constructs-to-llvm-ir.readthedocs.io/en/latest/))
* **Legacy vs New Pass Manager**
  * _"New PM: taming a custom pipeline of Falcon JIT"_, F. Sergeev, EuroLLVM 2018
    ([slides](http://llvm.org/devmtg/2018-04/slides/Sergeev-Taming%20a%20custom%20pipeline%20of%20Falcon%20JIT.pdf),
     [video](https://www.youtube.com/watch?v=6X12D46sRFw))
  * _"The LLVM Pass Manager Part 2"_, Ch. Carruth, LLVM Dev Meeting 2014
    ([slides](https://llvm.org/devmtg/2014-10/Slides/Carruth-TheLLVMPassManagerPart2.pdf),
     [video](http://web.archive.org/web/20160718071630/http://llvm.org/devmtg/2014-10/Videos/The%20LLVM%20Pass%20Manager%20Part%202-720.mov))
  * _”Passes in LLVM, Part 1”_, Ch. Carruth, EuroLLVM 2014 ([slides](https://llvm.org/devmtg/2014-04/PDFs/Talks/Passes.pdf), [video](https://www.youtube.com/watch?v=rY02LT08-J8))
* **Examples in LLVM**
	* The Hello example: [llvm/lib/Transforms/Hello](https://github.com/llvm/llvm-project/tree/release/10.x/llvm/lib/Transforms/Hello)
	* The Bye example: [llvm/examples/Bye](https://github.com/llvm/llvm-project/tree/release/10.x/llvm/examples/Bye)
	* Control flow graph (CFG) simplifications (presented in [this tutorial](https://www.youtube.com/watch?v=3QQuhL-dSys&t=826s)): [llvm/examples/IRTransforms/](https://github.com/llvm/llvm-project/tree/release/10.x/llvm/examples/IRTransforms)
* **LLVM Pass Development**
  * _"Getting Started With LLVM: Basics "_, J. Paquette, F. Hahn, LLVM Dev Meeting 2019 [video](https://www.youtube.com/watch?v=3QQuhL-dSys&t=826s)
  * _"Writing an LLVM Pass: 101"_, A. Warzyński, LLVM Dev Meeting 2019 [video](https://www.youtube.com/watch?v=ar7cJl2aBuU)
  * _"Writing LLVM Pass in 2018"_, Min-Yih Hsu, [blog series](https://medium.com/@mshockwave/writing-llvm-pass-in-2018-preface-6b90fa67ae82)
  * _"Building, Testing and Debugging a Simple out-of-tree LLVM Pass"_ Serge Guelton, Adrien Guinet, LLVM Dev Meeting 2015 ([slides](https://llvm.org/devmtg/2015-10/slides/GueltonGuinet-BuildingTestingDebuggingASimpleOutOfTreePass.pdf), [video](https://www.youtube.com/watch?v=BnlG-owSVTk&index=8&list=PL_R5A0lGi1AA4Lv2bBFSwhgDaHvvpVU21))
* **LLVM Based Tools Development**
  * _"Introduction to LLVM"_, M. Shah, Fosdem 2018, [link](http://www.mshah.io/fosdem18.html)
  *  [llvm-demo](https://github.com/nsumner/llvm-demo), by N Sumner
  * _"Building an LLVM-based tool. Lessons learned"_, A. Denisov, [blog post](https://lowlevelbits.org/building-an-llvm-based-tool.-lessons-learned/), [video](https://www.youtube.com/watch?reload=9&v=Yvj4G9B6pcU)

License
========
The MIT License (MIT)

Copyright (c) 2019 Andrzej Warzyński

Permission is hereby granted, free of charge, to any person obtaining a copy of
this software and associated documentation files (the "Software"), to deal in
the Software without restriction, including without limitation the rights to
use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies
of the Software, and to permit persons to whom the Software is furnished to do
so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
