================================================
Scala Native Interop for External Object Systems
================================================

Abstract
========
This document investigates use cases and possible design choices for Scala Native interop
with external object systems / class-based languages.
Only systems that can be accessed from pure C are considered (i.e. C++ classes are not in the scope of this document).

The analysis of required enhancements to support interop with these systems is based on the capabilities of Scala Native 0.3.6. 


Considered Object Systems
=========================

Informal C Objects
------------------
A common idiom in plain C is to use structs to store the properties of an object, and to use functions that
take the object instance (= struct ptr) as their first argument, as object methods.

This technique is usually combined with a naming convention for the function names, requring that all functions that
operate on a sepcific object type (struct) share a common prefix derived from the struct name. Furthermore, implementations
of this system usually also adhere to the convention that a new object instance is created by a function named `$prefix_new`
($prefix being the aforementioned common prefix), that takes the initial property values as arguments.
Type hierarchies and inheritance are not explicitly supported in this system (but can be partially modelled using conventions
or preprocessor macros).

There are two schemes for accessing the instance properties:

- direct access to properties via the struc type
- providing accessor functions for every property (this technique is utilized by libraries that
  treat instances as opaque pointers of type ``void*``).

**C Example**:

.. code:: C

  typedef struct Counter {
    int count;
  } Counter;

  Counter* counter_new(int initial) {
    Counter* new = (Counter*)malloc(sizeof(Counter));
    new->count = initial;
    return new;
  }

  int counter_add(Counter* self, int incr) {
    self->count += incr;
    return self->count;
  }
  
  Counter* c = counter_new(0);
  counter_add(c,1);
  printf("%d\n",c->count);
  
**Real world examples**: GLib, Gtk+ [1]_

Scala Interop
~~~~~~~~~~~~~
Libraries adhering to this convention can already be used from Scala Native (0.3.6) by calling the
external functions and providing the instance pointer explicitly. However, to use such libraries with idiomatic
Scala, we'd like to use dot calls instead, i.e.

.. code:: Scala

  val counter = Counter(0)  // calls counter_new(0)
  counter.add(2)            // calls counter_add(counter,2)
  println(counter.count)    // access C struct
  

Since there are no type hierarchies or interface contracts, there is usually no need to define such a type in Scala.
However, if need should arise, it is possible to do so using the exisiting C interop features
(provided it is possible to define arbitrary structures in Scala).




GObject
-------
TBD

Objective-C
-----------
TBD


Derived Requirements for Interop
================================
TBD


Design Proposals
================
TBD


.. [1] although Gtk+ it is actually based on GObject, Gtk+ applications can be created without explicit use of GObject features.
