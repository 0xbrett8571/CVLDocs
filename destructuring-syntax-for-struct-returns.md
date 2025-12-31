# Changes Introduced in CVL 2 — Certora Prover Documentation 0.0 documentation

# [Changes Introduced in CVL 2](#id2)[](#changes-introduced-in-cvl-2 "Link to this heading")

CVL 2 is a major overhaul to the type system of CVL. Though many of the changes are internal, we wanted to take this opportunity to introduce a few improvements to the syntax. The general goal of these changes is to make the behavior of CVL more explicit and predictable, and to bring the syntax more in line with Solidity’s syntax.

This document summarizes the changes to CVL syntax introduced by CVL 2.

The `CVLMigration` repository contains examples demonstrating each of the changes; the `cvl1` branch contains the examples in valid CVL 1 syntax, while the `cvl2` branch contains the same examples in CVL 2 syntax. You can see the differences [here](https://github.com/Certora/CVL2Migration/compare/cvl1..cvl2?diff=split), our you can clone [the repository](https://github.com/Certora/CVL2Migration) and compare the `cvl1` and `cvl2` branches using your favorite tools.

## [Superficial syntax changes](#id3)[](#superficial-syntax-changes "Link to this heading")

There are several simple changes to the syntax to make specs more uniform and consistent, and to reduce the superficial differences with Solidity.

### [`function` and `;` required for methods block entries](#id4)[](#function-and-required-for-methods-block-entries "Link to this heading")

In CVL 2, methods block entries must now start with `function` and end with `;` (semicolons were optional in CVL 1). For example ([CVL 1](https://github.com/Certora/CVL2Migration/blob/cvl1/certora/spec/MethodsEntries.spec), [CVL 2](https://github.com/Certora/CVL2Migration/blob/cvl2/certora/spec/MethodsEntries.spec), [diff](https://github.com/Certora/CVL2Migration/compare/cvl1..cvl2?diff=split#diff-9cd1ae6f2c8146e323568cb25c79d4f6671fcb690872dce33591bd514759fc24)):

transferFrom(address, address, uint) returns(bool) envfree

will become

function transferFrom(address, address, uint) external returns(bool) envfree;

(note also the addition of `external`, [described below](#cvl2-visibility)).

This is also true for entries with summaries:

balanceOf(address) returns(uint256) \=> ALWAYS(3)

will become

function balanceOf(address) external returns(uint256) \=> ALWAYS(3);

If you do not change this, you will get an error message like the following:

CRITICAL: \[main\] ERROR ALWAYS \- certora/spec/MethodsEntries.spec:4:5: Syntax error: unexpected token near ID(transferFrom)
CRITICAL: \[main\] ERROR ALWAYS \- certora/spec/MethodsEntries.spec:4:5: Couldn't repair and continue parse unexpected token near ID(transferFrom)

### [Required `;` in more places](#id5)[](#required-in-more-places "Link to this heading")

`using`, `import`, `use`, and `invariant` statements all require a `;` at the end. For example ([CVL 1](https://github.com/Certora/CVL2Migration/blob/cvl1/certora/spec/Semicolons.spec), [CVL 2](https://github.com/Certora/CVL2Migration/blob/cvl2/certora/spec/Semicolons.spec), [diff](https://github.com/Certora/CVL2Migration/compare/cvl1..cvl2?diff=split#diff-15fb1ef5e6524f8a661d83ae5160b6b072840c5c54bf8d07733aab32b9da73f7)):

invariant balanceOfZeroIsZero()
    balanceOf(0) \== 0

becomes

invariant balanceOfZeroIsZero()
    balanceOf(0) \== 0;

`use` and `invariant` statements do not require (and may not have) a semicolon if they are followed by a `preserved` or `filtered` block. For example, the following is valid in both CVL 1 and CVL 2:

invariant totalSupplyBoundsBalance(address a)
    balanceOf(a) <= totalSupply()
    { preserved { require false; } }

If you do not change this, you will see an error like the following:

CRITICAL: \[main\] ERROR ALWAYS \- certora/spec/Semicolons.spec:5:1: Syntax error: unexpected token near using
CRITICAL: \[main\] ERROR ALWAYS \- certora/spec/Semicolons.spec:5:1: Couldn't repair and continue parse unexpected token near using

### [Method literals require `sig:`](#id6)[](#method-literals-require-sig "Link to this heading")

In some places in CVL, you can refer to a contract method by its name and argument types. For example, you might write ([CVL 1](https://github.com/Certora/CVL2Migration/blob/cvl1/certora/spec/MethodLiterals.spec), [CVL 2](https://github.com/Certora/CVL2Migration/blob/cvl2/certora/spec/MethodLiterals.spec), [diff](https://github.com/Certora/CVL2Migration/compare/cvl1..cvl2?diff=split#diff-41df8240fa5faa12531baa82863891b94abf3fb3b859bdd10bafde73b60eda5d)):

f.selector \== approve(address, uint).selector

In this example, `approve(address,uint)` is a _method literal_. In CVL 2, these methods literals must now start with `sig:`. For example, the above would become:

f.selector \== sig:approve(address, uint).selector

If you do not change this, you will see the following error:

Error: Error in spec file (MethodLiterals.spec:14:5): Variable address is undefined (first instance only reported)
Error: Error in spec file (MethodLiterals.spec:14:5): Variable uint is undefined (first instance only reported)
Error: Error in spec file (MethodLiterals.spec:15:34): could not type expression "address", message: unknown variable "address"
Error: Error in spec file (MethodLiterals.spec:15:43): could not type expression "uint", message: unknown variable "uint"

### [Use of contract name instead of `using` variable](#id7)[](#use-of-contract-name-instead-of-using-variable "Link to this heading")

In CVL 1, the only way to refer to a contract in the [scene](../../user-guide/glossary.html#term-scene) was to first introduce a contract instance variable with a `using` statement, and then use that variable. For example, to access a struct type `S` defined in `PrimaryContract.sol`, you would need to write ([CVL 1](https://github.com/Certora/CVL2Migration/blob/cvl1/certora/spec/ContractNames.spec), [CVL 2](https://github.com/Certora/CVL2Migration/blob/cvl2/certora/spec/ContractNames.spec), [diff](https://github.com/Certora/CVL2Migration/compare/cvl1..cvl2?diff=split#diff-a6a2974b81074e87d755753c2e84ef1b0cb553bfdeb729827959e0c63f0d02d7)):

using PrimaryContract as primary;

rule structExample {
    primary.S x;
    ...
}

In CVL 2, you must now use the name of the contract, rather than the instance variable, when referring to user-defined types. The above example would now be written

rule structExample {
    PrimaryContract.S x;
    ...
}

There is no need for a `using` statement in this example.

If you don’t change this, you will an error like the following:

Error: Error in spec file (ContractNames.spec:12:19): Contract name primary does not exist in the scene. Make sure you are using a contract name and not a contract instance name.

Calling methods on secondary contracts still requires using a contract instance variable:

using SecondaryContract as secondary;

rule multicontractExample {
    ...
    secondary.balanceOf(0);
    ...
}

Entries in the `methods` block may use either the contract name or an instance variable:

using SecondaryContract as secondary;

methods {
    //// both are valid (and the effect is the same):
    secondary.balanceOf(address) returns(uint) envfree
    SecondaryContract.transfer(address, uint) returns(bool) envfree
}

Using the contract name in the methods block currently has the same effect as using an instance variable; this may change in future versions of CVL.

### [Rules must start with `rule`](#id8)[](#rules-must-start-with-rule "Link to this heading")

In CVL 1, you could omit the keyword `rule` when writing rules ([CVL 1](https://github.com/Certora/CVL2Migration/blob/cvl1/certora/spec/RuleKeyword.spec), [CVL 2](https://github.com/Certora/CVL2Migration/blob/cvl2/certora/spec/RuleKeyword.spec), [diff](https://github.com/Certora/CVL2Migration/compare/cvl1..cvl2?diff=split#diff-b39a57bffd39f86bc1f9555a487af389f501eab9e66a1c3059a89691319da248)):

transferReverts {
    ...
}

In CVL 2, the `rule` keyword is no longer optional:

rule transferReverts {
    ...
}

If you don’t change this, you will receive an error like the following:

CRITICAL: \[main\] ERROR ALWAYS \- certora/spec/RuleKeyword.spec:3:1: Syntax error: unexpected token near ID(transferReverts)

## [Changes to methods block entries](#id9)[](#changes-to-methods-block-entries "Link to this heading")

In addition to the superficial changes listed above, there are some changes to the way that methods block entries can be written (there are also a [few instances](#cvl2-wildcards) where the meanings of entries has changed). In CVL 1, `methods` block entries often had several different functions and meanings:

*   They were used to indicate targets for summarization
    
*   They were used to write generic specs that could apply to contracts with missing methods
    
*   They were used to declare targets `envfree`
    

The changes described in this section make these different uses more explicit:

### [Most Solidity types allowed as arguments](#id41)[](#most-solidity-types-allowed-as-arguments "Link to this heading")

CVL 1 had some restrictions on the types of arguments allowed in `methods` block entries. For example, user-defined types (such as enums and structs) were not fully supported.

CVL 2 `methods` block entries may use any Solidity types for arguments and return values, except for [function types](https://docs.soliditylang.org/en/v0.8.17/types.html#function-types) and contract or interface types.

To work around the missing types, CVL 1 allowed users to encode some user-defined types as primitive types in the `methods` block; these workarounds are no longer allowed in CVL 2. For example, consider the following [solidity function](https://TODO/):

contract Example {
    enum Permission { READ, WRITE };

    function f(Permission p) internal { ... }
}

In CVL 1, a methods block entry for `f` would need to declare that it takes a `uint8` argument:

methods {
    f(uint8 permission) \=> NONDET
}

In CVL 2, the methods block entry should use the same type as the Solidity implementations\[^contract-types\] ([compare files](https://github.com/Certora/CVL2Migration/compare/cvl1..cvl2?diff=split#diff-5b1b684b999817bab176753b548b9ca548c8e9a1b7ce72d355030a8e03f498d8)), except for function types and contract or interface types:

methods {
    function f(Example.Permission p) internal \=> NONDET;
}

The method can be called from CVL as follows:

rule example {
    f(Example.Permission.READ);
}

Contract functions that take or return contract or interface types should instead use `address` in the `methods` block declaration. For example, if the contract contains the following function:

function listToken(IERC20 token) internal { ... }

the `methods` block should use `address` for the `token` argument:

methods {
    function listToken(address token) internal;
}

Contract functions that take or return function types are not currently supported.

### [Required `internal` or `external` annotation](#id42)[](#required-internal-or-external-annotation "Link to this heading")

Every methods block entry must be marked either `internal` or `external`. The annotation must come after the argument list and before the `returns` clause.

If a function is declared `public` in Solidity, then the Solidity compiler creates an internal implementation method, and an external wrapper method that calls the internal implementation. Therefore, you can summarize a `public` method by marking the summarization `internal`.

Warning

The behavior of `internal` vs. `external` summarization for public methods can be confusing, especially because functions called directly from CVL are not summarized. See [Visibility modifiers](../methods.html#methods-visibility).

### [`optional` methods block entries](#id43)[](#optional-methods-block-entries "Link to this heading")

In CVL 1, you could write an entry in the methods block for a method that does not exist in the contract; rules that would call the non-existent method were skipped during verification.

This behavior can lead to confusion, because typos or name changes could silently cause a rule to be skipped.

In CVL 2, this behavior is still available, but the methods entry must contain the keyword `optional` somewhere after the `returns` clause and before the summarization (if any).

### [Required `calldata`, `memory`, or `storage` annotations for reference types](#id44)[](#required-calldata-memory-or-storage-annotations-for-reference-types "Link to this heading")

In CVL 2, methods block entries for internal functions must contain either `calldata`, `memory`, or `storage` annotations for all arguments with reference types (such as arrays).

For methods block entries of external functions the location annotation must be omitted unless it’s the `storage` annotation on an external library function, in which case it is required (the reasoning here is to have the information required in order to correctly calculate a function’s sighash).

### [Summaries only apply to one contract by default](#id45)[](#summaries-only-apply-to-one-contract-by-default "Link to this heading")

In CVL 1, a summary in the `methods` block applied to all methods with the given signature.

In CVL 2, summaries only apply to a single contract, unless the old behavior is explicitly requested by using `_` as the receiver. If no contract is specified, the default is `currentContract`.

Note

The receiver contract must be the contract where the method is defined. If a contract inherits a method defined in a supercontract, the receiver must be the supercontract, rather than the inheriting contract.

Entries that use `_` as the receiver are called [wildcard entries](../../user-guide/glossary.html#term-wildcard), summaries that do not are called [exact entries](../../user-guide/glossary.html#term-exact).

Consider the following example:

using C as c;

methods {
    function f(uint)   internal \=> NONDET;
    function c.g(uint) internal \=> ALWAYS(4);
    function h(uint)   internal \=> ALWAYS(1);
    function \_.h(uint) internal \=> NONDET;
}

In this example, the internal function `currentContract.f` has a `NONDET` summary, `c.g` has an `ALWAYS` summary, a call to `currentContact.h` has an `ALWAYS` summary and a call to `h(uint)` on any other contract will use a `NONDET` summary.

Summaries for specific contract methods (including the default `currentContract`) always override wildcard summaries.

Wildcard entries cannot be declared `optional` or `envfree`, since these annotations only make sense for specific contract methods.

Warning

The meaning of your summarizations will change from CVL 1 to CVL 2. In CVL 2, any entry without an `_` will only apply to a single contract!

### [Requirements on `returns`](#id46)[](#requirements-on-returns "Link to this heading")

In CVL 1, the `returns` clause on methods block entries was optional. CVL 2 has stricter requirements on the declared return types.

Entries that apply to specific contracts (i.e. those that don’t use the `_.f` [syntax](#cvl2-wildcards)) must include a `returns` clause if the contract method returns a value. A specific-contract entry may only omit the `returns` clause if the contract method does not return a value.

The Prover will report an error if the contract method’s return type differs from the type declared in the `methods` block entry.

Wildcard entries must not declare return types, because they may apply to multiple methods that return different types. If a wildcard entry is summarized with a ghost or function summary, the summary must include an `expect` clause; see [Expression summaries](../methods.html#expression-summary) for more details.

## [Changes to integer types](#id16)[](#changes-to-integer-types "Link to this heading")

In CVL 1, the rules for casting between integer types were complex; CVL 2 simplifies them.

The general rule of thumb is that you should use `mathint` whenever possible; only use `uint` or `int` types for data that will be passed as input to contract functions.

It is now impossible for CVL math expressions to cause overflow - all integer operations are exact. The remainder of this section describes the changes in detail.

### [Mathematical operations return `mathint`](#id17)[](#mathematical-operations-return-mathint "Link to this heading")

In CVL 2, the results of all arithmetic operators have type `mathint`, regardless of the input types. Arithmetic operators include `+`, `*`, `-`, `/`, `^`, and `%`, but not bitwise operators like `<<`, `xor`, and `|` (changes to bitwise operators are described [below](#cvl2-bitwise)).

The primary impact of this change is that you may need to declare more of your variables as `mathint` instead of `uint`. If you are passing the results of arithmetic operations to contract functions, you will need to be more explicit about the overflow behavior by using the [new casting operators](#cvl2-casting).

### [Implicit and explicit casting](#id18)[](#implicit-and-explicit-casting "Link to this heading")

If every number that can be represented by one type can also be represented by another type, then we say that the first type is a _subtype_ of the second type.

For example, a `uint8` variable could have any value between `0` and `2^8-1`, and all of these values can be stored in a `uint16` variable, so `uint8` is a subtype of `uint16`. An `int16` can also store any value between `0` and `2^8-1`, so `uint8` is also a subtype of `int16`.

All integer types are subtypes of `mathint`, since any integer can be represented by a `mathint`.

In CVL 1, the rules for converting between supertypes and subtypes were complicated; they depended not only on the types involved, but on the context in which the conversion happened. CVL 2 simplifies these rules and improves the clarity and predictability of casts.

In CVL 2, you can always use a subtype whenever the supertype is accepted. For example, you can always use a `uint8` where an `int16` is expected. We say that the subtype can be “implicitly cast” to the supertype.

In order to convert from a supertype to a subtype, you must use an explicit cast. In CVL 1, only a few casting operators (such as `to_uint256`) were supported.

CVL 2 replaces these casting operators with two new casting operators: _assert casts_ such as `assert_uint8(x)` or `assert_int256(x)`, and _require casts_ such as `require_uint8(x)` or `require_int256(x)`. Each of these casts checks that the value is in range; the `assert` cast will report a counterexample if the value is out of range, while the `require` cast will ignore counterexamples where the cast value is out of range.

Warning

As with normal `require` statements, require casts can cause vacuity and should be used with care.

CVL 2 supports assert and require casts on all numeric types.

Casts from `address` or `bytes1`…`bytes32` to integer types are not supported (see [Support for bytes1…bytes32](#bytesn-support) regarding casting in the other direction, and [Casting enums to integer types](#enum-casting) for information on casting enums), except for going from `bytes32` to the equivalent `uint256`.

`require` and `assert` casts are not allowed anywhere inside of a [quantified statement](../../user-guide/glossary.html#term-quantifier). You can work around this limitation by adding a second variable. For example, the following axiom is invalid because `x+1` is not a `uint`:

ghost mapping(uint \=> uint) a {
    axiom forall uint x . a\[x+1\] \== 0
}

However, it can be replaced with the following:

ghost mapping(uint \=> uint) a {
    axiom forall uint x . forall uint y . (to\_mathint(y) \== x + 1) \=> a\[y\] \== 0
}

### [Casting enums to integer types](#id19)[](#casting-enums-to-integer-types "Link to this heading")

In CVL2 enums are not directly comparable to the corresponding integer type (`uint8`). Instead one must use one of the new cast operators. For example

uint8 x \= MyContract.MyEnum.VAL; // will fail typechecking
uint8 x \= assert\_uint8(MyContract.MyEnum.VAL); // good
mathint x \= to\_mathint(MyContract.MyEnum.VAL); // good

Casting integer types to an enum is not supported.

### [Casting addresses to bytes32](#id20)[](#casting-addresses-to-bytes32 "Link to this heading")

CVL2 supports casting from the `address` type to the `bytes32` type. For example:

address a \= 0xa44f5d3d624DfD660ecc11FF777587AD0a19606d;
bytes32 b \= to\_bytes32(a);

The cast from `address` to `bytes32` behaves equivalently to the Solidity code:

address a \= 0xa44f5d3d624DfD660ecc11FF777587AD0a19606d;
bytes32 b \= bytes32(uint256(uint160(a)));

Among other things, this behavior means that the resulting `bytes32` value is right-aligned and zero-padded to the left.

CVL2 also supports casting from the `bytes32` type to the `address` type using either the `require_address()` or `assert_address()` cast functions.

bytes32 b \= to\_bytes32(0xa44f5d3d624DfD660ecc11FF777587AD0a19606d);
address a \= assert\_address(b);

Note that `require_address()` will silently allow a cast to continue when the `bytes32` variable contains a value that lies in the range `2^160 < var < 2^256`. The `assert_address()` cast function will fail when the `bytes32` variable contains a value in that same range.

bytes32 b \= to\_bytes32(0xa44f5d3d624DfD660ecc11FF777587AD0a19606d0e); // Note this contains one extra byte
address a \= require\_address(b);                                       // Silently does the cast.

While when using `assert_address`:

bytes32 b \= to\_bytes32(0xa44f5d3d624DfD660ecc11FF777587AD0a19606d0e); // Note this contains one extra byte
address a \= assert\_address(b);                                       // This will fail.

Casting from `bytes32` to `address` behaves equivalently to the Solidity code:

bytes32 b \= bytes32(0xa44f5d3d624DfD660ecc11FF777587AD0a19606d);
address a \= address(uint160(uint256(b)));

### [Modulo operator `%` always returns non-negative values](#id21)[](#modulo-operator-always-returns-non-negative-values "Link to this heading")

The modulo operator `%` follows the semantics of Solidity’s unsigned modulo and always returns non-negative values, regardless of the signs of its operands.

### [Support for `bytes1`…`bytes32`](#id22)[](#support-for-bytes1-bytes32 "Link to this heading")

CVL 2 supports the types `bytes1`, `bytes2`, …, `bytes32`, as in Solidity. Number literals must be explicitly cast to these types using `to_bytesN`; for example:

bytes32 x \= to\_bytes32(0);

Unlike Solidity, `bytes1`…`bytes32` literals do not need to be written in hex or padded to the correct length.

The only conversion between integer types and these types is from `uint<i*8>` to `bytes<i>` (i.e. unsigned integers with the same bitwidth as the target `bytes<i>` type) and from `bytes32` to `uint256`; For example:

uint24 u;
bytes3 x \= to\_bytes3(u); // This is OK
bytes4 y \= to\_bytes4(u); // This will fail
bytes32 b;
uint256 u \= assert\_uint256(b); // This is OK

### [Changes for bitwise operations](#id23)[](#changes-for-bitwise-operations "Link to this heading")

In CVL1, the exact details for bitwise operations (such as `&`, `|`, and `<<`) were not completely specified, especially for negative integers.

In CVL 2, all bitwise operations (`&`, `|`, `~`, `>>`, `>>>`, `<<`, and `xor`) on integer types first convert to a 256 bit word, then perform the operations on the full 256-bit word, then convert back to the expected type. Signed integer types use twos-complement encoding.

The two right-shifts differ in how they treat signed integers. `>>` is an arithmetic shift; it preserves the sign bit. `>>>` is a logical shift; it pads the shifted word with zero.

Bitwise operations cannot be performed on `mathint` values.

Note

By default, bitwise operators are [overapproximated](../../user-guide/glossary.html#term-overapproximation) (in both CVL 1 and CVL 2), so you may see counterexamples that incorrectly compute the results of bitwise operations. The approximations are still [sound](../../user-guide/glossary.html#term-sound): the Prover will not report a rule as verified if the original code does not satisfy the rule.

The [precise\_bitwise\_ops](../../prover/cli/options.html#precise-bitwise-ops) flag makes the Prover’s reasoning about bitwise operations more precise, but this flag is experimental in CVL 2.

## [Changes to the fallback function](#id24)[](#changes-to-the-fallback-function "Link to this heading")

In CVL 1, you could determine whether a `method` object was the fallback function by comparing its selector to `certorafallback().selector`:

assert f.selector \== certorafallback().selector,
    "f must be the fallback";

In CVL 2, `certorafallback()` is no longer valid. Instead, you can use the new field `f.isFallback` to detect the fallback method:

assert f.isFallback,
    "f must be the fallback";

## [Removed features](#id25)[](#removed-features "Link to this heading")

As we transit to CVL 2, we have removed several language features that are no longer used.

We have removed these features because we think they are no longer used and no longer useful. If you find that you do need one of these features, contact Certora support.

### [Methods entries for sighashes](#id26)[](#methods-entries-for-sighashes "Link to this heading")

In CVL 1, you could write a sighash instead of a method identifier in the `methods` block. This feature is no longer supported. You will need to have the name and argument types of the called method in order to provide an entry.

### [`invoke`, `sinvoke`, and `call`](#id27)[](#invoke-sinvoke-and-call "Link to this heading")

Older versions of CVL had special syntax for calling contract and CVL functions:

*   `invoke f(args);` should be replaced with `f@withrevert(args);`.
    
*   `sinvoke f(args);` should be replaced with `f(args);`.
    
*   `call f(args)` should be replaced with `f(args)`.
    

### [`static_assert` and `static_require`](#id28)[](#static-assert-and-static-require "Link to this heading")

These deprecated aliases for `assert` and `require` are being removed; replace them with `assert` and `require` respectively.

### [`invoke_fallback` and `certorafallback()`](#id29)[](#invoke-fallback-and-certorafallback "Link to this heading")

The `invoke_fallback` syntax is no longer supported; there is no longer a way to directly invoke the fallback method. You can work around this limitation by writing a parametric rule and filtering on `f.isFallback`. See [Changes to the fallback function](#cvl2-fallback-changes).

### [`invoke_whole`](#id30)[](#invoke-whole "Link to this heading")

The `invoke_whole` keyword is no longer supported.

### [Havocing local variables](#id31)[](#havocing-local-variables "Link to this heading")

In CVL 1, you could write the following:

calldataarg args; env e;
f(e, args);

havoc args;
g(e, args);

In CVL 2, you can only `havoc` ghost variables and ghost functions. Instead of havocing a local variable, replace the havoced variable with a new variable. For example, you should replace the above with

calldataarg args; env e;
f(e,args);

calldataarg args2;
g(e,args2);

### [Destructuring syntax for struct returns](#id32)[](#destructuring-syntax-for-struct-returns "Link to this heading")

In CVL 1, if a contract function returned a struct, you could use a destructuring syntax to get the return value in your spec. For example, consider the following contract:

contract Example {
    struct S {
        uint firstField;
        uint secondField;
        bool thirdField;
    }

    function f() returns(S) { ... }
    function g() returns(uint, uint) { ... }
}

To access the return value of `f` in CVL 1, you could write the following:

uint x; uint y; bool z;
x, y, z \= f();

This syntax is no longer supported. Instead, you should declare a variable with the struct type:

Example.S result \= f();
uint x \= result.firstField;

Destructuring assignments are still allowed for functions that return multiple values; the following is valid:

uint x; uint y;
x, y \= g();

### [`bytes[]` and `string[]`](#id33)[](#bytes-and-string "Link to this heading")

In CVL 1, you could declare variables of type `string[]` and `bytes[]`. You can no longer use these types in CVL.

You can still declare contract methods that use these types in the `methods` block. However, you can only call methods that take one of these types as an argument by passing a `calldataarg` variable, and you cannot access the return value of a method that returns one of these types.

### [`pragma`](#id34)[](#pragma "Link to this heading")

CVL 1 had a `pragma` command for specifying the CVL version, but this feature was not used and has been removed in CVL 2.

### [`events`](#id35)[](#events "Link to this heading")

CVL 1 had syntax for an `events` block, but it did nothing and has been removed.

## [Changes to the Command Line Interface (CLI)](#id36)[](#changes-to-the-command-line-interface-cli "Link to this heading")

As part of the transition to CVL 2 changes were made to enhanced clarity, uniformity, and readability on the Command-Line Interface (CLI). The complete CLI specification can be found [here](../../prover/cli/options.html)

Note

The changes will take effect starting v4.3.1 of `certora-cli`.

Note

To opt-out of the new CLI, one can set an environment variable `CERTORA_OLD_API` to `1`, e.g.: `export CERTORA_OLD_API=1`. **The old CLI will not be available in versions released after August 31st, 2023**

### [Flags Renaming](#id37)[](#flags-renaming "Link to this heading")

In CVL 2 some flags were renamed:

1.  flags with names that are generic or wrong
    
2.  flags that do not match their corresponding key in the `conf` file
    
3.  flags that do not follow the snake case format
    

This is the list of the flags that were renamed:

CVL 1

CVL 2

`--settings`

`--prover_args`

`--path`

`--solc_allow_path`

`--optimize`

`--solc_optimize`

`--optimize_map`

`--solc_optimize_map`

`--structLink`

`--struct_link`

`--javaArgs`

`--java_args`

### [`Prover Args`](#id38)[](#prover-args "Link to this heading")

`Prover args` are CLI flags that are sent to the Prover. `Prover args` can be set in one of two ways:

1.  Using specific CLI flags (e.g. `--loop_iter`)
    
2.  As parameters to the `--prover_args` (`--settings` in CVL 1)
    

Unlike CVL 1, if a `prover arg` is set using a specific CLI flag it cannot be set using `--prover_args`. In addition, the value commas and equal signs separators that were used in `--settings` were replaced with white-spaces in `--prover_args`.

Example:

Consider this call to `certoraRun` using CVL 1 syntax

certoraRun Compound.sol \\
    \--verify Compound:Compound.spec  \\
    \--solc solc8.13 \\
    \--settings \-smt\_bitVectorTheory\=true,-smt\_hashingScheme\=plainInjectivity,-assumeUnwindCond

In order to convert this call to CVL 2 we:

1.  renamed `--settings` to `--prover_args`
    
2.  replaced `-assumeUnwindCond` with the flag `--optimistic_loop`
    
3.  removed the comma and equal sign separators
    

certoraRun Compound.sol \\
    \--verify Compound:Compound.spec  \\
    \--solc solc8.13 \\
    \--optimistic\_loop \\
    \--prover\_args '-smt\_bitVectorTheory true -smt\_hashingScheme plainInjectivity'

### [`Solidity Compiler Args`](#id39)[](#solidity-compiler-args "Link to this heading")

The `Solidity Compiler Args` are CLI flags that are sent to the Solidity compiler. The behavior of the `Solidity Args` is similar to `Prover Args`. The flag `--solc_args` can only be used if there is no CLI flag that sets the Solidity flag and the value of `--solc_args` is a string that is sent as is to the Solidity compiler.

Example:

Consider this call to `certoraRun` using CVL 1 syntax

certoraRun Compound.sol \\
    \--verify Compound:Compound.spec  \\
    \--solc solc8.13 \\
    \--solc\_args "\['--optimize', '--optimize-runs', '200', '--experimental-via-ir'\]"

In CVL 2 calling optimize is using `--solc_optimize`

certoraRun Compound.sol \\
    \--verify Compound:Compound.spec  \\
    \--solc solc8.13 \\
    \--solc\_optimize 200 \\
    \--solc\_args "--experimental-via-ir"

### [Enhanced server support](#id40)[](#enhanced-server-support "Link to this heading")

In CVL 1, two server platforms were supported:

1.  `staging` was set using the flag `--staging [Branch/hotfix]`
    
2.  `production` was set using the flag `--cloud [Branch/hotfix]`
    

In CVL 2 the flag `--server` was added to replace `--staging` `--cloud` and to allow adding additional server platforms. `--server` gets as a parameter the platform name. `--prover_version` is a new flag in CVL 2 For setting the Branch/hot-fix
