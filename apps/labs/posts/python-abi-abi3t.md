---
title: 'What Every Python Developer Should Know About the CPython ABI'
authors: [nathan-goldbaum]
published: June 4, 2026
description: 'An introduction to the concept of the Application Binary Interface (ABI), the various CPython ABIs, and the new abi3t stable ABI in Python 3.15.'
category: [PyData ecosystem]
featuredImage:
  src: /posts/python-abi-abi3t/puzzle.png
  alt: 'A diagram showing three horizontally interlocking puzzle pieces illustrating how Python can call into a compiled extension. The puzzle piece on the left is purple and is labeled in a monospace font with `$ python run.py`. The middle piece is blue and labeled "CPython ABI" in a handwritten font. The right piece is green and labeled "Compiled Extension Module".'
hero:
  imageSrc: /posts/python-abi-abi3t/hero.png
  imageAlt: 'A diagram showing three horizontally interlocking puzzle pieces illustrating how Python can call into a compiled extension. The puzzle piece on the left is purple and is labeled in a monospace font with `$ python run.py`. The middle piece is blue and labeled "CPython ABI" in a handwritten font. The right piece is green and labeled "Compiled Extension Module".'
---

# What Every Python Developer Should Know About the CPython ABI

The CPython Application Binary Interface (ABI) backs Python's main superpower: the ability to easily call into native C, C++, Rust, or Fortran code and for that code to call back into the interpreter and update the state of Python objects.
But what exactly is an ABI? How does it differ from an Application Programming Interface (API)?
Why did I begin this post's the title with "What _Every_ Python Developer Should Know"?
Aren't these low-level details the kind of thing we can ignore most of the time in a high-level language like Python?

In this post I hope to answer all these questions and build up your intuition about these topics.
I also hope you'll learn some useful information about how Python projects that include native extensions are distributed, what the ABI compatibility tags that show up in wheel filenames mean, and how projects can choose to target different python ABIs depending on the tradeoffs they want to make.

## The CPython interpreter runtime and C API

