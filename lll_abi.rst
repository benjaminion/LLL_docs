***************
LLL and the ABI
***************

.. note::
    This documentation is a work in progress, so please exercise appropriate
    caution.  It a personal effort and has no formal connection with the
    Ethereum Foundation or anyone else.

In order for smart contracts to interoperate with the rest of the Ethereum
ecosystem, an `Application Binary Interface
<https://github.com/ethereum/wiki/wiki/Ethereum-Contract-ABI>`_ (ABI) has been
defined.

The ABI defines the way that data passed to and from contracts is represented
in binary form: the sizes, structures and layouts that can be used.

LLL programmers are under no obligation at all to implement the ABI, but if you
want any third-party to be able to read from or write to your contract then it
is pretty much necessary to do so.  And many tools within the Ethereum
ecosystem expect to find the ABI implemented.

What follows is an introduction to interacting with the ABI in LLL. It doesn't
aim to be at all comprehensive and the `ABI Specification
<https://github.com/ethereum/wiki/wiki/Ethereum-Contract-ABI>`_ remains the
authoritative source of information.


ABI Data format overview
========================

Data within the ABI is passed exclusively via blocks of 32-byte words.

There are two types of data defined in the ABI and handled differently:

  1. *Elementary types* are those that are represented entirely within a
     32-byte word, including ``bool``, ``uint8``, ``uint256``, ``address``,
     ``bytes32``.

  2. *Dynamic types* are those that have variable size and may require multiple
     words to specify, such as ``string``, ``bytes``, or arrays such as
     ``uint256[]``, ``bool[]``.  These are represented by a 32-byte pointer to
     where the actual data is stored along with its size.

In addition, only when calling a function within a contract, the four-byte
"signature" of that function is prepended to the call data.

A complete set of call data to a contract has a structure like this::

  // Function selector
  [4 bytes: function selector]

  // Argument list
  [32 bytes: ARG1]
  [32 bytes: ARG2]
  ...
  [32 bytes: ARGn]

  // Dynamic data
  [32 bytes: length of ARGi data, Ni]
  [Ni or 32*Ni bytes: actual data]
  [32 bytes: length of ARGj data, Nj]
  [Nj or 32*Nj bytes: actual data]
  ...

In the above, each ``ARGm`` in the argument list is either the actual data (for
an elementary type), or a pointer to the start of the data (for a dynamic type).

Encoding Example
----------------

Putting this all together, calling a function with signature
``foo(uint256,string,address)`` looks like this::

  0x00 [Function selector - 4 bytes 0xf2f69ca5]
  0x04 [ARG1, elementary type, uint256 - 32 bytes, right-aligned]
  0x24 [ARG2, dynamic type, pointer to start of the string data, 0x60]
  0x44 [ARG3, elementary type, address - 32 bytes, right aligned]
  0x64 [Length of ARG2 string - 32 bytes, right aligned]
  0x84 [Start of string contents of ARG2. Continues for as many full words as the string needs]

So, ``foo(42, "Hello, world!", 0x0123456789012345678901234567890123456789)``
would look like this, with 164 bytes of data::

  ;; Truncated Keccak-256 hash of foo(uint256,string,address)
  0x00 f2f69ca5
  ;; uint256 42 in a 32-byte right-aligned word
  0x04 000000000000000000000000000000000000000000000000000000000000002a
  ;; pointer to the start of the string data, 0x60 in this case
  0x24 0000000000000000000000000000000000000000000000000000000000000060
  ;; address in a 32-byte right-aligned word
  0x44 0000000000000000000000000123456789012345678901234567890123456789
  ;; the length of the string: 13 or 0x0d
  0x64 000000000000000000000000000000000000000000000000000000000000000d
  ;; the string text, *left* aligned in a 32-byte word
  0x84 48656c6c6f2c20776f726c642100000000000000000000000000000000000000

**Note** that for the dynamic data in ARG2---the string---the pointer to the
start of the string data ignores the function selector.  I.e. the string data
is considered to start at 0x60, not at 0x64.
  
Finally, then, the call data to send with the transaction is::

  f2f69ca5000000000000000000000000000000000000000000000000000000000000002a00000000000000000000000000000000000000000000000000000000000000600000000000000000000000000123456789012345678901234567890123456789000000000000000000000000000000000000000000000000000000000000000d48656c6c6f2c20776f726c642100000000000000000000000000000000000000
  
All those zeros look wasteful, but the gas cost for a zero byte of input data
is only 4, whereas it is 68 for a non-zero input byte.


Data Types
==========

