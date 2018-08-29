# Simple C++ Symbol Visibility Demo

A demonstration/test of the dangers of using default visibility for symbols in C++ shared libraries.

## Introduction

This document briefly walks through why and how to use the `-fvisibility=hidden` compilation flag along with `__attribute__((__visibility__("default")))` in symbol declarations.

On Linux, symbols have a default visibility that allows them to be loaded at runtime. If multiple libraries export the same symbol, this can lead to undefined behavior. In real terms, this means segmentation faults, deadlocks, or other quirky behavior can occur. While multiple solutions to this problem exist, this document discusses how to prevent this behavior by specifying symbol visibility during shared library compilation.

## Prerequisites

To run the demo, gcc and Make are required.

## An Example

To run the demonstration, run `demonstrate.sh`. This will build and run the demo twice -- building once with default visibility and building once with hidden visibility. On Linux, default visibility causes `libget_three.so` and `libget_seven.so` to effectively share the same definition for `internal_do_calculation()`, which produces incorrect results as far as the author of `PublicGetSeven()` is concerned:

```
PublicGetThree returned 3
PublicGetSeven returned 3
```

Hidden visibility causes the libraries to use their own definitions of `internal_do_calculation()`, so running the `.so`s with hidden visibility outputs correct results:

```
PublicGetThree returned 3
PublicGetSeven returned 7
```

### In Detail

`get_three.cpp` and `get_seven.cpp` are the implementations of two shared libraries, respectively named `libget_three.so` and `libget_seven.so`. `get_three.cpp` and `get_seven.cpp` each define a function with the signature `int internal_do_calculation()`. Meanwhile, `test.cpp` links to both of these libraries and calls functions from their public APIs, respectively named `PublicGetThree()` and `PublicGetSeven()` -- both of these functions call `internal_do_calculation()` to get the number to be returned:

 - `internal_do_calculation()` in `get_three.cpp` returns 3
 - `internal_do_calculation()` in `get_seven.cpp` returns 7

Naively, it seems like this should work, since each shared library doesn't know about the other shared library's version of `internal_do_calculation()`. However, on Linux, when the `test` program is executed (without mitigating factors) the following events occur at runtime:

 - the dynamic loader looks for `PublicGetThree` and finds it in `libget_three.so`
 - the dynamic loader looks for `PublicGetSeven` and finds it in `libget_seven.so`
 - the dynamic loader looks for `internal_do_calculation` and in this case finds it in `libget_three.so`

For the purposes of this demonstration, the loader is done at this point and doesn't care that `libget_seven.so` will call `internal_do_calculation()` as defined in `libget_three.so` instead of its own `internal_do_calculation()`:

 - `main()` in `test` calls:
 - `PublicGetSeven()` in `libget_seven.so`, which calls:
 - `internal_do_calculation()` in `libget_three.so`

