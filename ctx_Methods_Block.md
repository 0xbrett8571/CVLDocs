# The Methods Block — Certora Prover Documentation 0.0 documentation
The `methods` block contains additional information about contract methods. Although you can call contract functions from CVL even if they are not declared in the methods block, the methods block allows users to specify additional information about contract methods, and can help document the expected interface of the contract.

There are two kinds of declarations:

*   **Non-summary declarations** document the interface between the specification and the contracts used during verification (see envfree annotations). Non-summary declarations also support spec reuse by allowing specs written against a complete interface to be checked against a contract that only implements part of the interface (see optional annotations).
    
*   **Summary declarations** are used to replace calls to certain contract methods. Summaries allow the Prover to reason about external contracts whose code is unavailable. They can also be useful to simplify the code being verified to circumvent timeouts. See Summaries.
    

Caution

Summary declarations change the way that some function calls are interpreted, and are therefore unsound (with the exception of `HAVOC_ALL` summaries which are always sound, and `NONDET` summaries which are sound for `view` functions).

Contents

*   The Methods Block
    
    *   Syntax
        
    *   Methods entry patterns
        
        *   Exact entries
            
        *   Wildcard entries
            
        *   Catch-all entries
            
        *   Catch unresolved-calls entry
            
        *   Location annotations
            
        *   Visibility modifiers
            
    *   `envfree` annotations
        
    *   `optional` annotations
        
    *   `with(env e)` clauses
        
    *   Summaries
        
        *   Summary application
            
        *   Summary resolution
            
        *   Summary types
            
            *   View summaries: `ALWAYS`, `CONSTANT`, `PER_CALLEE_CONSTANT`, and `NONDET`
                
            *   Havoc summaries: `HAVOC_ALL` and `HAVOC_ECF`
                
            *   `DISPATCHER` summaries
                
            *   `AUTO` summaries
                
            *   `ASSERT_FALSE` summaries
                
            *   Expression summaries
                

Syntax

-------------------------------------------------

Changed in version 4.0: The syntax for methods block entries changed in CVL 2.

The syntax for the `methods` block is given by the following EBNF grammar:

```
methods          ::= "methods" "{" { method_spec } "}"

method_spec      ::= "function"
                     ( exact_pattern | wildcard_pattern | catch_all_pattern | catch_unresolved_calls_pattern )
                     [ "returns" "(" evm_types ")" ]
                     [ "envfree" |  "with" "(" "env" id ")" ]
                     [ "optional" ]
                     [ "=>" method_summary [ "" | "UNRESOLVED" | "ALL" | "DELETE" ] ]
                     ";"

exact_pattern    ::= [ id "." ] id "(" evm_params ")" visibility [ "returns" "(" evm_types ")" ]
wildcard_pattern ::= "_" "." id "(" evm_params ")" visibility
catch_all_pattern ::= id "." "_" "external"
catch_unresolved_calls_pattern ::= "_" "." "_" "external"

visibility ::= "internal" | "external"

evm_param ::= evm_type [ id ]

method_summary   ::= "ALWAYS" "(" value ")"
                   | "CONSTANT"
                   | "PER_CALLEE_CONSTANT"
                   | "NONDET"
                   | "HAVOC_ECF"
                   | "HAVOC_ALL"
                   | "DISPATCHER" [ "(" ( "true" | "false" ) ")" ]
                   | "AUTO"
                   | "ASSERT_FALSE"
                   | expr [ "expect" id ]
                   | "DISPATCH" [ "(optimistic=false)" ]  "[" dispatch_list_pattern [","] | empty "]" "default" method_summary
                   | "DISPATCH" [ "(optimistic=true)" ]  "[" dispatch_list_pattern [","] | empty "]"

dispatch_list_patterns ::= dispatch_list_patterns "," dispatch_pattern
                          | dispatch_pattern

dispatch_pattern ::= | "_" "." id "(" evm_params ")"
                     | id "." "_"
                     | id "." id "(" evm_params ")"

```


See Types for the `evm_type` production. See Basic Syntax for the `id` production. See Expressions for the `expr` production.

Methods entry patterns

