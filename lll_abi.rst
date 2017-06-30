***************
LLL and the ABI
***************

[Insert rationale for following the Solidity ABI standard]

[Insert cross-ref to ABI doc]


Techniques
==========

This section provides some patterns for handling common ABI implementations
in LLL.



Types
-----

[Almost all types are fundamentally bytes32.]



Strings
-------

[Strings are trickier...]



Functions
---------

[Explanation of function selector paradigm.]

[Hints on generating the function signature.]



Events
------

[How to use LOGn to generate events.]

[Explanation of the event signature and how to generate.]



Generating the ABI
==================

[Basically, create Solidity signatures for everything and feed them into ``solc
--abi`` or use the online solidity compiler.]

[Don't forget the constructor ABI.]

[NB. Identifying "constant" functions.]
