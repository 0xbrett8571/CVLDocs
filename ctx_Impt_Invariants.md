# Invariants — Certora Prover Documentation 0.0 documentation
Invariants describe a property of the state of a contract that is always expected to hold.

Caution

Certain features of invariants are unsound: the invariant can be verified by the Prover, but it may still be possible for the contract to violate it. The possible sources of unsoundness are Preserved blocks, Filters, and Invariants that revert. Invariant proofs are also unsound if some of the methods are filtered out using the method or parametric\_contracts flags. See the linked sections for details.

Contents

*   Invariants
    
    *   Syntax
        
    *   Overview
        
    *   Invariants that revert
        
    *   Preserved blocks
        
        *   Contract and method-specific preserved blocks
            
        *   Generic preserved blocks
            
        *   Binding the environment
            
    *   Filters
        
    *   Induction Step for Transient Storage
        
    *   Writing an invariant as a rule
        
    *   Invariants and induction
        

Syntax

-------------------------------------------------

The syntax for invariants is given by the following EBNF grammar:

```
invariant ::= [ "weak" | "strong" ] "invariant" id
              [ "(" params ")" ]
              expression
              [ "filtered" "{" id "->" expression "}" ]
              [ "{" { preserved_block } "}" ]

preserved_block ::= "preserved"
                    [ method_signature ]
                    [ "with" "(" params ")" ]
                    block

method_signature ::= [ contract_name "." ] id  "(" [ evm_type [ id ] { "," evm_type [ id ] } ] ")"
                     | "fallback" "(" ")"

contract_name ::= id
                | "_"

```


See Basic Syntax for the `id` production, Expressions for the `expression` production, and Statements for the `block` production.

Overview

-----------------------------------------------------

In CVL, we distinguish between strong and weak invariants. A _weak_ invariant is a property that is expected to be true whenever a contract method is not currently executing. This kind of invariant is sometimes called a “representation invariant”. A _strong_ invariant is an invariant that also holds before and after execution of an unresolved call, i.e. a call that potentially calls to another contract which could modify the current contract’s state. Essentially, a strong invariants ensures to hold whenever control is yielded to an external function, providing enhanced security, especially for contracts without global locks.

Each invariant has a name, possibly followed by a set of parameters, followed by a boolean expression. We say the invariant _holds_ if the expression evaluates to true in every reachable state of the contract, and for all possible values of the parameters.

While verifying a weak invariant, the Prover checks two things. First, it checks that the invariant is established after calling any constructor. Second, it checks that the invariant holds after the execution of any methods, assuming that it held before the method was executed (if it does hold, we say the method _preserves_ the invariant). By default, an invariant will be checked for any `public` or `external` (non-`view`/`pure`) method of any contract in the scene - the set of methods an invariant will be checked for can be further configured by filters Filters and by parametric\_contracts. `View` and `pure` methods are excluded from invariant checking as by definition they cannot change the state of their contract.

A strong invariant performs the same checks as a weak invariant - i.e. it will be checked for the constructor and for any other method, it will be assumed before executing the method (pre-state) and asserted afterward execution of the method (post-state). In addition to these steps, a strong invariant also asserts and assumes the invariant _during_ method execution at locations that potentially break the invariant. The invariant can be violated if there is an unresolved external call that can modify the state of the current contract. To verify the strong invariant, for every unresolved external call `c` (a call that will force the prover to havoc storage), a strong invariant will insert the following steps:

1.  _Before_ the call `c`: Assert that the invariant holds - if the invariant does not hold due to some logic of the current method, this will yield a counter example that ends with the `assert` before `c`.
    
2.  _After_ the call `c`: Assume the invariant holds. The semantics is that the call did not break the invariant.
    
3.  In the case `c` is a `delegatecall`, after assuming the invariant in step 2, havoc the current’s contact storage and assert the invariant once more. This step simulates the scenario that a `delegatecall` modifies the current contract’s storage.
    

A full example for `weak` and `strong invariant` can be found in our Examples Repository.

