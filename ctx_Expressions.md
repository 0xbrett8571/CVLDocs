# Expressions — Certora Prover Documentation 0.0 documentation
A CVL expression is anything that represents a value. This page documents all possible expressions in CVL and explains how they are evaluated.

Contents

*   Expressions
    
    *   Syntax
        
    *   Basic operations
        
    *   Struct Comparison
        
    *   Extended logical operations
        
    *   Ghost Mapping Sums
        
    *   Accessing fields and arrays
        
    *   Contracts, method signatures and their properties
        
    *   Special variables and fields
        
    *   Calling contract functions
        
        *   Type restrictions
            
    *   Comparing storage
        
    *   Direct storage access
        
        *   Direct storage havoc
            
        *   Direct immutable access
            
    *   Built-in Functions
        
        *   Hashing
            
            *   Example
                
        *   ECRecover
            
            *   Example
                

Syntax

-------------------------------------------------

The syntax for CVL expressions is given by the following EBNF grammar:

```
expr ::= literal
       | unop expr
       | expr binop expr
       | "(" exprs ")"
       | expr "?" expr ":" expr
       | [ "forall" | "exists" ] type id "." expr
       | [ "sum" | "usum" ] type id { "," type id } "." expr

       | expr "." id
       | id [ "[" expr "]" { "[" expr "]" } ]
       | id "(" types ")"

       | function_call

       | expr "in" id

function_call ::=
       | [ id "." ] id
         [ "@" ( "norevert" | "withrevert" | "dontsummarize" ) ]
         "(" exprs ")"
         [ "at" id ]


literal ::= "true" | "false" | number | string

unop  ::= "~" | "!" | "-"

binop ::= "+" | "-" | "*" | "/" | "%" | "^"
        | ">" | "<" | "==" | "!=" | ">=" | "<="
        | "&" | "|" | "<<" | ">>" | "&&" | "||"
        | "=>" | "<=>" | "xor" | ">>>"

specials_fields ::=
           | "block" "." [ "number" | "timestamp" ]
           | "msg"   "." [ "address" | "sender" | "value" ]
           | "tx"    "." [ "origin" ]
           | "length"
           | "selector" | "isPure" | "isView" | "numberOfArguments" | "isFallback"

special_vars ::=
           | "lastReverted"
           | "lastStorage"
           | "allContracts"
           | "lastMsgSig"
           | "_"
           | "max_uint" | "max_address" | "max_uint8" | ... | "max_uint256"
           | "nativeBalances"
           | "calledContract"
           | "executingContract"

cast_functions ::=
    | require_functions | to_functions | assert_functions

require_functions ::=
    | "require_uint8" | ... | "require_uint256" | "require_int8" | ... | "require_int256" | "require_address"

to_functions ::=
    | "to_mathint" | "to_bytes1" | ... | "to_bytes32"

assert_functions ::=
   | "assert_uint8" | ... | "assert_uint256" | "assert_int8" | ... | "assert_int256" | "assert_address"

contract ::= id | "currentContract"

```


See Basic Syntax for the `id`, `number`, and `string` productions. See Types for the `type` production.

Basic operations

---------------------------------------------------------------------

CVL provides the same basic arithmetic, comparison, bitwise, and logical operations for basic types that solidity does, with a few differences listed in this section and the next. The precedence and associativity rules are standard.

Caution

One significant difference between CVL and Solidity is that in Solidity, `^` denotes bitwise exclusive or and `**` denotes exponentiation, whereas in CVL, `^` denotes exponentiation and `xor` denotes exclusive or.

See Changes to integer types for more information about the interaction between mathematical types and the meaning of mathematical operations.

Struct Comparison

-----------------------------------------------------------------------

CVL supports equality comparison of structs under the following restrictions:

*   The structs must be of the same type.
    
*   The structs (or any nested structs) don’t contain dynamic arrays. (`string` and `bytes` can be part of the struct)
    
*   There’s no support for comparison for structs fetched using direct-storage-access.
    

Two structs will be evaluated as equal if and only if all the fields are equal.

For example:

```
rule example(MyContract.MyStruct s) {
    env e;
    assert s == currentContract.myStructGetter(e);
}

```


Extended logical operations

--------------------------------------------------------------------------------------------

CVL also adds several useful logical operations:

*   Like `&&` or `||`, an _implication_ expression `expr1 => expr2` requires `expr1` and `expr2` to be boolean expressions and is itself a boolean expression. `expr1 => expr2` evaluates to `false` if and only if `expr1` evaluates to `true` and `expr2` evaluates to `false`. `expr1 => expr2` is equivalent to `!expr1 || expr2`.
    
    For example, the statement `assert initialized => x > 0;` will only report counterexamples where `initialized` is true but `x` is not positive.
    
*   The short-circuiting behavior of implications (`=>`) and other boolean connectors in CVL mirrors the short-circuiting behavior seen in standard logical operators (`&&` and `||`). In practical terms, this implies that the evaluation process is terminated as soon as the final result can be determined without necessitating further computation. For example, when dealing with an implication expression like `expr1 => expr2`, if the evaluation of `expr1` results in false, there is no need to proceed with evaluating `expr2` since the overall result is already known. This aligns with the common short-circuiting behavior found in traditional logical operators.
    
*   Similarly, an _if and only if_ expression (also called a _bidirectional implication_) `expr1 <=> expr2` requires `expr1` and `expr2` to be boolean expressions and is itself a boolean expression. `expr1 <=> expr2` evaluates to `true` if `expr1` and `expr2` evaluate to the same boolean value.
    
    For example, the statement `assert balanceA > 0 <=> balanceB > 0;` will report a violation if exactly one of `balanceA` and `balanceB` is positive.
    
*   An _if-then-else_ (_ITE_) expression of the form `cond ? expr1 : expr2` requires `cond` to be a boolean expression and requires `expr1` and `expr2` to have the same type; the entire if-then-else expression has the same type as `expr1` and `expr2`. The expression `cond ? expr1 : expr2` should be read “if `cond` then `expr1` else `expr2`. If `cond` evaluates to `true` then the entire expression evaluates to `expr1`; otherwise the entire expression evaluates to `expr2`.
    
    For example, the expression
    
    ```
uint balance = address == owner ? ownerBalance()
                                : userBalance(address);

```

    
    will set `balance` to `ownerBalance()` if `address` is `owner`, and will set it to `userBalance(address)` otherwise.
    
    Conditional expressions are _short-circuiting_: if `expr1` or `expr2` have side-effects (such as updating a ghost variable), only the side-effects of the expression that is chosen are performed.
    
    Regarding the logical operator precedence, `=>` has higher precedence than `<=>`, and unlike math operators both are _right_ associative, so `expr1 => expr2 => expr3` is equivalent to `expr1 => (expr2 => expr3)`.
    
*   A _universal_ expression of the form `forall t v . expr` requires `t` to be a type (such as `uint256` or `address`) and `v` to be a variable name; `expr` should be a boolean expression and may refer to the identifier `v`. The expression evaluates to `true` if _every_ possible value of the variable `v` causes `expr` to evaluate to `true`.
    
    For example, the statement
    
    ```
require (forall address user . balance(user) <= balance(biggestUser));

```

    
    will ensure that every other user has a balance that is less than or equal to the balance of `biggestUser`.
    
*   Like a universal expression, an _existential_ expression of the form `exists t v . expr` requires `t` to be a type and `v` to be a variable name; `expr` should be a boolean expression and may refer to the variable `v`. The expression evaluates to `true` if there is _any_ possible value of the variable `v` that causes `expr` to evaluate to `true`.
    
    For example, the statement
    
    ```
require (exists uint t . priceAtTime(t) != 0);

```

    
    will ensure that there is some time for which the price is nonzero.
    

Note

The symbols `forall` and `exist` are sometimes referred to as quantifiers, and expressions of the form `forall type v . e` and `exist type v . e` are referred to as quantified expressions.

Caution

`forall` and `exists` expressions are powerful and elegant ways to express rules and invariants, but they require the Prover to consider all possible values of a given type. In some cases they can cause significant slowdowns for the Prover.

If you have rules or invariants using `exists` that are running slowly or timing out, you can remove the `exists` by manually computing the value that exists. For example, you might replace

```
require (exists uint t . priceAtTime(t) != 0);

```


with

```
require priceAtTime(startTime) != 0;

```


Caution

Calling contract functions within the body of a quantified expression is an experimental feature and may not work as intended.

Note

The Prover uses approximations that may cause spurious counterexamples in rules that use quantifiers. For example, a rule that requires a quantified statement may produce a counterexample that doesn’t satisfy the requirement. The approximation is sound: it won’t cause violations to be hidden. See Quantifier Grounding for more detail.

Ghost Mapping Sums