The Python interpreter does a bit of a magic trick when you execute a script
like in the cartoon below. The `np.array` function can be _called_ by Python
code, but it turns out this function is [implemented in
C](https://github.com/numpy/numpy/blob/f8c34f2927ba812a3efe9bc978d84aa47f27bff7/numpy/_core/src/multiarray/multiarraymodule.c#L1720). The interpreter runtime handles this transparently.

 <figure style={{ textAlign: 'center' }}>
   <img
     src="/posts/python-abi-abi3t/cpython_runtime_diagram.png"
     alt="A cartoon showing a Python script with NumPy code calling into a cloud representing the CPython interpreter runtime which in turn calls into a NumPy C extension."
     style={{position:'relative',left:'12%',width:'70%'}}
   />
 </figure>

How does it achieve this trick?
Via the [CPython C API](https://docs.python.org/3/c-api/index.html) and the corresponding [Python ABI](https://docs.python.org/3/c-api/stable.html#c-api-stability).
But what does that mean exactly and what even is a C API and how does it differ from the ABI?
It certainly doesn't help things that the two related and intertwined concepts are generally referred using very similar-sounding acronyms.

[CPython](https://github.com/python/cpython), as the name suggests, is implemented in the C programming language.
It began as a research project and was written to be easy to extend with new functionality.
To access the internals of the interpreter, one needed only to `#include "Python.h"` in a C program, which gave rich access to the internal state of the interpreter and hooks to call into the interpreter via the CPython C API, the very same C API that the core interpreter implementation is based on.

What is the C API exactly?
It's precisely the set of C macros, typedefs, functions, and structs exposed by the header that defines the C interface: `Python.h`
This constitutes an enormous number of symbols that collectively allow developers to write code in C that allows similar functionality to any arbitrary Python script.
The [Cython](https://cython.readthedocs.io/en/latest/) programming language takes advantage of this by (among other things) literally compiling Python code to a C "script" that when linked into a larger program behaves identically to the Python code it was compiled from.
Python JITs like [Numba](https://numba.pydata.org), [torch.compile](https://docs.pytorch.org/tutorials/intermediate/torch_compile_tutorial.html), and other third-party Python JITs do a similar trick at runtime, also via the C API.

What isn't the C API?
The C API is purely a construct of the C programming language.
Code written in languages that aren't C can call into a C API, but only be using the conventions of the C programming language and abstracting away functionality that is not expressible in C.
The C API contains minimal platform-specific details (besides 32 bit/64 bit differences), and attempts to make it possible for the same C code to compile machine code that functions identically across a wide variety of architectures despite substantial platform-specific differences in the machine code itself.

## The Python ABI

The ABI governs details of how exactly the C API works on each architecture - how a compiler translates calling a C function like [`PyDict_GetItemRef`](https://docs.python.org/3/c-api/dict.html#c.PyDict_GetItemRef) into a concrete set of machine code that sets CPU registers and other platform-specific details per the specification of whatever platform the code ultimately runs on.

### What is an ABI?

The Python interpreter is not special in how it behaves: all C programs running are translated from an abstract set of source code defined by an API into a concrete set of machine code that obey the platform ABI.

An ABI determines things like the exact way machine code needs to pass arguments to a function. For example, on the [`x86_64`](https://wiki.osdev.org/System_V_ABI#x86-64) architecture, only the first six non-floating-point arguments of a function are passed via registers, the remaining arguments are passed via the stack.
Normally, developers do not need to worry about details like this: compilers automatically generate machine code appropriate for whatever ABI a developer wants to target.
However, it _is_ important to know that different platforms and CPU architectures have unique ABIs and that code compiled for one ABI is completely unusable on another ABI.
This fact determines much of the design of the [binary wheel distribution format](https://packaging.python.org/en/latest/specifications/binary-distribution-format/), which we'll discuss in more detail below.
The biggest consequence is that each platform and CPU architecture, each with its own distinct ABI, requires their own unique builds.
This is one reason projects like `NumPy` distribute so many binary wheels with each release: each ABI needs its own full set of wheels for e.g. Python 3.12, 3.13, and newer that the project wants to support!

### ABI tags and wheel filenames: understanding wheel compatibility

Most popular Python packages ship binaries to the [Python Package Index](https://pypi.python.org) (PyPI) in the form of binary wheels. A wheel is a zip-compressed folder containing Python code and, optionally, compiled artifacts.

Other posts go into this in a lot more detail, but here we're particularly concerned with the filename of a wheel file. Let's consider what happens when I `pip install cryptography` on my development environment running on an ARM Mac laptop:

```bash
$ pip install cryptography
Collecting cryptography
  Downloading cryptography-48.0.0-cp311-abi3-macosx_10_9_universal2.whl.metadata (4.3 kB)
Collecting cffi>=2.0.0 (from cryptography)
  Downloading cffi-2.0.0-cp314-cp314-macosx_11_0_arm64.whl.metadata (2.6 kB)
Collecting pycparser (from cffi>=2.0.0->cryptography)
  Downloading pycparser-3.0-py3-none-any.whl.metadata (8.2 kB)
Downloading cryptography-48.0.0-cp311-abi3-macosx_10_9_universal2.whl (8.0 MB)
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 8.0/8.0 MB 11.6 MB/s  0:00:00
Downloading cffi-2.0.0-cp314-cp314-macosx_11_0_arm64.whl (181 kB)
Downloading pycparser-3.0-py3-none-any.whl (48 kB)
Installing collected packages: pycparser, cffi, cryptography
Successfully installed cffi-2.0.0 cryptography-48.0.0 pycparser-3.0
```

Let's pay particular attention to the filenames of the wheel files, because most of the information in the wheel file is related to ABI compatibility.

 <figure style={{ textAlign: 'center' }}>
   <img
     src="/posts/python-abi-abi3t/wheel-filename-anatomy.png"
     alt="A diagram illustrating the structure of a wheel filename. There are six pieces of information as part of each wheel file: the name of the distribution, the distribution's version, the Python version tag, the abi tag, and the platform tag, and the wheel filename extension."
     style={{position:'relative',left:'0%',width:'100%'}}
   />
 </figure>

The pieces that are interesting for ABI compatibility are the third, fourth, and fifth tag.
These are the [compatbility tags](https://packaging.python.org/en/latest/specifications/platform-compatibility-tags/) for the wheels. The "python" tag indicates either the exact version supported by the wheel or the minimum supported version.
Whether or not the version indicates a minimum or an exact version depends on the next ABI tag.
The `py3` tag used by `pycparser` indicates that this is usably by _any_ Python 3 interpreter running _any_ Python version, not necesarily just CPython.
The `cp311` and `cp314` tag indicate that the `cryptography` and `cffi` wheels are only usable with CPython. Other Python implementations will need a different wheel.

The ABI tag indicates the Python ABI the wheel file targets.
The `none` ABI tag used by `pycparser` indicates that the wheel doesn't target a particular native ABI in particular: the wheel includes only pure-python code and binary compatibility doesn't need to be considered.
The `cp314` tag used by `cffi` indicates that the wheel supports Python 3.14 exactly and no other minor Python version.
Finally, the `abi3` tag used by `cryptography` indicates that this wheel targets the [Python Stable ABI subset](https://docs.python.org/3/c-api/stable.html#stable-application-binary-interface), and is forward-compatible with all future Python 3 versions that support the `abi3` ABI.
Notably `abi3` wheels are not installable on [the free-threaded build](https://py-free-threading.github.io) of CPython.
We'll see below why this is the case and how the new `abi3t` ABI in Python 3.15 will help ameliorate this limitation going forward.

### What is the Python ABI?

Referring to only one Python ABI is a bit of a fib.
Each [target triple](https://mcyoung.xyz/2025/04/14/target-triples/) - the combination of CPU architecture, vendor, and operating system - has its own distinct ABI. However, one can logically group all the platform-specific details of a specific ABI into its own bucket and all the Python-specific details into another bucket, which I informally refer to as "the" Python ABI.
In practice, the Python ABI includes all of the variables and function declarations exposed in `Python.h`.
Here, a function definition is the name of the function, the number of arguments, the types of the arguments, and the type of the value returned by the function, if any.
Many variables exposed in the C headers are C structs, so the layout of these structs is also part of the ABI.
The layout of the struct is the order and types of all of the members of the struct.

Notably, the Python ABI does _not_ include things that _are_ in the C API like macros and typedefs. Since C macros are a feature of the C preprocessor, by the time a set of machine code is generated, the macro has long since been expanded into C code.
This can be a little confusing when working with the C API and thinking about the difference between the ABI and API because many symbols in the C API are implemented as macros but logically behave as functions.

### How the Python ABI exposes structs

By far the most important struct in the CPython C API is the `PyObject` struct.
All Python objects correspond in C to an instance of `PyObject` or a type that extends the `PyObject` struct.
Until Python 3.13, the `PyObject` struct had the following layout on 64-bit architectures:

```C
struct PyObject {
    Py_ssize_t ob_refcnt;
    PyTypeObject *ob_type;
}
```

Here, `Py_ssize_t` is a signed integer that is used to represent sizes in CPython.
In this case, it represents the [reference count](https://en.wikipedia.org/wiki/Reference_counting) of the object - the number of other data structures that reference this object.
This is used to manage memory: the reference count is incremented and decremented as an object is passed between different Python modules and functions.
When the reference count goes to zero the object is de-allocated.
If you're curious why a signed integer to represent a strictly positive count: using a signed integer would catostrophically wrap around to a large positive value if the reference count ever underflows, with a signed integer you would see a negative reference count: an obviously invalid state.

The other field is a pointer to the type object.
`PyTypeObject` is another struct that extends `PyObject` and is used to represent all Python types.

Taken together, these two pieces of information are always tracked on every Python object. In some sense, the object _is_ the reference count, type, and the address of the `PyObject` struct.

To make that all a little more concrete, `object()` in Python instantiates a `PyObject` instance in ther interpreter runtime, while `dict()` instantiates a `PyDictObject` struct - a different struct that extends `PyObject`.
That is, the first two fields of `PyDictObject` are exactly the same as `PyObject`: all Python objects correspond either to an instance of `PyObject` or a struct that extends `PyObject.

You may wonder: why go into all this detail on this struct? Because the layout of `PyObject` is one of the main sources of tension that led to the development of two new python ABIs in recent years. We'll get more into the history and future of the Python ABI below.

## The Past and Future of the Python ABI

### The Version-Specific ABI and the Python 2 -> 3 transition

In the early days, there wasn't much distinction between "the interpreter" and "what's exposed by the C API": one simply was able to monkey around at will with internal state in the interpreter.
While this enabled lots of cool stuff, it also had a cost: extensions regularly broke with different Python releases, requiring careful fixes to adapt to changes in interpreter internals.
This also introduced a design pressure to freeze internals, lest people building on Python complain about changes breaking their code.

### Python 3 and the Stable ABI: "abi3"

### The Free-Threaded ABI

## A new stable ABI for Python 3.15

### The Opaque PyObject ABI: "abi3t"
