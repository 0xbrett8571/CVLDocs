# Rules — Certora Prover Documentation 0.0 documentation
Rules (along with Invariants) are the main entry points for the Prover. A rule defines a sequence of commands that should be simulated during verification.

When the Prover is invoked with the verify option, it generates a report for each rule and invariant present in the spec file (as well as any imported rules).

You can find examples for rules in the Certora Prover and CVL Examples Repository. For example, the specs in /CVLByExample/Teams/ demonstrate some of these features.

Contents

*   Rules
    
    *   Syntax
        
    *   Overview
        
    *   Parametric rules
        
    *   Filters
        
    *   Multiple assertions
        
    *   Rule descriptions
        
    *   How rules are verified
        

Syntax

-------------------------------------------------

The syntax for rules is given by the following EBNF grammar:

```
rule ::= [ "rule" ]
         id
         [ "(" [ params ] ")" ]
         [ "filtered" "{" id "->" expression { "," id "->" expression } "}" ]
         [ "description" string ]
         [ "good_description" string ]
         block

params ::= cvl_type [ id ] { "," cvl_type [ id ] }


```


See Basic Syntax for the `id` and `string` productions; see Expressions for the `expression` production; see Types for the `cvl_type` production.

Overview

-----------------------------------------------------

A rule defines a sequence of commands that should be simulated during verification. These commands may be non-deterministic: they may contain unassigned variables whose value is not specified. The state of storage at the beginning of a rule is also unspecified. Rules may also be declared with a set of parameters; these parameters are treated the same way as undeclared variables.

In principal, the Prover will generate every possible combination of values for the undefined variables, and simulate the commands in the rule using those values. A particular combination of values is referred to as an example or a model. There are often an infinite number of models for a given rule; see How rules are verified for a brief explanation of how the Prover considers all of them.

If a rule contains a `require` statement that fails on a particular example, the example is ignored. Of the remaining examples, the Prover checks that all of the `assert` statements evaluate to true. If all of the `assert` statements evaluate to true on every example, the rule passes. Otherwise, the Prover will output a specific counterexample that causes the assertions to fail.

*   simple rule example
    
    ```
/// `deposit` must increase the pool's underlying asset balance
rule integrityOfDeposit {

    mathint balance_before = underlyingBalance();


    env e; uint256 amount;
    safeAssumptions(_, e);

    deposit(e, amount);

    mathint balance_after = underlyingBalance();

    assert balance_after == balance_before + amount,
        "deposit must increase the underlying balance of the pool";
}

```

    

Caution

`assert` statements in contract code are handled differently from `assert` statements in rules.

An `assert` statement in Solidity causes the transaction to revert, in the same way that a `require` statement in Solidity would. By default, examples that cause contract functions to revert are ignored by the prover, and these examples will _not_ be reported as counterexamples.

The multi\_assert\_check option causes assertions in the contract code to be reported as counterexamples.

Parametric rules

---------------------------------------------------------------------

Rules that contain undefined `method` variables are sometimes called parametric rules. See The method and calldataarg types for more details about how to use method variables.

Undefined variables of the `method` type are treated slightly differently from undefined variables of other types. If a rule uses one or more undefined `method` variables, the Prover will generate a separate report for each method (or combination of methods).

In particular, the Prover will generate a separate counterexample for each method that violates the rule, and will indicate if some contract methods always satisfy the rule.

You can request that the Prover only run with specific methods using the method and parametric\_contracts command line arguments. The set of methods can also be restricted using rule filters. The Prover will automatically skip any methods that have \`DELETE\` summaries.

If you wish to only invoke methods on a certain contract, you can call the `method` variable with an explicit receiver contract. The receiver must be a contract variable (either currentContract or a variable introduced with a `using` statement). For example, the following will only verify the rule `r` on methods of the contract `example`:

```
using Example as example;

rule r {
    method f; env e; calldataarg args;
    example.f(e,args);
    ...
}

```


It is an error to call the same `method` variable on two different contracts.

```
  rule sanity(method f) {
    env e;
    calldataarg args;
    f(e,args);
    assert false;
    }

```


*   parameteric rule example
    

Filters

---------------------------------------------------

A rule declaration may have a `filtered` block after the rule parameters. Rule filters allow you to prevent verification of parametric rules on certain methods. This can be less computationally expensive than using a `require` statement to ignore counterexamples for a method.

The `filtered` block consists of zero or more filters of the form `var -> expr`. `var` must match one of the `method` parameters to the rule, and `expr` must be a boolean expression that may refer to the variable `var`. The filter expression may not refer to other method parameters or any variables defined in the rule.

Before verifying that a method `m` satisfies a parametric rule, the `expr` is evaluated with `var` bound to a `method` object. This allows `expr` to refer to the fields of `var`, such as `var.selector` and `var.isView`. See The method and calldataarg types for a list of the fields available on `method` objects.

For example, the following rule has two filters. The rule will only be verified with `f` instantiated by a view method, and `g` instantiated by a method other than `exampleMethod(uint,uint)` or `otherExample(address)`:

*   filters example
    

```
rule r(method f, method g) filtered {
    f -> f.isView,
    g -> g.selector != exampleMethod(uint,uint).selector
      && g.selector != otherExample(address).selector
} {
    // rule body
    ...
}

```


See The method and calldataarg types for a list of the fields of the `method` type.

Multiple assertions

---------------------------------------------------------------------------

Rules may contain multiple assertions. By default, if any assertion fails, the Prover will report that the entire rule failed and give a counterexample that causes one of the assertions to fail.

Occasionally it is useful to consider different assert statements in a rule separately. With the multi\_assert\_check option, the Prover will try to generate separate counterexamples for each `assert` statement. The counterexamples generated for a particular `assert` statement will pass all earlier `assert` statements.

Rule descriptions

-----------------------------------------------------------------------

Rules may be annotated by writing `description` and/or `good_description` before the method body, followed by a string. These strings are displayed in the verification report.

How rules are verified

---------------------------------------------------------------------------------

While verifying a rule, the Prover does not actually enumerate every possible example and run the rule on the example. Instead, the Prover translates the contract code and the rule into a logical formula with logical variables representing the unspecified variables from the rule.

The logical formula is designed so that if a particular example satisfies the requirements and also causes an assertion to fail, then the formula will evaluate to `true` on that example; otherwise the formula will evaluate to false.

The Prover then uses off-the-shelf software called an SMT solver to determine whether there are any examples that cause the formula to evaluate to true. If there are, the SMT solver provides an example to the Prover, which then translates it into an example for the user. If the SMT solver reports that the formula is unsatisfiable, then we are guaranteed that whenever the `require` statements are true, the `assert` statements are also true.
