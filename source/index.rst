=================================================
Just-In-Time compilation using GCC (libgccjit.so)
=================================================

GNU Tools Cauldron 2014

David Malcolm <dmalcolm@redhat.com>

.. Abstract: This will be a report on http://gcc.gnu.org/wiki/JIT - the
   current status of this branch of gcc, along with a discussion of how we
   could go about merging it into the trunk.  I'll also talk about things
   we could do to both GCC and to the rest of the GNU toolchain (e.g. gas)
   to better support the JIT use-case from GCC.

.. Saturday 2014-07-19 4.30->5.15

.. What I want:
   (A) to get jit branch merged into next major gcc release (4.10/5.0)
       What would it take?
   (B) to get more people using it


What is libgccjit.so
=====================

* Experimental branch of GCC:
* Building GCC as a shared library
* Suitable for embedding inside other programs and libraries
* In-process code-generation, at runtime

See http://gcc.gnu.org/wiki/JIT


Why?
====
JIT compilation use cases:

* Language runtimes

  * JVM
  * Python
  * OpenGL shader language

* Other uses?

  * FFI?

* Your project?


Status of the project
=====================


Examples of use
===============
 * jittest
 * Octave
 * coconut?

.. * PyPy???


Method jit vs tracing jit
=========================


Why a dedicated API for JIT?
============================

* We need some kind of API that people can use to write JIT compilers

* We need to be able to provide API and ABI guarantees

* Existing internal APIs are in flux

* Existing internal APIs aren't easy to use - wrap them


Design decisions
================

* Assume that we're interfacing with C code

  * e.g. terminology

* All types are opaque.

* Support multithreaded client code


What the API looks like
=======================
A C header file; currently with:

  * 72 function prototypes
  * 12 opaque types
  * 8 enums


State and lifetime management
=============================
All state is hung off of context objects:

  .. code-block:: c

    typedef struct gcc_jit_context gcc_jit_context;

    extern gcc_jit_context *
    gcc_jit_context_acquire (void);

    extern void
    gcc_jit_context_release (gcc_jit_context *ctxt);

Simple memory mamangent for client code

  Everything that's created from a context is cleaned up when the
  context is released.


One-time setup vs per-compile state
===================================

A common pattern:

.. rst-class:: build

   1) one-time setup:

      The client code maps its own API into the JIT world:

        * create ``gcc_jit_type`` instances representing the structs
          and other types of interest

        * similar for globals, functions, etc

   2) repeatedly reuse (1) as each method becomes "hot", using (1)
      to compile each method to machine code

Seen e.g. in GNU Octave's JIT compiler.

.. nextslide::
   :increment:

How to handle this?

If we do it all in one context, we'll have a slow leak due to all of the
per-method state never going away.

.. nextslide::
   :increment:

Solution: nested contexts:

.. code-block:: c

  extern gcc_jit_context *
  gcc_jit_context_new_child_context (gcc_jit_context *parent_ctxt);

* Create a parent context, and do the one-time setup within it

* Create child context as each method becomes hot, compiling that
  method.

* Clean up the child context immediately.

* The parent context persists for the lifetime of the program.

.. nextslide::
   :increment:

* Arbitrary nesting is allowed.

* The child can reference objects created within the parent, but not
  vice-versa.

* The lifetime of the child context must be bounded by that of the
  parent: client code should release a child context before releasing
  the parent context.

Error-handling
==============
Inspired by OpenGL:

  * record errors

  * fail if an error has occurred

  * fail gracefully when called after an error

Client code only has to check for errors once.

.. code-block:: c

  extern const char *
  gcc_jit_context_get_first_error (gcc_jit_context *ctxt);


Comments as a first-class entity
================================

.. code-block:: c

  extern void
  gcc_jit_block_add_comment (gcc_jit_block *block,
                             gcc_jit_location *loc,
                             const char *text);

*Very* useful for debugging

e.g.

.. code-block:: c

  gcc_jit_block_add_comment (b_entry, NULL,
                             "for i in 0 to (ARRAY_SIZE - 1):");

Internally they are implemented as dummy labels.

Shouldn't affect optimization.

Visible in dumps of initial tree and of gimple.

.. I have an unfinished patch to add comments to gimple and to RTL


What the API doesn't do
=======================

* Type inference
* Unboxing

etc


The C++ API
===========
Methods, and (optionally) operator overloading:

.. code-block:: c++

  struct quadratic
  {
    double a;
    double b;
    double c;
    double discriminant;
  };

  gccjit::rvalue q_a = param_q.dereference_field (field_a);
  gccjit::rvalue q_b = param_q.dereference_field (field_b);
  gccjit::rvalue q_c = param_q.dereference_field (field_c);

  gccjit::rvalue four =
    ctxt.new_rvalue (double_type, 4);

.. nextslide::
   :increment:

.. code-block:: c++

  gccjit::block block = calc_discriminant.new_block ();
  block.add_comment ("(b^2 - 4ac)");

  block.add_assignment (
    /* q->discriminant =...  */
    param_q.dereference_field (testcase.discriminant),
    /* (q->b * q->b) - (4 * q->a * q->c) */
    (q_b * q_b) - (four * q_a * q_c));
  block.end_with_return ();


Python bindings
===============

See https://github.com/davidmalcolm/pygccjit:

.. code-block:: python

    # Create parameter "i":
    param_i = ctxt.new_param(int_type, b'i')
    # Create the function:
    fn = ctxt.new_function(gccjit.FunctionKind.EXPORTED,
                           int_type,
                           b"square",
                           [param_i])

.. nextslide::
   :increment:

.. code-block:: python

    # Create a basic block within the function:
    block = fn.new_block(b'entry')

    # This basic block is relatively simple:
    block.end_with_return(
        ctxt.new_binary_op(gccjit.BinaryOp.MULT,
                           int_type,
                           param_i, param_i))

    # Having populated the context, compile it.
    jit_result = ctxt.compile()

    # This is what you get back from ctxt.compile():
    assert isinstance(jit_result, gccjit.Result)


Bindings for other languages?
=============================

Yes please!


Internals of how it works
=========================
The original way it worked:

How it now works:


What would it take to get it merged?
====================================

State removal: the clean way vs the hack
========================================


.. Interesting commits:

    https://gcc.gnu.org/git/?p=gcc.git;a=commitdiff;h=96b218c9a1d5f39fb649e02c0e77586b180e8516

.. The TODO.rst list

.. Bug list?

.. FIXME:

   we can't implement the macro-based Py_DECREF until we have function ptrs (to jumping through tp_dealloc)
