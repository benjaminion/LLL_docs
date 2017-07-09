******************
Language Reference
******************

.. note::
    This documentation is a work in progress, so please exercise appropriate
    caution.  It a personal effort and has no formal connection with the
    Ethereum Foundation.

.. note::
    Everything in these docs pertains to the `Solidity/LLL
    <https://github.com/ethereum/solidity/>`_ implementation, specifically the
    *develop* branch.


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
   ``0x2a``. Negative numbers cannot be directly input. (You can do ``(sub 0
   N)`` or use the ``~`` `bitwise not operator`_.)

 - A string (see `rules for strings`_ below).
   
 - An atom that has been defined previously via an argument-less ``def``
   expression.  E.g. with an existing definition ``(def 'm (memsize))``, then
   ``m`` can be used as an atom---an expression without parentheses---which
   will expand to ``(msize)``.

 - An evaluated expression which takes the form of a parenthesised list of an
   operator followed by zero or more expressions which are the operands, ``(OP
   EXPR1 EXPR2 ...)``, e.g. ``(add 1 2)``.  The operator ``OP`` is either a
   built-in `EVM opcode`_, or a `parser expression`_, or a macro defined in
   terms of these.

The above items are the basic syntax, and the last of these explains why
parentheses are everywhere in LLL. There are some variations to the syntax
introduced by the compact form described below.

Expressions evaluate either to "void", or to a single value that is left on top
of the EVM stack, and can be worked on by further expressions::

  (mstore 0x00 42) ;; Evaluates to "void"; leaves nothing on the stack
  (mload 0x00)     ;; Evaluates to 42.


.. _compact notation:

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
  EXPR1.  A colon, ``:``, may optionally be added after the closing bracket if 
  desired for readability.

``$ EXPR``
  ``(calldataload EXPR)`` - reads a word of call data starting from byte EXPR.


.. _rules for strings:

Strings
-------

A string in LLL is represented internally as a single 32 byte word, with the
string left-aligned to the high-order byte in the (big-endian) word. Any
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

The only exception is when the ``;`` character is within a double-quoted
string, in which case it is treated as part of the string and does not begin a
comment.

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

Almost all valid EVM opcodes are automatically valid LLL operations, so
expressions like the following will work as standard::

  (add 1 2)
  (signextend 1 0xff00)
  (return 0x00 0x20)
  (calldatasize)
  (call allgas to value 0 0 0 0)

In terms of the EVM, the operands can be considered as items on the stack (the
left-most being at the top of the stack) upon which the EVM operation works.

The opcodes implmented by the EVM are listed in `libevmasm/Instruction.cpp
<https://github.com/ethereum/solidity/blob/develop/libevmasm/Instruction.cpp>`_
and described in detail in the Ethereum `Yellow Paper
<http://gavwood.com/Paper.pdf>`_.

In general, LLL is intended to protect the programmer from having to deal with
tedious stack manipulation, and directly operating on the stack is discouraged.

Nonetheless, ``pop`` creates a valid expression---``(pop EXPR)``---and it can
be used to throw away the result of its argument expression. This could be used
with ``raw`` to fine-tune which of the sub-expressions provides the final value
of the ``raw`` expression, for example.

``pushN`` is not available as an LLL expression. Literals like ``42`` are
automatically pushed to the stack, so this instruction is not necessary. Other
stack manipulation operations such as ``swapN`` and ``dupN`` are available but
best avoided.

``jump``, ``jumpi`` and ``jumpdest`` are also best avoided as there is no easy
way to compute the address of a ``JUMPDEST``. Better to make use of the LLL
control structures provided. However, a "throw" as it was originally
implemented in Solidity (a jump to a non-valid address) can be performed with
``(jump 0x00)``, as long as you can be sure that 0x00 is not a valid
``JUMPDEST``.  Nonetheless, the currently recommended way to do this is with
``(panic)``; Solidity has now deprecated ``throw``.


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

It is possible for macros to shadow built-in operators, EVM operators and
previously defined macros.  For example, the following works as a definition of
a unary negation operator::

  (def '- (n) (- 0 n))

After this, ``(- 42)`` evaluates to an int256 -42 rather than +42 as it
normally would.

This feature ought, perhaps, to be used sparingly, if at all.

Macro scope
^^^^^^^^^^^

From Gav Wood's `original documentation
<https://github.com/ethereum/cpp-ethereum/wiki/LLL-PoC-6/04fae9e627ac84d771faddcf60098ad09230ab58>`_,
the following applies to macro scoping. If anyone can make sense of this (or
the `source code
<https://github.com/ethereum/solidity/blob/develop/liblll/CodeFragment.cpp>`_)
so as to explain it more simply (i.e. so I can understand it), I would be most
grateful.

    Environmental definitions at the time of the macro's definition are
    recorded alongside the macro. When a definition must be resolved,
    definitions made within the present macro are treated preferentially, then
    arguments to the macro, then definitions & arguments made in the scope of
    which the macro was called (treated in the same order), then finally
    definitions & arguments stemming from the scope in which the macro was
    created (again, treated in the same order).

Whatever else this means, it does mean that macros cannot be defined
recursively, so the following does not compile. (Actually, the compiler just
chases its tail trying to recursively expand the macro until it eventually
coredumps.) ::

  (seq
    (def 'fac (n) (when (> n 1) (* n (fac (- n 1)))))
    (fac 5))
  
This is probably just as well, as the resulting code could be unexpected. It is
important to remember that *macros are not functions*. Macros get fully
expanded in place at each invocation. If you have 10 invocations in different
places, the same code will be duplicated ten times.


Macro example
^^^^^^^^^^^^^

Here's a simple four-argument macro for raising ERC20 "Transfer" and "Approval"
events::

  (def 'event3 (id addr1 addr2 value)
    (seq
      (mstore 0x00 value)
      (log3 0x00 0x20 id addr1 addr2)))

We can use plain macros to store the ABI event ID constants for convenience::

  ;; Event IDs
  (def 'transfer-event-id
    (sha3 0x00 (lit 0x00 "Transfer(address,address,uint256)")))
  
  (def 'approval-event-id
    (sha3 0x00 (lit 0x00 "Approval(address,address,uint256)")))

Now it's easy to raise an event::

  (event3 transfer-event-id (caller) to value)))


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

Filepaths may be absolute or relative to the current directory. A filename on
its own is looked for in the current directory.


Control structures
------------------


``seq``
^^^^^^^

``(seq EXPR1 EXPR2 ...)`` evaluates all following expressions in order.  It
evaluates to the result of the final expression given.


.. _raw:

``raw``
^^^^^^^

``(raw EXPR1 EXPR2 ...)`` evaluates all following expressions in order.  It
evaluates to the result of the first non-void expression (i.e. the first
expression that leaves anything on the stack - this can be manipulated with
``pop``), or void if there is none.

For example, ``(raw (pop 1) 2 (pop 3))`` evaluates to 2.

We can use ``raw`` to avoid assigning a temporary variable when implementing
Euclid's GCD algorithm::

  ;; Evaluates to GCD(a,b)
  (seq
    (set 'a 1071)
    (set 'b 462)
    (while @b
      [a]:(raw @b [b]:(mod @a @b)))
    @a)

Normally the ``while`` body would need explicit temporary storage: ``{[0x00]:@b
[b]:(mod @a @b) [a]:@0x00})``. ``raw``'s properties allow us to avoid this, as
above. It saves 36 gas in this example! (Much more with bigger problems.)

    
``if``
^^^^^^

This is an "if-then-else" construction.

In ``(if PRED Y N)``: when the predicate ``PRED`` evaluates to non-zero, ``Y``
is evaluated; when ``PRED`` evaluates to zero, ``N`` is evaluated.

The following calculates the absolute value of signed 256 bit input::

  (if (S< (calldataload 0x04) 0)
    (- 0 (calldataload 0x04))
    (calldataload 0x04))


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
    @0x00)

``(until PRED BODY)`` is the same as ``while`` except that it evaluates
``BODY`` when ``PRED`` is zero until and continues until it becomes non-zero.

Evaluates to the number of leading zero bytes in the call data (up to 32 max)::

  (seq
    [0x20]:(calldataload 0x04)
    (until
      (or (= @0x00 32) (byte @0x00 @0x20))
      [0x00]:(+ 1 @0x00))
    @0x00)


``for``
^^^^^^^

``(for INIT PRED POST BODY)`` evaluates ``INIT`` once (ignoring any result),
then evaluates ``BODY`` and ``POST`` (discarding the result of both) as long as
``PRED`` is true.

The following code computes factorials: 10! = 3628800 = 0x375f00 in this
case. ::
   
    (seq
      (for
        (seq (set 'i 1) (set 'j 1))       ; INIT
        (<= (get 'i) 10)                  ; PRED
        (mstore i (+ (get 'i) 1))         ; POST
        (mstore j (* (get 'j) (get 'i)))) ; BODY
      (get 'j))

This is one of the rare occasions where I think the compact notation is
actually an improvement. The following compiles to the same bytecode. ::

    (seq
      (for
        { (set 'i 1) (set 'j 1) } ; INIT
        (<= @i 10)                ; PRED
        [i]:(+ @i 1)              ; POST
        [j]:(* @j @i))            ; BODY
      @j)

      
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


Variables
---------

LLL has an analogue of variables. It is relatively cheap to write to and read
from memory, so it can be efficient to store intermediate quantities
temporarily in memory. Since LLL doesn't provide direct access to the EVM stack
this is a practical alternative.

Variables provide a convenient way to automatically assign names to memory
locations. This automatic approach may or may not be desirable, depending on
how much control you wish to have over memory allocation. In any case, it's
important to know how and where the variable storage is assigned so that it
does not conflict with memory you may assign by other means.

For each variable created using a ``set`` or ``with`` expression, 32 bytes of
memory are assigned, starting from memory location 0x80 = 128. So, for example,
in ``{(set 'x 1) (set 'y 2) (set 'z 3)}``, ``x`` is at 0x80, ``y`` is at
0xa0 and ``z`` is at 0xc0.  Note that when a variable is ``unset``, or
goes out of the ``with`` scope, the memory space is *not reclaimed or
reassigned*.  Thus, the following will use sixty-four bytes of memory: ``{(set
'foo 1) (unset 'foo) (set 'foo 2)}``.


Assigning - ``set``
^^^^^^^^^^^^^^^^^^^

A variable is created with ``(set NAME EXPR)``, where NAME is any valid string
but with no restriction on length. So all of the following are valid, although
not all may be wise choices... ::
 
  (set 'x 42)
  (set 'foo 42)
  (set '41 42)
  (set 'abcdefghijklmnopqrstuvwxyz0123456789 42)
  (set '' 42)
  (set " " 42)
  (set "a b c" 42)


Accessing - ``get``
^^^^^^^^^^^^^^^^^^^

The value of a variable can be accessed using the ``(get NAME)`` expression,
where NAME is the same string used in the ``set`` expression::

  (get 'foo)


Referencing - ``ref``
^^^^^^^^^^^^^^^^^^^^^

The memory address where a variable is stored can be found using the ``(ref
NAME)`` expression.

Alternatively, using the variable name unquoted evaluates to its address in
memory: ``foo`` and ``(ref 'foo)`` are equivalent.  This can be useful when
using `compact notation`_: ``@foo`` evaluates to the value of the variable and
is equivalent to ``(get 'foo)``.  Note that using variable names unquoted like
this restricts the space of variable names that may be assigned (no leading
numerals, spaces, etc.).


Unassigning - ``unset``
^^^^^^^^^^^^^^^^^^^^^^^

[Coming in `PR #2520 <https://github.com/ethereum/solidity/pull/2520>`_]

Variable names can unassigned with ``(unset NAME)``. After this the name may no
longer be referenced.  If the same name is reassigned with ``set`` or ``with``
then a new memory location is assigned.


Temporary - ``with``
^^^^^^^^^^^^^^^^^^^^

[Coming in `PR #2520 <https://github.com/ethereum/solidity/pull/2520>`_]

A temporary variable may be assigned using the ``(with NAME EXPR1 EXPR2)``
expression.  ``EXPR2`` is then evaluated with variable ``NAME`` set to
``EXPR1``. ``with`` expressions may be nested for multiple local variables.

When the ``with`` expression ends, the variable name is unset, but the memory
is not reclaimed or re-used.

The following evaluates to 5::

  (with 'x 2
    (with 'y 3
      (+ @x @y)))


Memory allocation - ``alloc``
-----------------------------

[Exact behaviour still TBD - see `PR #2545
<https://github.com/ethereum/solidity/pull/2545>`_]

``(alloc SIZE)`` provides ``SIZE`` contiguous bytes of memory starting from the
current top of memory.  It returns the start of the memory space allocated.
This is memory that has not been previously written to (or read from), and is
all initialised to zero.

Since memory is allocated in multiples of 32 bytes, the actual amount allocated
is rounded up to the next 32 byte boundary::

  (alloc 0)  ;; Does nothing, returns (msize) unchanged
  (alloc 1)  ;; Allocates 32 bytes, returns the original (msize)
  (alloc 32) ;; Allocates 32 bytes, returns the original (msize)

It isn't necessary at all to use ``alloc`` to reserve memory; the LLL
programmer has complete control over how memory is laid out and used.  However,
``alloc`` could be useful for macros that need to find some unused space in
which to write return data, for example.

Note that the gas cost of memory is proportional to the number of bytes used up
to 724 bytes, and increases super-linearly above that.

Assembler - ``asm``
-------------------

Low-level assembler may be included in line with one caveat; it must have
transparent stack usage.  This basically means that ``JUMP`` or ``JUMPI`` are
best avoided; if used then ignoring their jump effects (and thus assuming the
jump doesn't happen and the PC just gets incremented) must have a valid final
result in terms of items deposited on the stack.  Usage is::

  (asm ATOM1 ATOM2 ...)

Where the ``ATOM``\ s may be either valid, non-``PUSH`` VM instructions or
literals (in which case they will result in an appropriate ``PUSH``
instruction). The EVM assembler language is defined in the `Yellow Paper
<http://gavwood.com/Paper.pdf>`_.

For example, ``(asm 69 42 ADD)`` evaluates to the value 111. Note any assembler
fragment that results in fewer than zero items being deposited on the stack or
greater than 1 will almost certainly not interoperate with the rest of the
language and thus cause compile errors.


Code - ``lll``
--------------

For handling cases where code needs to be compiled and passed around, there is
the ``lll`` expression::

  (lll EXPR POS MAXSIZE)
  (lll EXPR POS)

This places the EVM-code as compiled from ``EXPR`` into memory at position
``POS`` if and only if said EVM-code is at most ``MAXSIZE`` bytes, or if
``MAXSIZE`` is not provided. It evaluates to the number of bytes of memory
written, i.e. either 0 or the number of bytes of EVM-code; if provided, this is
always at most ``MAXSIZE``.

Contract creation code will typically look something like::

  {
    ;; Initialisation code goes here
    ;; This just records who the original creator is
    [[0]] (caller)
    
    ;; Return the contract code
    (return 0 (lll {
      ;; Contract code goes here
      ;; This just self-destructs if called by the original creator
      (when (= (caller) @@0) (selfdestruct (caller)))
    } 0))
  }

There is a built-in macro, ``returnlll`` described below, that simplifies this
pattern.


Code size - ``bytecodesize``
----------------------------

``(bytecodesize)`` evaluates to the total size of the compiled EVM bytecode in
bytes.

This is useful when creating a constructor for a contract: arguments passed at
contract creation are appended to the contract bytecode and can be accessed
through a combination of the ``codecopy`` EVM instruction and ``bytecodesize``.

The following will evermore return the initial argument that was appended to
the bytecode used for contract creation::

  (seq

    ;; constructor: store the passed-in data word which is appended to the bytecode
    (codecopy 0x00 (bytecodesize) 32)
    (sstore 0x00 @0x00)

    ;; contract body
    (returnlll
      (return (sload 0x00))))


Built-in Macros
===============

A number of LLL macros are pre-defined by the compiler for convenience. They
can be seen in the source file `liblll/CompilerState.cpp
<https://github.com/ethereum/solidity/blob/develop/liblll/CompilerState.cpp>`_.
There is test code for most of the macros in `test/liblll/EndToEndTests.cpp
<https://github.com/ethereum/solidity/blob/develop/test/liblll/EndToEndTest.cpp#L404>`_
which may be a useful reference.

Utility
-------

``(def 'panic () (asm INVALID))``
  Inserts an invalid instruction. It is conventional to use this to "throw" on
  an internal error.

``(def 'allgas (- (gas) 21))``
  A helper used by some of the message-call macros below.

Message-calls
-------------

``(def 'send (to value) (call allgas to value 0 0 0 0))``
  Transfer ``value`` Wei to the address ``to``. No call data or return
  data.  Evaluates to 1 on success of the transfer and 0 on failure.

``(def 'send (gaslimit to value) (call gaslimit to value 0 0 0 0))``
  As above, but provides the opportunity to specify the gas limit
  explicitly. This would be 21000 for a simple value transfer to an account.

``(def 'msg (to data) { [0]:data (msg allgas to 0 0 32) })``
  Message-call into an account with no transfer value and a single 32 byte word
  of ``data``.  Evaluates to a 32 byte word returned from the call, which also
  overwrites memory location 0x00.

``(def 'msg (to value data) { [0]:data (msg allgas to value 0 32) })``
  As above, but also transfers ``value`` Wei.

``(def 'msg (gaslimit to value data) { [0]:data (msg gaslimit to value 0 32) })``
  As above, but allows ``gaslimit`` to be set (the above calls use the
  ``allgas`` macro as the default.)

``(def 'msg (gaslimit to value data datasize) { (call gaslimit to value data datasize 0 32) @0 })``
  As above, but can handle arbitrary amounts of input data. ``data`` is now the
  starting memory location, and ``datasize`` its length in bytes.

``(def 'msg (gaslimit to value data datasize outsize) { [0]:0 [0]:(msize) (call gaslimit to value data datasize @0 outsize) @0 })``
  This version with six arguments allows all call parameters to be set, except
  that it will automatically allocate memory space for the arbitrary length
  returned data (``outsize`` bytes of it) at the current top of
  memory. Evaluates to the memory location of the start of the return data.


Contract creation
-----------------

``(def 'create (value code) { [0]:0 [0]:(msize) (create value @0 (lll code @0)) })``
  ``create`` with two arguments uses the built-in EVM CREATE opcode (which has
  three arguments) to create a new account with the associated ``code``
  (as delivered by ``returnlll``)). The value ``value`` is transferred to the
  new account. Returns 0 on failure, the new account's address on success.

``(def 'create (code) { [0]:0 [0]:(msize) (create 0 @0 (lll code @0)) })``
  As above, but without a value transfer. This could be defined more
  succinctly as ``(def 'create (code) (create 0 code))``.

Note that in the above macros, memory location 0x00 is first written to in
order to "reserve" it. This avoids an edge case where ``msize`` is initially
zero and data gets overwritten by the ``lll`` operation.

Keccak256/SHA3 functions
------------------------

``(def 'sha3 (loc len) (keccak256 loc len))``
  The EVM opcode for ``SHA3`` was changed to ``KECCAK256`` to reduce
  confusion. this macro ensures that legacy code continues to compile. It
  calculates the Keccak256 hash of the data in memory starting from ``loc`` and
  with length ``len``. The expression evaluates to the result.

``(def 'sha3 (val) { [0]:val (sha3 0 32) })``
  With one argument ``sha3`` evaluates to the Keccak256 hash of 32 byte input
  ``val``. Note that memory location 0x00 is overwritten with the input
  parameter.

``(def 'sha3pair (a b) { [0]:a [32]:b (sha3 0 64) })``
  Concatenates the two 32 byte arguments and returns the Keccak256 hash over
  the resulting 64 bytes. Overwrites memory locations 0x00-0x3f with the input
  parameters. The expression evaluates to the result.

``(def 'sha3trip (a b c) { [0]:a [32]:b [64]:c (sha3 0 96) })``
  Concatenates the three 32 byte arguments and returns the Keccak256 hash over
  the resulting 96 bytes. Overwrites memory locations 0x00-0x5f with the input
  parameters. The expression evaluates to the result.


Returns
-------

``(def 'return (val) { [0]:val (return 0 32) })``
  Halt execution and return the 32 byte/256 bit argument, ``val``, to the
  caller.

``(def 'returnlll (code) (return 0 (lll code 0)) )``
  This is a convenience macro for handling the byte code of the body of the
  contract. Typically an LLL contract will have a structure on these lines: ``{
  CONSTRUCTOR-EXPRESSIONS (returnlll {BODY-EXPRESSIONS})}``.

Storage handling
----------------

``(def 'makeperm (name pos) { (def name (sload pos)) (def name (v) (sstore pos v)) } )``
  Helper macro for ``perm``.

``(def 'permcount 0)``
  Helper macro for ``perm``.

``(def 'perm (name) { (makeperm name permcount) (def 'permcount (+ permcount 1))} )``
  This allows named references to storage locations. ``(perm 'foo)`` creates
  two macros: ``(foo EXPR)`` that will store the value of ``EXPR`` in permanent
  storage; and ``foo`` which will evaluate to the value stored. Storage
  locations are assigned consecutively, starting numbering from zero. (The
  starting point could be changed by redefining ``permcount`` at the top of
  your code if desired.)

Built-in contracts
------------------

``(def 'ecrecover (hash v r s) { [0] hash [32] v [64] r [96] s (msg allgas 1 0 0 128) })``
  Uses the built-in contract at address 0x01 to verify Ethereum signatures. If
  the signature is good then it will return the correct signing address. See
  the `test cases
  <https://github.com/ethereum/solidity/blob/develop/test/liblll/EndToEndTest.cpp#L601>`_
  for an example.  Overwrites memory locations 0x00 - 0x7f and evaluates to the
  resulting address, or zero on failure.

``(def 'sha256 (data datasize) (msg allgas 2 0 data datasize))``
  Uses the built-in contract at address 0x02 to calculate the SHA256 hash of
  arbitrary quantities of data stored beginning from memory location
  ``data``. ``datasize`` is in bytes.  Places the resulting hash at memory
  location 0x00.

``(def 'ripemd160 (data datasize) (msg allgas 3 0 data datasize))``
  Uses the built-in contract at address 0x03 to calculate the `RIPEMD-160
  <https://en.wikipedia.org/wiki/RIPEMD>`_ hash of arbitrary quantities of data
  stored beginning from memory location ``data``. ``datasize`` is in bytes.
  Places the resulting hash at memory location 0x00.

``(def 'sha256 (val) { [0]:val (sha256 0 32) })``
  Uses the built-in contract at address 0x02 to calculate the SHA256 hash of a
  32 byte word of data.  Places the resulting hash at memory location 0x00.
  
``(def 'ripemd160 (val) { [0]:val (ripemd160 0 32) })``
  Uses the built-in contract at address 0x03 to calculate the RIPEMD-160 hash
  of a 32 byte word of data.  Places the resulting hash at memory location
  0x00.


Ether sub-units
---------------

``(def 'wei 1)``
  The smallest subunit. One Ether is ``(* 1000000000000000000 wei)``.

``(def 'szabo 1000000000000)``
  The number of Wei in a Szabo. One Ether is ``(* 1000000 szabo)``.

``(def 'finney 1000000000000000)``
  The number of Wei in a Finney. One Ether is ``(* 1000 finney)``.

``(def 'ether 1000000000000000000)``
  The number of Wei in an Ether.

Shift instructions
------------------

These should be replaced by native instructions once supported by EVM

``(def 'shl (val shift) (mul val (exp 2 shift)))``
  Shift ``val`` left by ``shift`` bits, filling with zero bits. This is a
  relatively expensive operation. When the EVM finally has support for native
  ``SHL`` (`EIP #145
  <https://github.com/axic/EIPs/blob/4218665af978444201d685c8fef23a360500befd/EIPS/eip-145.md>`_)
  then this macro should be removed.

``(def 'shr (val shift) (div val (exp 2 shift)))``
  Shift ``val`` right by ``shift`` bits, filling with zero bits. This is a
  relatively expensive operation. When the EVM finally has support for native
  ``SHR`` (`EIP #145
  <https://github.com/axic/EIPs/blob/4218665af978444201d685c8fef23a360500befd/EIPS/eip-145.md>`_)
  then this macro should be removed. ``(shr (calldataload 0x00) 224)`` is a
  convenient way to extract the ABI function reference from the call data.
