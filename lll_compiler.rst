************
LLL Compiler
************

.. note::
    This documentation is a work in progress, so please exercise appropriate
    caution!

Installing the compiler
=======================

[Does this need to be covered? Link to DE's screencast? Move to resources
section?]

[Reference the Solidity "building from source" section]



Invoking the compiler
=====================

[And what to do with the output...]



Compiler Options
================

The ``lllc`` compiler options are reasonably self-explanatory and are as
follows.

When multiple options are used the following rules apply.

 * ``-h`` and ``-V`` take precendence over all others, and the *first* one of
   these listed is executed.

 * ``-a``, ``-b``, ``-t``, ``-x``, ``-d`` are mutually exclusive output
   formats. The *last* of these listed defines the output format created.

 * ``-o`` can be used with any of the output formats.



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

    

``-x,--hex``
------------

Parse, compile and assemble; output byte code in hex.

This is the default.

::

   > echo '(add 2 3)' | lllc
   6003600201

   > echo '(add 2 3)' | lllc --hex
   6003600201



``-a,--assembly``
-----------------

Only parse and compile; show assembly.

Outputs the intermediate assembly language which is shared with Solidity and
compiled into the final bytecode. If the ``-o`` flag is used as well then the
assembly language is displayed after optimisation.

::

   > lllc -a erc20.lll
     jumpi(tag_42, iszero(callvalue))
      0x0
     dup1
     revert
   tag_42:
     sstore(caller, 0x186a0)
     ...



``-t,--parse-tree``
-------------------

Only parse; show parse tree.

The "parse tree" is the clean version of the source code which is fed to the
LLL parser: all comments and linebreaks are removed, whitespace is normalised,
numbers are all converted to decimal and quoted strings standardised.

::

   > echo "(def 'foo (mload 0x0a)) ; define foo" | lllc -t
   ( def "foo" ( mload 10 ) )



``-d,--disassemble``
--------------------

The ``-d`` option is not documented in the ``--help`` output. It decompiles
hexadecimal EVM code into readable opcodes.

::

   > echo 602a600055 | lllc -d
   PUSH1 0x2A PUSH1 0x0 SSTORE



``-b,--binary``
---------------

Parse, compile and assemble; output byte code in binary.

I haven't found a use for this yet.



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


 
``-V,--version``
----------------

Show the version and exit. Note that the short form is a capital ``V``.

::

   > lllc -V
   LLLC, the Lovely Little Language Compiler 
   Version: 0.4.12-develop.2017.6.27+commit.b83f77e0.Linux.g++
