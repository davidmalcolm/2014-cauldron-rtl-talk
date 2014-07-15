A proposal for typesafe RTL
---------------------------

.. Typesafe RTL talk Sunday 2014-07-20 12.45->1.30

.. Title: A proposal for typesafe RTL

.. Author: David Malcolm <dmalcolm@redhat.com>

.. Abstract: The backend code's ENABLE_RTL_CHECKING ensures that only valid
   combinations of RTL are constructed, but this option is painfully slow to
   use, so AFAIK very few people with it, and only when close to release time.

   I'll talk about an approach I've been experimenting with in which the RTL
   codes are expressed in a simple way in the C++ type-system, so that this
   checking can be done when the compiler is built, rather than at run-time,
   allowing the checking to be on for all developers, throughout the
   development cycle.  As a further benefit, I believe it makes the backend
   code significantly more readable.


Introduction
============

* Who am I?  (and caveat)

* Motivation:

  * Readability

  * Type-checking

Patches can be seen at:
  http://dmalcolm.fedorapeople.org/gcc/patch-backups/rtx-classes/


Instructions vs Expressions
===========================

The backend code is full of variables of type ``rtx``.

Sometimes these are arbitrary expressions.

But often they are *instructions*, intended to be part
of a linked-list.

Sometimes naming conventions help e.g. "insn", "seq".

But not always.

I want to use types to identify instructions.


Proposed class hierarchy
========================

Subclasses of ``rtx_def``:

.. code-block:: c++

  class rtx_def;
    class rtx_insn; /* (INSN_P (X) || NOTE_P (X)
                       || JUMP_TABLE_DATA_P (X) || BARRIER_P (X)
                       || LABEL_P (X)) */
      class rtx_real_insn;         /* INSN_P (X) */
        class rtx_debug_insn;      /* DEBUG_INSN_P (X) */
        class rtx_nonjump_insn;    /* NONJUMP_INSN_P (X) */
        class rtx_jump_insn;       /* JUMP_P (X) */
        class rtx_call_insn;       /* CALL_P (X) */
      class rtx_jump_table_data;   /* JUMP_TABLE_DATA_P (X) */
      class rtx_barrier;           /* BARRIER_P (X) */
      class rtx_code_label;        /* LABEL_P (X) */
      class rtx_note;              /* NOTE_P (X) */


How many places?
================
Number of "grep" hits in my current working copy
(including "config" subdirs):

.. code-block:: c++

  class rtx_def;                   /* 25731 for "rtx" -w */
    class rtx_insn;                /* 3606 */
      class rtx_real_insn;         /* 9 */
        class rtx_debug_insn;      /* 5 */
        class rtx_nonjump_insn;    /* 3 */
        class rtx_jump_insn;       /* 5 */
        class rtx_call_insn;       /* 17 */
      class rtx_jump_table_data;   /* 32 */
      class rtx_barrier;           /* 14 */
      class rtx_code_label;        /* 162 */
      class rtx_note;              /* 49 */


First attempt
=============

Megapatch!

(Not going to fly)


Second attempt "v1"
===================

Goals:

  (A) Reviewable: series of *small* logically-divided patches

  (B) Incremental: Able to compile and run without regressions at each
      patch, on every configuration

.. I managed (A), kind-of (though I didn't write ChangeLogs)

.. I didn't manage (B); only builds on 70 configrations out of ~200


Dealing with interdependencies
==============================

Current approach: 4 phases:
  1) Add scaffolding (currently 44 patches)
  2) Per-source file (currently 77 patches)
  3) Per-config dir (currently 40 patches)
  4) Remove scaffolding (currently 42 patches)


Some of the places that I have using ``rtx_insn *``
===================================================
* Basic blocks: ``BB_HEAD``, ``BB_END``, ``BB_HEADER``, ``BB_FOOTER``.
* Result of ``NEXT_INSN`` and ``PREV_INSN``.
* Results of ``next_insn``, ``prev_nonnote_insn`` et al
* Hundreds of function params, struct fields, etc.

  * e.g. within register allocators, schedulers