----------------------------------------------------------------------------------

Each entry in the methods block contains a pattern that matches some set of contract functions.

*   Exact entries match a single method of a single contract.
    
*   Wildcard entries match a single method signature on all contracts.
    
*   Catch-all entries apply a single summary to all methods of a specific contract.
    
*   Catch unresolved-calls entry apply a summary to all calls that cannot be statically resolved in any contract.
    

### Exact entries


An exact methods block entry matches a single method of a single contract. If the contract name is omitted, the default is `currentContract`. For example,

```
methods {
    function C.f(uint x) external returns(uint);
}

```


will match the external function `f` of the contract `C`.

Exact methods block entries must include a return type; the Prover will check that the declared return type matches the return type of the contract function.

Exact entries may contain summaries, envfree annotations, optional annotations, and with(env e) clauses.

It is possible for an exact entry to overlap with another entry; see Summary resolution for information on how summaries are resolved.

### Wildcard entries


A wildcard entry matches any function in any contract with the indicated name, argument types, and visibility. For example,

```
methods {
    function _.f(uint x) external => NONDET;
}

```


will match any external function called `f(uint)` in any contract.

Wildcard entries must not declare a return type, since different matched methods may return different types.

Wildcard entries may not have envfree annotations or optional annotations; their only purpose is summarization. Therefore, wildcard entries must have a summary.

It is possible for a wildcard entry to overlap with another entry; see Summary resolution for information on how summaries are resolved.

### Catch-all entries


Sometimes the behavior of a contract in the scene is irrelevant to the properties being verified. For example, the exact behavior of an external library contract may be unimportant for a particular verification project.

So-called “catch-all” entries are useful in these situations. A catch-all entry is used to apply a single summary to all functions that are declared in a given contract. For example:

```
methods {
   function SomeLibrary._ external => NONDET;
}

```


Will apply the `NONDET` havoc summary in place of _every_ call to a function in the `SomeLibrary` contract. Note that there are no parameter types _or_ return types for this entry: it refers to all methods in a contract, and cannot be further refined with parameter type information. Catch-all summaries apply only to `external` methods, and therefore the `external` visibility modifier is required. Further, the only purpose of catch-all entries is to apply a summary to all external methods in a contract, so a summary is required. However, only havocing summaries are allowed for these entries. Finally, envfree annotations and optional annotations keywords are not allowed for catch-all entries.

It is possible for a catch-all summary to overlap with another entry; see Summary resolution for information on how summaries are resolved.

Note

Catch-all summaries are only applied when the Prover can definitively show that the target of a call resolves to the contract mentioned in the catch-all summary. For library contracts (a common use case for these catch-all summaries) the Prover is almost always able to resolve the target contract.

For example, if you have an entry `function Token._ external => NONDET;`, where the contract `Token` has a `burn()` method, the Prover will _not_ apply the `NONDET` summary for the call `t.burn()`, unless it can prove that `t` must hold the address of the `Token` contract. The “Rule Call Resolution” panel shown in the web report can indicate whether a summary was applied.

### Catch unresolved-calls entry


Example:

```
methods {
    // Applies to all unresolved calls called within `C.foo()`
    unresolved external in C.foo() => DISPATCH [
        D.baz()
    ] default HAVOC_ECF;

    // Applies to all unresolved calls in the scene (except ones specified by more refined catch-unresolved-calls entries)
    unresolved external in _._ => DISPATCH [
        C.foo(uint),
        _.bar(address), // Will resolve to all available functions with the signature "bar(address)", specifically Other.bar(address)
        C._ // Will resolve to all functions in C, specifically C.foo(uint) and C.baz(bool)
    ] default NONDET;

    // An optimistic dispatcher can be used to enforce resolving all unresolved calls to a specific method.
    // Be aware: In case the method C.foo(uint) doesn't exist or the sighash doesn't match, this create vacuity.
    unresolved external in _._ => DISPATCH(optimistic=true) [
        C.foo(uint)
    ];
}

```