The following is a quick overview. Much more detailed descriptions and examples
are provided in the `ABI Specification
<https://github.com/ethereum/wiki/wiki/Ethereum-Contract-ABI>`_.


Elementary Types
----------------

Each of the `elementary types
<https://github.com/ethereum/wiki/wiki/Ethereum-Contract-ABI#types>`_ listed in
the ABI specification is represented in the call data as a 32 byte word.  Any
smaller types, such as booleans or addresses, must be padded on the left (high
order bytes) or right (low order bytes) with zero bytes: ``bytes`` types are
left-aligned; others are right aligned.

bool(true):
  ``0x0000000000000000000000000000000000000000000000000000000000000001``

uint8(42):
  ``0x000000000000000000000000000000000000000000000000000000000000002a``

uint32(42):
  ``0x000000000000000000000000000000000000000000000000000000000000002a``

int256(-1):
  ``0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff``

int8(-1):
  ``0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff``

address(0x314159265dD8dbb310642f98f50C066173C1259b)
  ``0x000000000000000000000000314159265dD8dbb310642f98f50C066173C1259b``

bytes32('0x1234'):
  ``0x1234000000000000000000000000000000000000000000000000000000000000``


Dynamic Types
-------------

Data that is variable in length and could exceed the bounds of a 32-byte word
is treated as a dynamic type.

Within the argument list part of the data, a dynamic type is represented by a
32 byte pointer to where the actual data is stored, which will be after the end
of the argment list. The pointer is the offset in bytes from the beginning of
the argument list to the word where the data's length is stored.

For most dynamic types, the length is stored in a 32 byte word as (effectively)
a uint256. Immediately after the length comes the data.

The data occupies as much space as required by the length, rounded up to a
multiple of 32 bytes/whole words.  For ``string`` and ``bytes`` types, the data
occupies one byte per unit of length specified.  Simple one-dimensional arrays
occupy one 32 byte word per element.

The ABI Specification has a `good example
<https://github.com/ethereum/wiki/wiki/Ethereum-Contract-ABI#use-of-dynamic-types>`_.


Passing data
============

There are essentially four situations where we are passing data around in this
format, if we include Event non-indexed data.  In each case the data are in
the same format as above, but are passed by different mechanisms.


Passing data to the Constructor
-------------------------------

Constructor data at contract deployment is simply appended to the contract
code as a block of 32-byte words with no function selector.

Accessing this constructor data is described in Daniel Ellison's `2nd article
<https://media.consensys.net/the-structure-of-an-lll-contract-part-2-bf57a5a91829>`_
on the structure of an LLL contract.

Essentially, the first word of the ABI data can be copied to memory at position
0x00 using::

  (codecopy 0x00 (bytecodesize) 32)

and you can continue parsing and processing the data from there.


Passing data to a function
--------------------------

When calling a function in a contract, all the necessary information is
contained in the "call data" that forms part of the transaction.  You can check
the length of the call data with ``(calldatasize)`` - this evaluates to the
number of bytes of call data available.  Reading beyond the end of the call
data is not an error, it just results in zero bytes being read.

Function call data at run time is prepended with the four-byte function
selector as described below, but otherwise follows the same format of 32-byte
blocks described above.

A convenient way to access the function selector is as follows. ::

  (seq
    (mstore 0x00 0)
    (calldatacopy 0x1c 0x00 4))

This first zeroes all the bytes in memory location ``0x00`` and then copies
the first four bytes of the call data to the last four bytes of the word at
memory ``0x00``. This can then easily be used in comparisons to find the right
function::

  (when (eq @0 0xf2f69ca5)
    (execute-function-foo))

See the [TODO] design patterns page for guidelines on implementing functions.


Returning data from a function
------------------------------

Returning data from a function follows exactly the same format of composing the
data (whether elementary or dynamic) into 32-byte blocks, but omits any
function selector.

Once the data has been marshalled into contiguous memory, it is returned as
follows::

  (return start length)

``start`` is the start location of the data in memory to be returned in bytes,
``length`` is the length in bytes of data to be returned. To be ABI compliant,
length must be a multiple of 32.


Events
------

Another way to expose internal data to the outside world is via what the ABI
(and Solidity) calls "Events".  These are just executions of the EVM ``LOGn``
opcodes.

EVM log entries and Events as specified by the ABI relate to each as follows.

EVM Log Entry
^^^^^^^^^^^^^

An EVM log entry comprises,

  * An arbitrary length ``data`` blob.

  * ``n`` topics, ``topic[0]`` to ``topic[n-1]``, each of which is a 32-byte
    word with the corresponding data type specified in the event signature.