If an invariant is proven, it is safe to assume that it holds in other rules and invariants. The requireInvariant command makes it easy to add this assumption to another rule, and is a quick way to rule out counterexamples that start in impossible states. See also Listing Safe Assumptions.

Note

Invariants are intended to describe the state of a contract at a particular point in time. Therefore, you should only use view functions inside of an invariant. Non-view functions are allowed, but the behavior is undefined.

Invariants that revert

---------------------------------------------------------------------------------

There is well-known unsoundness in the Prover’s handling of invariants that occurs if an invariant expression reverts in the “before” state but not in the “after” state. In this case, the assumption that the invariant holds before calling the contract method will revert, causing any counterexample to be discarded.

For example, consider the following contract:

```
contract Example {
    private uint[] a;

    public function add(uint i) external {
        a.push(i);
    }

    public function get(uint i) external returns(uint) {
        return a[i];
    }
}

```


This contract simply wraps an array of integers and allows you to add integers to the array. The following invariant states that all elements of the array are 0:

```
invariant all_elements_are_zero(uint i) get(i) == 0;

```


This property is clearly false; you can invalidate it by calling `add(2)`. Nevertheless, the invariant will pass. The reason is that before a call to `add` pushes a nonzero integer into `a[i]`, the length of `a` was `i-1`, so the call to `get(i)` will revert. Therefore, the Prover would discard the counterexample instead of reporting it. As above, an invariant stating that `supply() == token.totalSupply()` would be verified, but a method on `token` might change the total supply without updating the `SupplyTracker` contract. Since the Prover only checks the main contract’s methods for preservation, it will not report that the invariant can be falsified.

For this reason, invariants that depend on the environment or on the state of external contracts are a potential source of unsoundness, and should be used with care.

There is an additional source of unsoundness that occurs if the invariant expression reverts in the before state but not in the after state.

Preserved blocks

---------------------------------------------------------------------

Often, the proof that an invariant is preserved depends on another invariant, or on an external assumption about the system. These assumptions can be written in `preserved` blocks.

Caution

Adding `require` statements to preserved blocks can be a source of unsoundness, since the invariants are only guaranteed to hold if the requirements are true for every method invocation.

Recall that the Prover checks that a method preserves an invariant by first requiring the invariant (the prestate check), then executing the method, and then asserting the invariant (the poststate check). Preserved blocks are executed after the prestate check but before executing the method. They usually consist of `require` or `requireInvariant` statements, although other commands are also possible.

Preserved blocks are listed after the invariant expression (and after the filter block, if any), inside a set of curly braces (`{ ... }`). Each preserved block consists of the keyword `preserved` followed by an optional method signature, an optional `with` declaration, and finally the block of commands to execute.

### Contract and method-specific preserved blocks


The method signature of the preserved block may optionally contain a contract name followed by a `.` character followed by a contract method name.

*   In the case where the preserved block does not have a contract name but does have a method name (not the `fallback` case), the preserved block will apply only to methods that match in the main contract. For example, here the preserved block will apply only to the method `withdrawExcess(address)` that appears in the main contract:
    

```
invariant solvencyAsInv() asset.balanceOf() >= internalAccounting() {
  preserved withdrawExcess(address token)  {
      require token != asset; 
  }
}

```


*   If the method signature includes a specific contract name, then the Prover only applies the preserved block to the methods in the named contract. For example, here the preserved block only applies to the `asset` contract method `transfer(address,uint)`. The preserved block does not apply to the `transfer(address,uint)` method in any other contract.
    

```
invariant solvencyAsInv() asset.balanceOf() >= internalAccounting() {
  preserved asset.transfer(address x, uint y) with (env e) {
      require e.msg.sender != currentContract 
  }
}

```


*   If the contract name is the wildcard character `_`, the Prover applies the preserved block to instances of the method in all contracts in the scene. For example, this preserve block applies to all contracts containing a method matching the `transfer(address,uint)` method signature.
    

```
invariant solvencyAsInv() asset.balanceOf() >= internalAccounting() {
  preserved _.transfer(address x, uint y) with (env e) {
      require e.msg.sender != currentContract 
  }
}

```


