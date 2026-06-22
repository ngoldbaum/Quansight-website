---
title: 'What Every Python Developer Should Know About the CPython ABI'
authors: [nathan-goldbaum]
published: June 4, 2026
description: 'An introduction to the concept of the Application Binary Interface (ABI), the various CPython ABIs, and the new abi3t stable ABI in Python 3.15.'
category: [PyData ecosystem]
featuredImage:
  src: /posts/python-abi-abi3t/cpython_api_layers_listing.png
  alt: 'Five nested ellispes illustrating the layering of the Python C API. The outermost ellipse is gray and labeled "Internal API". The next enclosed ellipse is red and is labeled "Private API`. The next enclosed ellipse is yellow and is labeled "Unstable API". The next enclosed ellipse is blue and labeled "Version-specific API". The next enclosed ellipse is green and is labeled "Limited API".'
hero:
  imageSrc: /posts/python-abi-abi3t/cpython_api_layers_hero.png
  imageAlt: 'Five nested ellispes illustrating the layering of the Python C API. The outermost ellipse is gray and labeled "Internal API". The next enclosed ellipse is red and is labeled "Private API`. The next enclosed ellipse is yellow and is labeled "Unstable API". The next enclosed ellipse is blue and labeled "Version-specific API". The next enclosed ellipse is green and is labeled "Limited API".'
---

# What Every Python Developer Should Know About the CPython ABI

The CPython Application Binary Interface (ABI) backs Python's main superpower: the ability to easily call into native C, C++, Rust, or Fortran code and for that code to call back into the interpreter and update the state of Python objects.
But what exactly is an ABI? How does it differ from an Application Programming Interface (API)?
Why did I begin this post's title with "What _Every_ Python Developer Should Know"?
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
It certainly doesn't help things that the two related and intertwined concepts are referred to using similar-sounding acronyms.

