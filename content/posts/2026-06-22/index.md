---
title: "2026-06-22"
description: "Modules, Lies, and VHS Tapes"
date: 2026-06-22T20:05:00+02:00
math: false
license: "CC BY-NC-SA 4.0"
hidden: false
comments: true
draft: false
tags:
    - Python
    - C
categories:
    - Archeo
---

>The original version of this article was written in 2006.
>
>Back then:
>
>* Python 2 was still alive,
>* distutils existed,
>* XEmacs was technically not dead,
>* and people still owned VHS tapes.
>
>Let's see how much of this still works in 2026.

# Modules, Lies, and VHS Tapes

Sometimes you need to write Python bindings for a strange third-party library. It happens.

Sometimes you need to write the library itself. That happens too.

And sometimes you have to do all of that on Windows using GNU tools.

This article serves both as a professional tutorial and as a public service announcement suggesting that you abandon everything and move to a remote mountain cabin. Mostly because I can never remember the correct order of the steps involved.

## Prerequisites on Windows

The easiest approach is to use MSYS2 and GCC.

You should have the following tools available somewhere in your PATH:

* GCC
* Python
* `dlltool`

Nothing particularly exotic.

## A Sample Library

Our example library is intentionally simple. It contains only two functions: one that adds two integers and one that multiplies them.

```c
int mul2x(int x, int y)
{
    return x * y;
}

int add2x(int x, int y)
{
    return x + y;
}
```

Since the library is simple, let's call it `SimpLib`.

Assume the source code is stored in a file called `simplib.c`.

## Building the Library

As always, source files should first be compiled into object files.

To produce a shared library, GCC requires the `-shared` option.

> Following Unix naming conventions, shared libraries usually have a `lib` prefix and a `.so` extension. Linking with `-lname` assumes the presence of a library named `libname.so` somewhere on the library search path. Many GNU projects ported to Windows continue to use this naming convention internally, even though the final artifact is a `.dll`.

### Linux

```bash
gcc -o simplib.o -c simplib.c
gcc -o libsimp.so simplib.o -shared
```

Of course, for a single source file you could simply write:

```bash
gcc -o libsimp.so simplib.c -shared
```

However, once a project grows beyond a trivial example, compiling each source file separately and linking everything at the end is usually a much better idea.

Congratulations. The library has been built and is ready for use.

*Cue triumphant fanfare.*

### Windows

Although we're using exactly the same compiler, Windows requires one additional step.

We need to create an **import library** (sometimes called an interface library) that other applications can link against. This is where `dlltool` enters the picture.

#### Creating the Import Library

```bash
gcc -o simplib.o -c simplib.c
dlltool -l libsimp.lib simplib.o
gcc -o libsimp.dll simplib.o -shared
```

The DLL is now ready.

The import library is also ready.

*Cue Windows-themed fanfare.*

## Header File

Providing a header file is generally considered a polite thing to do.

Other developers will appreciate not having to reverse-engineer your function declarations, and they will be less inclined to say unpleasant things about you.

```c
#ifndef __SIMPLIB_H__
#define __SIMPLIB_H__

int mul2x(int x, int y);
int add2x(int x, int y);

#endif
```

Assume this file is called `simplib.h`.

## A Sample C Program

The best way to test a library is to write a small program that uses it.

> There are not many activities one can perform while fully clothed that are more enjoyable than programming, although surprisingly few people seem aware of this fact.

```c
#include <stdio.h>
#include "simplib.h"

int main(int argc, char *argv[])
{
    printf("Adding 5 to 10: %d\n", add2x(5, 10));
    printf("Multiplying 5 by 10: %d\n", mul2x(5, 10));

    return 0;
}
```

Assume this program is stored as `simprg.c`.

## Building the Sample Program

### Linux

```bash
gcc -o simprg.o -c simprg.c
gcc -o simprg simprg.o -lsimp
```

### Windows

Remember the import library:

```bash
gcc -o simprg.o -c simprg.c
gcc -o simprg simprg.o libsimp.lib -lsimp
```

### A Brief Note About Flags

If the library is not located on the system library search path, remember to use the `-L` option.

Windows searches the current directory by default.

Linux does not.

For example:

```bash
gcc -o simprg simprg.o -L. -lsimp
```

## Writing a Python Extension Module

> Historical note
>
> The code below targets Python 2.x and reflects the state of the CPython API around 2006.
> Modern Python 3 extension modules use a different initialization mechanism based on
> `PyModuleDef` and `PyModule_Create`.

