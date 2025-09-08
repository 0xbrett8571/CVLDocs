
*   replace `sinvoke f(...)` with `f(...)`
    
*   replace `invoke f(...)` with `f@withrevert(...)`
    
*   replace `f(...).selector` with `sig:f(...).selector`
    
*   ensure that rules start with `rule`
    
*   replace `static_assert` with `assert`
    
*   replace `static_require` with `require`
    
*   add `;` to the end of `pragma`, `import`, `using`, and `use` statements
    
*   add a `;` to the end of a methods block entry if it doesn’t seem to continue to the next line
    
*   add `function` to the beginning of a methods block entry
    
*   add `external` to unsummarized or `DISPATCHER` methods block entries
    
*   change `function f(...)` to `function _.f(...)` for summarized external functions
    

In particular, as the script only consumes spec files, there are decisions that it cannot make, as they are based on the Solidity code. Some of those are listed here.

## [Step 3: Fix type errors](#id6)[](#step-3-fix-type-errors "Link to this heading")

This is a good time to try running `certoraRun` on your spec. The command-line interface to `certoraRun` has not changed in CVL 2, so you should try to verify your contract the same way you usually would.

If your spec verifies without errors, move on to [Step 4: Review your methods blocks](#cvl2-migration-summaries)! If `certoraRun` reports errors, you will need to fix them manually. Here are some of the more common errors that you may come across:

This section contains specific advice for these situations; if you come across problems that are not covered here, consult the [Changes Introduced in CVL 2](changes.html) or ask!

### [Syntax errors introduced by the migration script](#id9)[](#syntax-errors-introduced-by-the-migration-script "Link to this heading")

The migration script is not perfect, and can make syntax mistakes in some cases, such as adding an extra semicolon or omitting a keyword. We hope these will be easy to identify and fix, but if you have syntax errors you can’t understand, consult [Superficial syntax changes](changes.html#cvl2-superficial-syntax-changes).

### [Type errors in arithmetic and casts](#id10)[](#type-errors-in-arithmetic-and-casts "Link to this heading")

CVL 2 is more careful about converting between different integer types. See [Changes to integer types](changes.html#cvl2-integer-types) in the changes guide for complete details and examples.

If you have errors that indicate problems with number types, try the following:

*   Try to change most of your integers to `mathint`. The only integers that should _not_ be `mathint` are those that you are passing as arguments to contract functions.
    
*   If you have a type error in a `havoc ... assuming` statement, consider using the [newer ghost variable syntax](../ghosts.html#ghost-variables). This can avoid potential vacuity pitfalls caused by mixing `to_mathint` and `havoc ... assuming`.
    
*   If you need to modify the output of one contract function and pass it to another contract function, you will need to think carefully about how you want to handle overflow. If you think the computation won’t go out of bounds, you can use an `assert_` cast to assert that the value is in bounds. If you want to ignore cases where the value goes out of bounds, you can use a `require_` cast (but think twice first: `require_` casts are dangerous!). See [Changes to integer types](changes.html#cvl2-integer-types) for more details.
    

Warning

Use `assert_` and `require_` casts sparingly! `assert_` casts can lead to unnecessary counterexamples, and `require_` casts can hide bugs in your contracts (just as any `require` statement can).

*   You cannot use `assert_` and `require_` casts inside [quantified statements](../../user-guide/glossary.html#term-quantifier). To solve that issue, you can introduce an additional universally quantified variable of type `uint256`, and require it to be equal to the expression using an upcast to `mathint`.
    
    For example, if there is a ghost array access `forall uint x. a[x+1] == 0`, rewrite it as follows:
    
    forall uint x. forall uint y. to\_mathint(y) \== x+1 \=> a\[y\] \== 0
    

### [`using` statements](#id11)[](#using-statements "Link to this heading")

Multi-contract declaration using `using` statements are not imported. If you have a spec `a.spec` importing `b.spec`, with `b.spec` declaring a multicontract contract usage, which you need in `a.spec`, repeat the declaration from `b.spec`, and rename the alias.

_The next minor version of CVL2 will improve this behavior._

### [Problems with `certorafallback` or `invoke_fallback`](#id12)[](#problems-with-certorafallback-or-invoke-fallback "Link to this heading")

CVL2 does not allow you to refer to the fallback function explicitly as it was seldom used and not well-defined. The most common use case for having to refer to the fallback was to check if a parametric method is the fallback function. For that, one can use the `.isFallback` field of any variable of type `method`.

See [Changes to the fallback function](changes.html#cvl2-fallback-changes) for examples.

## [Step 4: Review your `methods` blocks](#id7)[](#step-4-review-your-methods-blocks "Link to this heading")

CVL 2 changes the requirements for and meanings of methods block entries; you should manually review all of your methods block entries to make sure they have the intended meanings. Here are the things to consider:

The remainder of this section describes these considerations. See [Changes to methods block entries](changes.html#cvl2-methods-blocks) for more details.

If you have complex methods blocks, we encourage you to examine the call resolution tab on the rule report to double-check that your summaries are applied as you expect them to be.

### [`internal` and `external` methods](#id13)[](#internal-and-external-methods "Link to this heading")

In CVL 2, you must mark `methods` block entries as either `internal` or `external`. Unlike Solidity, you cannot mark entries as `private` or `public`.

The Prover does not distinguish between `private` and `internal` methods; if you want to summarize a `private` method, use `internal` in the `methods` block.

To understand how to work with public Solidity methods, it is important to understand how Solidity compiles public functions. When a contract contains a public method, the Solidity compiler generates an internal method that executes the code, and an external method that calls the internal method.

You can add methods block entries for either (or both) of those methods, and they will have different effects. See [Required internal or external annotation](changes.html#cvl2-visibility) for the details.

### [Receiver contracts](#id14)[](#receiver-contracts "Link to this heading")

In CVL 1, method summaries applied to all methods in all contracts that match the specified signature. In CVL 2, summaries only apply to one contract by default.

You specify the receiver contract just before the method name. For example, to refer to the `exampleMethod` method of the `ExampleContract` contract, you would write:

methods {
    function ExampleContract.exampleMethod(uint) external returns(uint);
}

If no contract is specified, the default contract is `currentContract`.

If you want to write an entry that applies to methods in all contracts with the given signature, you can use the special `_` receiver:

methods {
    function \_.exampleMethod(uint) external \=> NONDET;
}

Wildcard entries cannot specify return types. If you summarize them with a CVL function or ghost, you will need to supply an `expect` clause. See [Summaries only apply to one contract by default](changes.html#cvl2-wildcards) for details.

## [Step 5: Profit!](#id8)[](#step-5-profit "Link to this heading")

Hopefully this guide has helped you successfully migrate to CVL 2. Although the functional changes in CVL 2.0 are relatively small, the internal changes lay the groundwork for many exciting features. We promise that the effort involved in migration will pay off in the next few releases!
