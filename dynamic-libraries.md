---
title: Building Dynamic Libraries
---

Although the vast majority of Common Lisp implementations have
some kind of [foreign function interface](ffi.html) which allows you to
call functions from libraries which use C ABI, the other way around,
i.e. compiling your CL library as a library callable via C ABI from
other languages, might be rare.

Commercial implementations like LispWorks and Allegro CL usually
offer this functionality and they are well documented.

This chapter describes a project called [SBCL-Librarian](https://github.com/quil-lang/sbcl-librarian), an opinionated way to create libraries callable from C (anything which has C FFI) and Python using an open-source and free-to-use implementation [Steel Bank Common Lisp](https://www.sbcl.org).


## Preparing Environment

### Make SBCL Shared Library

Binary distributions of SBCL usually don't come with shared library which is needed for SBCL-Librarian. You can download either from [SBCL git repository](https://github.com/sbcl/sbcl) or [using Roswell](getting-started.html#with-roswell) by running command `ros install sbcl-source`.

SBCL also needs a working Common Lisp system to bootstrap the compiliation
process. Easy trick is to download a binary installation from Roswell and
add it to your `PATH` variable.

SBCL depends on `zstd` library. On Linux-based system, you can obtain both
the library and its header files from the package manager, usually it's called
something like `libzstd-dev`. On Windows, recommended way is to use
[MSYS2](https://www.msys2.org) which includes both Roswell, `zstd` and its headers.

Go to the directory with sources and run:

~~~bash
# Bash

# (assuming the version of your SBCL installed via Roswell is 2.4.1)
export PATH=~/.roswell/impls/x86-64/linux/sbcl-bin/2.4.1/bin/:$PATH

./make-config.sh --fancy
./make.sh --fancy
./make-shared-library.sh --fancy
~~~

Note that the shared library has `.so` extension even on Windows and Mac
but it seems to work just fine. If you use Roswell in MSYS2, it can sometimes
use you Windows home directory rather than your MSYS2 home diretory, which
are different paths. So the path to Roswell is `/C/Users/<username>/.roswell`,
not `~/.roswell/`.

### Download SBCL-Librarian

