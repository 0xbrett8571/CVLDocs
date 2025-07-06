# Parametric rules — Certora Prover Documentation 0.0 documentation
General
-------------------------------------------

A parametric rule is a rule that uses a `method f` parameter. Such a rule will be tested against all possible methods `f`, including methods from other contracts in the scene. It is possible to limit the methods tested, see Parametric rules.

Parametric rules can be used to verify properties of the changes in storage values. The template for such checks is:

Parametric rule template

```
rule parametricExample(method f) {
    // Get storage values before
    uint before = ...;
    ...

    // Function call
    env e;
    calldataarg args;
    f(e, args)

    // Get storage values after
    uint after = ...;
    ...

    // Assert property of the change, e.g.:
    assert after - before == ...;
}

```


The main differences between parametric rules and invariants are:

1.  Invariants are also tested after the constructor.
    
2.  Invariants are used to assert properties of the storage (between function calls), while parametric rules are used to assert properties of _changes_ in the storage (caused by function calls).
    

ERC20 example
-------------------------------------------------------

Here is a parametric rule example from the spec file ERC20Full.spec – a spec for an ERC20 implementation. The parametric rule `onlyAllowedMethodsMayChangeBalance` asserts two things:

1.  A user’s balance can increase only by calls to `mint`, `transfer`, and `transferFrom`.
    
2.  A user’s balance can decrease only by calls to `burn`, `transfer`, and `transferFrom`.
    

It follows that all other functions do not change balances. The rule is shown below, with the lines for getting the storage value before and after the function call highlighted. The two helper functions used in the rule are explained below the rule.

```
rule onlyAllowedMethodsMayChangeBalance(env e){
    requireInvariant totalSupplyIsSumOfBalances();

    method f;
    calldataarg args;

    address holder;
    uint256 balanceBefore = balanceOf(holder);
    f(e, args);
    uint256 balanceAfter = balanceOf(holder);
    
    assert balanceAfter > balanceBefore => canIncreaseBalance(f);
    assert balanceAfter < balanceBefore => canDecreaseBalance(f);
}

```


*   The function `canIncreaseBalance(f)` returns true if `f` is one of the functions `mint`, `transfer`, or `transferFrom`.
    
*   Similarly, `canDecreaseBalance(f)` returns true if `f` is one of `burn`, `transfer`, or `transferFrom`.
    

```
definition canIncreaseBalance(method f) returns bool = 
	f.selector == sig:mint(address,uint256).selector || 
	f.selector == sig:transfer(address,uint256).selector ||
	f.selector == sig:transferFrom(address,address,uint256).selector;

definition canDecreaseBalance(method f) returns bool = 
	f.selector == sig:burn(address,uint256).selector || 
	f.selector == sig:transfer(address,uint256).selector ||
	f.selector == sig:transferFrom(address,address,uint256).selector;

```
