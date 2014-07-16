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
 * coconut

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

* Very high-level API


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


Entities within a context
=========================

.. code-block:: c

  extern const char *
  gcc_jit_object_get_debug_string (gcc_jit_object *obj);

* Very useful for debugging

* Internal note: if you call this on an object, the ``const char *``
  has the same lifetime as the object.

.. nextslide::
   :increment:

.. blockdiag::

  diagram {
    gcc_jit_object <- gcc_jit_location;
    gcc_jit_object <- gcc_jit_type;
        gcc_jit_type <- gcc_jit_struct;
    gcc_jit_object <- gcc_jit_field;
    gcc_jit_object <- gcc_jit_function;
    gcc_jit_object <- gcc_jit_block;
    gcc_jit_object <- gcc_jit_rvalue;
      gcc_jit_rvalue <- gcc_jit_lvalue;
        gcc_jit_lvalue <- gcc_jit_param;
  }

Source-Code Locations
=====================

Optional, but useful to end-users

.. code-block:: c

  /* Use this to create locations: */
  extern gcc_jit_location *
  gcc_jit_context_new_location (gcc_jit_context *ctxt,
                                const char *filename,
                                int line,
                                int column);

  /* Need to turn on generation of debuginfo: */
  gcc_jit_context_set_bool_option (
    ctxt, GCC_JIT_BOOL_OPTION_DEBUGINFO, 1);


.. nextslide::
   :increment:

We can use this to single-step through the machine code
e.g. generated for bytecode::

  (gdb) break fibonacci
  (gdb) run
  Breakpoint 1, fibonacci (input=8) at main.cc:43
  43      DUP,
  (gdb) next
  47      PUSH_INT_CONST, 2,
  (gdb) next
  51      BINARY_INT_COMPARE_LT,
  (gdb) next
  55      JUMP_ABS_IF_TRUE, 17,
  (gdb) next
  59      DUP,
  (gdb) next
  63      PUSH_INT_CONST,  1,
  (gdb) next
  67      BINARY_INT_SUBTRACT,

Types
=====

Access to simple C types:

.. code-block:: c

   gcc_jit_type *int_type =
      gcc_jit_context_get_type (ctxt, GCC_JIT_TYPE_INT);

   gcc_jit_type *double_type =
      gcc_jit_context_get_type (ctxt, GCC_JIT_TYPE_DOUBLE);

   /* etc */

.. nextslide::
   :increment:

* structs
* function pointers
* const, volatile
* etc

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

Functions
=========

How to generate the equivalent of:

.. code-block:: c

     const char *
     test_string_literal (void)
     {
        return "hello world";
     }

.. nextslide::
   :increment:

.. code-block:: c

  gcc_jit_type *const_char_ptr_type =
    gcc_jit_context_get_type (ctxt, GCC_JIT_TYPE_CONST_CHAR_PTR);

  /* Build the test_fn.  */
  gcc_jit_function *test_fn =
    gcc_jit_context_new_function (ctxt, NULL,
                                  GCC_JIT_FUNCTION_EXPORTED,
                                  const_char_ptr_type,
                                  "test_string_literal",
                                  0, NULL,
                                  0);
  gcc_jit_block *block = gcc_jit_function_new_block (test_fn, NULL);

  gcc_jit_block_end_with_return (
    block, NULL,
    gcc_jit_context_new_string_literal (ctxt, "hello world"));

.. nextslide::
   :increment:

Example of a conditional:

.. code-block:: c

  /* if (i >= n) */
  gcc_jit_block_end_with_conditional (
    loop_cond, NULL,
    gcc_jit_context_new_comparison (
       ctxt, NULL,
       GCC_JIT_COMPARISON_GE,
       gcc_jit_lvalue_as_rvalue (i),
       gcc_jit_param_as_rvalue (n)),
    after_loop,
    loop_body);

.. nextslide::
   :increment:

.. code-block:: c

  /* sum += i * i */
  gcc_jit_block_add_assignment_op (
    loop_body, NULL,
    sum, /* lvalue */
    GCC_JIT_BINARY_OP_PLUS,
    gcc_jit_context_new_binary_op ( /* rvalue */
       ctxt, NULL,
       GCC_JIT_BINARY_OP_MULT, the_type,
       gcc_jit_lvalue_as_rvalue (i),
       gcc_jit_lvalue_as_rvalue (i)));


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


How it originally worked
========================
The original way it worked:

.. actdiag::

   diagram {
     api_calls -> compile -> toplev_main -> parse_file -> callback -> more_api_calls;

     lane client_code {
        label = "Client code";
        api_calls [label = "API calls"];
        callback [label = "Callback"];
        more_api_calls [label = "More API calls"];
     }
     lane jit_api {
        label = "JIT API";
        compile [label = "compile"];
     }
     lane jit_frontend {
        label = "JIT \"Frontend\"";
        parse_file [label = "parse_file"];
     }
     lane libbackend_a {
        label = "libbackend.a";
        toplev_main [label = "toplev_main"];
     }
  }

How it now works
================

.. actdiag::

   diagram {
     api_calls -> recording -> compile -> toplev_main
       -> parse_file -> playback;

     lane client_code {
        label = "Client code";
        api_calls [label = "API calls"];
        compile [label = "compile"];
     }
     lane jit_api {
        label = "JIT API";
        recording [label = "Recording"];
     }
     lane jit_frontend {
        label = "JIT \"Frontend\"";
        parse_file [label = "parse_file"];
     }
     lane libbackend_a {
        label = "libbackend.a";
        toplev_main [label = "toplev_main"];
        playback [label = "playback"];
     }
  }


State removal: the clean way vs the hack
========================================


What would it take to get it merged?
====================================


.. The TODO.rst list

.. Bug list?
