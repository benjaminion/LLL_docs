******************
Language Reference
******************

LLL Syntax
==========


Expressions
-----------


Comments
--------


Allowed Characters
------------------

[NB string lengths; case-sensitivity]



Common conventions
------------------

[E.g. use of lower case. What else?]


EVM Opcodes
===========

[Link to the source file where these are listed.]


Parser expressions
==================


Arithmetic Operators
--------------------



Macro expansion - ``def``
-------------------------



Including files - ``include``
-----------------------------



Control structures
------------------


``seq``
^^^^^^^


``raw``
^^^^^^^


``if``
^^^^^^


``when``, ``unless``
^^^^^^^^^^^^^^^^^^^^


``while``, ``until``
^^^^^^^^^^^^^^^^^^^^


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

This is one of the few occasions where I think the compact notation is actually
an improvement. The following compiles to the same bytecode.
      
::

    (seq
      (for
        { (set 'i 1) (set 'j 1) } ; INIT
        (<= @i 10)                ; PRED
        [i]:(+ @i 1)              ; POST
        [j]:(* @j @i))            ; BODY
      (return j 0x20)))

      
      
``lll``
^^^^^^^

[Here or elsewhere?]


``&&``, ``||``
^^^^^^^^^^^^^^


Literals - ``lit``
------------------



Compact notation
----------------

[I've discovered an undocumented ``$`` short-form!!  ``$0x04`` expands to
``(calldataload 0x04)``.]



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
