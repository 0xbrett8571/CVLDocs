# Glossary — Certora Prover Documentation 0.0 documentation
axiom

a statement accepted as true without proof.

call trace

A call trace is the Prover’s visualization of either a counterexample or a witness example.

A call trace illustrates a rule execution that leads up to the violation of an `assert` statement or the fulfillment of a `satisfy` statement. The trace is a sequence of commands in the rule (or in the contracts the rule was calling into), starting at the beginning of the rule and ending with the violated `assert` or fulfilled `satisfy` statement. In addition to the commands, the call trace also does a best effort at showing information about the program state at each point in the execution. It contains information about the state of global variables at crucial points as well as the values of call parameters, return values, and more.

If a call trace exists, it can be found in the “Call Trace” tab in the report after selecting the corresponding (sub-)rule.

CFG

control flow graph

control flow path

Control flow graphs (short: CFGs) are a program representation that illustrates in which order the program’s instructions are processed during program execution. The nodes in a control flow graph represent single non-branching sequences of commands. The edges in a control flow graph represent the possibility of control passing from the last command of the source node to the first command of the target node. For instance, an `if`\-statement in the program will lead to a branching, i.e., a node with two outgoing edges, in the control flow graph. A CVL rule can be seen as a program with some extra “assert” commands, thus a rule has a CFG like regular programs. The Certora Prover’s TAC reports contain a control flow graph of the TAC intermediate representation of each given CVL rule. The control flow paths are the paths from source to sink in a given CFG. In general (and in practice) the number of control flow paths grows exponentially with the size of the CFG. This is known as the path explosion problem. Further reading: Wikipedia: Control-flow graph Wikipedia: Path explosion problem

environment

The environment of a method call refers to the global variables that solidity provides, including `msg`, `block`, and `tx`. CVL represents these variables in a structure of type env. The environment does _not_ include the contract state or the state of other contracts — these are referred to as the storage.

EVM

Ethereum Virtual Machine

EVM bytecode

EVM is short for Ethereum Virtual Machine. EVM bytecode is one of the source languages that the Certora Prover internally can take as input for verification. It is produced by the Solidity and Vyper compilers, among others. For details on what the EVM is and how it works, the following links provide good entry points. Official documentation, Wikipedia

EVM memory

EVM storage

The EVM has two major concepts of memory, called _memory_ and _storage_. In brief, memory variables keep data only for the duration of a single EVM transaction, while storage variables are stored persistently in the Ethereum blockchain. Official documentation

havoc

Havoc means that variables are assigned values chosen non-deterministically. A havoc happens in two cases: the first, at the beginning of the rule all variables “havoced”. The second, during certain events when the Certora Prover should assume that some variables can change in an unknown way. For example, an external function on an unknown contract may have an arbitrary effect on the state of a third contract. In this case, we also say that the variable was “havoced”. See Havoc summaries: HAVOC\_ALL and HAVOC\_ECF and Havoc Statements for more details.

hyperproperty

A hyperproperty describes a relationship between two hypothetical sequences of operations starting from the same initial state. For example, a statement like “two small deposits will have the same effect as one large deposit” is a hyperproperty. See The storage type for more details.

invariant

An invariant (or representation invariant) is a property of the contract state that is expected to hold between invocations of contract methods. See Invariants.

model

example

counterexample

witness example

We use the terms “model” and “example” interchangeably. In the context of a CVL rule, they refer to an assignment of values to all of the CVL variables and contract storage that either violates an `assert` statement or fulfills a `satisfy` statement. In the `assert` case, we also call the model a “counterexample”. In the `satisfy` case, we also call the model “witness example”. See Overview. In the context of SMT solvers, a model is a valuation of the logical constants and uninterpreted functions in the input formula that makes the formula evaluate to `true`, also see SAT result.

linear arithmetic

nonlinear arithmetic

An arithmetic expression is called linear if it consists only of additions, subtractions, and multiplications by constant. Division and modulo where the second parameter is a constant are also linear arithmetic. Examples for linear expressions are `x * 3`, `x / 3`, `5 * (x + 3 * y)`. Every arithmetic expression that is not linear is nonlinear. Examples for nonlinear expressions are `x * y`, `x * (1 + y)`, `x * x`, `3 / x`, `3 ^ x`.

overapproximation

underapproximation

Sometimes it is useful to replace a complex piece of code with something simpler that is easier to reason about. If the approximation includes all of the possible behaviors of the original code (and possibly others), it is called an “overapproximation”; if it does not then it is called an “underapproximation”.

Example: A NONDET summary is an overapproximation because every possible value that the original implementation could return is considered by the Certora Prover, while an ALWAYS summary is an underapproximation if the summarized method could return more than one value.

Proofs on overapproximated programs are sound, but there may be spurious counterexamples caused by behavior that the original code did not exhibit. Underapproximations are more dangerous because a property that is successfully verified on the underapproximation may not hold on the approximated code.

optimistic assumptions

pessimistic assertions

Some input programs contain constructs that the Prover can only handle in an approximative way. This approximation entails that the Prover will disregard some specific parts of the programs behavior, like for example the behavior induced by a loop being unrolled beyond a fixed number of times. For each of these constructs the Prover provides a flag controlling whether it should handle them optimistically or pessimistically. (See the links at the end of this paragraph for examples of “optimistic” flags.)

In pessimistic mode (which is the default) _pessimistic assertions_ are inserted into the program that check whether there is any behavior that needs to be approximated, for instance whether loops are present with bounds exceeding loop\_iter. If this is the case, the rule will fail with a corresponding message.

In optimistic mode, instead of the assertions, _optimistic assumptions_ are introduced in each of the places where an approximation happens. Each assumption excludes the relevant behavior from checking for one occurrence of the problematic construct, e.g., for each loop.

For a list of all “optimistic” settings see CLI Options. Examples include optimistic\_hashing, optimistic\_loop, optimistic\_summary\_recursion, and more. Also see Prover Approximations for more background on some of the approximations.

parametric rule

A parametric rule is a rule that calls an ambiguous method, either using a method variable, or using an overloaded function name. The Certora Prover will generate a separate report for each possible instantiation of the method. See Parametric rules for more information.

quantifier

quantified expression

The symbols `forall` and `exist` are sometimes referred to as _quantifiers_, and expressions of the form `forall type v . e` and `exist type v . e` are referred to as _quantified expressions_. See Extended logical operations for details about quantifiers in CVL.

receiveOrFallback

A special function we introduce in every contract to model the behavior of solidity for calls with no data or that do not resolve to any contract function. It will call the receive function if present for calls with no data, and otherwise the fallback function. Shows up in the parametric rules or invariants, as well as in the call trace for such calls, written `<receiveOrFallback>()`. See also Solidity Documentation.

sanity

SAT

UNSAT

SAT result

UNSAT result

_SAT_ and _UNSAT_ are the results that an SMT solver returns on a successful run (i.e. not a timeout). SAT means that the input formula is satisfiable and a model has been found. UNSAT means that the input formula is unsatisfiable (and thus there is no model for it). Within the Certora Prover, what SAT means depends on the type of rule being checked: For an `assert` rule, SAT means the rule is violated and the SMT model corresponds to a counterexample. For a `satisfy` rule, SAT means the rule is not violated and the SMT model corresponds to a witness example. Conversely, UNSAT means that an `assert` is never violated or a `satisfy` never fulfilled respectively. See also Overview.

(scene)=

scene

The _scene_ refers to the set of contract instances that the Certora Prover knows about.

SMT

SMT solver

“SMT” is short for “Satisfiability Modulo Theories”. An SMT solver takes as input a formula in predicate logic and returns whether the formula is satisfiable (short “SAT”) or unsatisfiable (short: “UNSAT”). The “Modulo Theory” part means that the solver assumes a meaning for certain symbols in the formula. For instance the theory of integer arithmetic stipulates that the symbols `+`, `-`, `*`, etc. have their regular everyday mathematical meaning. When the formula is satisfiable, the SMT solver can also return a model for the formula. I.e. an assignment of the formula’s variables that makes the formula evaluate to “true”. For instance, on the formula “x > 5 /\\ x = y \* y”, a solver will return SAT, and produce any valuation where x is the square of an integer and larger than 5, and y is the root of x. Further reading: Wikipedia

sound

unsound

Soundness means that any rule violations in the code being verified are guaranteed to be reported by the Certora Prover. Unsound approximations such as loop unrolling or certain kinds of harnessing may cause real bugs to be missed by the Prover, and should therefore be used with caution. See Prover Approximations for more details.

split

split leaf

split leaves

Control flow splitting is a technique to speed up verification by splitting the program into smaller parts and verifying them separately. These smaller programs are called splits. Splits that cannot be split further are called split leaves. See Control flow splitting.

summary

summarize

A method summary is a user-provided approximation of the behavior of a contract method. Summaries are useful if the implementation of a method is not available or if the implementation is too complex for the Certora Prover to analyze without timing out. See The Methods Block for complete information on different types of method summaries.

TAC

TAC (originally short for “three address code”) is an intermediate representation (Wikipedia) used by the Certora Prover. TAC code is kept invisible to the user most of the time, so its details are not in the scope of this documentation. We provide a working understanding, which is helpful for some advanced proving tasks, in the TAC Reports section.

tautology

A tautology is a logical statement that is always true.

vacuous

vacuity

A logical statement is _vacuous_ if it is technically true but only because it doesn’t say anything. For example, “every integer that is both greater than 5 and less than 3 is a perfect square” is technically true, but only because there are no numbers that are both greater than 5 and less than 3.

Similarly, a rule or assertion can pass, but only because the `require` statements rule out all of the models. In this case, the rule doesn’t say anything about the program being verified. The Rule Sanity Checks help detect vacuous rules.

verification condition

The Certora Prover works by translating a program an a specification into a single logical formula that is satisfiable if and only if the program violates the specification. This formula is called a _verification condition_. Usually, a run of the Certora Prover generates many verification conditions. For instance a verification condition is generated for every parametric rule, and also for each of the sanity checks triggered by rule\_sanity. See also Certora Technology White Paper, Certora User’s Guide.

wildcard

exact

A methods block entry that explicitly uses `_` as a receiver is a _wildcard entry_; all other entries are called _exact entries_. See The Methods Block.
