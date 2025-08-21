::

    Status: Draft
    Type: Feature
    Created: 2023-01-04
    Authors: Aljaž Mur Eržen <aljaz@edgedb.com>

===========================================
RFC 1020: Multi choice control flow
===========================================

This RFC proposes new syntax for control flow with multiple branches.


Motivation
==========

Usually known as switch, match or CASE statements, these flow control
constructs provide great convenience over using IF syntax in some cases.

Specification
=============

Syntactically, the operator requires a value to be matched and a 
block of cases that each have a condition and a value.

  MATCH <val> {
    <cond1> -> <expr1>,
    <cond2> -> <expr2>,
    ...
    <condN> -> <exprN>
  }


Semantically, MATCH operator is evaluated as follows:
1. evaluate <val>,
2. evaluate <cond1>,
3. compare <cond1> to <val> using `==` operator. 
   If true, return <expr1>
   If false, repeat steps 2. and 3. for next case.
4. If no case has matched, return empty set.


Rationale
=========


Syntactically, there are two dimensions of design to be considered:
- the leading keyword (common ones are `switch`, `match` and `CASE`),
- the cases syntax.

Even though `switch` has high adoption among c-style programming
languages, it is not a reserved keyword and would make this RFC
backwards incompatible. `match` seems to be the second most adopted.

For cases, arrow syntax was chosen based on similarity with function
definition syntax, which is semantically the closest construct in
EdgeQL.

Backwards Compatibility
=======================

There should not be any backwards compatibility issues,
as SWITCH is already a reserved keyword.


Security Implications
=====================

There are no security implications.


Rejected Alternative Ideas
==========================

There are no rejected alternative ideas.