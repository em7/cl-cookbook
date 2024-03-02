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

## Let's Start: Callback Example

SBCL-Librarian comes with a couple of examples, the simple one is a callback to Python code.

### ASD File

ASD file `libcallback.asd` declares a dependency on SBCL-Librarian

~~~lisp
:defsystem-depends-on (#:sbcl-librarian)
:depends-on (#:sbcl-librarian)
~~~

ASDF system need to know where to look for SBCL-Librarian sources. One way is to set its directory your `CL_SOURCE_REGISTRY` environment variable.

### Bindings.lisp

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

### Compile LISP Code

Now we can compile the LISP code and generate the C sources for compiling our library and the Python wrapper.

A couple of environment variables can be set for convenience:

~~~bash
# Directory with SBCL sources
export SBCL_SRC=~/.roswell/src/sbcl-2.4.1
# Directory with this project, don't forget the double slash at the end
# or it might not work
export CL_SOURCE_REGISTRY="~/prg/sbcl-librarian//"
~~~

On more modern Linux-based systems, libraries are usually not searched for in the current directory. The same goes for paths which Python searches for libraries.

~~~bash
export LD_LIBRARY_PATH=.:
export PATH=.:$PATH
~~~

`script.lisp` is a fairly simple LISP script for compiling the LISP sources and outputting the wrapper code and the LISP core.

~~~lisp
(require '#:asdf)

(asdf:load-system '#:libcallback)

(in-package #:sbcl-librarian/example/libcallback)

(build-bindings libcallback ".")
(build-python-bindings libcallback ".")
(build-core-and-die libcallback "." :compression t)
~~~

Now, we have a couple of new files.

`libcallback.c`, is the source code for our library.

~~~c
#define CALLBACKING_API_BUILD

#include "libcallback.h"

void (*lisp_release_handle)(void* handle);
int (*lisp_handle_eq)(void* a, void* b);
void (*lisp_enable_debugger)();
void (*lisp_disable_debugger)();
void (*lisp_gc)();
err_t (*callback_call_callback)(void* fn, char* out_buffer);

extern int initialize_lisp(int argc, char **argv);

CALLBACKING_API int init(char* core) {
  static int initialized = 0;
  char *init_args[] = {"", "--core", core, "--noinform", };
  if (initialized) return 1;
  if (initialize_lisp(4, init_args) != 0) return -1;
  initialized = 1;
  return 0; }
~~~

At the top, we see a couple of SBCL-related functions like `lisp_gc` for telling the LISP garbagec collector that it's a good time to run. Then there is a pointer to our function `callback_call_callback`. Finaly, the `init` function which we should run before any LISP code is run.

There is currently no way to de-initialize the LISP core.

`libcallback.h` is a header file which should be included in both `lispcallback.c` and in any calling C code. Apart of prototypes of functions and function pointers in `lispcallback.c` it includes the error enum and any comments that were added in `bindings.lisp`:

~~~C
typedef enum { ERR_SUCCESS = 0, ERR_FAIL = 1, } err_t;
~~~

The last file is `lispcallback.py`, the Python wrapper around our library. The most interesting part is this:

~~~Python
from ctypes import *
from ctypes.util import find_library

try:
    libpath = Path(find_library('libcallback')).resolve()
except TypeError as e:
    raise Exception('Unable to locate libcallback') from e
~~~

The rest of the file is similar to the C header file.

As you can see, it loads a compiled C library (shared object, DLL, dylib) and tells the Python interpreter what can be done with the library. It also initializes the LISP core when it's loaded.


### Compile C Code

~~~bash
cc -shared -fpic -o libcallback.so libcallback.c -L$SBCL_SRC/src/runtime -lsbcl
~~~

On Mac OS the command might be a bit different:

~~~bash
cc -dynamiclib -o libcallback.dylib libcallback.c -L$SBCL_SRC/src/runtime -lsbcl
~~~

If you don't have `$SBCL_SRC/src/runtime` in your `$PATH`, copy the `$SBCL_SRC/src/runtime/libsbcl.so` file to the current directory.

### Run

Now, everything is ready. You can run the example code using

~~~bash
$ python3 ./example.py 
I guess  it works!
~~~

If you see a rather cryptic error

~~~bash
$ python3 ./example.py 
Traceback (most recent call last):
  File "/home/user/prg/sbcl-librarian/examples/callback/./example.py", line 2, in <module>
    import libcallback
ImportError: dynamic module does not define module export function (PyInit_libcallback)
~~~

it means that Python is trying to load `libcallback.so` directly as if it was a compiled Python module (written in C). Since it isn't, I suggest to rename `libcallback.py` to something else like `callback.py` and from `example.py` import `callback` rather than `libcallback`.

## Makefile

Each example has a Makefile for building it on Mac. It even automatically build the `libsbcl.so` library and copies it into the current directory. However the command for building the project (e.g. `libcallback` itself) needs to be modified to be usable on Linux-based operating systems and on Windows (MSYS2).

## CMake

Using CMake is rather straightforward, unfortunately there is currently no CMake-aware library or `vcpgk`/ `conan` package so we have to use `HINTS` with `find_library`.

Assuming you'd like to compile a project called `my_project` and would like to add a LISP library:

~~~CMake
# If there is a better way, let me know.
if(WIN32)
    set(DIR_SEPARATOR ";")
else()
    set(DIR_SEPARATOR ":")
endif()

# Set the ENV Vars for building the LISP part
set(SBCL_SRC "$ENV{SBCL_SRC}" CACHE PATH "Path to SBCL sources directory.")
set(SBCL_LIBRARIAN_DIR "${CMAKE_CURRENT_SOURCE_DIR}/../sbcl-librarian" CACHE PATH "Source codes of SBCL-LIBRARIAN project.")
set(CL_SOURCE_REGISTRY "${CMAKE_CURRENT_SOURCE_DIR}${DIR_SEPARATOR}${SBCL_LIBRARIAN_DIR}" CACHE PATH "ASDF registry for building of the libray.")

# Find the SBCL library
find_library(libsbcl NAMES sbcl HINTS ${SBCL_SRC}/src/runtime/)

# Link the library to the C project
target_link_libraries(my_project ${libsbcl})

# Build LISP part of the project
add_custom_command(OUTPUT my_project-lisp.core my_project-lisp.c my_project-lisp.h my_project-lisp.py
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
    COMMAND ${CMAKE_COMMAND} -E env CL_SOURCE_REGISTRY="${CL_SOURCE_REGISTRY}"
        ${SBCL_SRC}/run-sbcl.sh ARGS --script script.lisp
    COMMAND ${CMAKE_COMMAND} -E copy_if_different my_project-lisp.core $<TARGET_FILE_DIR:my_project>
    COMMAND ${CMAKE_COMMAND} -E copy_if_different my_project-lisp.c $<TARGET_FILE_DIR:my_project>
    COMMAND ${CMAKE_COMMAND} -E copy_if_different my_project-lisp.h $<TARGET_FILE_DIR:my_project>
    COMMAND ${CMAKE_COMMAND} -E copy_if_different my_project-lisp.py $<TARGET_FILE_DIR:my_project>
    COMMAND ${CMAKE_COMMAND} -E rm my_project-lisp.core my_project-lisp.c my_project-lisp.h my_project-lisp.py

# Copy SBCL library if newer
add_custom_command(TARGET my_project POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E copy_if_different
        "${libsbcl}"
        $<TARGET_FILE_DIR:my_project>)
~~~