If an invariant has multiple preserved blocks with the same method signature where one signature is more specific and the other is more general (as in the `_.method` case), then the more specific preserved block will apply.

If a preserved block specifies a method signature, the signature must either be `fallback()` or match one of the contract methods, and the preserved block only applies when checking preservation of that contract method. The `fallback()` preserved block applies only to the `fallback()` function that should be defined in the contract. The arguments of the method are in scope within the preserved block.

### Generic preserved blocks


If there is no method signature, the preserved block is a default block that is used for all methods that don’t have a specific preserved block, including the `fallback()` method. If an invariant has both a default preserved block and a specific preserved block for a method, the specific preserved block is used; the default preserved block will not be executed.

### Binding the environment


The `with` declaration is used to give a name to the environment used while invoking the method. It can be used to restrict the transactions that are considered. For example, the following preserved block rules out counterexamples where the `msg.sender` is the 0 address:

```
invariant zero_address_has_no_balance()
    balanceOf(0) == 0
    { preserved with (env e) { require e.msg.sender != 0; } }

```


The variables defined as parameters to the invariant are also available in preserved blocks, which allows restricting the arguments that are considered when checking that a method preserves an invariant. As always, you should use caution when adding additional `require` statements, as they can rule out important cases.

Caution

A common source of confusion is the difference between `env` parameters to an invariant and the `env` variables defined by the `with` declaration. Compare the following to the previous example:

```
invariant zero_address_has_no_balance_v2(env e)
    balanceOf(e, 0) == 0
    { preserved { require e.msg.sender != 0; } }

```


In this example, we require the `msg.sender` argument to `balanceOf` to be nonzero, but makes no restrictions on the environment for the call to the method we are checking for preservation.

To see why this is not the desired behavior, consider a `deposit` method that increases the message sender’s balance. When the `zero_address_has_no_balance_v2` invariant is checked on `deposit`, the Prover will effectively check the following (see Writing an invariant as a rule):

```
env e;
require balanceOf(e,0) == 0;

env calledEnv;
require e.msg.sender != 0; // from the `preserved` block
deposit(calledEnv, ...);

assert balanceOf(e,0) == 0;

```


Notice that the `calledEnv` is not restricted by the `preserved` block.

The Prover will report a violation with the `msg.sender` set to 0 in the call to `deposit` and set to a nonzero value in the calls to `balanceOf`. This counterexample is not ruled out by the `preserved` block because the `preserved` block only places restrictions on the environment passed to `balanceOf`.

In general, you should be cautious of invariants that depend on an environment.

Filters

----------------------------------------------------

For performance reasons, you may want to avoid checking that an invariant is preserved by a particular method or set of methods. Invariant filters provide a method for skipping verification on a method-by-method basis.

Caution

Filtering out methods while checking invariants is unsound. If you are filtering out a method because the invariant doesn’t pass, consider using a `preserved` block instead; this allows you to add assumptions in a fine-grained way (although `preserved` blocks can also be unsound).

To filter out methods from an invariant, add a `filtered` block after the expression defining the invariant. The body of the `filtered` block must contain a single filter of the form `var -> expr`, where `var` is a variable name, and `expr` is a boolean expression that may depend on `var`.

Before verifying that a method preserves an invariant, the `expr` is evaluated with `var` bound to a `method` object. This allows `expr` to refer to the checked method using `var`’s fields, such as `var.selector`, `var.contract`, and `var.isView`. See The method and calldataarg types for a list of the fields available on `method` objects.

If the expression evaluates to `false` with `var` replaced by a given method, the Prover will not check that the method preserves the invariant. For example, the following invariant will not be checked on the `deposit(uint)` method:

```
invariant balance_is_0(address a)
    balanceOf(a) == 0
    filtered {
        f -> f.selector != sig:deposit(uint).selector
    }

```


In this example, when the variable `f` is bound to `deposit(uint)`, the expression `f.selector != sig:deposit(uint).selector` evaluates to `false`, so the method will be skipped.

Note

If there is a preserved block for a method, the method will be verified even if the filter would normally exclude it.

