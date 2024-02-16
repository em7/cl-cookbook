---
title: Building Dynamic Libraries
---

Although the vast majority of Common Lisp implementations have
some kind of [foreign function interface](ffi.html) which allows you to
call functions from libraries which use C ABI, the other way around,
i.e. compiling your CL library as a library callable via C ABI from
other languages, might be rare.

Commercial implementations like LispWorks and Allegro CL usually
offer this functionality and is well documented.

This chapter describes a project called [SBCL-Librarian](https://github.com/quil-lang/sbcl-librarian), an opinionated way to create libraries callable from C (and anything which has C FFI) and Python using [Steel Bank Common Lisp](https://www.sbcl.org).


## Preparing your Environment

todo

