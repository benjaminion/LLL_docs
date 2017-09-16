****************
LLL Introduction
****************

.. note::
    This documentation is a work in progress, so please exercise appropriate
    caution.  It a personal effort and has no formal connection with the
    Ethereum Foundation.

.. note::
    Everything in these docs pertains to the `Solidity/LLL
    <https://github.com/ethereum/solidity/>`_ implementation, specifically the
    *develop* branch.


Background
========== 

LLL is one of the original Ethereum smart contract programming languages and
provides a different perspective and programming discipline when compared to
the ubiquitous Solidity language.

According to the Ethereum `Homestead Documentation
<http://www.ethdocs.org/en/latest/contracts-and-transactions/contracts.html#id4>`_,

    Lisp Like Language (LLL) is a low level language similar to Assembly. It is
    meant to be very simple and minimalistic; essentially just a tiny wrapper
    over coding in EVM directly.

In particular, LLL doesn't hide from you the highly resource-constrained nature
of the EVM and enables efficient use of those limited resources.  LLL
facilitates the creation of very clean EVM code whilst removing the worst of
the pain of coding for the EVM directly: namely stack management and jump
management.

These pages aim to provide a reference resource for LLL contract development.

While these pages are based on the `Solidity/LLL
<https://github.com/ethereum/solidity/>`_ implementation, there is also a
`Viper/LLL
<https://github.com/ethereum/viper/blob/master/viper/compile_lll.py>`_
implementation. Some of the info here may carry across; some may not.


Resources
=========

The list of LLL-related resources currently available is quite short.


Authoritative Resources
-----------------------

The sole authoritative resource on LLL is the `compiler source code
<https://github.com/ethereum/solidity/tree/develop/liblll>`_ itself.

(While this documentation aims to be accurate, there will certainly be
errors and omissions.)


Tutorials
---------

Daniel Ellison has put together a number of tutorial articles and screencasts
on getting started with LLL.

 * A seven part series of articles entitled `The Resurrection of LLL
   <http://blog.syrinx.net/the-resurrection-of-lll-part-1/>`_.

 * An ongoing series of articles on the `Consensys media pages
   <https://media.consensys.net/@zigguratt>`_ with links to screencasts for
   some of them.


Original Resources
------------------

The `original LLL documentation
<https://github.com/ethereum/cpp-ethereum/wiki/LLL-PoC-6/04fae9e627ac84d771faddcf60098ad09230ab58>`_
is still available on GitHub and remains the starting point for this
documentation set. However, that documentation was last updated in 2014,
and significant things have changed since then.


Example Code
------------

 * The deployed Ethereum Name Service Registry was `written in LLL
   <https://github.com/ethereum/ens/blob/master/contracts/ENS.lll>`_.

 * There is also a sample `ENS Resolver in LLL
   <https://github.com/ethereum/ens/blob/master/contracts/PublicResolver.lll>`_.

 * An ERC20 token `implementation in LLL
   <https://github.com/benjaminion/LLL_erc20>`_.
   
 * The compiler `built-in macros
   <https://github.com/ethereum/solidity/blob/develop/liblll/CompilerState.cpp>`_
   and `end-to-end test suite
   <https://github.com/ethereum/solidity/blob/develop/test/liblll/EndToEndTest.cpp>`_
   are also useful references.

 * Once again, Daniel Ellison has some `code examples
   <https://github.com/zigguratt>`_ and demonstrations of useful techniques.

**Warning** some of the following examples may use features that have changed
or been removed. Some examples may no longer compile.

 * The original `LLL Examples for PoC 5
   <https://github.com/ethereum/cpp-ethereum/wiki/LLL-Examples-for-PoC-5/04fae9e627ac84d771faddcf60098ad09230ab58>`_.

 * A `GavCoin <https://github.com/ethereum/dapp-bin/blob/master/coin/coin.lll>`_
   contract.