Catch unresolved-calls entries are a special type of summary declaration that instructs the Prover to replace calls to unresolved external function calls with a specific kind of summary, dispatch list. By default, the Prover will use an AUTO summary for unresolved function calls, but that may produce spurious counter examples. Catch unresolved-calls entries let the user refine the summary used for unresolved function calls.

Note that the catch unresolved-calls entry _only applies_ in cases where the called function’s _sighash_ is unresolved. In the example below there is a function call `target.call(data)`. The sighash of the called function depends on the parameter `data` and cannot be known beforehand.

```
    function flashLoan(
        uint256 borrowAmount,
        address borrower,
        address target,
        bytes calldata data
    )
        external
    {
        uint256 balanceBefore = damnValuableToken.balanceOf(address(this));
        require(balanceBefore >= borrowAmount, "Not enough tokens in pool");
        
        damnValuableToken.transfer(borrower, borrowAmount);
        (bool success, ) = target.call(data);
        require(success, "External call failed");

        uint256 balanceAfter = damnValuableToken.balanceOf(address(this));
        require(balanceAfter >= balanceBefore, "Flash loan hasn't been paid back");
    }

```


One can specify the scope (`unresolved external in <scope>`) for which the unresolved summary will apply. The options are:

*   `Contract.functionSignature()` for summarizing unresolved calls within this function
    
*   `_.functionSignature()` for summarizing unresolved calls within this function in any contract
    
*   `Contract._` for summarizing unresolved calls in any function of the given contract
    
*   `_._` for summarizing all unresolved calls in the scene.
    

If multiple catch unresolved-calls entries exist, the order of precedence is the order of the above list, from top to bottom.

Note

If `C.foo` has a (resolved) external call to `D.bar`, and `D.bar` contains an unresolved call, a catch-unresolved-calls entry that applies to `C.foo` will _not_ be applied to this unresolved call - only an entry that matches `D.bar` will be used.

Catch unresolved-calls entries can only be summarized with a dispatch list summary (and a dispatch list summary is only applicable for a catch unresolved-calls entries).

As with `DISPATCH`, there are optimistic and pessimistic dispatch lists. This can be specified via `DISPATCHER(optimistic=<true|false>). When the` optimistic\` option is not specified in parentheses, the Prover will use a pessimistic dispatch list to ensure sound reasoning.

A dispatch list summary directs the Prover to consider each of the methods described in the list as possible candidates for this unresolved call. The Prover will choose dynamically, that is, for each potential run of the program, which of them to call. It is done accurately by matching the selector from the call’s arguments to that of the methods described in the dispatch list. When using a pessimistic dispatch list and no method from the list matches, it will use the `default` summary. When using the optimistic dispatch list, an `ASSUME FALSE;` is inlined by the Prover; see the example below. The dispatch list will contain a list of patterns and the default summary to use in case no function matched the selector. The possible patterns are:

1.  Exact function - a pattern specifying both a contract, and the function signature. Example: `C.foo(uint)`
    
2.  Wildcard contract - a pattern specifying the function signature to match this signature on all available contracts (including the primary contract). Example: `_.bar(address)`
    
3.  Wildcard function - a pattern specifying a contract, and matches all external functions in specified contract (This pattern will also include the contract’s fallback if it’s implemented). Example: `C._`
    

The example entry at the head of this section will specify three functions to route calls to:

1.  `C.foo(uint)`
    
2.  `Other.bar(address)`
    
3.  `C.baz(bool)`
    

Entry annotations (envfree annotations, optional annotations) and the `returns` clause are not allowed on an unresolved-calls entry. Also, the visibility is always external, and no policy should be specified.

For an unresolved function call being summarized with the dispatch list above, the Prover will replace the call with a dynamic resolution of the function call. That is something in the lines of:

```
function summarized(address a, bytes calldata data) external {
  if (uint32(data[0:4]) == 0x11111111 && address == address(c)) {
    // Function selector was equal to foo's
    // Call C.foo(...)
  } else if (uint32(data[0:4]) == 0x22222222 && address == address(o)) {
    // Function selector was equal to bar's
    // Call O.bar(...)
  } else if (uint32(data[0:4]) == 0x33333333 && address == address(c)) {
    // Function selector was equal to baz's
    // Call C.baz(...)
  } else {
    // In the case of the DISPATCHER(optimistic=false), the summary 
    // specified after the "default" is inlined here. This is typically a HAVOC_ALL,
    // HAVOC_ECF or a NONDET.
    // In the case of the DISPATCHER(optimistic=true) option, no default is used 
    // and the Prover inlines an ASSUME FALSE; at this location marking 
    // this branch as unreachable.
  }
}