--------------------------------------------------------------------------

The prover is capable of calculating the `sum` of ghost mappings over specified indices. For example, if we have a ghost mapping `ghost mapping(address => mapping(bytes32 => mathint)) myGhost`, and want to sum all the values of the ghost over all addresses for a given `bytes32` value `b`, one can write `sum address a. myGhost[a][b]`. This returns a `mathint`\-typed number that sums all known values of `myGhost` over the first index. The full syntax for this is `sum type1 t1, type2 t2, ... typeN tN. ghostName[...]`. See here for an example.

Note

The prover support only summation of ghosts. If one wants to sum e.g. some storage mapping, one could implement a ghost that mirrors the storage and sum over it.

An extension of the summing logic is the unsigned sum which uses the `usum` keyword. It follows the same rules as the regular sum, but adds some extra logic to ensure the value of the sum is always larger than its parts.

Note

The keyword `usum` indicates that all entries are non-negative (unsigned). It can only be used on ghost mappings with unsigned or `mathint` value types. In the case of `mathint` an assertion is added on each write to the ghost that reports an error if the written value is negative. When using this keyword, the solver will introduce the additional assumption that the `usum` is larger than any value in the ghost mapping and that it is even larger than the sum of any finite subset of values. This additional assumption is valid, because the other values in the ghost mapping can make the total sum only larger.

Accessing fields and arrays

--------------------------------------------------------------------------------------------

One can access the special fields of built-in types, fields of user-defined `struct` types, and members of user-defined `enum` types using the normal `expr.field` notation. Note that as described in User-defined types, access to user-defined types must be qualified by a contract name.

Access to arrays also uses the same syntax as Solidity.

Contracts, method signatures and their properties

---------------------------------------------------------------------------------------------------------------------------------------

Writing the ABI signature for a contract method produces a `method` object, which can be used to access the method fields.

For example,

```
method m;
require m.selector == sig:balanceOf(address).selector
     || m.selector == sig:transfer(address, uint256).selector;

```


will constrain `m` to be either the `balanceOf` or the `transfer` method.

