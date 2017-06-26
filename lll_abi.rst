.. index:: ! abi

***************
LLL and the ABI
***************

[Insert rationale for following the Solidity ABI standard]

[Insert cross-ref to ABI doc]


.. index:: ! abi;techniques

Techniques
==========

This section provides some patterns for handling common ABI implementations
in LLL.

.. index:: ! abi;techniques;types

Types
-----

[Almost all types are fundamentally bytes32.]


.. index:: ! abi;techniques;strings

Strings
-------

[Strings are trickier...]



.. index:: ! abi;techniques;functions

Functions
---------

[Explanation of function selector paradigm.]

[Hints on generating the function signature.]


.. index:: ! abi;techniques;events

Events
------

[How to use LOGn to generate events.]

[Explanation of the event signature and how to generate.]


.. index:: ! abi;generating

Generating the ABI
==================

[Basically, create Solidity signatures for everything and feed them into ``solc
--abi`` or use the online solidity compiler.]

[Don't forget the constructor ABI.]

[NB. Identifying "constant" functions.]