In addition, the EVM provides the address of the contract emitting the event.

In terms of LLL, the following generates a three-topic log entry with 32 bytes
of ``data`` (read from memory starting at 0x00), ``topic[0]`` is the
``event-id`` as described below, and ``topic[1]`` and ``topic[2]`` are each an
address::

   (log3 0x00 32 event-id addr1 addr2)

  
ABI Event
^^^^^^^^^

The `ABI <https://github.com/ethereum/wiki/wiki/Ethereum-Contract-ABI#events>`_
specifies that the EVM Log entry maps to ABI Events as follows.

  * ``topic[0]`` is the Event signature. This is like a function signature, but
    is the full 32-byte Keccak-256 hash over the event name and arguments.

    For example, an ERC20 "Transfer" Event has the signature,
    ``keccak-256("Transfer(address,address,uint256)")``, which is
    ``0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef``

  * Further topics correspond to the first ``n-1`` arguments in the Event
    signature (the "indexed args").

  * The ``data`` blob corresponds to the final argument of the Event signature
    or the "non-indexed" arguments (simplifying here; see the ABI Specification
    for the details).

For example, to produce an ERC20 ``Transfer(address,address,uint256)`` Event,
we can use the following macro in LLL. ::

    (def 'event3 (id addr1 addr2 value)
      (seq
        (mstore 0x00 value)
        (log3 0x00 0x20 id addr1 addr2)))

``id`` is the 32-byte Event signature for ``Transfer`` as described above. This
is recorded as ``topic[0]`` in the event log.  ``addr1`` and ``addr2`` are two
Ethereum addresses and are ``topic[1]`` and ``topic[2]`` respectively in the
event log.  The amount of the transfer, ``value`` is a ``uint256`` and is first
written to memory and then recorded as the ``data`` element of the Event.


Techniques
==========

This section aims to provide some practical suggestions around working with LLL
and the ABI.


Functions
---------

When calling a function in a contract in accordance with the ABI then the first
four bytes of the call data are a truncated Keccak256 hash over the function
signature. The left-most, highest-order four bytes of the hash are used. We
will call this the function selector.

For example::

  name()
  → 0x06fdde0383f15d582d1a74511486c9ddf862a882fb7904b3d9fe9b8b8e58a796
  → 0x06fdde03
 
  transferFrom(address,address,uint256)
  → 0x23b872dd7302113369cda2901243429419bec145408fa8b352b3dd92b66c680b
  → 0x23b872dd
 

The function signature is the case-sensitive function name followed by a
parenthesised list of its argument types in order. Allowable types are listed
in the `ABI Specification
<https://github.com/ethereum/wiki/wiki/Ethereum-Contract-ABI>`_. Note that no argument names or spaces are included in the function signature string
that is hashed.

Once again, this has nothing to do with LLL *per se*, only with how external
entitities will interact with your contract written in LLL. The contract itself
only sees the four byte function selector hash at the front of a block of data
containing the function arguments (the "call data").

You can generate the function selector by pasting the function signature into a
`Keccak256 hash generator
<https://emn178.github.io/online-tools/keccak_256.html>`_ and taking the first
four bytes only.  Alternatively, from a web3.js 1.0.0 enabled console, you can
do as follows::

  > web3.utils.sha3("name()")
  '0x06fdde0383f15d582d1a74511486c9ddf862a882fb7904b3d9fe9b8b8e58a796'

  > web3.utils.sha3("transferFrom(address,address,uint256)")
  '0x23b872dd7302113369cda2901243429419bec145408fa8b352b3dd92b66c680b'

See also Remix in the next section.


Generating the JSON ABI
-----------------------

To share your contract's interface with others, a JSON format for the
contract's ABI is defined.

One way to generate the ABI for your contract relatively painlessly is to feed
the function definitions into the Solidity compiler with the ``--abi`` flag.
On the Linux command line, as follows::

  echo 'interface Foo{function totalSupply() constant returns (uint256); function transfer(address,uint256) returns (bool); event Transfer(address,address,uint256);}' | solc --abi

  Contract JSON ABI 
  [{"constant":true,"inputs":[],"name":"totalSupply","outputs":[{"name":"","type":"uint256"}],"payable":false,"type":"function"},{"constant":false,"inputs":[{"name":"","type":"address"},{"name":"","type":"uint256"}],"name":"transfer","outputs":[{"name":"","type":"bool"}],"payable":false,"type":"function"},{"anonymous":false,"inputs":[{"indexed":false,"name":"","type":"address"},{"indexed":false,"name":"","type":"address"},{"indexed":false,"name":"","type":"uint256"}],"name":"Transfer","type":"event"}]

