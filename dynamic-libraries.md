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

Binary distributions of SBCL usually don't come with SBCL built as a shared library which is needed for SBCL-Librarian. You can download either from [SBCL git repository](https://github.com/sbcl/sbcl) or [using Roswell](getting-started.html#with-roswell) by running command `ros install sbcl-source`.

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

### Download and setup SBCL-Librarian

Clone the SBCL-Librarian repostiory

~~~bash
git clone https://github.com/quil-lang/sbcl-librarian.git
~~~

### Let's start with callback example

SBCL-Librarian comes with a couple of examples, the simple one is a callback to Python code.

ASD file `libcallback.asd` declares a dependency on SBCL-Librarian

~~~lisp
:defsystem-depends-on (#:sbcl-librarian)
:depends-on (#:sbcl-librarian)
~~~

ASDF system need to know where to look for SBCL-Librarian sources. One way is to set its directory your `CL_SOURCE_REGISTRY` environment variable.

`bindings.lisp` contains the important parts for generating the C bindings.

~~~lisp
(defun call-callback (callback outbuffer)
  (sb-alien:with-alien ((str sb-alien:c-string "I guess "))
    (sb-alien:alien-funcall callback str outbuffer)))
~~~

This is the most important function from the example - this one is called from Python code and calls back a Python method (`callback` parameter). Since SBCL-Librarian creates a C library and a Python module which wraps it, this function can be called from both C and Python. This example uses Python.

SBCL-Librarian uses `sb-alien` which is a SBCL package for calling C functions.`with-alien` creates a resource (in this case `str` of type `c-string`) which valid within its body and is disposed of when it gets out of scope, preventing memory leaks. `alien-funcall` calls a C function, in this case its `callback`. It is called with the newly created string and a string buffer which was passed in as an argument.

~~~lisp
(sbcl-librarian::define-type :callback
  :c-type "void*"
  :alien-type (sb-alien:* (sb-alien:function sb-alien:void sb-alien:c-string (sb-alien:* sb-alien:char)))
  :python-type "c_void_p")

(sbcl-librarian::define-type :char-buffer
  :c-type "char*"
  :alien-type (sb-alien:* sb-alien:char)
  :python-type "c_char_p")
~~~

This part defines types `callback` and `char-buffer`, in all three languages - C, Python and Common Lisp. C and Python types are `void *`. Common Lisp type has a proper function prototype. `sb-alien:*` means pointer, so `:callback` is a pointer to function, the result type of which is `void`, and which takes two parameters, a `c-string` and a `char *` (SB-Alien differentiates between a string and a pointer to a character).

`:char-buffer` type is a `char *` in all three languages.

~~~lisp
(define-enum-type error-type "err_t"
  ("ERR_SUCCESS" 0)
  ("ERR_FAIL" 1))

(define-error-map error-map error-type 0
  ((t (lambda (condition)
        (declare (ignore condition))
        (return-from error-map 1)))))
~~~

This creates a mapping between conditions signalled by Common Lisp functions and a return type of the wrapping C functions. If a condition is signalled from Common Lisp, it is translated to a number - a C function return value - within `define-error-map`. The enumeration type adds a C enum so instead of

~~~C
if (1 == cl_function()) {
~~~

you can type

~~~C
if (ERR_FAIL == cl_function()) {
~~~

which is easier to read.

~~~lisp
(define-api libcallback-api (:error-map error-map
                             :function-prefix "callback_")
    (:literal "/* types */")
  (:type error-type)
  (:literal "/* functions */")
  (:function
   (call-callback :void ((fn :callback) (out_buffer :char-buffer)))))

(define-aggregate-library libcallback (:function-linkage "CALLBACKING_API")
  sbcl-librarian:handles sbcl-librarian:environment libcallback-api)
~~~

Finally, `define-api` describes the structure of the code of library to create - what should be the error map, what types and fuctions to include and in which order (`:literal` is just a literal, used for comments in this case). Note taht the function named `call-callback` uses previously defined types for its arguments, `callback` type for first argument called `fn` and `:char-buffer` type for its second argument `out_buffer`. Since `:function-prefix` is used, the actual name of the exported function will be `callback_call_callback`.

The last part, `define-aggregate-library` defines the whole library, what should be included and in which order.




--------------------------------------------


**!!!!TODO!!!!!** This is ... well, a total mess. ! Probably start with very simple example, then this....

SBCL-Librarian expects the path to SBCL sources in `SBCL_SRC` environment variable. Shred libraries are not commonly searched for in the current working directory on newer Linux-based operating systems. This can be fixed by setting environment variable `LD_LIBRARY_PATH`.

~~~bash
export SBCL_SRC=~/.roswell/src/sbcl-2.4.1
export LD_LIBRARY_PATH=.:$LD_LIBRARY_PATH
~~~







SBCL Librarian has two examples - a simple callback and a more complex calculator. Let's explore the callback.

It already contains a Makefile for Mac. For a Linxu-based OS, we should change the name of the created library from `libcallback.dylib` to `libcallback.so` (on Windows to `libcallback.dll`) and the `$(CC)` parameters for `libcallback.so` (or `dll`) target should be changed from `-dynamiclib` to `-shared -fpic`:

~~~Makefile
EXAMPLES_DIR := $(shell pwd)
ROOT_DIR := $(shell dirname $(shell dirname $(EXAMPLES_DIR)))


.PHONY: all clean

all: libsbcl.so libcallback.so

libsbcl.so: $(SBCL_SRC)
	test ! -e $(SBCL_SRC)/src/runtime/libsbcl.so && cd $(SBCL_SRC) && ./make-shared-library.sh || true
	cp $(SBCL_SRC)/src/runtime/libsbcl.so ./

libcallback.core libcallback.c libcallback.h libcallback.py: 
	CL_SOURCE_REGISTRY="$(ROOT_DIR)//" $(SBCL_SRC)/run-sbcl.sh --script "script.lisp"

libcallback.so: libcallback.core libcallback.c
	$(CC) -shared -fpic -o $@ libcallback.c -L$(SBCL_SRC)/src/runtime -lsbcl
clean:
	rm -f libsbcl.so libcallback.c libcallback.h libcallback.core libcallback.py libcallback.so
~~~

Windows:

~~~Makefile
EXAMPLES_DIR := $(shell pwd)
ROOT_DIR := $(shell dirname $(shell dirname $(EXAMPLES_DIR)))


.PHONY: all clean

all: libsbcl.so libcallback.dll

# For some reason, even on OSX SBCL builds its library with *.so
# extension, but this works!
libsbcl.so: $(SBCL_SRC)
	test ! -e $(SBCL_SRC)/src/runtime/libsbcl.so && cd $(SBCL_SRC) && ./make-shared-library.sh || true
	cp $(SBCL_SRC)/src/runtime/libsbcl.so ./libsbcl.dll

libcallback.core libcallback.c libcallback.h libcallback.py: 
	export CL_SOURCE_REGISTRY="$(ROOT_DIR):$(EXAMPLES_DIR)" && \
	echo "cl_source_registry ${CL_SOURCE_REGISTRY}" && \
	$(SBCL_SRC)/run-sbcl.sh --script "script.lisp"

libcallback.dll: libcallback.core libcallback.c
	$(CC) -shared -fpic -o $@ libcallback.c -L$(SBCL_SRC)/src/runtime -L$(EXAMPLES_DIR) -lsbcl
clean:
	rm -f libsbcl.dll libcallback.c libcallback.h libcallback.core libcallback.py libcallback.so
~~~

Python function find_library on Windows / MSYS: export PATH=.:$PATH