[CPython](https://github.com/python/cpython), as the name suggests, is implemented in the C programming language.
It began as a research project and was written to be easy to extend with new functionality.
To access the internals of the interpreter, one needed only to `#include "Python.h"` in a C program, which gave rich access to the internal state of the interpreter and hooks to call into the interpreter via the CPython C API, the very same C API that the core interpreter implementation is based on.

What is the C API exactly?
It's precisely the set of C macros, typedefs, functions, and structs exposed by the header that defines the C interface: `Python.h`.
This constitutes an enormous number of symbols.
It's possible to write code in C that allows similar functionality to any arbitrary Python script at the cost of compilation, verbosity, and exposure to the pitfalls of the C programming language.
The reward is raw execution speed.
It is often possible to achieve order-of-magnitude or even several order-or-magnitude size speedups by translating Python code to a compiled language that can call into the C API.
The [Cython](https://cython.readthedocs.io/en/latest/) programming language takes advantage of this by (among other things) literally compiling Python code to a C "script" that when linked into a larger program behaves identically to the Python code it was compiled from.

What isn't the C API?
The C API is purely a construct of the C programming language.
Code written in languages that aren't C can call into a C API, but only by using the conventions of the C programming language and abstracting away functionality that is not expressible in C.
This is when it becomes important to think about the Python ABI.
The concrete ABI conventions used by machine code that C API calls compile to are _also_ used by other programming languages like C++ and Rust to interact with the Python interpreter despite the fact that CPython does not expose C++ or Rust APIs.

## The Python ABI

In this section I will explain exactly what an application binary interface (ABI) is. To separate out Python-specific details from platform-specific details I will also refer to the Python ABI and the platform ABI, respectively, in the discussion below.

### What is a platofrm ABI?

The platform ABI governs details of how exactly machine code executes on each architecture: how a compiler translates calling a C function like [`PyDict_GetItemRef`](https://docs.python.org/3/c-api/dict.html#c.PyDict_GetItemRef) into a concrete set of machine code that sets CPU registers and other platform-specific details per the specification of whatever platform the code ultimately runs on.

A platform ABI determines things like the exact way machine code needs to pass arguments to a function. For example, on the [`x86_64`](https://wiki.osdev.org/System_V_ABI#x86-64) architecture, only the first six non-floating-point arguments of a function are passed via registers, the remaining arguments are passed via the stack.
Normally, developers do not need to worry about details like this: compilers automatically generate machine code appropriate for whatever platform ABI a developer wants to target.
However, it _is_ important to know that different operating systems and CPU architectures have unique ABIs and that code compiled for one platform ABI is completely unusable on another platform ABI.
This fact determines much of the design of the [binary wheel distribution format](https://packaging.python.org/en/latest/specifications/binary-distribution-format/), which we'll discuss in more detail below.

The biggest consequence is that each platform and CPU architecture, each with its own distinct platform ABI, requires its own unique builds.
This is one reason projects like `NumPy` distribute so many binary wheels with each release: projects need to build binaries for each platform ABI they want to support.

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
These are the [compatibility tags](https://packaging.python.org/en/latest/specifications/platform-compatibility-tags/) for the wheels. The "python" tag indicates either the exact version supported by the wheel or the minimum supported version.
Whether or not the version indicates a minimum or an exact version depends on the next ABI tag.
The `py3` tag used by `pycparser` indicates that this is usable by _any_ Python 3 interpreter running _any_ Python version, not necessarily just CPython.
The `cp311` and `cp314` tag indicate that the `cryptography` and `cffi` wheels are only usable with CPython. Other Python implementations will need a different wheel.

The ABI tag indicates the Python ABI the wheel file targets.
The `none` ABI tag used by `pycparser` indicates that the wheel doesn't target a particular native ABI in particular: the wheel includes only pure-Python code and binary compatibility doesn't need to be considered.
The `cp314` tag used by `cffi` indicates that the wheel supports Python 3.14 exactly and no other minor Python version.
Finally, the `abi3` tag used by `cryptography` indicates that this wheel targets the [Python Stable ABI subset](https://docs.python.org/3/c-api/stable.html#stable-application-binary-interface), and is forward-compatible with all future Python 3 versions that support the `abi3` ABI.
Notably `abi3` wheels are not installable on [the free-threaded build](https://py-free-threading.github.io) of CPython.
We'll see below why this is the case and how the new `abi3t` ABI in Python 3.15 will help ameliorate this limitation going forward.

### What is the Python ABI?

The Python ABI includes all of the symbols - the variables and function declarations exposed in `Python.h`.
A symbol in the ABI corresponding to a C API function holds the name of the function, the number of arguments, the types of the arguments, and the type of the value returned by the function, if any.
Many variables exposed in the C headers are C structs, so the layout of these structs is also part of the ABI.
The layout of the struct is the order and types of all of the members of the struct.

Notably, the Python ABI does _not_ include things that _are_ in the C API.
This includes all items that are particular to the conventions of the C language or the C preprocessor like [macros](<https://www.cs.yale.edu/homes/aspnes/pinewiki/C(2f)Macros.html>), [typedefs](https://en.wikipedia.org/wiki/Typedef), and [inline functions](<https://en.wikipedia.org/wiki/Inline_(C_and_C%2B%2B)>).
The Python ABI also doesn't depend on the names of the function arguments: all it cares about is how to store and lay out the instances of different types exposed in a C API in memory.

This can be a little confusing when working with the CPython C API and thinking about the difference between the ABI and API because many names in the C API are implemented as macros but logically behave as functions.
Despite that, because of how they are implemented, these items do not appear in the Python ABI.
This means that programming languages that cannot compile C syntax, like Rust, cannot use any C API items that are defined as C macros or inline functions.
Instead, Rust extensions rely on Rust re-implementations of macros and static inline functions exposed by the C API based on items that _are_ in the Python ABI.
C++ _is_ compatible with the C preprocessor, so you can use typedefs, macros, and inline functions exposed in the CPython C API in C++ extensions.

### The PyObject struct

In this section we do a bit of a deep dive on one of the structs that are part of the Python ABI.
Why go into all this detail on this struct?
For one, the `PyObject` struct is by far the most important struct in the CPython C API and the Python ABI.
Also, because the layout of `PyObject` is one of the main sources of tension that led to the development of two new Python ABIs in recent years.
We'll get more into the history and future of the Python ABI below.

All Python objects correspond in C to an instance of `PyObject` or a struct that extends the `PyObject` struct.
Until free-threaded Python (more on that below), the `PyObject` struct had the following layout on 64-bit architectures:

```C
struct PyObject {
    Py_ssize_t ob_refcnt;
    PyTypeObject *ob_type;
}
```

Here, [`Py_ssize_t`](https://docs.python.org/3/c-api/intro.html#c.Py_ssize_t) is a signed integer type that is used to represent sizes in the CPython C API.
In this case, the `ob_refcnt` field represents the [reference count](https://en.wikipedia.org/wiki/Reference_counting) of the object - the number of other data structures that reference an instance of the `PyObject` struct.
This is used to manage memory: the reference count is incremented and decremented as an object is passed between different Python modules and functions.
When the reference count goes to zero, the object is de-allocated.
If you're curious why Python uses a signed integer to represent a strictly positive count: a signed integer would catastrophically wrap around to a large positive value if the reference count ever underflows.
With a signed integer you would see a negative reference count: an obviously invalid state.

The other field, `ob_type`, is a pointer to the type of the object.
Since this is Python and even types are objects, `PyTypeObject` is another struct that extends `PyObject`.

Taken together, these two pieces of information, the reference count and the type, are always tracked on every Python object. In some sense, an object _is_ the address and content of a `PyObject` instance or a instance of a struct that extends `PyObject`.

To make that all a little more concrete, `object()` in Python instantiates a `PyObject` instance in the interpreter runtime, while `dict()` instantiates a `PyDictObject` struct - a different struct that extends `PyObject`.
That is, the first two fields of `PyDictObject` are exactly the same as `PyObject`.

## The Layers of the CPython C API

In the early days, there wasn't much distinction between "the interpreter" and "what's exposed by the C API": one simply was able to monkey around at will with internal state in the interpreter.
While this enabled lots of cool stuff, it also had a cost: extensions regularly broke with different Python releases, requiring careful fixes to adapt to changes in interpreter internals.
This also introduced a design pressure to freeze internals, lest people building on Python complain about changes breaking their code.

These days there is a much better distinction between CPython internals and publicly exposed API, with all signs pointing towards the trend of isolating interpreter internals to continue into the future.

This is managed by breaking up the "full" C API surface used by the interpreter into five different layers, illustrated in the diagram below.

 <figure style={{ textAlign: 'center' }}>
   <img
     src="/posts/python-abi-abi3t/cpython_api_layers.png"
     alt='Five nested ellipses illustrating the layering of the Python C API. The outermost ellipse is gray and labeled "Internal API, exposed only if `Py_BUILD_CORE` is defined. The next enclosed ellipse is red and outlined with a dashed line defined in the legend to mean "Usable with `#include "Python.h"`". The red ellipse is labeled "Private API `_Py*` prefix`. The next enclosed ellipse is yellow and is labeled "Unstable API `PyUnstable_*` prefix". The next enclosed ellipse is blue and labeled "Version-specific API". The next enclosed ellipse is green with a dark shaded outline the legend defines to mean "Usable if `Py_LIMITED_API` is defined and is labeled "Limited API".'
     style={{position:'relative',left:'12%',width:'70%'}}
   />
 </figure>

Below we describe each of these layers and explain how this separation enables many different usecases.

### Internal C API

The outermost layer is for people who work on the CPython implementation.
In order to use it, a C or C++ program must define the `Py_BUILD_CORE` compiler macro, which is equivalent to declaring that your program is a part of the CPython interpreter itself.
As [the CPython developer guide](https://devguide.python.org/developer-workflow/c-api/#the-internal-api) indicates, the internal API is subject to change at any moment and should not be relied on.
Defining `Py_BUILD_CORE` and opting into using the internal C API is equivalent to saying you're willing to take on maintenance for code that can and will break at any time.
For 99.9% of people who are not CPython contributors, the internal API should not be used.

### The Public C API

The public API is the innermost four layers in the diagram above. This is the full set of API symbols available to a C program that has `#include "Python.h"` at the top and no other special preprocessor macros defined at build time.

Despite the name, it also includes many details that the CPython project considers private according to their formal stability policty.

### Private C API

For historical reasons, the public API includes many private implementation details whose names are prefixed by a leading underscore.
To pick a random example: [`_PyDict_GetItem_KnownHash`](https://github.com/python/cpython/blob/2e5843e13fcfd768a435d82e6182af403844432c/Include/cpython/dictobject.h#L45-L46) allows users to bypass the normal Python `dict` interface to directly access a dict entry with an already-computed hash.
This optimization is useful sometimes in the interpreter, so it's defined, but it's not a documented primitive, so it may either be deprecated and eventually removed or possibly get promoted to an official API function.
Sometimes symbols are exposed like this for technical reasons, sometimes for historical reasons, and sometimes because a CPython user requested the ability to do so with the knowledge that there is no API stability guarantee for these functions.

### Unstable C API

The next innermost layer encloses all of the documented functionality in the CPython C API.
There is an extra layer here that's outside the "regular" C API because not everything that's documented is stable.
The outermost layer is [the unstable C API](https://docs.python.org/3/c-api/stable.html#unstable-c-api).
If a serious defect is discovered in an unstable C API item it may be radically changed or removed entirely in a CPython minor release.
These are documented symbols that serve a real purpose, but that the CPython developers are not ready to commit to supporting long-term because they rely on interpreter implementation details.
The unstable API is most useful for projects or organizations who can regularly follow and contribute to CPython C API development and can make appropriate judgement calls about the suitability of using these API functions.

### The Version-Specific API

The next innermost layer is the Python version-specific C API and includes all symbols that follow the CPython C API [stability policy](https://docs.python.org/3/c-api/stable.html#c-api-stability), as originally adopted back in 2009 for [PEP 387](https://peps.python.org/pep-0387/).
While public symbols in the CPython C API can be removed following a deprecation period or to fix a serious defect, this only happens after a period of public discussion and consideration of the community impact. Critically, CPython has a policy not to add or remove items in the version-specific API from the first beta release of a Python minor version onward.
For example, this year Python 3.15.0b1 came out in May and that release froze the Python 3.15 version-specific C API.
All subsequent betas, release candidates, and official releases share the same set of API items, including function signatures and type layouts, with no changes allowed until the next Python minor release.

### Limited C API

Finally, the most restricted innermost layer corresponds, fittingly, to the "Limited" C API.
A somewhat confusing point that is often glossed over is that the limited C API _also_ changes from Python version to Python version. One can build extensions that target _either one_ of, say, the Python 3.13 version-specific API or the limited API.
Either by defining a compiler macro or selecting an option in a build backend, one selects a particular limited API version to target.
It's possible to target limited API versions older than the Python version used to build an extension.
An extension or wheel build targeting, say, the Python 3.10 limited API can be imported on Python 3.10 and _any newer Python release_.

There is a big "but" in the above claim that targeting the limited API allows supporting _any_ Python version newer than Python 3.10.
It turns out that this guarantee depends critically on the CPython ABI and how free-threaded Python has introduced two new ABIs to target on new Python versions.

## CPython C ABI stability

There are two levels of ABI stability provided by CPython:

- The ABI is frozen for all releases within a Python minor version, say from Python 3.13.0 to the last Python 3.13.x bugfix release.
- ABI items that are in the limited C API will never be removed, even if a future version of the limited API removes an API item.

Of course "never" is a little strong, and if a particularly bad design flaw is discovered then the _implementation_ of the ABI entry corresponding to the pernicious symbol that is eventually removed from the limited API would probably trigger a runtime error.
The _symbol_ would still be available in the ABI though, so an extension that happens to use the removed function would still compile, link, and import successfully, it just might trigger a runtime error when a new Python version uses functionality that _calls_ the removed function.

These two ABI stability guarantees enable two different kinds of builds for distributors of extension modules:

### The version-specific ABI: `cp3XY`

The guarantee that the Python version-specific C API does not change within a Python minor release series corresponds to a stability policty for the CPython version-specific ABI.
In practice, it means that extension modules and binary wheels built using Python 3.15.0b1 can be imported using any subsequent version of Python 3.15, even years into the future.

Projects shipping binary wheels using the version-specific ABI have ABI compatibility tags in the wheel filename like `cp314-cp314`. If a wheel has this particular tag, that indicates the wheel has binaries that are built using the GIL-enabled version-specific ABI for Python 3.14.
We'll cover shortly why I had to add "GIL-enabled" there and why that's important.

Because the version-specific ABI is specific to a particular minor release of CPython, this means a project must explicitly add support for new Python releases every year.
Some projects, like NumPy, target this ABI because they have the contributor resources to manage that level of complexity, with an annual deadline corresponding to the release of CPython in the Fall.
Many projects do not have the resources to track CPython like that and instead tend to fall behind, adding support for new CPython versions months or even years after the CPython release happened.

### The Stable ABI: `abi3`

The guarantee that items in the limited C API will never go away enables a different kind of _forward-compatible_ ABI.
One can build binaries using the limited API as it was defined in, say, Python 3.10.
Because of the stability guarantees provided by the stable subset of the CPython ABI, you can rely on the fact future Python versions will be able to import your binary: all the symbols it needs from the CPython ABI should still be there.

There is one major problem with this scheme: `abi3` as it was originally defined shared a detail with the version-specific ABI: the layout of the `PyObject` struct is exposed.

### The CPython ABI and the GIL

As you may have heard, it's possible to use a version of CPython that does not have a [global interpreter lock](https://docs.python.org/3/glossary.html#term-global-interpreter-lock) (the GIL)

What is the GIL and why is it important for this discussion? It's actually very much related to the layout of the `PyObject` struct. As we saw above, `PyObject` contains a reference count field `ob_refcnt`, that holds the number of live referenes to the Python object.
If the reference count goes to zero the object is deallocated.
In CPython, the object is deallocated immediately.
Critically, Python also exposes access to OS-level threads via the `threading` module and in principle, without a global lock preventing it, could expose _simultaneous_ access to the same Python object.

So you have a situation where a C struct has a signed integer that may be incremented or decremented at arbitrary times -- even simultaneously -- from any arbitrary number of threads.
This is a classic case of a problem that is susceptible to pitfalls of concurrency: [races](https://en.wikipedia.org/wiki/Race_condition).
This particular case, where the race to increment or decrement the integer, happens in C code, is suseptible to a particularly bad kind of multithreaded race: a [data race](https://en.wikipedia.org/wiki/Race_condition#Data_race).
In the specification of the C, C++, and Rust languages, the result of what happens after [these sorts of races are undefined behavior](https://www.hboehm.info/c++mm/why_undef.html).

This problem was realized quite early on in the development of CPython.
To fix it, the language grew the global interpreter lock.
In order to interact with Python objects, you need to acquire a global [mutex](<https://en.wikipedia.org/wiki/Lock_(computer_science)>) and release it as soon as you are done using the Python objects or the CPython interpreter runtime.

Since incrementing and decrementing reference counts should only happen with the GIL mutex held, it's therefore perfectly safe to arbitrarily share Python objects: one can be confident that only one thread at a time can actually see or mutate the state of hte object.

There is a big downside to this design decision: the GIL means it is impossible to actually use more than one CPU core to run Python code.
In particular: only one core can run code that holds the GIL.
Here, "code that holds the GIL" corresponds to all "pure" Python code that uses only builtins and does not use third-party code or the standard library.
Some code in third-party extensions and the standard library does release the GIL and allow more than one CPU to simultaneously execute code, but in practice one often finds that scaling is abysmal due to the impact of [Amdahl's "law"](https://en.wikipedia.org/wiki/Amdahl%27s_law): any time-slice of an application's execution that must run on only one CPU will eventually prevent improving parallel scaling as a function of thread count.

That is why there have been several efforts over the years -- to varying degrees of success -- to remove the GIL from the CPython implementation.
Over the past few years, one of these approaches seems to have stuck.
It looks like a free-threaded Python, one with no global interpreter lock and no limit to multithreaded concurrency in pure Python code, is the future of CPython.

### The Free-Threaded ABI: `cp3XYt`

## A new stable ABI for Python 3.15

### The Free-Threaded Stable ABI: `abi3t`

### Advice for Project Maintainers

The free-threaded Stable ABI, `abi3t`, gives extension maintainers a path to
one artifact per platform for Python 3.15 and newer. Older non-free-threaded
versions can still be covered by `abi3`:

<div className="overflow-x-auto">
  <table className="mx-auto w-auto min-w-[42rem]">
    <thead>
      <tr>
        <th>CPython version</th>
        <th>Non-free-threaded</th>
        <th>Free-threaded</th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <td>3.12</td>
        <td rowSpan={3} className="align-middle text-center"><code>abi3</code></td>
        <td>&mdash;</td>
      </tr>
      <tr>
        <td>3.13</td>
        <td>&mdash;</td>
      </tr>
      <tr>
        <td>3.14</td>
        <td><code>cp314t</code></td>
      </tr>
      <tr>
        <td>3.15</td>
        <td colSpan={2} rowSpan={3} className="align-middle text-center"><code>abi3t</code></td>
      </tr>
      <tr>
        <td>3.16</td>
      </tr>
      <tr>
        <td>3.17 and later</td>
      </tr>
    </tbody>
  </table>
</div>