```


The dispatch list summary will create a dynamic resolution process that determines the specific function to call at runtime based on the function signature and the target contract address. In the provided example, when an unresolved function call is encountered, the Prover dynamically resolves it by inspecting the function selector in the transaction data and the target contract address. By comparing the function selector against known signatures and verifying the contract address, the Prover identifies the appropriate function to call.

This dynamic resolution mechanism is crucial for refining specifications because it enables the Prover to accurately model the behavior of smart contracts, even when the exact function being called is not known statically. By replacing unresolved calls with dynamically resolved calls in the dispatch list summary, the specification becomes more precise, leading to more accurate verification results and improved assurance in the correctness of the smart contract.

### Location annotations


Methods block entries for internal functions must contain either `calldata`, `memory`, or `storage` annotations for all arguments with reference types (such as arrays).

Entries for external functions may have `storage` annotations for argument references (in Solidity, external library functions may have storage arguments). If a reference-type argument does not have a `storage` annotation, the entry will apply to a function that has either a `calldata` or a `memory` annotation on the argument.

### Visibility modifiers


Entries in the methods block must be marked either `internal` or `external`; the entry will only match a function with the indicated visibility.

If a function is declared `public` in Solidity, then the Solidity compiler creates an internal implementation method, and an external wrapper method that calls the internal implementation. An `internal` methods block entry will apply to the generated implementation method, while an `external` entry will apply to the generated external wrapper method.

This summarization behavior can be confusing, especially because functions called directly from CVL are not summarized.

Consider a public function `f`. Suppose we provide an `internal` summary for `f`:

*   Calls from CVL to `f` _will_ effectively be summarized, because CVL will call the external function, which will then call the internal implementation, and the internal implementation will be summarized.
    
*   Calls from another contract to `f` (or calls to `this.f` from `f`’s contract) _will_ effectively be summarized, again because the external function immediately calls the summarized internal implementation.
    
*   Internal calls to `f` will be summarized.
    

On the other hand, suppose we provide an `external` summary for `f`. In this case:

*   Calls from CVL to `f` _will not_ be summarized, because direct calls from CVL to contract functions do not use summaries.
    
*   Internal calls to `f` _will not_ be summarized - they will use the original implementation.
    
*   External calls to `f` (from Solidity code that calls `this.f` or `c.f`) will be summarized
    

In most cases, public functions should use an `internal` summary, since this effectively summarizes both internal and external calls to the function.

`envfree` annotations

------------------------------------------------------------------------------

Following the `returns` clause of an exact methods entry is an optional `envfree` tag. Marking a method with `envfree` has two effects. First, calls to the method from CVL do not need to explicitly pass an environment value as the first argument. Second, the Prover will verify that the method implementation in the contract being verified does not depend on any of the environment variables. The results of this check are displayed on the verification report as separate rules called `envfreeFuncsStaticCheck` and `envfreeFuncsAreNonpayable`[1].

`optional` annotations

--------------------------------------------------------------------------------

Added in version 4.0: Prior to CVL 2, all methods entries used the `optional` behavior, and there was no `optional` annotation.

When multiple contracts implement a shared interface, it is convenient to write a generic spec of generic rules. Some interfaces specify optional methods that some implementations provide and others don’t. For example, some ERC20 implementations contain a `mint` method, but others don’t.

In this situation, you might like to write rules that are checked if the contract contains the `mint` method and are skipped otherwise.

To do so, you can add the `optional` annotation to the exact methods block entry for the function. Any rules that reference an optional method will be skipped if the method does not exist in the contract. For example:

```
methods {
    function mint(address _to, uint256 _amount, bytes calldata _data) external optional;
}

