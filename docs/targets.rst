Target Specific
===============


Parity Substrate
________________

Solang works with Parity Substrate 2.0 or later. This target is the most mature and has received the most testing so far.

The Parity Substrate has the following differences to Ethereum Solidity:

- The address type is 32 bytes, not 20 bytes. This is what Substrate calls an "account"
- An address literal has to be specified using the ``address"5GBWmgdFAMqm8ZgAHGobqDqX6tjLxJhv53ygjNtaaAn3sjeZ"`` syntax
- ABI encoding and decoding is done using the `SCALE <https://substrate.dev/docs/en/knowledgebase/advanced/codec>`_ encoding
- Multiple constructors are allowed, and can be overloaded
- There is no ``ecrecover()`` builtin function, or any other function to recover or verify cryptographic signatures at runtime
- Only functions called via rpc may return values; when calling a function in a transaction, the return values cannot be accessed
- An `assert()`, `require()`, or `revert()` executes the wasm unreachable instruction. The reason code is lost

There is an solidity example which can be found in the
`examples <https://github.com/hyperledger-labs/solang/tree/main/examples>`_
directory. Write this to flipper.sol and run:

.. code-block:: bash

  solang --target substrate flipper.sol

Now you should have a file called ``flipper.contract``. The file contains both the ABI and contract wasm.
It can be used directly in the
`Polkadot UI <https://substrate.dev/substrate-contracts-workshop/#/0/deploy-contract>`_, as if the contract was written in ink!.


Solana
______

The Solana target requires `Solana <https://www.solana.com/>`_ v1.8.1. There a few missing features:

- Balance transfers are not implemented

This is how to build your Solidity for Solana:

.. code-block:: bash

  solang --target solana flipper.sol -v

This will produce two files called `flipper.abi` and `bundle.so`. The first is an ethereum style abi file and the latter being
the ELF BPF shared object which can be deployed on Solana. For each contract, an abi file will be created; a single `bundle.so`
is created which contains the code all the contracts provided on the command line.

The contract storage model in Solana is different from Ethereum; it is consists of a contigious piece of memory, which can be
accessed directly from the smart contract. This means that there are no `storage slots`, and that a `mapping` must be implemented
using a simple hashmap. The same hashmap is used for fixed-length arrays which are a larger than 1kb. So, if you declare an
contract storage array of ``int[10000]``, then this is implemented using a hashmap.

Solana has execution model which allows one program to interact with multiple accounts. Those accounts can
be used for different purposes. In Solang's case, each time the contract is executed, it needs two accounts.
One account is the program, which contains the compiled BPF program. The other account contains the contract storage
variables, and also the return variables for the last invocation.

The output of the compiler will tell you how large the second account needs to be. For the `flipper.sol` example,
the output contains *"info: contract flipper uses at least 17 bytes account data"*. This means the second account
should be 17 bytes plus space for the return data, and any dynamic storage. If the account is too small, the transaction
will fail with the error *account data too small for instruction*.

Before any function on a smart contract can be used, the constructor must be first be called. This ensures that
the constructor as declared in the solidity code is executed, and that the contract storage account is
correctly initialized. To call the constructor, abi encode (using ethereum abi encoding) the constructor
arguments, and pass in two accounts to the call, the 2nd being the contract storage account.

Once that is done, any function on the contract can be called. To do that, abi encode the function call,
pass this as input, and provide the two accounts on the call, plus any accounts that may be called.

The return data (i.e. the return values or the revert error) is provided in the
program log in base64. Any emitted events are written to the program log in base64
too.

There is `an example of this written in node
<https://github.com/hyperledger-labs/solang/tree/main/integration/solana>`_.

Hyperledger Burrow (ewasm)
__________________________

The ewasm specification is not finalized yet. There is no `create2` or `chainid` call, and the keccak256 precompile
contract has not been finalized yet.

In Burrow, Solang is used transparently by the ``burrow deploy`` tool if it is given the ``--wasm`` argument.
When building and deploying a Solidity contract, rather than running the ``solc`` compiler, it will run
the ``solang`` compiler and deploy it as a wasm contract.

This is documented in the `burrow documentation <https://hyperledger.github.io/burrow/#/reference/wasm>`_.

ewasm has been tested with `Hyperledger Burrow <https://github.com/hyperledger/burrow>`_.
Please use the latest master version of burrow, as ewasm support is still maturing in Burrow.

Some language features have not been fully implemented yet on ewasm:

- Contract storage variables types ``string``, ``bytes`` and function types are not implemented