* "insn" and "curr_insn" within .md files (peephole, attributes,
  define_bypass guards)

.. nextslide::
   :increment:

* ``insn_t`` in `sel-sched-ir.h`
* Target hooks: updated params of 25 of them
* Debug hooks: "label" and "var_location"
* Result of ``DF_REF_INSN``
* ``DEP_PRO`` and ``DEP_CON``
* ``VINSN_INSN_RTX``
* ``BB_NOTE_LIST``
* etc

jump tables
===========

The current prototype for ``tablejump_p``:

.. code-block:: c++

   extern bool tablejump_p (const_rtx, rtx *, rtx *);

Aside: can we please add names to parameters in header files?
I'd much rather this was written:

.. code-block:: c++

   extern bool
   tablejump_p (const_rtx insn, rtx *labelp, rtx *tablep);


.. nextslide::
   :increment:

The current prototype (with param names added):

.. code-block:: c++

   extern bool
   tablejump_p (const_rtx insn, rtx *labelp, rtx *tablep);

Using subclasses:

.. code-block:: c++

   extern bool
   tablejump_p (const rtx_insn *insn,
                rtx_code_label **labelp,
                rtx_jump_table_data **tablep);

.. nextslide::
   :increment:

Code that looks like this (from cfgbuild.c):

.. code-block:: c++

     else if (tablejump_p (insn, NULL, &table))
       {
         rtvec vec;
         int j;

         /* This happens in 5 places in the backend */
         if (GET_CODE (PATTERN (table)) == ADDR_VEC)
           vec = XVEC (PATTERN (table), 0);
         else
           vec = XVEC (PATTERN (table), 1);

         for (j = GET_NUM_ELEM (vec) - 1; j >= 0; --j)
           make_label_edge (edge_cache, bb,
                            XEXP (RTVEC_ELT (vec, j), 0), 0);


.. nextslide::
   :increment:

can be simplified by adding a ``get_labels`` method to the
``JUMP_TABLE_DATA`` subclass:

.. code-block:: c++

    else if (tablejump_p (insn, NULL, &table))
      {
        rtvec vec = table->get_labels (); /* do the work here */
        int j;

        for (j = GET_NUM_ELEM (vec) - 1; j >= 0; --j)
          make_label_edge (edge_cache, bb,
                           XEXP (RTVEC_ELT (vec, j), 0), 0);


.. nextslide::
   :increment:

and further simplified by making it a vec of ``LABEL_REF``, assuming that we
can have a ``rtx_label_ref::label`` method for getting the ``CODE_LABEL``:

.. code-block:: c++

    else if (tablejump_p (insn, NULL, &table))
      {
        vec <rtx_label_ref *> vec = table->get_labels ();
        int j;

        for (j = GET_NUM_ELEM (vec) - 1; j >= 0; --j)
          make_label_edge (edge_cache, bb,
                           vec [j]->label (), 0);


Status of insn separation
=========================
Currently at 209 patches:

.. code-block:: c++

  class rtx_def;                   /* 25731 for "rtx" -w */
    class rtx_insn;                /* 3606 */
      class rtx_real_insn;         /* 9 */
        class rtx_debug_insn;      /* 5 */
        class rtx_nonjump_insn;    /* 3 */
        class rtx_jump_insn;       /* 5 */
        class rtx_call_insn;       /* 17 */
      class rtx_jump_table_data;   /* 32 */
      class rtx_barrier;           /* 14 */
      class rtx_code_label;        /* 162 */
      class rtx_note;              /* 49 */


Full separation?
================
Is it worthwhile/desirable to pursue a full separation of instructions
from rtx nodes?

e.g. something like this as the base class:

.. code-block:: c++

  class rtx_insn /* we can bikeshed over the name */
  {
  public:
    rtx_insn *m_prev;
    rtx_insn *m_next;
    int m_uid;
  };

  #define PREV_INSN(INSN) ((INSN)->m_prev)
  #define NEXT_INSN(INSN) ((INSN)->m_next)
  #define INSN_UID(INSN)  ((INSN)->m_uid)
    /* or we could convert them to functions returning
       references, I guess */

.. nextslide::
   :increment:

Tricky - what about:
  * params of every ``PREV_INSN``, ``NEXT_INSN``, ``INSN_UID``
  * ``PATTERN(INSN)``
  * ``BLOCK_FOR_INSN(INSN)``
  * etc


NULL_RTX
========

We have:

.. code-block:: c++

  #define NULL_RTX (rtx) 0

Do we want a ``NULL_INSN``?

Where do we draw the line?

(NULL_CODE_LABEL, NULL_JUMP_TABLE_DATA etc???)


Other classes:
==============
  * INSN_LIST
  * EXPR_LIST
  * SEQUENCE
  * SET (and single_set)??
  * PARALLEL?
      .. perhaps a "rtx_compound" parent class for both SEQUENCE and
         PARALLEL? see var-tracking.c: insn_stack_adjust_offset_pre_post

"Phase 5" of my patch kit

EXPR_LIST
=========

From reload1.c: set_initial_label_offsets:

.. code-block:: c++

  for (x = forced_labels; x; x = XEXP (x, 1))
    if (XEXP (x, 0))
      set_label_offsets (XEXP (x, 0), NULL_RTX, 1);

  for (x = nonlocal_goto_handler_labels; x; x = XEXP (x, 1))
    if (XEXP (x, 0))
      set_label_offsets (XEXP (x, 0), NULL_RTX, 1);

.. nextslide::
   :increment:

Using subclasses:

.. code-block:: c++

  for (rtx_expr_list *x = forced_labels; x; x = x->next ())
    if (x->element ())
      set_label_offsets (x->element (), NULL, 1);

  for (rtx_expr_list *x = nonlocal_goto_handler_labels; x; x = x->next ())
    if (x->element ())
      set_label_offsets (x->element (), NULL, 1);

INSN_LIST
=========

e.g. in sched-int.h struct deps_desc::

     /* A list of the last function calls we have seen.  We use a list to
        represent last function calls from multiple predecessor blocks.
        Used to prevent register lifetimes from expanding unnecessarily.  */
  -  rtx last_function_call;
  +  rtx_insn_list *last_function_call;

(9 of these in this struct)

SEQUENCE
========

From resource.c:find_dead_or_set_registers:

.. code-block:: c++

      for (i = 1; i < XVECLEN (PATTERN (insn), 0); i++)
        INSN_FROM_TARGET_P (XVECEXP (PATTERN (insn), 0, i))
          = ! INSN_FROM_TARGET_P (XVECEXP (PATTERN (insn), 0, i));

Can be rewritten as:

.. code-block:: c++

      rtx_sequence *seq = as_a <rtx_sequence *> (PATTERN (insn));
      for (i = 1; i < seq->len (); i++)
        INSN_FROM_TARGET_P (seq->element (i))
          = ! INSN_FROM_TARGET_P (seq->element (i));

Difficulties
============

Using a single "tmp" local for multiple things:

.. code-block:: c++

  rtx tmp;

Or reusing a local for both a pattern and an insn e.g.:

.. code-block:: c++

  /* Emit a debug bind insn before the insn in which
    reg dies.  */
  bind = gen_rtx_VAR_LOCATION (GET_MODE (SET_DEST (set)),
                               DEBUG_EXPR_TREE_DECL (dval),
                               SET_SRC (set),
                               VAR_INIT_STATUS_INITIALIZED);
  count_reg_usage (bind, counts + nreg, NULL_RTX, 1);

  bind = emit_debug_insn_before (bind, insn);
  df_insn_rescan (bind);

.. from cse.c:delete_trivially_dead_insns

.. nextslide::
   :increment:

We can fix the above by splitting local "bind" into:

  * an ``rtx`` for the ``VAR_LOCATION`` and

  * an ``rtx_insn *`` for the ``DEBUG_INSN``

.. code-block:: c++

  bind_var_loc = gen_rtx_VAR_LOCATION (GET_MODE (SET_DEST (set)),
                                       DEBUG_EXPR_TREE_DECL (dval),
                                       SET_SRC (set),
                                       VAR_INIT_STATUS_INITIALIZED);
  count_reg_usage (bind_var_loc, counts + nreg, NULL_RTX, 1);

  bind_insn = emit_debug_insn_before (bind_var_loc, insn);
  df_insn_rescan (bind_insn);


Taking it further?
==================
Adding classes per ``DEF_RTL_EXPR``?

* Converting operands to actual fields, with types
* Converting e.g. ``XINT(RTX, N)`` to lookup of ``m_fieldN``

This would give us the equivalent of today's ``ENABLE_RTL_CHECKING``
with no compile-time cost.

But very invasive.

.. nextslide::
   :increment:

e.g. introducing named accessors as well as types for operands
of ``DEF_RTL_EXPR``

Kind of silly with e.g. ``RTX_BIN_ARITH``?


Summary
=======

* I think a ``rtx_insn`` subclass of ``rtx_def`` is a "sweet spot" of

  * big gain in type-safety, readability

  * not too big an unheaval

* I want to get this into trunk for stage 1 of 4.10/5.0 after 4.9.1
  is released.


Questions & Discusssion
=======================

Thanks for listening!

.. Notes:

    the mn10300 thing

    other insn subclasses

      maybe "struct deps_desc" in sched-int.h???
      (perhaps use for CALL_INSN_FUNCTION_USAGE?)
    etc... any others?
      maybe SET (and single_set)??
      maybe PARALLEL?  (perhaps a "rtx_compound" parent class for
        both SEQUENCE and PARALLEL? see
        var-tracking.c: insn_stack_adjust_offset_pre_post)
      see my TODO.txt class hierarchy:
        e.g. unary ops/binary ops??
      maybe INT_LIST ??????
    generate rtl.def from a meta.md; names and types for attributes as an alternate access strategy?
      what was Oleg's suggestion?

    NULL_RTX vs NULL.  Do we want NULL_INSN, NULL_INSN_LIST, NULL_EXPR_LIST etc?

    I apologize in advance to the arc port maintainer (Note to self: Joern Rennecke)

    From gcc/config/arc.c:arc_reorg (with line numbers):
    5769      if (GET_CODE (insn) == JUMP_INSN
    5770          && recog_memoized (insn) == CODE_FOR_doloop_end_i)
    5771        {
    5772           rtx top_label = (XEXP (XEXP (SET_SRC (XVECEXP (PATTERN (insn), 0, 0)), 1), 0));

    Quick quiz: what does line 5772 do?

      rtx top_label = (XEXP (XEXP (SET_SRC (XVECEXP (PATTERN (insn), 0, 0)), 1), 0));

    Adding gratuitious indentation to show structure:

      rtx top_label = (XEXP (XEXP (SET_SRC (XVECEXP (PATTERN (insn),
                                                     0,
                                                     0)
                                            ),
                                   1),
                             0)
                       );

    This could be rewritten with locals as:

      rtx pattern = PATTERN (insn);
      rtx pattern_elem0 = XVECEXP (pattern, 0, 0);
      rtx set_src_pattern_elem0 = SET_SRC (pattern_elem0);
      rtx set_src_pattern_elem0_1 = XEXP (set_src_pattern_elem0, 1);
      rtx top_label = (XEXP (set_src_pattern_elem0_1, 0));

    TODO: it's not clear to me what this actually is doing.