In real-world situations, this can be problematic, especially since common libraries may be statically compiled into multiple shared libraries, such as via .a archive files or header-only include files. Conflicting versions of such common libraries can be loaded at runtime, resulting in segfaults, deadlocks, and other undefined behavior. For example, [`libLabJackM.so`](https://labjack.com/ljm) (tested with version 1.18.5) and [`libroscpp.so`](http://wiki.ros.org/roscpp) (tested with version Kinetic Kame) both contain external symbol references related to the [Boost C++ libraries](https://www.boost.org/). Specifically, `boost::thread_group` can conflict at runtime, causing deadlocks to occur. (`libLabJackM.so` version 1.18.6 and later is compiled with hidden visibility to solve this issue.)

### Solution: `-fvisibility=hidden` and `__attribute__((__visibility__("default")))`

To solve the above problem using hidden visibility:

 1. Use the `-fvisibility=hidden` compilation flag to compile all symbols with hidden visibility (unless a symbol is explicitly specified to be public).
 2. Use the `__attribute__((__visibility__("default")))` attribute to explicitly define the public API as public.

#### Part 1: `-fvisibility=hidden`

Hidden visibility for a given symbol causes a shared library to never load that hidden symbol from any other component's symbol definitions, but instead load that symbol from the shared library's own symbol definitions. Additionally, symbols with hidden visibility are not exported -- other components will not be able to use them. (More on this below.)

In the second build of `demonstrate.sh`, the libraries are compiled with `-fvisibility=hidden` to make all symbols hidden (except for symbols that have been explicitly marked with a different visibility). This means that at runtime:

 - `libget_three.so` will load its own `internal_do_calculation()` and
 - `libget_seven.so` will load its own `internal_do_calculation()`.

#### Part 2: `__attribute__((__visibility__("default")))`

Outside a component, trying to explicitly call a function with hidden visibility results in undefined reference errors. To get around this and allow a shared library's public API functions to be called, the visibility for those chosen public functions should be set to the default visibility.

("Default" is an unfortunate name for this visibility type since the `-fvisibility` compiler flag sort of sets a default visibility type -- it sets the visibility type for all symbols that don't have a more explicit visibility type. In other words, it may be helpful to think of the "default" visibility type as the visible or exported visibility type, while the `-fvisibility` flag sets the default visibility type.)

To expose a given symbol, you can use `__attribute__((__visibility__("default")))` to decorate a symbol, e.g.:

```
__attribute__((__visibility__("default"))) int PublicGetSeven();
```

Other methods to specify visibility exist, such as using `#pragma` directives. See below for more details.

#### Benefit: Compatibility

Compiling with hidden visibility means that:

 - Symbols definitions are assumed to be located within the component using those symbols
 - Symbols are not exported

Therefore, if a shared library is compiled with hidden visibility:

 - It won't accidentally use symbols from another component
 - It won't let its own symbols be accidentally used by another component

This is an advantage over `-Bsymbolic` or `-Bsymbolic-functions` because it doesn't matter as much if other shared libraries adhere to correct symbol visibility.

#### Benefit: Optimizations

Another benefit to hidden symbols is that additional optimizations can be made. In short:

 - Library load time is improved
 - Runtime speed can be improved because the compiler knows that it can optimize via devirtualization and inlining of functions
 - Shared library binary size can be reduced because the compiler can omit hidden symbols from the exported symbols table

#### Downside: Unit Testing is Harder

Since symbols are not exported when using hidden visibility, unit testing of internal functions requires building the library with default visibility. This may not be much of a problem, however, since there are different compilation flags that you'd want to apply to debug and release builds.

Example debug build flags:

 - `-g` - add debug info
 - `-O0` - no optimization (improves experience using debuggers)

 Example release build flags:

 - `-fvisibility=hidden`
 - `-O2` - moderate size and speed optimizations (many other optimization options exist as well)

Alternately to building for both debug and release, source code could be compiled into a single component with unit test code, so that compiled unit test code has access to compiled source code symbols.

#### Downside: Problems with C++ exceptions

To use hidden visibility, any defined type that is passed out of a shared library must have exported symbols (i.e. default visibility), including exceptions. This is because when binary code in a given component catches an exception, the exception requires a typeinfo lookup. However, typeinfo symbols for symbols with hidden visibility are also hidden.

In short, hidden visibility can still be used with C++ exceptions, but if the shared library's API conceptually does not include exception throwing, it's even simpler because there's no danger of throwing an exception of a hidden type.

### On macOS

`-fvisibility=hidden` is not needed to fix this on macOS. From the [GNU documentation](https://gcc.gnu.org/onlinedocs/gcc-4.6.4/gcc/Function-Attributes.html):

```
On Darwin, default visibility means that the declaration is visible to other modules.
```

This is in contrast to ELF format (i.e. Linux), which allows declarations to also be overridden:

```
On ELF, default visibility means that the declaration is visible to other modules and, in shared libraries, means that the declared entity may be overridden.
```

Running `demonstrate.sh` on macOS produces the following output for both runs of `test`, with and without hidden visibility:

```
PublicGetThree returned 3
PublicGetSeven returned 7
```

### LD_PRELOAD

The environment variable LD_PRELOAD can be used to force the runtime loader to load symbols from one library instead of others. However:

 - LD_PRELOAD is ignored for symbols that were compiled with hidden visibility.
 - LD_PRELOAD is ignored for libraries that were linked with `-Bsymbolic`/`-Bsymbolic-functions`.

## Other Solutions

Other solutions include:

 - The `-Bsymbolic` Linker Flag(s)
 - `-fvisibility=protected`
 - Namespaces
 - Dynamic Linking

### The `-Bsymbolic` Linker Flag(s)

The `-Bsymbolic` or `-Bsymbolic-functions` linker flags (usually run as `-Wl,-Bsymbolic` or `-Wl,-Bsymbolic-functions`) is another way to solve this issue. For example, you could compile and link this demo as follows:

```
g++ -g -Wall -o get_three.o -c -fPIC get_three.cpp
g++ -Wl,-Bsymbolic -o libget_three.so -shared get_three.o
g++ -g -Wall -o get_seven.o -c -fPIC get_seven.cpp
g++ -Wl,-Bsymbolic -o libget_seven.so -shared get_seven.o
g++ -g -Wall -Wl,-rpath=. -o test test.cpp -L. -lget_three -lget_seven
```

This would produce the following (desired) output when running `./test`:

```
PublicGetThree returned 3
PublicGetSeven returned 7
```

One advantage hidden visibility has over `-Bsymbolic` or `-Bsymbolic-functions` is that `-Bsymbolic` or `-Bsymbolic-functions` still export their symbols, so another library can accidentally use them. For example, while compiling `libget_seven.so` with `-Bsymbolic` may give the correct results:

```
...
g++ -o libget_three.so -shared get_three.o
g++ -o libget_seven.so -shared get_seven.o -Wl,-Bsymbolic
g++ -g -Wall -Wl,-rpath=. -o test test.cpp -L. -lget_three -lget_seven
./test
PublicGetThree returned 3
PublicGetSeven returned 7
```

But if the linking order while linking `test` is swapped so that `libget_seven.so` is seen be the loader first,, `./test` gives incorrect results:

```
g++ -g -Wall -Wl,-rpath=. -o test test.cpp -L. -lget_seven -lget_three
./test
PublicGetThree returned 7
PublicGetSeven returned 7
```

This is not a problem when compiling with hidden visibility -- either `get_three.o` or `get_seven.o` could be compiled with `-fvisibility=hidden` and the correct results are still achieved regardless.

Another advantage hidden visibility has over `-Bsymbolic` or `-Bsymbolic-functions` is that hidden visibility helps enable additional size and speed optimizations

An advantage `-Bsymbolic` or `-Bsymbolic-functions` has over hidden visibility is that unit testing internal functions can be done without building the library twice.

### `-fvisibility=protected`

Protected visibility is similar to hidden visibility in that it binds symbol references within a component so that a shared library compiled with `-fvisibility=protected` will not have its symbols overridden. However, protected visibility is similar to default visibility in that it can still "pollute" the symbol table so that unaware libraries might still get their symbols overridden.

For example this link order is successful:

```
g++ -g -Wall -o get_three.o -c -fPIC get_three.cpp
g++ -g -Wall -o get_seven.o -c -fPIC get_seven.cpp -fvisibility=protected
...
g++ -g -Wall -Wl,-rpath=. -o test test.cpp -L. -lget_three -lget_seven
./test
PublicGetThree returned 3
PublicGetSeven returned 7
```

But this link order is not:

```
g++ -g -Wall -o get_three.o -c -fPIC get_three.cpp -fvisibility=protected
g++ -g -Wall -o get_seven.o -c -fPIC get_seven.cpp
...
g++ -g -Wall -Wl,-rpath=. -o test test.cpp -L. -lget_three -lget_seven
./test
PublicGetThree returned 3
PublicGetSeven returned 3
```

### Namespaces

Namespaces can solve this issue if you have control of the source code being compiled. However, if you're writing a shared library that statically links a popular dependent library, your library's version of a function defined in that popular library may conflict with another shared library's version of that popular library. For an example, see above for the problem with `libLabJackM.so` (pre 1.18.6) and `libroscpp.so` both statically linking Boost headers.

In such a case, hidden visibility circumvents the need to modify third-party code or third-party binaries.

### Dynamic Linking

If libraries A and B both use a dependent library C, it can be a problem if A and B use different versions of C, because the internal implementation in C may have changed such that problems occur. Imagine an old version of library C defines a given data structure one way and a newer version of library C defines it in a different way. It could be that the size of a given data structure changes or the order of data fields in that data structure. In this case, if an instance of that data structure is initialized in the older version of library C and used in the newer version of library C (or vice versa), problems will occur.

However, if A and B are both compatible with a given major version of library C, dynamic linking solves this issue. All data structures will be initialized and used with the same version of library C.

In some cases, dynamic linking can present a disadvantage. One reason to statically link is to have minimal dependencies for the sake of simple distribution. Another reason is that it's difficult to test all versions of a dependency, so statically linking to a particular version of a dependency allows for consistent behavior of that dependency.

## More Details

This document is narrowly focused. More details have been documented elsewhere.

 - More on visibility: [https://gcc.gnu.org/wiki/Visibility](https://gcc.gnu.org/wiki/Visibility)
 - More on writing a shared library: [https://www.akkadia.org/drepper/dsohowto.pdf](https://www.akkadia.org/drepper/dsohowto.pdf)
 - More on symbol loading: [https://flameeyes.blog/2012/10/07/symbolism-and-elf-files-or-what-does-bsymbolic-do/](https://flameeyes.blog/2012/10/07/symbolism-and-elf-files-or-what-does-bsymbolic-do/)
 - More on compiler optimizations when using `-fvisibility=hidden`: [https://kristerw.blogspot.com/2016/11/inlining-shared-libraries-are-special.html](https://kristerw.blogspot.com/2016/11/inlining-shared-libraries-are-special.html)
