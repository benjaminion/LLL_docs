******************
Language Reference
******************

LLL Syntax
==========


Expressions
-----------

Influenced by Lisp, everything in LLL should be thought of as expressions to be
evaluated, rather than instructions to be executed. To quote `Wikipedia on Lisp
<https://en.wikipedia.org/wiki/Lisp_(programming_language)>`_, "The
interchangeability of code and data gives Lisp its instantly recognizable
syntax."

An LLL expression is any of the following.

 - An integer, optionally prefixed with "0x" for hex base: ``42`` or
   ``0x2a``. Negative numbers cannot be directly input (do ``(sub 0 N)`` or use
   the ``~`` `bitwise not operator`_).

 - A string (see below).
   
 - An atom that has been defined previously via an argument-less ``def``
   expression.  E.g. with an existing definition ``(def 'm (memsize))``, then
   ``m`` can be used as an atom - an expression without parentheses - which
   will expand to ``(msize)``.

 - An evaluated expression which takes the form of a parenthesised list of an
   operator followed by zero or more expressions which are the operands, ``(OP
   EXPR1 EXPR2 ...)``, e.g. ``(add 1 2)``.  The operator ``OP`` is either a
   built-in `EVM opcode`_, or a `parser expression`_, or a macro defined in
   terms of these.

The above items are the basic syntax, and the last of these explains why
parentheses are everywhere in LLL. There are some variations to the syntax
introduced by the compact form described below.

   
Compact notation
----------------

A compact form is available for certain expressions, particularly memory
accesses. Used wisely it can enhance the readability of the code. Using the
compact forms does not affect the code generated; the parser simply makes the
substitutions below. Note that in all cases any whitespace is optional.

``{ ... }``
  ``(seq ...)`` - evaluates a sequence of expressions in order.

``@ EXPR``
  ``(mload EXPR)`` - loads the value of memory location EXPR.

``@@ EXPR``
  ``(sload EXPR)`` - loads the value of storage location EXPR.

``[ EXPR1 ] EXPR2``
  ``(mstore EXPR1 EXPR2)`` - writes the value of EXPR2 into memory location
  EXPR1.  A colon, ``:``, may optionally be added after the colsing bracket if
  desired for readability.

``[[ EXPR1 ]] EXPR2``
  ``(sstore EXPR1 EXPR2)`` - writes the value of EXPR2 into storage location
  EXPR1.  A colon, ``:``, may optionally be added after the colsing bracket if 
  desired for readability.

``$ EXPR``
  ``(calldataload EXPR)`` - reads a word of call data starting from byte EXPR.


.. _rules for strings:

Strings
-------

A string in LLL is represented internally as a single 32 byte word, with the
string left-alighed to the high-order byte in the (big endian) word. Any
characters beyond the 32nd are ignored by the compiler.  Permissible characters
in strings vary according to which representation is used.

There are two ways to represent strings in LLL:

 1. As a fully quoted string: ``"Hello, world!"``.  Note that there is no
    escaping mechanism to allow a double quote character, ``"``, to be included
    within the string. With that sole exception, all characters are OK
    including whitespace and newlines.

 2. With a single quote at the beginning: ``'forty-two``.  The string may not
    include any whitespace characters, or the characters used in compact
    notation, parentheses or comments (``{``, ``}``, ``[``, ``]``, ``@``,
    ``$``, ``(``, ``)``, ``:``, ``;``). However, both single and double quotes
    *are* allowed.  ``'forty-two`` is identical to ``"forty-two"`` as far as
    the parser is concerned.

The following are all valid strings, also shown with their internal hexadecimal
representations::

  "Hello, world!"
  0x48656c6c6f2c20776f726c642100000000000000000000000000000000000000

  "$£¥€ - {}[]@():;"
  0x24c2a3c2a5e282ac202d207b7d5b5d4028293a3b000000000000000000000000

  'forty-two
  0x666f7274792d74776f0000000000000000000000000000000000000000000000

  '"forty-two"
  0x22666f7274792d74776f22000000000000000000000000000000000000000000

  'こんにちは世界
  0xe38193e38293e381abe381a1e381afe4b896e7958c0000000000000000000000

  
Comments
--------

A comment begins with a semicolon, ``;``, and finishes at the end of the line.
In common with Lisp, varying numbers of semicolons may be used to begin a
comment depending on context, but this is purely stylistic.


Common conventions
------------------

LLL code is generally written with everything in lower case, except in `asm`
expressions where the EVM opcodes tend to be uppercased. Internally for
parsing, all expressions are converted to uppercase except for
literals/strings.

Literals, such as definition names created by ``def`` expressions, are case
sensitive, so ``(def 'x 42)`` and ``(def 'X 42)`` define two distinct macros.

.. _EVM opcode:

EVM Opcodes
===========

All valid EVM opcodes are automatically valid LLL operations, so expressions
like the following all work as standard::

  (add 1 2)
  (return 0x00 0x20)
  (calldatasize)
  (call allgas to value 0 0 0 0)

In terms of the EVM, the operands can be considered as items on the stack (the
left-most being at the top of the stack) upon which the EVM operation works.

[Link to the source file where these are listed:
solidity/libevmasm/Instruction.cpp]

[Warnings about push, pop, and jump]

.. _parser expression:

Parser expressions
==================

In addition to the EVM opcodes, the the LLL parser provides a number of other
operators for convenience.

Arithmetic Operators
--------------------

Multi-ary
^^^^^^^^^

The following arithmetic operators can take one or more arguments.

``+``
  ``(+ 1 2 3 4 5)`` evaluates to 15.
  
``-``
  ``(- 1 2 3 4 5)`` evaluates to -13.
  
``*``
  ``(* 1 2 3 4 5)`` evaluates to 120.

``/``
  ``(/ 60 2 3)`` evaluates to 10.
  
``%`` - modulus operation.
  ``(% 67 10 3)`` evaluates to 1, i.e. (67%10)%3.

``&`` - bitwise `and`.
  ``(& 15 6 4)`` evaluates to 4.

``|`` - bitwise `or`.
  ``(| 4 5 6)`` evaluates to 7.

``^`` - bitwise `xor`.
  ``(^ 1 2 3)`` evaluates to 0.

When only one argument is provided then the expression evaluates to the value
of that argument. I.e. ``(/ 5)`` evaluates to 5.


Binary
^^^^^^

Binary comparison operators are available with the usual meanings: ``<`` ,
``<=`` , ``>`` , ``>=`` , ``=``, ``!=``.  If the comparison is true then they
evaluate to 1: ``(< 4 5)`` -> 1.  If the comparison is false they evaluate to
0: ``(> 4 5)`` -> 0.

Note that ``<`` , ``<=`` , ``>`` , ``>=`` all perform unsigned comparisons. So,
``(> 1 (- 0 1))`` evaluates to false, for example, which may be unexpected.

In addition, there are four signed comparison operators: ``S<`` , ``S<=`` ,
``S>`` , ``S>=``. Thus, ``(S> 1 (- 0 1))`` evaluates as true.


Unary
^^^^^

.. _bitwise not operator:

``~`` is a bitwise not, corresponding to the EVM's ``NOT`` operation - it
inverts all the bits in the operand (treated as a 32 byte word).

With care, this provides a compact way to specify negative numbers In the EVM's
`twos-complement arithmetic
<https://en.wikipedia.org/wiki/Two%27s_complement>`_. ``(~ 4)`` is equivalent
to -5, so ``(+ 5 (~ 4))`` evaluates to zero.


Macro definition - ``def``
--------------------------

Overview
^^^^^^^^

LLL macros provide a powerful way to make writing LLL code efficient.

There are two forms of macro definition. In the following, ``NAME`` is a quoted
macro name as per the rules below, and ``name`` is the unquoted version,
i.e. ``'foo`` and ``foo`` respectively.

  * ``(def NAME EXPR)`` defines a macro without arguments, such as a
    constant. Wherever the atom ``name`` appears, it will be substituted with
    ``EXPR``.

    E.g. ``{(def 'foo 42) foo)}`` evaluates to 42. 

  * ``(def NAME (ARG1 ARG2 ...) EXPR)`` defines a macro with zero or more
    arguments. When the expression ``(name ARG1 ARG2 ...)`` appears it will be
    substituted with the arguments passed to EXPR.

    E.g after defining ``(def 'sum (l r) (+ l r))``, the expression ``(sum 2
    3)`` will evaluate to 5.  to 5. And after defining ``(def 'panic () 0xfe)``
    then the expression ``(panic)`` will insert an invalid opcode, causing the
    EVM to throw.

    Macros with the same name but differing numbers of arguments are treated as
    different macros and do not conflict with each other.

Macros can be defined in terms of other macros and expansion will occur
recursively until only native expressions remain.

Here's a single-argument macro for returning a string value
(``string-literal``) in the correct ABI format. The string literal here is not
limited to 32 bytes. ::

  (def 'return-string (string-literal)
    (seq
      (mstore 0x00 0x20)           ; Points to our string's memory location
      (mstore 0x20 string-literal) ; Length. String itself is copied to 0x40.
      (return 0x00 (+ 0x40 (mload 0x20)))))

  (return-string (lit 0x40 "Hello, world!"))

[todo - less clunky example?]


Macro names
^^^^^^^^^^^

Although the ``def`` expression allows a wide latitude in assigning macro
names, some restrictions apply if the macro name is to be usable. Essentially,
the same rules apply as for single quoted strings, except that,

 * there is no upper bound on length,
 * a double quote mark may not be used in the name (single quote is OK), and
 * the name may not begin with a numeral.

All of the following correctly evaluate to 100, but are perhaps ill-advised::

  {(def '£ 100) £}
  {(def 'a' 100) a'}
  {(def 'a (sub 0 100)) (def '-a (sub 0 a)) -a}
  {(def 'thismacronameislongerthan32characters 100) thismacronameislongerthan32characters}


Macro scope
^^^^^^^^^^^

From Gav Wood's `original documentation
<https://github.com/ethereum/cpp-ethereum/wiki/LLL-PoC-6/04fae9e627ac84d771faddcf60098ad09230ab58>`_,
the following applies to macro scoping. If anyone can make sense of this (or
the `source code
<https://github.com/ethereum/solidity/blob/develop/liblll/CodeFragment.cpp>`_)
so as to explain it more simply. I would be most grateful.

    Environmental definitions at the time of the macro's definition are
    recorded alongside the macro. When a definition must be resolved,
    definitions made within the present macro are treated preferentially, then
    arguments to the macro, then definitions & arguments made in the scope of
    which the macro was called (treated in the same order), then finally
    definitions & arguments stemming from the scope in which the macro was
    created (again, treated in the same order).


Including files - ``include``
-----------------------------

``(include "filename.lll")`` inserts the contents of *filename.lll* at this
point in the code being parsed. Note that, as ever, subject to the `rules for
strings`_ (except that the length is unlimited), the filename can be given as a
single quoted string: ``(include 'filename.lll)``.

``include`` can appear anywhere an expression would be valid. For example, this
is fine and returns whatever the code in *foo.lll* evaluated to: ``(return
(include "foo.lll"))``.

More often, ``include`` might be used to insert external libraries of common
macro definitions shared between projects.


Control structures
------------------


``seq``
^^^^^^^

``(seq EXPR1 EXPR2 ...)`` evaluates all following expressions in order.  It
evaluates to the result of the final expression given.


``raw``
^^^^^^^

``(raw EXPR1 EXPR2 ...)`` evaluates all following expressions in order.  It
evaluates to the result of the first non-void expression (i.e. the first
expression that leaves anything on the stack), or void if there is none.


``if``
^^^^^^

This is an "if-then-else" construction.

In ``(if PRED Y N)``: when the predicate ``PRED`` evaluates to non-zero, ``Y``
is evaluated; when ``PRED`` evaluates to zero, ``N`` is evaluated.


``when``, ``unless``
^^^^^^^^^^^^^^^^^^^^

``(when PRED BODY)`` evaluates ``BODY``, discarding any result, if and only if
``PRED`` evaluates to a non-zero value.

For example, a "not-payable" guard::

   (when (callvalue) revert)

``(unless PRED BODY)`` evaluates ``BODY``, discarding any result, if and only
if ``PRED`` evaluates to zero.

A guard for checking that exactly one argument has been passed in the call
data::

  (unless
    (= 0x24 (calldatasize))
    revert)


``while``, ``until``
^^^^^^^^^^^^^^^^^^^^

``(while PRED BODY)`` evaluates ``PRED`` and if the result is non-zero
evaluates ``BODY``, discarding the result. This is repeated while ``PRED``
remains non-zero.

Let's say you are putting data into contract storage at consecutive locations
starting at zero. The following will count how many items you have. (For fewer
than a hundred or so items it's likely cheaper to re-count them than to store a
count separately.) ::

  (seq
    [0x00]:0
    (while (sload @0x00) [0x00]:(+ 1 @0x00))
    (return 0x00 0x20))

``(until PRED BODY)`` is the same as ``while`` except that it evaluates
``BODY`` when ``PRED`` is zero until and continues until it becomes non-zero.

Return the number of leading zero bytes in the call data (up to 32 max)::

  (seq
    [0x20]:(calldataload 0x04)
    (until
      (or (= @0x00 32) (byte @0x00 @0x20))
      [0x00]:(+ 1 @0x00))
    (return 0x00 0x20))


``for``
^^^^^^^

``(for INIT PRED POST BODY)`` evaluates ``INIT`` once (ignoring any result),
then evaluates ``BODY`` and ``POST`` (discarding the result of both) as long as
``PRED`` is true.

The following code computes factorials: 10! = 3628800 = 0x375f00 in this
case. ::
   
    (seq
      (for
        (seq (set 'i 1) (set 'j 1))      ; INIT
        (<= (get 'i) 10)                 ; PRED
        (mstore i (+ (get 'i) 1))        ; POST
        (mstore j (* (get 'j) (get 'i))) ; BODY
        )
      (return j 0x20))

This is one of the rare occasions where I think the compact notation is
actually an improvement. The following compiles to the same bytecode. ::

    (seq
      (for
        { (set 'i 1) (set 'j 1) } ; INIT
        (<= @i 10)                ; PRED
        [i]:(+ @i 1)              ; POST
        [j]:(* @j @i))            ; BODY
      (return j 0x20)))

      
Logical Operators ``&&``, ``||``, ``!``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Logical "and", "or" and "not".

Both ``&&`` and ``||`` can take any non-zero number of arguments. They evaluate
the arguments from left to right and perform `short circuit evaluation
<https://en.wikipedia.org/wiki/Short-circuit_evaluation>`_ so that evaluation
of arguments stops as soon as the outcome is known. I.e. ``(&& EXPR1 EXPR2
...)`` will stop evaluating after encountering an expression that evaluates to
zero; ``(|| EXPR1 EXPR2 ...)`` will stop evaluating after encountering an
expression that evaluates to non-zero.

``!`` is a unary logical not operator, thus it takes one argument. ``(! EXPR)``
evaluates to zero when ``EXPR`` evaluates to non-zero, and to one when ``EXPR``
evaluates to zero. It is equivalent to ``(iszero EXPR)``.


Literals - ``lit``
------------------

When literals must be included that can be placed into memory, there is the
``lit`` operation.

 * ``(lit POS STRING)`` Places the string ``STRING`` into memory at ``POS`` and
   evaluates to its length. The usual `rules for strings`_ apply, except that
   there is no limit on the length.
   
 * ``(lit POS BIGINT)`` Places ``BIGINT`` into memory at POS and evaluates to
   the number of bytes it takes. Unlike for the previous case, ``BIGINT`` may
   be arbitrarily large, and thus if specified as hex, can facilitate storing
   arbitrary binary data.

So, ``(lit 0x40 "Hello, world!")`` copies the string to memory starting at byte
0x40 and returns 13, the length of the string.

[Note that the `former
<https://github.com/ethereum/cpp-ethereum/wiki/LLL-PoC-6/04fae9e627ac84d771faddcf60098ad09230ab58#literals--code>`_
``(lit POS INT1 INT2 ...)`` functionality was changed in `PR #1329
<https://github.com/ethereum/solidity/pull/1329/files>`_. I'm not sure of the
background to this as it does look potentially useful.]


Code - ``lll``
--------------




Variables - ``set``, ``get``, ``ref``
-------------------------------------

[Comments on memory layout]



Memory allocation - ``alloc``
-----------------------------



Assembler - ``asm``
-------------------



Code size - ``bytecodesize``
----------------------------



Built-in Macros
===============

[Reference the source code and the test code.]
