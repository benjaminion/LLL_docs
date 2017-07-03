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
   ``0x2a``. Negative numbers cannot be directly represented (do ``(sub 0
   N)``).

 - A string (see below).
   
 - An atom that has been defined previously via an argument-less ``def``
   expression.  E.g. with an existing definition ``(def 'm (memsize))``, then
   ``m`` can be used as an atom - an expression without parentheses - which
   will expand to ``(msize)``.

 - An evaluated expression which takes the form of a parenthesised list of an
   operator followed by one or more expressions which are the operands, ``(OP
   EXPR1 EXPR2 ...)``, e.g. ``(add 1 2)``. In terms of the EVM, the operands
   can be considered as items on the stack (the left-most being at the top of
   the stack) upon which the operator works. This is indeed how the built-in
   EVM operators described below work.

The above items are the basic syntax, and the last of these explains why
parentheses are everywhere in LLL. There are some variations to the syntax
introduced by the compact form [todo: link] described below.

   
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


Common conventions
------------------

LLL code is generally written with everything in lower case, except in `asm`
expressions where the EVM opcodes tend to be uppercased. Internally all
expressions are converted to uppercase, except for literals/strings.

Literals, such as definition names created by ``def`` expressions, are case
sensitive, so ``(def 'x 42)`` and ``(def 'X 42)`` define two distinct macros.


EVM Opcodes
===========

[Link to the source file where these are listed.]


Parser expressions
==================


Arithmetic Operators
--------------------



Macro expansion - ``def``
-------------------------


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


Including files - ``include``
-----------------------------





Control structures
------------------


``seq``
^^^^^^^

``(seq EXPR1 EXPR2 ...)`` evaluates all following expressions in
order. Evaluates to the result of the final expression given.


``raw``
^^^^^^^

``(raw EXPR1 EXPR2 ...)`` evaluates all following expressions in
order. Evaluates to the result of the first non-void expression (i.e. the first
expression that leaves anything atop the stack), or void if there is none.


``if``
^^^^^^

This is an "if-then-else" construction.

In ``(if PRED Y N)``: when the predicate ``PRED`` evaluates to non-zero, ``Y``
is evaluated; when ``PRED`` evaluates to zero, ``N`` is evaluated.


``when``, ``unless``
^^^^^^^^^^^^^^^^^^^^

``(when PRED BODY)`` evaluates ``BODY``, discarding any result, if and only if
``PRED`` evaluates to a non-zero value.

A "not-payable" guard:

::

   (when (callvalue) revert)

``(unless PRED BODY)`` evaluates ``BODY``, discarding any result, if and only
if ``PRED`` evaluates to zero.

A guard for checking that exactly one argument has been passed in the call data:

::

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
count separately.)

::

  (seq
    [0x00]:0
    (while (sload @0x00) [0x00]:(+ 1 @0x00))
    (return 0x00 0x20))

``(until PRED BODY)`` is the same as ``while`` except that it evaluates
``BODY`` when ``PRED`` is zero until and continues until it becomes non-zero.

Return the number of leading zero bytes in the call data (up to 32 max):

::

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

The following code computes factorials: 10! = 3628800 = 0x375f00 in this case.

::
   
    (seq
      (for
        (seq (set 'i 1) (set 'j 1))      ; INIT
        (<= (get 'i) 10)                 ; PRED
        (mstore i (+ (get 'i) 1))        ; POST
        (mstore j (* (get 'j) (get 'i))) ; BODY
        )
      (return j 0x20))

This is one of the rare occasions where I think the compact notation is
actually an improvement. The following compiles to the same bytecode.
      
::

    (seq
      (for
        { (set 'i 1) (set 'j 1) } ; INIT
        (<= @i 10)                ; PRED
        [i]:(+ @i 1)              ; POST
        [j]:(* @j @i))            ; BODY
      (return j 0x20)))

      
``&&``, ``||``, ``!``
^^^^^^^^^^^^^^^^^^^^^

Logical "and", "or" and "not".

Both ``&&`` and ``||`` can take any non-zero number of arguments. They evaluate
the arguments from left to right and perform `short circuit evaluation
<https://en.wikipedia.org/wiki/Short-circuit_evaluation>`_ so that evaluation
of arguments stops as soon as the outcome is known. I.e. ``(&& EXPR1 EXPR2
...)`` will stop evaluating after encountering an expression that evaluates to
zero; ``(|| EXPR1 EXPR2 ...)`` will stop evaluating after encountering an
expression that evaluates to non-zero.

``!`` is a unary not operator, thus it takes one argument. ``(! EXPR)``
evaluates to zero when ``EXPR`` evaluates to non-zero, and to one when ``EXPR``
evaluates to zero. It is equivalent to ``(iszero EXPR)``.


Literals - ``lit``
------------------


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
