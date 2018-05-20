PEP: 9999
Title: A native C calling interface for Python callables
Author: Stefan Behnel, Robert Bradshaw, Jeroen Demeyer, Jim Pivarski
Status: Draft
Type: Standards Track
Content-Type: text/x-rst
Created: 04-May-2018
Python-Version: 3.8
Post-History: 30-Aug-2002


Abstract
========

This PEP proposes a low-level calling interface for C implemented callables
in CPython.


Rationale
=========

In CPython, many callables are implemented in C.  This includes the builtins
as well as wrapped functions of external native libraries.  Calling these
objects will eventually pass through a C signature internally.  On the other
hand, many use cases of Python callables also involve calling them from native
code, such as applying a Python callable to a series of values in a low-level
data array.  This means that in many cases, especially performance critical
cases, both sides, the caller and the callee, will often be implemented in
native code, without knowing from each other.

The current calling interface in CPython requires the arguments and the return
value to be passed as Python objects.  This introduces a substantial overhead,
especially in the case where the arguments and/or the return value must first
be converted between native types and Python objects.

This PEP defines a native C calling interface that native code can use to
call into other native code with a simple C function call, instead of passing
through a Python function call first.

Similar to the buffer protocol, which specifies an interface for passing
native memory blocks between Python libraries, this PEP defines a way to
pass native callables around and call them efficiently through their native
C interface.


The current calling interface
=============================

* tp_call
* PyCFunction

Before CPython 3.6, this required either passing only 0 or 1 argument into
the callee, for which there is an efficient calling signature, or passing
a tuple and/or dict of Python arguments.

* METH_NOARGS
* METH_O
* METH_VARARGS
* METH_KEYWORDS

In CPython 3.6, there is an additional signature for PyCFunction, but not
for arbitrary callables.

* METH_FASTCALL


Constraints
===========

- should be an instance thing, not a type thing, to limit the number of callable types
- one or more C/C++ signatures should be available from a Python callable
- should be supportable also by ctypes, i.e. you wrap something in ctypes and pass it into Numba or Cython
- should support C++ signatures, not only C
- argument clinic could populate signatures for builtins
- default arguments would be nice to support, or could be exploded into overloaded signatures
- support PyMethodDef? e.g. with additional flags?
- exceptions and nogil state must be supported somehow, i.e. the actual call signature might actually have to be different than the simple expected one (Numba generates a function with status return and return value pointer argument)


The native calling interface
============================

- reuse unused "tp_print" type slot for an offset to a signature description array (maybe call it "tp_ccall" ?)
- should include FASTCALL signatures if available
- Python call signatures ("METH_*") could be included as well
- PyMethodDef could be supported with an additional flag that means "the function pointer does not point to a C function but to the signature array instead"
- hope for signature strings to be interned by all providers and do fast pointer comparisons first; intern Python strings to get the same string across the entire Python runtime
- expect signature providers to do complete "normalisation" on it, and expand all signatures into all equivalent signatures, e.g. with+without GIL
- normalisation: exactly one space between types (to make it human readable), "i32, i32, i32" instead of "3 x i32", except for SIMD vectors
- (put some kind of signature hash at the beginning of the signature string to speed up left-to-right comparison for mismatches) -- interning strings is probably even better
- use llvm type system to represent signatures; also has a string representation
- extend string representation with Python call information: 1) can raise exceptions, 2) can be called without GIL, 3) borrowed references
- signatures should be composable somehow, e.g. with escape characters to separate llvm syntax and Python connotations? "@" is used for functions names in llvm, which we don't use => we could use those!


Mapping signatures
==================



Resources
=========

- llvm type system: https://llvm.org/docs/LangRef.html#type-system
- ctypes module: https://docs.python.org/3/library/ctypes.html
- struct module: https://docs.python.org/3/library/struct.html
- Java function descriptors: https://docs.oracle.com/javase/specs/jvms/se7/html/jvms-4.html#jvms-4.3
- "tp_print" slot: https://docs.python.org/3/c-api/typeobj.html?highlight=tp_print#c.PyTypeObject.tp_print


References
==========



Copyright
=========

This document has been placed in the public domain.



..
   Local Variables:
   mode: indented-text
   indent-tabs-mode: nil
   sentence-end-double-space: t
   fill-column: 70
   coding: utf-8
   End:
