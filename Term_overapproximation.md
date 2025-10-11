
A statement accepted as true without proof.

call trace[](#term-call-trace "Link to this term")

A call trace is the Prover’s visualization of either a [counterexample](#term-counterexample) or a [witness example](#term-witness-example).

A call trace illustrates a rule execution that leads to the violation of an `assert` statement or the fulfillment of a `satisfy` statement. The trace is a sequence of commands in the rule (or in the contracts the rule was calling into), starting at the beginning of the rule and ending with the violated `assert` or fulfilled `satisfy` statement. In addition to the commands, the call trace also makes the best effort to show information about the program state at each point in the execution. It contains information about the state of global variables at crucial points as well as the values of call parameters, return values, and more.

If a call trace exists, it can be found in the “Call Trace” tab in the report after selecting the corresponding (sub-)rule.

CFG[](#term-CFG "Link to this term")

control flow graph[](#term-control-flow-graph "Link to this term")

control flow path[](#term-control-flow-path "Link to this term")

Control flow graphs (short: CFGs) are a program representation that illustrates in which order the program’s instructions are processed during program execution. The nodes in a control flow graph represent single non-branching sequences of commands. The edges in a control flow graph represent the possibility of control passing from the last command of the source node to the first command of the target node. For instance, an `if`\-statement in the program will lead to a branching, i.e., a node with two outgoing edges, in the control flow graph. A CVL rule can be seen as a program with some extra “assert” commands, thus a rule has a CFG like regular programs. The Certora Prover’s [TAC reports](../prover/diagnosis/index.html#tac-reports) contain a control flow graph of the [TAC](#term-TAC) intermediate representation of each given CVL rule. The control flow paths are the paths from source to sink in a given CFG. In general (and in practice), the number of control flow paths grows exponentially with the size of the CFG. This is known as the path explosion problem. Further reading: [Wikipedia: Control-flow graph](https://en.wikipedia.org/wiki/Control-flow_graph) [Wikipedia: Path explosion problem](https://en.wikipedia.org/wiki/Path_explosion)

environment[](#term-environment "Link to this term")

The environment of a method call refers to the global variables that Solidity provides, including `msg`, `block`, and `tx`. CVL represents these variables in a structure of type [env](../cvl/types.html#env). The environment does _not_ include the contract state or the state of other contracts — these are referred to as the [storage](../cvl/types.html#storage-type).

EVM[](#term-EVM "Link to this term")

Ethereum Virtual Machine[](#term-Ethereum-Virtual-Machine "Link to this term")

EVM bytecode[](#term-EVM-bytecode "Link to this term")

EVM is short for Ethereum Virtual Machine. EVM bytecode is one of the source languages the Certora Prover can take as input for verification. It is produced by the Solidity and Vyper compilers, among others. The following links provide good entry points for details on what the EVM is and how it works: [Official documentation](https://ethereum.org/en/developers/docs/evm/), [Wikipedia](https://en.wikipedia.org/wiki/Ethereum#Virtual_machine)

EVM memory[](#term-EVM-memory "Link to this term")

EVM storage[](#term-EVM-storage "Link to this term")

The [EVM](#term-EVM) has two major concepts of memory, called _memory_ and _storage_. In brief, memory variables keep data only for the duration of a single EVM transaction, while storage variables are stored persistently in the Ethereum blockchain. [Official documentation](https://ethereum.org/en/developers/docs/smart-contracts/anatomy)

havoc[](#term-havoc "Link to this term")

Havoc refers to assigning variables arbitrary, non-deterministic values. This occurs in two main cases:

1.  At the beginning of a rule, all variables are havoced to model an unknown initial state.
    
2.  During rule execution, certain events may cause specific variables to be havoced. For example, when calling an external function on an unknown contract, the Prover assumes it could arbitrarily affect the state of a third contract.
    

For more information, see [Havoc summaries: HAVOC\_ALL and HAVOC\_ECF](../cvl/methods.html#havoc-summary) and [Havoc Statements](../cvl/statements.html#havoc-stmt).

hyperproperty[](#term-hyperproperty "Link to this term")

A hyperproperty describes a relationship between two hypothetical sequences of operations starting from the same initial state. For example, a statement like “two small deposits will have the same effect as one large deposit” is a hyperproperty. See [The storage type](../cvl/types.html#storage-type) for more details.

invariant[](#term-invariant "Link to this term")

An invariant (or representation invariant) is a property of the contract state that is expected to hold between invocations of contract methods. See [Invariants](../cvl/invariants.html#invariants).

model[](#term-model "Link to this term")

example[](#term-example "Link to this term")

counterexample[](#term-counterexample "Link to this term")

witness example[](#term-witness-example "Link to this term")

We use the terms “model” and “example” interchangeably. In the context of a CVL rule, they refer to an assignment of values to all CVL variables and contract storage that either:

*   violates an `assert` statement, in which case the model is also called a **counterexample**, or
    
*   satisfies a `satisfy` statement, in which case it is also called a **witness example**.
    

See [Overview](../cvl/rules.html#rule-overview) for more on how these are used.

In the context of SMT solvers, a _model_ refers to a valuation of the logical constants and uninterpreted functions in the input formula that makes the formula evaluate to `true`. See [SAT result](#term-SAT-result) for more details.

linear arithmetic[](#term-linear-arithmetic "Link to this term")

nonlinear arithmetic[](#term-nonlinear-arithmetic "Link to this term")

An arithmetic expression is called linear if it consists only of additions, subtractions, and multiplications by constants. Division and modulo where the second parameter is a constant are also linear arithmetic. Examples for linear expressions are `x * 3`, `x / 3`, `5 * (x + 3 * y)`. Every arithmetic expression that is not linear is nonlinear. Examples for nonlinear expressions are `x * y`, `x * (1 + y)`, `x * x`, `3 / x`, `3 ^ x`.

overapproximation[](#term-overapproximation "Link to this term")

underapproximation[](#term-underapproximation "Link to this term")

Sometimes, it is useful to replace a complex piece of code with something simpler that is easier to reason about. If the approximation includes all of the possible behaviors of the original code (and possibly others), it is called an “overapproximation”; if it does not, it is called an “underapproximation”.

Example: A [NONDET](../cvl/methods.html#view-summary) summary is an overapproximation because the Certora Prover considers every possible value that the original implementation could return, while an [ALWAYS](../cvl/methods.html#view-summary) summary is an underapproximation if the summarized method could return more than one value.

Proofs on overapproximated programs are [sound](#term-sound), but spurious [counterexample](#term-counterexample)s may be caused by behavior that the original code did not exhibit. Underapproximations are more dangerous because a property that is successfully verified on the underapproximation may not hold on the approximated code.

optimistic assumptions[](#term-optimistic-assumptions "Link to this term")

pessimistic assertions[](#term-pessimistic-assertions "Link to this term")

Some input programs contain constructs that the Prover can only handle in an approximative way. This approximation entails that the Prover will disregard some specific parts of the programs behavior, like for example the behavior induced by a loop being unrolled beyond a fixed number of times. For each of these constructs the Prover provides a flag controlling whether it should handle them optimistically or pessimistically. (See the links at the end of this paragraph for examples of “optimistic” flags.)

In pessimistic mode (which is the default) _pessimistic assertions_ are inserted into the program that check whether there is any behavior that needs to be approximated, for instance whether loops are present with bounds exceeding [loop\_iter](../prover/cli/options.html#loop-iter). If this is the case, the rule will fail with a corresponding message.

In optimistic mode, instead of the assertions, _optimistic assumptions_ are introduced in each of the places where an approximation happens. Each assumption excludes the relevant behavior from checking for one occurrence of the problematic construct, e.g., for each loop.

In a more general sense, “pessimistic” behavior rather reports a failure, making sure that no error is missed, but risking to report irrelevant errors, and vice versa for “optimistic” behavior. Pessimistic behavior thus corresponds to [overapproximation](#term-overapproximation)s, and optimistic to [underapproximation](#term-underapproximation)s.

For a list of all “optimistic” settings see [CLI Options](../prover/cli/options.html#prover-cli-options). Examples include [optimistic\_hashing](../prover/cli/options.html#optimistic-hashing), [optimistic\_loop](../prover/cli/options.html#optimistic-loop), [optimistic\_summary\_recursion](../prover/cli/options.html#optimistic-summary-recursion), and more. Also see [Prover Approximations](../prover/approx/index.html#prover-approximations) for more background on some of the approximations.

parametric rule[](#term-parametric-rule "Link to this term")

A parametric rule is a rule that calls an ambiguous method, either using a method variable, or using an overloaded function name. The Certora Prover will generate a separate report for each possible instantiation of the method. See [Parametric rules](../cvl/rules.html#parametric-rules) for more information.

quantifier[](#term-quantifier "Link to this term")

quantified expression[](#term-quantified-expression "Link to this term")

The symbols `forall` and `exist` are sometimes referred to as _quantifiers_, and expressions of the form `forall type v . e` and `exist type v . e` are referred to as _quantified expressions_. See [Extended logical operations](../cvl/expr.html#logic-exprs) for details about quantifiers in CVL.

receiveOrFallback[](#term-receiveOrFallback "Link to this term")

A special function automatically added to every contract to model how Solidity handles calls that either have no `calldata` or do not match any existing function signature.

1.  If the call has no data and a `receive` function is present, `receive` is invoked.
    
2.  Otherwise, the `fallback` function is called.
    

This behavior is modeled in CVL using the synthetic function `receiveOrFallback`, which may appear in parametric rules, invariants, or call traces as `<receiveOrFallback>()`.

For more details, see the [Solidity Documentation](https://docs.soliditylang.org/en/latest/contracts.html#fallback-function) on fallback functions.

rule name pattern[](#term-rule-name-pattern "Link to this term")

Rule names, like all CVL identifiers, have the same format as Solidity identifiers: they consist of a combination of letters, digits, dollar signs, and underscores, but cannot start with a digit (see [here](https://docs.soliditylang.org/en/v0.8.16/path-resolution.html#allowed-paths)). When used in client options (like [rule](../prover/cli/options.html#rule)), rule name patterns can also include the wildcard `*` that can replace any sequence of valid identifier characters. For example, the rule pattern `withdraw_*` can be used instead of listing all rules that start with the string `withdraw_`. This wildcard functionality is part of the client interface and does not apply within CVL spec files.

sanity[](#term-sanity "Link to this term")

SAT[](#term-SAT "Link to this term")

UNSAT[](#term-UNSAT "Link to this term")

SAT result[](#term-SAT-result "Link to this term")

UNSAT result[](#term-UNSAT-result "Link to this term")

`SAT` and `UNSAT` are the two possible results returned by an [SMT solver](#term-SMT-solver) if it does not time out.

*   `SAT` (satisfiable) means that the input formula can be satisfied, and a corresponding [model](#term-model) has been found.
    
*   `UNSAT` (unsatisfiable) means that no such [model](#term-model) exists, as the formula cannot be satisfied.
    

In the context of the Certora Prover, the interpretation of `SAT` depends on the type of rule being checked:

*   For an `assert` rule, `SAT` means the rule is violated; the [model](#term-model) returned serves as a counterexample.
    
*   For a `satisfy` rule, `SAT` means the rule is fulfilled; the [model](#term-model) is a witness example.
    

Conversely, `UNSAT` indicates:

*   An `assert` rule is never violated.
    
*   A `satisfy` rule is never fulfilled.
    

See also the [Overview](../cvl/rules.html#rule-overview) for more background.

scene[](#term-scene "Link to this term")

The set of contract instances that the Certora Prover knows about.

SMT[](#term-SMT "Link to this term")

SMT solver[](#term-SMT-solver "Link to this term")

SMT stands for Satisfiability Modulo Theories. An SMT solver takes as input a formula written in predicate logic and determines whether it is satisfiable ([SAT](#term-SAT)) or unsatisfiable ([UNSAT](#term-UNSAT)).

The “Modulo Theories” part refers to the solver’s ability to reason about specific background theories, such as integer arithmetic, arrays, or bitvectors. For example, under the theory of integer arithmetic, symbols like `+`, `-`, and `*` are interpreted according to their standard mathematical meaning.

When a formula is satisfiable, the solver may also return a [model](#term-model): an assignment of values to variables that makes the formula evaluate to `true`. For instance, given the formula `x > 5 ∧ x = y * y`, the solver might return `x = 9, y = 3` as a valid model.

For more background, see [Wikipedia](https://en.wikipedia.org/wiki/Satisfiability_modulo_theories).

sound[](#term-sound "Link to this term")

unsound[](#term-unsound "Link to this term")

Soundness means that the Certora Prover is guaranteed to report any rule violations in the code being verified. Unsound approximations, such as loop unrolling or certain kinds of harnessing, may cause the Prover to miss real bugs, and should, therefore, be used with caution. See [Prover Approximations](../prover/approx/index.html) for more details.

split[](#term-split "Link to this term")

split leaf[](#term-split-leaf "Link to this term")

split leaves[](#term-split-leaves "Link to this term")

Control flow splitting is a technique to speed up verification by splitting the program into smaller parts and verifying them separately. These smaller programs are called splits. Splits that cannot be split further are called split leaves. See [Control flow splitting](../prover/techniques/index.html#control-flow-splitting).

summary[](#term-summary "Link to this term")

summarize[](#term-summarize "Link to this term")

summarization[](#term-summarization "Link to this term")

Summaries[](#term-Summaries "Link to this term")

A method summary is a user-provided approximation of the behavior of a contract method.  
Summaries are useful if the implementation of a method is not available or if the implementation is too complex for the Certora Prover to analyze without timing out.  
See [Summaries](../cvl/methods.html#summaries) for complete information on different types of method summaries.

TAC[](#term-TAC "Link to this term")

TAC (originally short for “three address code”) is an intermediate representation ([Wikipedia](https://en.wikipedia.org/wiki/Intermediate_representation)) used by the Certora Prover. TAC code is kept invisible to the user most of the time, so its details are not included in the scope of this documentation. In the [TAC Reports](../prover/diagnosis/index.html#tac-reports) section We provide a working understanding, which is helpful for some advanced proving tasks.

tautology[](#term-tautology "Link to this term")

A tautology is a logical statement that is always true.

vacuous[](#term-vacuous "Link to this term")

vacuity[](#term-vacuity "Link to this term")

A logical statement is _vacuous_ if it is technically true but only because it doesn’t say anything. For example, “every integer that is both greater than 5 and less than 3 is a perfect square” is technically true, but only because there are no numbers that are both greater than 5 and less than 3.

Similarly, a rule or assertion can pass, but only because the `require` statements rule out all of the [model](#term-model)s. In this case, the rule doesn’t say anything about the program being verified. The [Rule Sanity Checks](../prover/checking/sanity.html) help detect vacuous rules.

verification condition[](#term-verification-condition "Link to this term")

The Certora Prover translates a program and a specification into a single logical formula that is satisfiable if and only if the program violates the specification. This formula is called a _verification condition_. Usually, a run of the Certora Prover generates many verification conditions. For instance, the Prover generates a verification condition for every [parametric rule](#term-parametric-rule) and sanity checks triggered by [rule\_sanity](../prover/cli/options.html#rule-sanity). See also the [Certora Technology White Paper](../whitepaper/index.html#white-paper) and the [Certora User’s Guide](index.html#user-guide).

wildcard[](#term-wildcard "Link to this term")

exact[](#term-exact "Link to this term")

A methods block entry that explicitly uses `_` as a receiver is a _wildcard entry_; all other entries are called _exact entries_. See [The Methods Block](../cvl/methods.html).