```


`with(env e)` clauses

-----------------------------------------------------------------------------

After the `optional` annotation, an entry may contain a `with(env e)` clause. The `with` clause introduces a new variable (`e` for `with(env e)`) to represent the environment that is passed to a summarized function; the variable can be used in function summaries. `with` clauses may only be used if the entry has a function summary. See Expression summaries below for more information about the environment provided by the `with` clause.

Summaries

--------------------------------------------------------

**Summary declarations** are used to replace calls to methods having the given signature with something that is simpler for the Prover to reason about. Summaries allow the Prover to reason about external contracts whose code is unavailable. They can also be useful to simplify the code being verified to circumvent timeouts.

A summary is indicated by adding `=>` followed by the summary to the end of the entry in the methods block. For example,

```
function f(uint) external returns(uint) => ALWAYS(0);

```


will replace calls to `f` with an `ALWAYS` summary, while

```
function f(uint x) external returns(uint) => cvl_function(x);

```


will replace calls to `f` with the CVL function `cvl_function`.

There are several kinds of summaries available:

*   View summaries: ALWAYS, CONSTANT, PER\_CALLEE\_CONSTANT, and NONDET. These assume that the called method have no side-effects and simply replace them with a specific value.
    
*   Havoc summaries: HAVOC\_ALL and HAVOC\_ECF. These assume that the called method can have arbitrary side-effects on the storage of some contracts.
    
*   DISPATCHER summaries assume that the receiver of the method call could be any contract that implements the method.
    
*   Expression summaries replace calls to the summarized method with a CVL expression, typically Functions or Ghost axioms.
    
*   AUTO summaries are the default for unresolved calls.
    
*   ASSERT\_FALSE summaries. These replace the method with an assert false, effectively checking that no such method is called.
    

### Summary application


To decide whether to summarize a given function call at a given call site, the Prover first determines whether it matches any of the declarations in the methods block, and then uses the declaration and the _calling context_ to determine whether the call should be replaced by an approximation.

Specifically, the matching is based on three attributes:

(1) The contract in which the method is defined, or a wildcard contract denoted with `_`.

(2) The method signature, with optional named parameters.

(3) The context in which it is called, either `external` or `internal`. A Solidity function which is defined as `public` can be specified in the methods block as either `external` or `internal`, and this affects which call sites of the function will be summarized.

The ability of the Prover to match a particular call site to a method declaration depends on whether the call was _resolved_ or not, i.e., whether we know which target contract is called and which method signature is called. Internal calls are always resolved, but for external calls it is not always the case. For example, the target contract may be given by a user input, and there is no single match for the target contract:

```
function callIt(address it) external {
  IERC20(it).transfer(...); // cast `it` to an IERC20 contract and call the `transfer` method
}

```


Similarly, the method signature may also be not resolvable:

```
function callIt(bytes memory data) external {
  address(this).call(data);
}

```


To determine whether a function call is replaced by an approximation summary, the Prover considers all three aforementioned attributes, the resolved information, and in addition to that, also the application policy. If present, the application policy must be either `ALL`, `UNRESOLVED`, or `DELETE`. The `ALL` policy indicates the summary should be applied to all instances of the specified method, while `UNRESOLVED` applies only to methods that cannot be fully resolved (i.e., either target contract or the method signature are unknown). For internal summaries, the default is `ALL`, as all internal functions are always resolvable; thus `UNRESOLVED` is impossible and will yield an error. Similarly, for external summaries with contract-specific entries, the default policy is `ALL`. Conversely, for any external summary on wildcard contracts, the default policy is `UNRESOLVED`. One may apply the `ALL` policy to make the summary apply on all instances of the wildcard method, even on target contracts for which it was resolved, e.g. by linking.

A `DELETE` summary is similar to an `ALL` summary, except that the `DELETE` summary removes the method from the scene entirely. Calling the method from CVL will produce a rule violation, and parametric rules will not be instantiated on the deleted method. This can drastically improve performance if the deleted method is complex.

The decision to replace a call by an approximation is made as follows:

*   If the function is called from CVL rather than from contract code then it is never replaced by a summary.
    
*   If the code for the function is known at verification time, either because it is a method of `currentContract` or because the receiver contract is `linked`, then the function is only summarized if the resolution type is `ALL`.
    
*   If the code for the function is not known at verification time, then the function call must be summarized. If no summary is given, the default summary type is AUTO, whose behavior is determined by the type of function call. In this case, the verification report will contain a contract call resolution warning.
    

### Summary resolution


With wildcard entries, catch-all entries, and exact entries, multiple entries could apply to a method.

For example, given a call to `Token.burn()` with a methods block that contains the following entries:

```
methods {
   function Token.burn() external => HAVOC_ECF;
   function _.burn() external => HAVOC_ALL;
   function Token._ external => NONDET;
}