The constructor ABI should also be included if relevant. Of course it's easier
if you read from and write to files in practice.

You can also use the online `Remix IDE
<https://ethereum.github.io/browser-solidity/>`_ to do this. Click on "Contract
details (bytecode, interface etc.)" to see the Interface ABI generated.  Remix
will also tell you the function selector hashes, so you can do it all in one
place.

Note that "constant" functions are those that don't change the blockchain
state: i.e. they don't transfer value, change anything in storage or emit any
events. These functions can be evaluated at zero gas cost on a local node
without broadcasting a transaction to the blockchain.


Using web3.js to call the ABI
-----------------------------

Once you have the JSON ABI descriptor for your contract then you can interact
with it using standard tools such as
`web3.js <https://www.npmjs.com/package/web3>`_
(`documentation <http://web3js.readthedocs.io/en/1.0/index.html>`_), which is
easier than messing around with the call data directly.

.. note::
    The following examples all use web3.js version 1.0.0-beta.

You can see the raw data that would get sent to your contract without even
deploying it. Web3.js will calculate it for you from the provided ABI and input
arguments. ::

  > var Web3 = require('web3');
  undefined
  > var web3 = new Web3();
  undefined
  > var myContract = new web3.eth.Contract([{inputs: [{type:'string'}], name: 'foo_string', outputs: [], type: 'function'}]);
  undefined
  > myContract.methods.foo_string("abc").encodeABI()
  '0x1099ee88000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000036162630000000000000000000000000000000000000000000000000000000000'

This is what will be sent to the function as the transaction's call data when
it is invoked on the blockchain.

  * The first four bytes of ``keccak-256("foo_string(string)")`` are
    ``0x1099ee88`` which we see at the beginning as the function selector.

  * Then we see the ``uint256`` quantity ``0x20``, which is the start of the
    string in the call data (ignoring the function selector).

  * Next is the ``uint256`` value ``3``, the length of the string.

  * Finally the three left-justified ASCII values ``0x61``, ``0x62``, ``0x63``,
    which are just the string "abc".


Worked example
--------------

The following LLL code simply returns its input data (excluding the function
selector) as an ABI compliant ``bytes32[]`` array. ::

  (returnlll
    (seq

      ;; The size in bytes of the calldata minus the function descriptor
      (def 'datalen (- (calldatasize) 4))

      ;; Point to the start of the dynamic data to return (a bytes32[] array)
      [0x00]:0x20

      ;; First word of the dynamic data is the length (in words for bytes32)
      [0x20]:(/ datalen 32)

      ;; Copy the call data to memory as the bytes32[] contents.
      (calldatacopy 0x40 0x04 datalen)

      ;; Return the whole structure we have built
      (return 0x00 (msize))))


.. note::
    The following breaks somewhere between web3.js 1.0.0-beta.11 and
    1.0.0-beta.15 - aaagh!

You can use the version 1.0.0 web3.js interface to interact with this contract
as follows.  When setting the ``from`` address below, use an address that you
have access to in your test environment.  I'm using ``testrpc -d`` which
generates a set of accounts automatically. ::

   var Web3 = require('web3');
   var web3 = new Web3('http://localhost:8545');

   // You can define different "inputs" types here to play with the ABI.
   // The LLL code doesn't care what they are.
   var myContract = new web3.eth.Contract([{inputs: [{type:'uint256'},{type:'string'},{type:'address'}], name: 'foo', outputs: [{type:'bytes32[]'}], type: 'function'}]);

   // Put your from address in the below.
   myContract.options.from='0x90f8bf6a479f320ead074411a4b0e7944ea8c9c1';

   // This is the compiled bytecode of the LLL contract.
   myContract.options.data = '0x601c80600c6000396000f300602060005260206004360304602052600436036004604037596000f3';

   myContract.deploy().send().then(function(x){x.methods.foo(42,"LLL rocks!","0x1234567890123456789012345678901234567890").call().then(console.log)});
   
The output should look like this::

   [ '0x000000000000000000000000000000000000000000000000000000000000002a',
     '0x0000000000000000000000000000000000000000000000000000000000000060',
     '0x0000000000000000000000001234567890123456789012345678901234567890',
     '0x000000000000000000000000000000000000000000000000000000000000000a',
     '0x4c4c4c20726f636b732100000000000000000000000000000000000000000000' ]

This is just the input call data passed back to us as a ``bytes32[]`` array,
as we intended.
