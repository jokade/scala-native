================================================
Scala Native Interop for External Object Systems
================================================

Abstract
========
This document investigates use cases and possible design choices for Scala Native interop
with external object systems / class-based languages.
Only systems that can be accessed from pure C are considered (i.e. C++ classes are not in the scope of this document).

The analysis of required enhancements to support interop with these systems is based on the capabilities of Scala Native 0.3.6. 

.. contents:: Contents
  :depth: 3

Informal C Objects
==================
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

C Example
---------

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
  
  Counter* counter_copy(Counter* self) {
    Counter* new = (Counter*)malloc(sizeof(Counter));
    new->count = self->count;
    return new;
  }
  
  Counter* counter1 = counter_new(0);
  
  Counter* counter2 = counter_clone(counter1);
  counter_add(counter1,2);
  printf("counter1=%d  counter2=%d\n",counter->count);
  
**Real world examples**: GLib, Gtk+ [1]_

Scala Interop
-------------
Libraries following this convention can already be used from Scala Native (0.3.6) by calling the
external functions and providing the instance pointer explicitly. However, to use such libraries with idiomatic
Scala, we'd like to use dot notation instead, i.e.

.. code:: Scala

  val counter1 = Counter(0)        // calls counter_new(0)
  val counter2 = counter1.copy()   // calls counter_copy(counter1)
  counter.add(2)                   // calls counter_add(counter1,2)
  println(s"counter1=${counter1.count} counter2=${counter2.count}") // access C structs
  

Since there are no type hierarchies or interface contracts, there is usually no need to define such a type in Scala.
However, if need should arise, it is possible to do so using the exisiting C interop features
(provided it is possible to define arbitrary structures in Scala).

Current Implementation Patterns
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Currently there are two ways to represent an informal C object in Scala and support method call syntax for it.

The first wraps the reference to the external object in a plain Scala class, and uses the companion object to define the external bindings and the constructor function:

.. code:: Scala

  /* First pattern: use a wrapper class */
  class Counter private(ref: Ptr[Byte]) {
    def this(init: Int) = this(Counter.ext.counter_new(init))
    
    @inline def add(incr: Int): Int = Counter.ext.counter_add(ref,incr)
    @inline def copy(): Counter = new Counter( Counter.ext.counter_clone(ref) )
  }
  object Counter {
    def apply(init: Int): Counter = new Counter(ext.counter_new(init))
    
    // we need to define a separate object for external bindings
    // since we cannot mix normal code with external bindings in SN 0.3.6
    @extern
    object ext {
      def counter_new(init: Int): Ptr[Byte] = extern
      def counter_add(self: Ptr[Byte], incr: Int): Int = extern
      def counter_copy(self: Ptr[Byte]): Ptr[Byte] = extern
    }
  }

This pattern allows instantiation and method calls as required above. In addition, it also supports object instantiation
with ``new``. Property access is also straigthforward: if the library uses accessors for properties, we just add them to the class definition. If it uses direct access via the C struct, we can use the struct type for the `ref`:

.. code:: Scala

  class Counter private(ref: Ptr[CStruct1[Int]]) {
    @inline def count: Int = !ref._1
    ...
  }

An alternative pattern is to use a class (or trait) to represent the reference directly in Scala, and to cast ``this``
into a ``Ptr[Byte]`` or ``Ptr[CStruct]`` whenever it is accessed:

.. code:: Scala

  /* Second pattern: use class as type for external reference */
  class Counter private() {
    @inline def count: Int = !(this.cast[Ptr[CStruct1[Int]]])._1
    @inline def add(incr: Int): Int = Counter.ext.counter_add(this.cast[Ptr[Byte]], incr)
    @inline def copy(): Counter = Counter.ext.counter.clone(this.cast[Ptr[Byte]]).cast[Object].asInstanceOf[Counter]
  }
  object Counter {
    def apply(init: Int): Counter = ext.counter_new(init).cast[Object].asInstanceOf[Counter]
    @extern
    object ext {
      def counter_new(init: Int): Ptr[Byte] = extern
      def counter_add(self: Ptr[Byte], incr: Int): Int = extern
      def counter_copy(self: Ptr[Byte]): Ptr[Byte] = extern
    }
  }
  
This pattern supports all requirements, with the exception that instantiation with ``new`` is not possible.

However, this is semantically a gray area: although it works fine for every-day use cases, we're misusing
a normal Scala type to represent an external reference. This requires for example, that method calls don't
require data stored with the runtime representation of a class instance, since it doesn't exist.

Most obviously, vals/vars cannot be used with this pattern:

.. code:: Scala

  class Counter ... {
    // This compiles with both patterns,
    // but access to incr will segfault with the second pattern at runtime,
    // while it works as expected for the first pattern
    var incr: Int = 1
    
    ...
  }

Syntactic Sugar
~~~~~~~~~~~~~~~

Advanced Use Cases
------------------
Mixing Scala Code with Externals
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
It should be possible to add rich extension methods to classes representing external objects, e.g.

.. code:: Scala

  class Counter ... {
    var defaultIncr = 42  // not supported by pattern (2)
    
    def add2(): Int = add(2)  // possible with both patterns
    def addDefault(): Int = add(defaultIncr)  // not supported with pattern (2)
    ...
  }
  
Modeling Implicit Type Hierarchies
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Although informal C object systems don't have an explicit notion of class hierarchies or interface contracts,
they can be implicitly present due to design and conventions, or partially formalized using preprocessor macros, unions, etc.

One paradigm in this category is the convention, where every object type must provide a set of functions with prescribed semantics, the canonical example being collection libraries.

Suppose we have two types, one for singly-linked lists, another for doubly-linked ones. Both provide a ``size`` and ``append`` operations:

.. code:: C

  typedef struct {
    ...
  } SList;
  
  SList* slist_new(void);
  SList* slist_append(SList* self, void* elem);
  int slist_size(SList self);
  
  typedef struct {
    ...
  } DList;
  
  DList* dlist_new(void);
  DList* dlist_append(DList* self, void* elem);
  int slist_size(DList self);
  
  
Of course, we'd like to abstract over both types in Scala and use them interchangeably, e.g.:

.. code:: Scala

  @extern(C)
  trait Appendable {
    def append(elem: Ptr[Byte]): Appendable = extern
    def size: Int = extern
  }
  
  @extern(C)
  class SList extends Appendable
  
  @extern(C)
  class DList extends Appendable
  
  
W.r.t external bindings, this must be semantically equivalent to:

.. code:: Scala

  class SList ... {
    @inline def append(elem: Ptr[Byte]): SList = SList( SList.ext.slist_append(...) )
    @inline def size: Int = SList.ext.slist_size(...)
  }
  object SList {
    @extern
    object ext {
      def slist_append(self: Ptr[Byte], elem: Ptr[Byte]): Ptr[Byte] = extern
      def slist_size(self: Ptr[Byte]): Int = extern
    }
  }

  class DList ... {
    @inline def append(elem: Ptr[Byte]): DList = DList( DList.ext.dlist_append(...) )
    @inline def size: Int = DList.ext.dlist_size(...)
  }
  object DList {
    @extern
    object ext {
      def dlist_append(self: Ptr[Byte], elem: Ptr[Byte]): Ptr[Byte] = extern
      def dlist_size(self: Ptr[Byte]): Int = extern
    }
  }
  
  
**Real world example**: GLib

GObject
=======
TBD

Objective-C
===========
TBD


Derived Requirements for Interop
================================
TBD


Design Proposals
================
TBD


.. [1] although Gtk+ it is actually based on GObject, Gtk+ applications can be created without explicit use of GObject features.