```


which summary will apply? In CVL, precedence is given to the summary attached to the _most specific signature_. Exact entries are considered more exact than wildcard entries, which are themselves more exact than catch-all entries. In other words, the order of precedence for summaries are:

1.  Summaries given for exact entries
    
2.  Summaries given for wildcard entries
    
3.  Summaries given for catch-all entries
    

Thus, in this example, the `HAVOC_ECF` summary would apply.

Note

An entry that does not have a summary attached does _not_ factor into the precedence of summary application. For example, if the first entry in the above was instead `function Token.burn() external envfree;` without a summary, the `HAVOC_ALL` of the wildcard entry will apply.

### Summary types


#### View summaries: `ALWAYS`, `CONSTANT`, `PER_CALLEE_CONSTANT`, and `NONDET`


These four summary types treat the summarized methods as view methods: the summarized methods are replaced by approximations that do not update the state of any contract (aside from any balances transferred with the method call itself). They differ in the assumptions made about the return value:

*   The `ALWAYS(v)` approximation assumes that the method always returns `v`. The value `v` must be a literal boolean or integer.
    
*   The `CONSTANT` approximation assumes that all calls to methods with the given signature always return the same result. If the summarized method is expected to return multiple results, the approximation returns the correct number of values.
    
*   The `PER_CALLEE_CONSTANT` approximation assumes that all calls to the method on a given receiver contract must return the same result, but that the returned value may be different for different receiver contracts. If the summarized method is expected to return multiple results, the approximation returns the correct number of values.
    
*   The `NONDET` approximation makes no assumptions about the return values; each call to the summarized method may return a different result. The number of returned values is _not_ assumed to match the requested number, unless optimisticReturnsize is specified.
    

Warning

Using `CONSTANT` and `PER_CALLEE_CONSTANT` summaries for functions that have variable-sized outputs is a potential source of vacuity and should be avoided. Prefer a `NONDET` summary where possible.

#### Havoc summaries: `HAVOC_ALL` and `HAVOC_ECF`


The most conservative summary type is `HAVOC_ALL`. This summary makes no assumptions at all about the called function: it is allowed to have arbitrary side effects on the state of any contract (including the calling contract), and may return any value. It can also change any contract’s ETH balance in an arbitrary way. In effect, calling a method that is summarized by `HAVOC_ALL` obliterates all knowledge that the Prover has about the state of the contract before the call.

The `HAVOC_ALL` approximation is sound, but it can be overly restrictive in practice. In reality, a contract’s state cannot be changed in arbitrary ways, but only according to the contract’s methods. However, the Prover does not currently have support for more fine-grained reasoning about the side effects of unknown methods.

A useful middle ground is the `HAVOC_ECF` summary type. A `HAVOC_ECF` summarization for a method encodes the assumption that the called method is not reentrant. This summarization approximates a method call by assuming it can have arbitrary effects on contracts other than the contract being verified, but that it can neither change the current contract’s state nor decrease its ETH balance (aside from value transferred by the method call itself).

The Prover makes no assumptions about the return value of a havoc summary. For methods that return multiple values, the approximations are allowed to return the incorrect number of results. In most cases, this will cause the calling method to revert. If you want to ignore this particular revert condition, you can pass the optimisticReturnsize option.

#### `DISPATCHER` summaries


The `DISPATCHER` summary type provides a useful approximation for methods of interfaces that are implemented by multiple contracts. For example, the methods defined by the ERC20 specification are often summarized using the `DISPATCHER` summary type.

If a function with a `DISPATCHER` summary is called, the Prover will assume that the receiver of the call is one of the known contract implementations containing the given signature; the call will then behave the same way that a normal method call on the receiver would. The Prover will consider examples with every possible implementing contract, but multiple `DISPATCHER` method calls on the same receiver address in the same example will use the same receiver contract.

The set of contract implementations that the Prover chooses from contains the set of contracts passed as arguments to the CLI. In addition, the Prover may consider an unknown target contract whose methods are all interpreted using the AUTO summary. The presence of the unknown contract is determined by the optional boolean argument to the `DISPATCHER` summary:

*   With `DISPATCHER(false)` or just `DISPATCHER`, the unknown contract is considered as a possibility
    
*   With `DISPATCHER(true)`, only the known contract instances are considered
    

There is an alternative syntax for determining the presence or absence of the unknown contract:

*   `DISPATCHER(optimistic=<true|false>)` with `true` and ‘false\` having the same meaning as in the other syntax.
    

In some cases there’s a proxy contract that only has a fallback function and that fallback then delegates function calls it receives to some other contract. For this case it could be useful for `DISPATCHER` summaries to also inline the `fallback` function of known contracts. To enable this use the following syntax:

*   `DISPATCHER(optimistic=<true|false>, use_fallback<true|false>)`
    

Note

The most commonly used dispatcher mode is `DISPATCHER(true)`, because in almost all cases `DISPATCHER(false)` and `AUTO` report the same set of violations. Since Certora CLI version 7.7.0 when using `_.someFunc() => DISPATCHER(true)` the Prover first tests that a method `someFunc()` exists in the scene, and if not will fail. Before this version, this may cause vacuous results.

Note

`DISPATCHER` summaries cannot be used to summarize library calls.

#### `AUTO` summaries


The behavior of the `AUTO` summary depends on the type of call[2]:

*   Calls to non-library `view` and `pure` methods use the `NONDET` approximation: they keep all state unchanged.
    
*   Calls to library methods and `delegatecall`s are assumed to change the caller’s storage in an arbitrary way, but are assumed to leave ETH balances and the storage of other contracts unchanged.
    
*   All other calls and constructors use the `HAVOC_ECF` approximation: they are assumed to change the state of external contracts arbitrarily but to leave the caller’s state unchanged. `AUTO` summary behavior for the `CALL` opcode with 0 length `calldata` can be changed with optimistic\_fallback.
    

#### `ASSERT_FALSE` summaries


This summary is a short syntax for a summary that contains an `assert false;` and checks that the summarized method is not reached. This can be useful for instance, in the presence of unresolved calls in combination with the `unresolved external` syntax to ensure that every unresolved call is actually dispatched correctly (i.e. use `unresolved external in _._ => DISPATCH [...] default ASSERT_FALSE`). It also enables more optimizations in the Prover and may lead to shorter running times.

#### Expression summaries


Contract methods can also be summarized using CVL expressions, typically Functions or Ghost axioms as approximations. Contract calls to the summarized method are replaced by evaluation of the CVL expression.

To use a CVL function or ghost as a summary, use a call to the function in place of the summary type.

If a wildcard entry has a ghost or function summary, the user must explicitly provide an `expect` clause to the summary. The `expect` clause tells the Prover how to interpret the value returned by the summary. For example:

```
methods {
    function _.foo() external => fooImpl() expect uint256 ALL;
}