The following code demonstrates Python bindings for `SimpLib`:

```c
#include <Python.h>
#include "simplib.h"

static PyObject *simp_mul2x(PyObject *self, PyObject *args)
{
  int x, y, result;
  PyObject *pyresult;
  if (!PyArg_ParseTuple(args, "ii", &x;, &y;))
  {
    return NULL;
  }
  result = mul2x(x, y);
  pyresult = Py_BuildValue("i", result);
  return pyresult;
}

static PyObject *simp_add2x(PyObject *self, PyObject *args)
{
  int x, y, result;
  PyObject *pyresult;
  if (!PyArg_ParseTuple(args, "ii", &x;, &y;))
  {
    return NULL;
  }
  result = add2x(x, y);
  pyresult = Py_BuildValue("i", result);
  return pyresult;
}

static PyMethodDef SimpMethods[] = {
  {"mul2x", simp_mul2x, METH_VARARGS, "Multiply two integer values"},
  {"add2x", simp_add2x, METH_VARARGS, "Add two integer values"},
  {NULL, NULL, 0, NULL}
};
 
PyMODINIT_FUNC initsimp(void)
{
  (void) Py_InitModule("simp", SimpMethods);
}
```

Assume the file is called `simpmodule.c`.

### What Is Happening Here?

Every extension module requires an initialization function.

Historically, for Python 2.x, this function was named `init` followed by the module name, hence:

```c
initsimp()
```

The initialization function creates the module object itself.

Every function listed in the module's method table becomes a method of that module.

In our case:

```c
static PyMethodDef SimpMethods[]
```

contains:

* the names visible from Python,
* the addresses of the corresponding C functions,
* additional flags describing how arguments are passed.

The `METH_VARARGS` flag indicates that Python will pass arguments as a tuple.

The binding functions themselves follow a simple pattern:

1. Validate input arguments.
2. Perform the actual computation.
3. Convert the result into a Python object.

Argument validation is handled by:

```c
PyArg_ParseTuple(args, "ii", &x, &y)
```

The format string `"ii"` means:

> I expect two arguments, and both of them must be integers.

If validation fails, the function should simply return `NULL`.

Building the return value is similarly straightforward:

```c
Py_BuildValue("i", result);
```

which constructs a Python integer object.

The Python documentation traditionally recommends declaring all binding functions as `static`.

## Building the Module

One way to build the extension is to use `distutils`.

### Example setup.py

```python
#!/usr/bin/env python3

from distutils.core import setup, Extension

simpmodule = Extension(
    'simp',
    sources=['simpmodule.c'],
    libraries=['simp'],
    library_dirs=['.']
)

setup(
    name='SimpLib',
    version='1.0',
    description='This is SimpLib bindings package',
    ext_modules=[simpmodule]
)
```

The `Extension` object describes:

* source files,
* required libraries,
* library search paths.

The `setup()` function handles the rest.

### Linux

```bash
python setup.py build
```

### Windows

Unfortunately, Python traditionally expects Visual C++ when building extension modules.

If that doesn't bother you, feel free to use it.

That would, however, make this article considerably shorter.

To force MinGW:

```bash
python setup.py build --compiler=mingw32
```

## Installing the Module

### Linux

```bash
python setup.py install
```

The extension and all required files will be installed into the appropriate Python directories.

### Windows

The generated `simp.pyd` file should be copied into either:

```text
Lib\site-packages
```

or

```text
%APPDATA%\Roaming\Python\site-packages
```

Make sure you know which Python installation you're targeting before copying files around.

> If you happen to own Visual C++, and if the sight of it does not cause emotional distress, installation works exactly like it does on Linux.

## A Short Test Program

Naturally, the best way to test the module is to use it.

```python
from simp import *

print("Adding 212 to 384:", add2x(212, 384))
print("Multiplying 212 by 384:", mul2x(212, 384))
```

You should also verify that incorrect input is handled properly.

For example:

```python
print(mul2x(212, "A tester walks into a bar"))
```

or:

```python
print(mul2x(212, 384, 25))
```

Both cases should result in Python exceptions being raised by the argument validation code.

## Final Thoughts

Keep your spirits up.

Rome wasn't built in a day.

Neither are C libraries, Python bindings, build systems, or debugging skills.

Compilation errors, broken builds, failed tests, and nights spent chasing mysterious bugs are simply part of the profession.

The important thing is to remember that every experienced developer has been through exactly the same process.

Some of them are still going through it.
