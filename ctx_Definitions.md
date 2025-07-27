# Definitions — Certora Prover Documentation 0.0 documentation

# Definitions[](#definitions "Link to this heading")

## Basic Usage[](#basic-usage "Link to this heading")

In CVL, **definitions** serve as type-checked macros, encapsulating commonly used expressions. They are declared at the top level of a specification and are in scope inside every rule, function, and other definitions. The basic usage involves binding parameters for use in an expression on the right-hand side, with the result evaluating to the declared return type. Definitions can take any number of arguments of any primitive types, including uninterpreted sorts, and evaluate to a single primitive type, including uninterpreted sorts.

### Example:[](#example "Link to this heading")

`is_even` binds the variable `x` as a `uint256`. Definitions are applied just as any function would be.

methods {
  foo(uint256) returns bool envfree
}

definition MAX\_UINT256() returns uint256 \= 0xffffffffffffffffffffffffffffffff;
definition is\_even(uint256 x) returns bool \= exists uint256 y . 2 \* y \== x;

rule my\_rule(uint256 x) {
  require is\_even(x) && x <= MAX\_UINT256();
  foo@withrevert(x);
  assert !lastReverted;
}

## Advanced Functionality[](#advanced-functionality "Link to this heading")

### Include an Application of Another Definition[](#include-an-application-of-another-definition "Link to this heading")

Definitions can include an application of another definition, allowing for arbitrarily deep nesting. However, circular dependencies are not allowed, and the type checker will report an error if detected.

#### Example:[](#id1 "Link to this heading")

`is_odd` and `is_odd_no_overflow` both reference other definitions:

definition MAX\_UINT256() returns uint256 \= 0xffffffffffffffffffffffffffffffff;
definition is\_even(uint256 x) returns bool \= exists uint256 y . 2 \* y \== x;
definition is\_odd(uint256 x) returns bool \= !is\_even(x);
definition is\_odd\_no\_overflow(uint256 x) returns bool \=
    is\_odd(x) && x <= MAX\_UINT256();

### Type Error circular dependency[](#type-error-circular-dependency "Link to this heading")

The following examples would result in a type error due to a circular dependency:

// example 1
// cycle: is\_even -> is\_odd -> is\_even
definition is\_even(uint256 x) returns bool \= !is\_odd(x);
definition is\_odd(uint256 x) returns bool \= !is\_even(x);

// example 2
// cycle: circular1->circular2->circular3->circular1
definition circular1(uint x) returns uint \= circular2(x) + 5;
definition circular2(uint x) returns uint \= circular3(x \- 2) + 7;
definition circular3(uint x) returns uint \= circular1(x) + circular1(x);

### Reference Ghost Functions[](#reference-ghost-functions "Link to this heading")

Definitions may reference ghost functions. This means that definitions are not always “pure” and can affect ghosts, which are considered a “global” construct.

#### Example:[](#id2 "Link to this heading")

ghost foo(uint256) returns uint256;

definition is\_even(uint256 x) returns bool \= x % 2 \== 0;
definition foo\_is\_even\_at(uint256 x) returns bool \= is\_even(foo(x));

rule rule\_assuming\_foo\_is\_even\_at(uint256 x) {
  require foo\_is\_even\_at(x);
  // ...
}

More interestingly, the two-context version of ghosts can be used in a definition by adding the `@new` or `@old` annotations. If a two-context version is used, the ghost must not be used without an `@new` or `@old` annotation, and the definition must be used in a two-state context for that ghost function (e.g., at the right side of a `havoc assuming` statement for that ghost).

#### Example:[](#id3 "Link to this heading")

ghost foo(uint256) returns uint256;

definition is\_even(uint256 x) returns bool \= x % 2 \== 0;
definition foo\_add\_even(uint256 x) returns bool \= is\_even(foo@new(x)) &&
    forall uint256 a. is\_even(foo@old(x)) \=> is\_even(foo@new(x));

rule rule\_assuming\_old\_evens(uint256 x) {
  // havoc foo, assuming all old even entries are still even, and that
  // the entry at x is also even
  havoc foo assuming foo\_add\_even(x);
  // ...
}

**Note:** The type checker will notify you if a two-state version of a variable is used incorrectly.

## Filter Example[](#filter-example "Link to this heading")

The following example introduces a definition called `filterDef`:

definition filterDef(method f) returns bool \= f.selector \== sig:someUInt().selector;

This definition serves as shorthand for `f.selector == sig:someUInt().selector` and is used in the filter for the `parametricRule`:

rule parametricRuleInBase(method f) filtered { f \-> filterDef(f)  }
{
  // ...
}

This is equivalent to:

rule parametricRuleInBase(method f) filtered { f \-> f.selector \== sig:someUInt().selector  } {
  // ...
}

## Syntax[](#syntax "Link to this heading")

The syntax for definitions is given by the following EBNF grammar:

definition ::= \[ "override" \]
               "definition" id \[ "(" params ")" \]
               "returns" cvl\_type
               "=" expression ";"

See [Types](types.html), [Expressions](expr.html), and [Identifiers](basics.html#identifiers) for descriptions of the `cvl_type`, `expression`, and `id` productions, respectively.

In this syntax, the `definition` keyword is followed by the definition’s identifier (`id`). Parameters can be specified in parentheses, and the return type is declared using the `returns` keyword. The body of the definition is provided after the equal sign (`=`) and should end with a semicolon (`;`).