```


This entry will replace any call to any external function `foo()` with a call to the CVL function `fooImpl()` and will interpret the output of `fooImpl` as a `uint256`.

If a function does not return any value, the summary should be declared with `expect void`.

Warning

You must check that your `expect` clauses are correct.

The Prover cannot always check that the return type declared in the `expect` clause matches the return type that the contract expects. Continuing the above example, suppose the contract being verified declared a method `foo()` that returns a type other than `uint256`:

```
function foo() external returns(address) {
    ...
}

function bar() internal {
    address x = y.foo();
}

```


In this case, the Prover would encode the value returned by `fooImpl()` as a `uint256`, and the `bar` method would then attempt to decode this value as an `address`. This will cause undefined behavior, and in some cases the Prover will not be able to detect the error.

The function call can only refer directly to the variables defined as arguments in the summary declarations; expressions that combine those variables are not supported.

The function call may also use the special variable `calledContract`, which gives the address of the contract on which the summarized method was called. More precisely, it equates to `address(this)` in the context of the original call that is being summarized. This is useful for identifying the called contract in wildcard summaries. For internal functions, the `calledContract` is also the calling contract, since they are the same. For library functions and delegate calls the `calledContract` is the contract calling the function. Similarly, there is another special variable `executingContract`, which gives the address of the contract making the call to the function that is summarized. For internal, delegate and library calls, `executingContract` is the same as `calledContract`. They differ only in non-delegate external calls, where `calledContract` will be the receiver of the call and `executingContract` will be the caller. The `calledContract` and `executingContract` keywords may only be used inside the `methods` block, and `executingContract` also in hook bodies.

For example, a wildcard summary for a `transferFrom` method may apply to multiple ERC20 contracts; the summary can update the correct ghost variables as follows:

```
methods {
    function _.transferFrom(address from, address to, uint256 amount) external
        => cvlTransferFrom(calledContract, from, to, amount);
}

