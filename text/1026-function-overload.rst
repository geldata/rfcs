.. code::

   Status: Draft
   Type: Feature
   Created: 2024-01-23
   Authors: Aljaž Mur Eržen <aljaz@edgedb.com>

#######################################
 RFC 1026: Remove function overloading
#######################################

**********
 Abstract
**********

Current behavior of function and operator overloading requires a
complicated call resolution algorithm. This RFC proposes a new syntax
for defining multiple function and operator implementations and removing
implicit function overloading.

************
 Motivation
************

Currently, EdgeQL allows overloading function definitions. For example,
there are 11 functions defined under the name ``std::contains``:

.. code::

   CREATE FUNCTION std::contains(
       haystack: std::str, needle: std::str
   ) -> std::bool using (...);
   CREATE FUNCTION std::contains(
       haystack: std::bytes, needle: std::bytes
   ) -> std::bool using (...);
   CREATE FUNCTION std::contains(
       haystack: array<anytype>, needle: anytype
   ) -> std::bool using (...);
   CREATE FUNCTION std::contains(
       haystack: json, needle: json
   ) -> std::bool using (...);
   CREATE FUNCTION std::contains(
       haystack: range<anypoint>, needle: range<anypoint>
   ) -> std::bool using (...);
   CREATE FUNCTION std::contains(
       haystack: range<anypoint>, needle: anypoint
   ) -> std::bool using (...);
   CREATE FUNCTION std::contains(
       haystack: multirange<anypoint>, needle: multirange<anypoint>
   ) -> std::bool using (...);
   CREATE FUNCTION std::contains(
       haystack: multirange<anypoint>, needle: range<anypoint>
   ) -> std::bool using (...);
   CREATE FUNCTION std::contains(
       haystack: multirange<anypoint>, needle: anypoint
   ) -> std::bool using (...);
   CREATE FUNCTION std::contains(
       haystack: range<cal::local_date>, needle: cal::local_date
   ) -> std::bool using (...);
   CREATE FUNCTION std::contains(
       haystack: multirange<cal::local_date>, needle: cal::local_date
   ) -> std::bool using (...);

This behavior is desired, as it is useful to implement a function for
multiple types with slight variations. However, it complicates resolving
which definition should be used for a given call.

As functions can be defined at any location, these definitions do not
have any order. When a function is called, all definitions that match in
name are compared with the function call signature. First, all
definitions whose parameters are not super types of call's arguments are
discarded. Then, we compute a "call distance" between types of arguments
and parameters and pick the function definition with the lowest
distance.

It might happen that multiple definitions have an equal call distance,
in which case an error saying "please disambiguate" is raised.

Similar behavior is used for operators.

This behavior is mostly used in combination with pseudo-types:

   create function foo(a: anyscalar) -> str using ("matched anyscalar");
   create function foo(a: str) -> str using ("matched str");

   select foo("hello"); # {"matched str"} select foo(12); # {"matched
   anyscalar"}

Note that in user-defined functions, it is not allowed to use
pseudo-types or overload functions that take object types. Overloading
of functions that take scalar types does not work correctly. This means
that this overloading behavior cannot used outside of EdgeDB-defined
functions.

**********
 Proposal
**********

#. Add the following syntax:

   .. code::

      CREATE FUNCTION std::contains
          (haystack: std::str, needle: std::str) -> std::bool using (...) OR
          (haystack: std::bytes, needle: std::bytes) -> std::bool using (...) OR
          (haystack: array<anytype>, needle: anytype) -> std::bool using (...);

   Here, all implementations must be defined in a single declaration.
   Since they have an inherent order, this order can be used during call
   resolution. To resolve a call, we first find the single definition
   matching in name. Then we traverse the list of implementations from
   first to last and compare their signature to the call. First
   signature that matches is resolved to, regardless of potential match
   in the following implementations.

   Individual implementations in the list would be allowed to define any
   number of parameters with arbitrary types.

#. Add similar syntax for operators.
#. Forbid implicit overloading of functions and operators.

***************
 Justification
***************

This change would vastly simplify call resolution code.

It would also make it much more predictable. This is currently
problematic, as there is 36 implementations of ``std::`=```, including
implementations for ``anytype``, ``anyscalar``, ``anynumber``, each of
the numeric types and their combinations.

The simplified behavior would be easier to implement dynamically (i.e.
for dynamic dispatch), as it has to match a linear list, as opposed to
an inheritance tree. Currently, dynamic dispatch is supported only for
pseudo-types, using appropriate PostgreSQL pseudo-types.

******************
 Future expansion
******************

Function overloading is needed when an extension defines or extends a
type. Specifically, there is a need to implement ``std::`=```.

As extensions would not be able to alter the original function
definition, they do require overloading. That could be accomplished via
explicit an OVERLOADED keyword. Implementations created this way would
be inject in front of the list, before all other implementations. To
disambiguate between two OVERLOADED implementations, explicit references
between the implementations could be used.

************************
 Backward compatibility
************************

Because function overloading is currently useless in user-code, we can
consider it not part of current EdgeQL spec. This means that this change
is fully backwards compatible, if the proposed change can resolve all
calls to EdgeDB-defined functions and operators as they are resolved
currently.

That *should* be possible, although I am not certain.
