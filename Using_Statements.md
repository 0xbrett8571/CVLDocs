# Using Statements — Certora Prover Documentation 0.0 documentation

# [Using Statements](#id1)[](#using-statements "Link to this heading")

The `using` statement introduces a variable that can be used to call methods on contracts other than the main contract being verified.

## [Examples](#id2)[](#examples "Link to this heading")

*   [Accessing additional contracts from CVL](../user-guide/multicontract/index.html#using-example)
    
*   [An example for `using`]

```solidity
//// This spec extends `pool_havoc.spec` using the Certora Prover's support for linking
//// and accessing methods from other contracts.
////
//// You can verify this spec by running the following from the command line:
////
////      sh certora/scripts/verifyWithLinking.sh
////
//// See [the multicontract section of the user guide][guide] for a complete
//// discussion of this example.
////
//// [guide]: https://docs.certora.com/en/latest/docs/user-guide/multicontract/index.html
////

using Asset as underlying;

methods
{
    function balanceOf(address)                      external returns(uint256) envfree;
    function totalSupply()                           external returns(uint256) envfree;
    function transfer(address, uint256)              external returns(bool);
    function transferFrom(address, address, uint256) external returns(bool);

    function deposit(uint256)                        external returns(uint256);
    function withdraw(uint256)                       external returns(uint256);

    function flashLoan(address, uint256) external;

    function underlying.balanceOf(address)           external returns(uint256) envfree;
}

/// `deposit` must increase the pool's underlying asset balance
rule integrityOfDeposit {

    mathint balance_before = underlying.balanceOf(currentContract);

    env e; uint256 amount;
    require e.msg.sender != currentContract;

    deposit(e, amount);

    mathint balance_after = underlying.balanceOf(currentContract);

    assert balance_after == balance_before + amount,
        "deposit must increase the underlying balance of the pool";
}
```

## [Syntax](#id3)[](#syntax "Link to this heading")

The syntax for `using` statements is given by the following [EBNF grammar](overview.html#ebnf-syntax):

using ::= "using" id "as" id

See [Identifiers](basics.html#identifiers) for the `id` production.
