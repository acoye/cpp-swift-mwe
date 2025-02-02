# cpp-swift-mwe

fork: Bump to SPM 5.3

A minimal example that investigates how to build a Swift project that depends on a compiled, non-system C++ library.

Built from [@aciidb0mb3r](https://github.com/aciidb0mb3r)'s [blog post](http://ankit.im/swift/2016/05/21/creating-objc-cpp-packages-with-swift-package-manager/).

All of this is executed with

    Apple Swift Package Manager - Swift 3.0.2 (swiftpm-11750)

This is the latest Toolchain available through Apple's channels at the time of
this writing.


## Starting Point

Following the blog post closely, the initial project (commit 565752f6e27633b41b6c62c1ead700ba4e8d7d95)
is structured as follows:
the Swift module depends on a C wrapper around a C++ module with sources.

`xcrun swift run` will prints `5 -- all the way from C++!`.


## Goal

The ultimate goal is to use a C++ library -- let's call it C -- in Swift.

 * C is open source.
 * We can not expect C to be available on target systems.
 * We want to use a custom (minimal, platform specific) build of C.

Thus, as long as SwiftPM does not allow us to specify custom build instructions
for dependencies,
we have to supply the library as binary file, with headers to compile against.
(In settings where sharing the sources of C is not an option, the need is even more immediate.)

Ideally, we want to receive a build result that can be easily referenced
in other Swift projects, in particular such developed in XCode.

In case that is relevant, the library we want to build is to be used by
iOS apps.


## Attempt 1: No C++ sources, binary library

Add `libcpplib.dylib` built with the Starting Point configuration.
Remove `Sources/cpplib/cpplib.cpp`.
Add `build.sh` for convenience.  
(Forgot to exclude the non-module stuff, but does not matter here; see Attempt 2.)

Commit 85700aa9aacca73b13e2082bba68853a13b37add

Build with command:

~~~bash
swift build -Xlinker -L/path/to/Dependencies \
            -Xlinker -lcpplib
~~~

Output:

~~~
error: the module at /path/to/cpp-swift-mwe/Sources/cpplib
       does not contain any source files
fix:   either remove the module folder, or add a source file to the module
~~~


## Attempt 2: Dummy C++ sources, binary library

Add an empty file `Sources/cpplib/empty.cpp`.
Update `Package.swift` to exclude `Dependencies`, `build.sh`.

Commit 7db39811f72128db9cc6cf77ee15b5c70c19b32a

Build with the same command as Attempt 1. Output:

~~~
Compile cpplib empty.cpp
Linking cpplib
Linking cwrapper
Undefined symbols for architecture x86_64:
  "cpplib::five()", referenced from:
      _cwrapperfive in cwrapper.cpp.o
ld: symbol(s) not found for architecture x86_64
clang: error: linker command failed with exit code 1 (use -v to see invocation)
<unknown>:0: error: build had 1 command failures
~~~

Apparently, the folder provided by `-L` does not take precedence as the
documentation promises.


## Interlude: A Workaround

The problem is with who looks for libraries where.

@aciidb0mb3r proposes a workaround in
[swiftpm.slack](https://swiftpm.slack.com/archives/help/p1486035484001308):

> Run these two commands once:
>
> ~~~bash
> mv Dependencies/libcpplib.dylib Dependencies/libcpplibVendored.dylib
> install_name_tool -id @executable_path/../../Dependencies/libcpplibVendored.dylib \
>                   Dependencies/libcpplibVendored.dylib
> ~~~
>
> Then use option `-Xlinker -lcpplibVendored` instead of `-Xlinker -lcpplib`.

This fixes the build but does not generate a shippable product,
since in production the paths will be different.

I filed a [feature request](https://bugs.swift.org/browse/SR-3832) towards
support of non-system binary dependencies.

## Attempt 3: A Makefile

Needed to do [this](http://stackoverflow.com/a/16058799/539599), but as `.cpp`.

Then, it is mostly copying `.build/debug.yaml` (for now; release later) from Starting Point
into another build tool and replace compilation of the C++ library with some copying
instructions.

In order to make the process less arduous the next time around, I have built
    [a `configure` script](https://github.com/reitzig/cpp-swift-mwe/blob/master/configure.rb)
that should deal with other projects that use the same structure.

This now works:

~~~bash
$ make clean
$ ./configure.rb
$ make
$ .build/debug/swift
5 -- all the way from C++!
~~~

### Open Problems

 * Does this work with `libswift.a` just as well?
 * Generated modulemap files are probably broken if there is more than one C header.
 * Will need adapting for building Swift libraries.
 * Will probably not work properly if the (Swift) source folders contain subfolders.
 * Will need adapting for building for iOS.
 * `configure.rb` has its parameters hard-coded for now.


## Thoughts

It seems clear that the current design of SwiftPM does not consider dependencies
that are provided as non-system binaries.
Maybe there should be a protocol `Dependency` -- currently `Package.dependencies`
is of type `[Package]`! -- with a new subtype `Binary` with initializer

~~~swift
Binary(binary: String, header: String)
~~~

or similar. Note that the file may have to depend on the target platform and
architecture.  
(Would we need parameters for the header language and whether to
integrate the binary into the package binary?)

Could this be implemented by SwiftPM by creating pkgconfig files?
