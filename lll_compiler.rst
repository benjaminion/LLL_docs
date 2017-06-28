.. index:: ! compiler

************
LLL Compiler
************

.. index:: ! compiler;installing

Installing the compiler
=======================

[Does this need to be covered? Link to DE's screencast? Move to resources
section?]


.. index:: ! compiler;invoking

Invoking the compiler
=====================


.. index:: ! compiler;options

Compiler Options
================

The ``lllc`` compiler options are reasonably self-explanatory and are as
follows.

[Which options can be conbined, and which are exclusive?]


.. index:: ! compiler;options;help

``-h,--help``
-------------

Displays the following compiler options.

::

    -b,--binary  Parse, compile and assemble; output byte code in binary.
    -x,--hex  Parse, compile and assemble; output byte code in hex.
    -a,--assembly  Only parse and compile; show assembly.
    -t,--parse-tree  Only parse; show parse tree.
    -o,--optimise  Turn on/off the optimiser; off by default.
    -h,--help  Show this help message and exit.
    -V,--version  Show the version and exit.

    
.. index:: ! compiler;options;hex

``-x,--hex``
------------

Parse, compile and assemble; output byte code in hex.

This is the default.

::

   > echo '(add 2 3)' | lllc
   6003600201

   > echo '(add 2 3)' | lllc --hex
   6003600201


.. index:: ! compiler;options;binary

``-b,--binary``
---------------

Parse, compile and assemble; output byte code in binary.


.. index:: ! compiler;options;assembly

``-a,--assembly``
-----------------

Only parse and compile; show assembly.


.. index:: ! compiler;options;parse

``-t,--parse-tree``
-------------------

Only parse; show parse tree.

The "parse tree" is the clean version of the source code which is fed to the
LLL parser: all comments and linebreaks are removed, whitespace is normalised,
numbers are all converted to decimal and quoted strings standardised.

::

   > echo "(def 'foo (mload 0x0a)) ; define foo" | lllc -t
   ( def "foo" ( mload 10 ) )


.. index:: ! compiler;options;optimise

``-o,--optimise``
-----------------

Turn on/off the optimiser; off by default.

The optimiser passes the assembly output through Solidity's optimiser. The main
useful thing the optimiser can do is the replacement of constant expressions,
but it doesn't always manage to spot all opportunities for this.

::

   > echo '(add 1 (mul 2 (add 3 4)))' | lllc
   6004600301600202600101
   
   > echo '(add 1 (mul 2 (add 3 4)))' | lllc -o
   600f


.. index:: ! compiler;options;version
   
``-V,--version``
----------------

Show the version and exit. Note that the short form is a capital ``V``.

::

   > lllc -V
   LLLC, the Lovely Little Language Compiler 
   Version: 0.4.12-develop.2017.6.27+commit.b83f77e0.Linux.g++