Induction Step for Transient Storage

--------------------------------------------------------------------------------------------------------------

With the introduction of transient storage in Solidity (EIP-1153, Solidity contracts can now use a `tload` or `tstore` instruction to perform `load` and `store` on transient storage. The transient storage will be reset after a transaction has terminated.

The Prover will automatically detect if any contract in the scene uses `tload` and `tstore` and adds another induction step for transient storage. This induction step verifies the invariant is independent from the transient storage, i.e, it will assume the invariant in pre-state, perform a reset of the transient storage and `assert` the invariant in post-state.

Writing an invariant as a rule

--------------------------------------------------------------------------------------------------

Above we explained that verifying an invariant requires two checks: an initial-state check that the constructor establishes the invariant, and a preservation check that each method preserves the invariant.

Invariants are the only mechanism in CVL for specifying properties of constructors, but parametric rules can be used to write the preservation check in a different way. This is useful for two reasons: First, it can help you understand what the preservation check is doing. Second, it can help break down a complicated invariant by defining new intermediate variables.

The following example demonstrates all of the features of invariants:

```
invariant complex_example(env e1, uint arg)
    property_of(e1, arg)
    filtered {
        m -> m.selector != sig:ignored(uint, address).selector
    }
    {
        preserved with (env e2) {
            require e2.msg.sender != 0;
        }
        preserved special_method(address a) with (env e3) {
            require a != 0;
            require e3.block.timestamp > 0;
        }
    }

```


The preservation check for this invariant could be written as a parametric rule as follows:

```
rule complex_example_as_rule(env e1, uint arg, method f)
filtered {
    f -> f.selector != sig:ignored(uint, address).selector
}
{
    // pre-state check
    require property_of(e1, arg);

    if (f.selector == sig:special_method(address).selector) {
        // special_method preserved block
        address a;
        env e3;
        require a != 0;
        require e3.block.timestamp > 0;

        // method execution
        special_method(e3, a);
    } else {
        // general preserved block
        calldataarg args;
        env e2;
        require e2.msg.sender != 0;

        // method execution
        f(e2, args);
    }

    // post-state check
    assert property_of(e1, arg);
}

```


Invariants and induction

--------------------------------------------------------------------------------------

This section describes the logical justification for invariant checks. You do not need to understand this section to use the Prover correctly, but it helps explain the connection between the invariant checks and mathematical proofs for those who are familiar with writing proofs. This section also justifies the safety of arbitrary `requireInvariant` statements in `preserved` blocks.

This section assumes familiarity with basic proofs by induction. We use the symbols \\(∀\\), \\(⇒\\), and \\(∧\\) to stand for “for all”, “implies”, and “and” respectively.

Consider an invariant `i(x)` that is verified by the Prover. For the moment, let’s assume that `i(x)` has no `preserved` blocks. We will prove that for all reachable states of the contract, `i(x)` is `true`.

A state `s` is reachable if we can start with an newly created state (that is, where all storage variables are 0), apply any constructor, and then call any number of contract methods to produce `s`.

Let \\(P\_i(x,n)\\) be the statement “if we start from the newly created state, apply any constructor, and then call \\(n\\) contract methods, then the resulting state satisfies `i(x)`.” Our goal is then to prove \\(∀ n, ∀ x, P\_i(x,n)\\). We will prove this by induction on \\(n\\).

In the base case we want to show that for any \\(x\\), if we apply any constructor to the newly created contract, that the resulting state satisfies `i(x)`. This is exactly what the Prover verifies in the initial state check. In other words, the initial state check proves that \\(∀ x, P\_i(x,0)\\).

For the inductive step, we assume that any \\(n\\) contract calls produce a state that satisfies `i(x)`, and we want to show that a state produced after \\(n+1\\) calls also satisfies `i(x)`. This is exactly what the Prover verifies in the preservation check: that if the state before the last method call satisfies `i(x)` then after the last method call it still satisfies `i(x)`. In other words, the preservation check proves that \\(∀ n, ∀ x, P\_i(x,n) ⇒ P\_i(x, n+1)\\).

This completes the proof that together, the initial state check and the preservation check ensure that the invariant `i` holds on all reachable states.

Now, let us consider preserved blocks. Adding `require` statements to a `preserved` block for invariant `i` adds an additional assumption `Q` to the preservation check. Now, instead of

\\\[∀ n, ∀ x, P\_i(x,n) ⇒ P\_i(x, n+1),\\\]

the preservation check only proves

\\\[∀ n, ∀ x, P\_i(x,n) {\\bf ∧ Q} ⇒ P\_i(x, n+1).\\\]

The addition of the assumption \\(Q\\) invalidates the above proof if we don’t have reason to believe that \\(Q\\) actually holds, which is why we caution against adding `require` statements to `preserved` blocks.

However, it is important to note that adding `requireInvariant j(y)` to a `preserved` block is safe (assuming that `j` is verified), even if the `preserved` block for `j` requires the invariant `i`. To demonstrate this, we consider three examples.

For the first example, consider the spec

```
invariant i(uint x) ... { preserved { requireInvariant i(x); } }

```


Although this may seem like circular logic (we require `i` in the proof of `i`), it is not. The verification of the preservation check for `i` proves the statement

\\\[∀ n, ∀ x, P\_i(x, n) ∧ P\_i(x, n) ⇒ P\_i(x, n+1),\\\]

which is logically equivalent to the preservation check without the `preserved` block (since \\(P\_i(x,n) ∧ P\_i(x,n)\\) is equivalent to just \\(P\_i(x,n)\\)).

For the second example, consider the following spec:

```
invariant i(uint x) ...  { preserved { requireInvariant j(x); } }

invariant j(uint x) ...  { preserved { requireInvariant i(x); } }

```


Verifying these invariants gives us the preservation check for `i`:

\\\[∀ n, ∀ x, P\_i(x, n) ∧ P\_j(x, n) ⇒ P\_i(x, n + 1)\\\]

and for `j`:

\\\[∀ n, ∀ x, P\_j(x, n) ∧ P\_i(x, n) ⇒ P\_j(x, n + 1)\\\]

Putting these together allows us to conclude

\\\[∀ n, ∀ x, P\_i(x,n) ∧ P\_j(x,n) ⇒ P\_i(x,n+1) ∧ P\_j(x,n+1)\\\]

which is exactly what we need for an inductive proof of the statement \\(∀ n, ∀ x, P\_i(x,n) ∧ P\_j(x,n)\\). This statement then shows that both `i(x)` and `j(x)` are true in all reachable states.

For the third example, consider the following spec:

```
invariant i(uint x) ... { preserved { requireInvariant i(f(x)); } }

```


The preservation check now proves

\\\[∀ n, ∀ x, P\_i(x,n) ∧ P\_i(f(x), n) ⇒ P\_i(x, n+1).\\\]

Seeing that this gives us enough to write an inductive proof that \\(∀ n, ∀ x, P\_i(x,n)\\) takes a little more effort, but it only requires a simple trick. Let \\(Q(n)\\) be the statement \\(∀ x, P\_i(x,n)\\). We prove \\(∀ n, Q(n)\\) by induction.

The base case comes directly from the initial state check for `i`.

For the inductive step, choose an arbitrary \\(n\\) and assume \\(Q(n)\\). We want to show \\(Q(n+1)\\), i.e. that \\(∀ x, P\_i(x, n+1)\\). Fix an arbitrary \\(x\\). We can apply \\(Q(n)\\) to \\(x\\) to conclude \\(P\_i(x,n)\\). We can also apply \\(Q(n)\\) to \\(f(x)\\) to conclude \\(P\_i(f(x), n)\\). These facts together with the preservation check show \\(P\_i(x, n+1)\\). Since \\(x\\) was arbitrary, we can conclude \\(∀ x, P(x, n+1)\\), which is \\(Q(n+1)\\). This completes the inductive step, and thus the proof.

The techniques used in these three examples can be used to demonstrate that it is always logically sound to add a `requireInvariant` to a `preserved` block, even for complicated interdependent invariants (as long as the required invariants have been verified).