One can determine whether a contract has a particular method using the `s in c` where `s` is a method selector and `c` is a contract (either `currentContract` or a contract introduced with a using statement. For example, the statement

```
if (burnFrom(address,uint256).selector in currentContract) {
  ...
}

```


will check that the current contract supports the optional `burnFrom` method.

Special variables and fields

----------------------------------------------------------------------------------------------

Several of the CVL types have special fields; see Types (particularly The env type, The method and calldataarg types, and Array access).

There are also several built-in variables:

*   `address currentContract` always refers to the main contract being verified (that is, the contract named in the verify option).
    
*   `bool lastReverted` indicates whether the most recent contract function reverted or threw an exception.
    
    Caution
    
    The variable `lastReverted` is updated after each contract call, even those called without `@withrevert` (see Calling contract functions). This is a common source of errors. For example, the following rule is vacuous:
    
    ```
rule revert_if_paused() {
  withdraw@withrevert();
  assert isPaused() => lastReverted;
}

```

    
    In this rule, the call to `isPaused` will update `lastReverted` to `true`, overwriting the value set by `withdraw`.
    
*   `lastStorage` refers to the most recent state of the EVM storage. See The storage type for more details.
    
*   You can use the variable `_` as a placeholder for a value you are not interested in.
    
*   The maximum values for the different integer types are available as the variables `max_uint`, `max_address`, `max_uint8`, `max_uint16` etc.
    
*   `nativeBalances` is a mapping of the native token balances, i.e. ETH for Ethereum. The balance of an `address a` can be expressed using `nativeBalances[a]`.
    
*   `calledContract` is only available in expression summaries. It refers to the receiver contract of a summarized method call.
    
*   `executingContract` is only available in hooks. It refers to the contract that is executing when the hook is triggered.
    

CVL also has several built-in functions for converting between numeric types. See Basic operations for details.

Calling contract functions

------------------------------------------------------------------------------------------

There are many kinds of function-like things that can be called from CVL:

*   Contract functions
    
*   Method variables
    
*   Ghost Functions
    
*   Functions
    
*   Definitions
    

There are several additional features that can be used when calling contract functions (including calling them through method variables).

The method name can optionally be prefixed by a contract name (introduced via a using statement). If a contract is not explicitly named, the method will be called with `currentContract` as the receiver.

It is possible for multiple contract methods to match the method call. This can happen in two ways:

1.  The method to be called is a method variable
    
2.  The method to be called is overloaded in the contract (i.e. there are two methods of the same name), and the method is called with a calldataarg argument.
    

In either case, the Prover will consider every possible resolution of the method while verifying the rule, and will provide a separate verification report for each checked method. Rules that use this feature are referred to as parametric rules.

Another possible way to have the Prover consider options for a function is by using an `address` typed variable and “calling” the function on it, e.g. `address a; a.foo(...);`. In this case the Prover will consider every possible contract in the {ref}scene that implements a function that matches the signature provided by the call (if no such function exists in the {ref}scene the prover will fail with an error). Note: The values that the address variable can take are the addresses that are associated with the relevant contracts in the scene. Notably, other values would not be possible: Given an address variable `a`, on which we call a some method implemented by contracts `A` and `B`, we will have an implicit `require a == A || a == B`.

After the function name, but before the arguments, you can write an optional method tag, one of `@norevert` or `@withrevert`.

*   `@norevert` indicates that examples where the method revert should not be considered. This is the default behavior if no tag is provided
    
*   `@withrevert` indicates that examples that would revert should still be considered. In this case, the method will set the `lastReverted` variable to `true` in case the called method reverts or throws an exception.
    
    `withrevert` example
    

After the method tag, the method arguments are provided. Unless the method is declared envfree, the first argument must be an environment value. The remaining arguments must either be a single calldataarg value, or the same types of arguments that would normally be passed to the contract method.

After the method arguments, a method invocation can optionally include `at s` where `s` is a storage variable. This indicates that before the method is executed, the EVM state should be restored to the saved state `s`.

### Type restrictions


When calling a contract function, the Prover must convert the arguments and return values from their Solidity types to their CVL types and vice-versa. There are some restrictions on the types that can be converted. See Conversions between CVL and Solidity types for more details.

Comparing storage

------------------------------------------------------------------------

As described in the documentation on storage types, CVL represents the entirety of the EVM and its ghost state in variables with `storage` type. Variables of this type can be checked for equality and inequality.

The basic form of this expression is `s1 == s2`, where `s1` and `s2` are variables of type `storage`. This expression compares the states represented by `s1` and `s2`; that is, it checks equality of the following:

1.  The values in storage for all contracts,
    
2.  The balances of all contracts,
    
3.  The state of all ghost variables and functions
    

Thus, if any field in any contract’s storage differs between `s1` and `s2`, the expression will return `false`. The expression `s1 != s2` is shorthand for `!(s1 == s2)`.

Storage comparisons also support narrowing the scope of comparison to specific components of the global state represented by `storage` variables. This syntax is `s1[r] == s2[r]` or `s1[r] != s2[r]`, where `r` is a “storage comparison basis”, and `s1` and `s2` are variables of type `storage`. The valid bases of comparison are:

1.  The name of a contract imported with a using statement,
    
2.  The keyword `nativeBalances`, or
    
3.  The name of a ghost variable or function
    

It is an error to use different bases on different sides of the comparison operator, and it is also an error to use a comparison basis on one side and not the other. The application of the basis restricts the comparison to only consider the portion of global state identified by the basis.

If the qualifier is a contract identifier imported via `using`, then the comparison operation will only consider the storage fields of that contract. For example:

```
using MyContract as c;
using OtherContract as o;

rule compare_state_of_c(env e) {
   storage init = lastStorage;
   o.mutateOtherState(e); // changes `o` but not `c`
   assert lastStorage[c] == init[c];
}

```


will pass verification whereas:

```
using MyContract as c;
using OtherContract as o;

rule compare_state_of_c(env e) {
   storage init = lastStorage;
   c.mutateContractState(e); // changes `c`
   assert lastStorage[c] == init[c];
}

```


will not.

Note

Comparing contract’s state using this method will **not** compare the balance of the contract between the two states.

If the qualifier is the identifier `nativeBalances`, then the account balances of all contracts are compared between the two storage states. Finally, if the basis is the name of a ghost function or variable, the values of that function/variable are compared between storage states.

Two ghost functions are considered equal if they have the same outputs for all input arguments.

Warning

The default behavior of the Prover on unresolved external calls is to pessimistically havoc contract state and balances. This behavior will render most storage comparisons that incorporate such state useless. Care should be taken (using summarization) to ensure that rules that compare storage do not encounter this behavior.

Warning

The grammar admits storage comparisons for both equality and inequality that occur arbitrarily nested within expressions. However, support within the Prover for these comparisons is primarily aimed at assertions of storage equality, e.g., `assert s1 == s2`. Support for storage inequality as well as nesting comparisons within other expressions is considered experimental.

Warning

The storage comparison checks for exact equality between every single slot of storage which can lead to surprising failures of storage equality assertions. In particular, these failures can happen if an uninitialized storage slot is written and then later cleared by Solidity (via the `pop()` function or the `delete` keyword). After the clear operation the slot will definitely hold 0, but the Prover will not make any assumptions about the value of the uninitialized slot which means they can be considered different.

Direct storage access

--------------------------------------------------------------------------------

The value of contract state variables can be directly accessed from CVL. These direct storage accesses are written using the state variable names and struct fields defined in the contract. For example, to access the state variable `uint x` defined in the `currentContract`, one can simply write `currentContract.x`. More complex structs can be accessed by chaining field selects and array/map dereference operations together. For example, if the current contract has the following type definitions and state variables:

```
contract Example {
   struct Foo {
      mapping (address => uint[]) bar;
   }
   Foo[3] myState;
   uint32 luckyNumber;
   address[] public addresses;
}

```


one can write `currentContract.myState[0].bar[addr][0]`, where `addr` is a CVL variable of type `address`.

The storage of contracts other than the `currentContract` can be accessed by writing the contract identifier bound with a using statement. For example, if the `myState` definition above appeared in a contract called `Test` and the current CVL file included `using Test as t;` one could write `t.myState[0].bar[addr][0]`.

Note

A contract identifier (or `currentContract`) _must_ be included in the direct storage access. In other words, writing just `myState[0].bar[addr][0]` will not work, even if `myState` is declared in the current contract.

Currently only primitive values (e.g., `uint`, `bytes3`, `bool`, enums, and user defined value types) can be directly accessed. Attempting to access more complex types will yield a type checking error. For example, attempting to access an entire array with `currentContract.myState[0].bar[addr]` will fail.

Note

Although entire arrays cannot be accessed, the _length_ or the _number of elements_ of the dynamic arrays can be accessed with `.length`, e.g., `currentContract.myState[0].bar[addr].length`.

Warning

Direct storage access is an experimental feature, and relies on several internal program analyses which can sometimes fail. For example, attempts to use direct storage access to refer to variable which is actually unused or inaccessible in the contract. If these internal static analyses fail, any rules that use direct storage access will fail during processing. If this occurs, check the “Global Problems” view of the web report and contact Certora for assistance.

### Direct storage havoc


The same direct storage syntax can also be used in `havoc` statements. With the previously-mentioned `Example` contract and `using Example as ex`, you can write `havoc ex.luckyNumber` or `havoc addresses[10]` or even `havoc addresses.length`.

While you may use a `havoc assuming` statement, unlike ghosts, you cannot directly refer to the havoced storage path in the `assuming` expression using the `@old` and `@new` syntax. This generally means `assuming` expressions are not as useful with direct storage access, so consider using and unconditional `havoc` statements instead of `havoc assuming`.

Warning

As with direct storage access in general, direct storage havoc is experimental and limited to primitive types. In particular, this mean you _cannot_ currently havoc

*   entire arrays or entire mappings (only arrays at a specific index, or mappings at a specific key)
    
*   user-defined types such as structs, or arrays/mappings of such types
    
*   enums
    

### Direct immutable access


The Certora Prover allows to access immutable variables in a contract, in a similar way to direct storage access. For example, given a contract:

```
contract WithImmutables {
  address private immutable myImmutAddr;
  bool public immutable myImmutBool;

  constructor() { ... }
  function publicGetterForPrivateImmutableAddr() external returns (address) {
    return myImmutAddr;
  }
}

```


We can access both `myImmutAddr` and `myImmutBool` directly from CVL like this:

```
using WithImmutables as withImmutables;

methods {
  function publicGetterForPrivateImmutableAddr() external returns (address) envfree;
  function myImmutBool() external returns (bool) envfree;
}

rule accessPrivateImmut {
  assert withImmutables.myImmutAddr == publicGetterForPrivateImmutableAddr();
}

rule accessPublicImmut {
  assert withImmutables.myImmutBool == withImmutables.myImmutBool();
}

```


The advantages of direct immutable access is that there is no need to declare `envfree` methods for the public immutables, and even more importantly, nor is there a need to harness contracts in order to expose the private immutables.

Built-in Functions

--------------------------------------------------------------------------

### Hashing


CVL allows to use Solidity’s `keccak256` hashing function directly in spec. Below are two usage examples: one using a `bytes` array, another using primitives. As `bytes32` is the return type of `keccak256` and is a primitive type, calls to `keccak256` can be nested.

(Currently, only the `keccak256` hash is supported in CVL as a built-in.)

#### Example


Given the following Solidity snippet:

```
contract HashingExample {
  struct SignedMessage {
    address sender;
    uint256 nonce;
    bytes signature;
  }

  mapping (bytes32 => uint256) messageToValue;

  function hashingScheme1(SignedMessage memory s) public pure returns (bytes32) {
    return keccak256(abi.encode(s.sender, s.nonce));
  }

  function hashingScheme2(SignedMessage memory s) public pure returns (bytes32) {
    return keccak256(s.signature);
  }

  function hashingScheme3(SignedMessage memory s) public pure returns (bytes32) {
    return keccak256(abi.encode(s.sender, s.nonce, keccak256(s.signature)));
  }

  function hashingScheme4(SignedMessage memory s) public pure returns (bytes32) {
    return keccak256(abi.encode(s.sender, s.nonce, s.signature));
  }
}

```


The hashing schemes described by `hashingScheme1`, `hashingScheme2`, and `hashingScheme3` can be replicated in CVL as follows:

```
function hashingScheme1CVL(HashingExample.SignedMessage s) returns bytes32 {
  return keccak256(s.sender, s.nonce);
}

function hashingScheme2CVL(HashingExample.SignedMessage s) returns bytes32 {
  return keccak256(s.signature);
}

function hashingScheme3CVL(HashingExample.SignedMessage s) returns bytes32 {
  return keccak256(s.sender, s.nonce, keccak256(s.signature));
}

```


The scheme implemented in `hashingScheme4` is not supported at the moment, as it combines a `bytes` type with primitives. The `keccak256` built-in function supports two kinds of inputs:

*   a single `bytes` parameter
    
*   a list of primitive (e.g., `uint256`, `uint8`, `addresss`) parameters
    

Note

`keccak256` is currently _**unsupported**_ in quantified expressions.

### ECRecover


The `ecrecover` function in Solidity is helpful in recovering the signer’s address from a signed message. It exists in very similar form in CVL and receives exactly the same parameter types as its Solidity counterpart.

Note

`ecrecover` is _**supported**_ in quantified expressions.

The Prover’s model of `ecrecover` does not actually implement the elliptical curve recovery algorithm, and is instead implemented using an ghost function. Like all ghost functions, axioms can be added to make the behavior of CVL’s `ecrecover` more faithfully model the actual key recovery algorithm.

There is a useful set of axioms that can be encoded in CVL to make the modeled behavior of `ecrecover` more precise and less likely to create false counterexamples:

```
function ecrecoverAxioms() {
  // zero value:
  require (forall uint8 v. forall bytes32 r. forall bytes32 s. ecrecover(to_bytes32(0), v, r, s) == 0);
  // uniqueness of signature
  require (forall uint8 v. forall bytes32 r. forall bytes32 s. forall bytes32 h1. forall bytes32 h2.
    h1 != h2 => ecrecover(h1, v, r, s) != 0 => ecrecover(h2, v, r, s) == 0);
  // dependency on r and s
  require (forall bytes32 h. forall uint8 v. forall bytes32 s. forall bytes32 r1. forall bytes32 r2.
    r1 != r2 => ecrecover(h, v, r1, s) != 0 => ecrecover(h, v, r2, s) == 0);
  require (forall bytes32 h. forall uint8 v. forall bytes32 r. forall bytes32 s1. forall bytes32 s2.
    s1 != s2 => ecrecover(h, v, r, s1) != 0 => ecrecover(h, v, r, s2) == 0);
}

```


#### Example


Given the following Solidity snippet:

```
contract ECExample {
  function wrap_ecrecover(bytes32 digest, uint8 v, bytes32 r, bytes32 s) public pure returns (address) {
    return ecrecover(digest,v,r,s);
  }
}

```


The following CVL function is equivalent to the `wrap_ecrecover` function in the Solidity snippet:

```
function wrap_ecrecoverCVL(bytes32 digest, uint8 v, bytes32 r, bytes32 s) returns address {
  return ecrecover(digest,v,r,s);
}

```