ghost mapping(address => mapping(address => mathint)) tokenBalances;

function cvlTransferFrom(address token, address from, address to, uint amount) {
    if (...) {
        tokenBalances[token][from] -= amount;
        tokenBalances[token][to]   += amount;
    }
}

```


When summarizing an internal library function to an expression, you cannot refer to a variable that is `storage`, since CVL functions cannot take variables that are `storage`. You can refer to other variables, or use a summarization that doesn’t take parameters:

```
methods {
    function MyLibrary.guess(int[] storage numbers) internal returns (int) => goodGuess1(numbers); // not allowed
    function MyLibrary.guess(int[] storage numbers, int myGuess) internal returns (int) => goodGuess2(myGuess); // allowed
    function MyLibrary.guess(int[] storage numbers) internal returns (int) => ALWAYS(42); // allowed
}

function goodGuess1(int[] numbers) returns int { return 4; }
function goodGuess2(int myGuess) returns int { return myGuess; }

```


The call can also refer to a variable of type `env` introduced by a with(env) clause. Here `e` may be replaced with any valid identifier.

The variable defined by the `with` clause contains an env type giving the context for the summarized function. This context may be different from the `env` passed to the original call from the spec. In particular:

*   `e.msg.sender` and `e.msg.value` refer to the sender and value from the most recent call to a non-library[3] external function (as in Solidity)
    
*   The variables `e.tx.origin`, `e.block.number`, and `e.block.timestamp` will be the same as the the environment for the outermost function call.
    

Continuing the above example, one can use the `env` to summarize the `transfer` method:

```
methods {
    function _.transfer(address to, uint256 amount) external with(env e)
        => cvlTransfer(calledContract, e, to, amount) expect void;
}

function cvlTransfer(address token, env passedEnv, address to, uint amount) {
    ...
}

rule example {
    env e;
    address sender;
    require e.msg.sender == sender;
    c.process(e);
}

```


In this example, if the `process` method calls `t.transfer(...)`, then in the `cvlTransfer` function, `token` will be `t`, `passedEnv.msg.sender` will be `c`, and `passedEnv.tx.origin` will be `sender`.

There is a restriction on the functions that can be used as approximations. Namely, the types of any arguments passed to or values returned from the summary must be convertible between CVL and Solidity types. Arguments that are not accessed in the summary may have any type.

You can still summarize functions that take unconvertible types as arguments, but you cannot access those arguments in your summary.

In case of recursive calls due to the summarization, the recursion limit can be set with `--summary_recursion_limit N'` where `N` is the number of recursive calls allowed (default 0). If `--optimistic_summary_recursion` is set, the recursion limit is assumed, i.e. one will never get a counterexample going above the recursion limit. Otherwise, if it is possible to go above the recursion limit, an assert will fire, producing a counterexample to the rule.

* * *
