# Hooks — Certora Prover Documentation 0.0 documentation

# [Hooks](#id5)[](#hooks "Link to this heading")

Hooks are used to attach CVL code to certain low-level operations, such as loads and stores to specific storage variables.

Each hook contains a pattern that describes what operations cause the hook to be invoked, and a block of code that is executed when the contract performs those operations.

The remainder of this document describes the operation of hooks in detail. For examples of idiomatic hook usage, see [Tracking changes with ghosts and hooks](../user-guide/ghosts.html#tracking-changes) and [Using Opcode Hooks](../user-guide/opcodes.html#using-opcodes).

## [Syntax](#id6)[](#syntax "Link to this heading")

The syntax for hooks is given by the following EBNF grammar:

hook ::= "hook" pattern block

pattern ::= "Sstore" access\_path param \[ "(" param ")" \]
          | "Sload"  param access\_path
          | opcode   \[ "(" params ")" \] \[ param \]

access\_path ::= id
              | \[ id "." \] "(" "slot" number ")"
              | access\_path "." id
              | access\_path "\[" "KEY"   basic\_type id "\]"
              | access\_path "\[" "INDEX" basic\_type id "\]"
              | access\_path "." "(" "offset" number ")"

opcode ::= "ALL\_SLOAD" | "ALL\_SSTORE" | "ALL\_TLOAD" | "ALL\_TSTORE" | ...

param  ::= evm\_type id

See [EVM opcode hooks](#opcode-hooks) below for the list of available opcodes.

See [Statements](statements.html) for information about the `statement` production; see [Types](types.html) for the `evm_type` production; see [Basic Syntax](basics.html) for the `number` production.

It is prohibited to have multiple hooks with the same hook pattern. Two hooks have the same hook pattern if both are `Sstore` hooks with the same access path, both are `Sload` hooks with the same access path, or both are opcode hooks with the same opcode. Doing so will result in an error like this: `The declared hook pattern <second hook> duplicates the hook pattern <first hook> at <spec file>. A hook pattern may only be used once.` Note that two access paths are considered to be “the same” if they resolve to the same storage address. Syntactically different access paths can alias, e.g., when accessing a member by name (`contract.member`) or by slot (`contract.(slot n)`).

## [Examples](#id7)[](#examples "Link to this heading")

*   [store hook example](https://github.com/Certora/Examples/blob/14668d39a6ddc67af349bc5b82f73db73349ef18/CVLByExample/ERC20/certora/specs/ERC20.spec#L117)
    
*   [load hook example](https://github.com/Certora/Examples/blob/14668d39a6ddc67af349bc5b82f73db73349ef18/CVLByExample/structs/BankAccounts/certora/specs/Bank.spec#L141)
    

See [Using Opcode Hooks](../user-guide/opcodes.html#using-opcodes) and [Tracking changes with ghosts and hooks](../user-guide/ghosts.html#tracking-changes) for additional examples.

## [Load and store hooks](#id8)[](#load-and-store-hooks "Link to this heading")

Load hooks are executed before a read from a specific location in storage, while store hooks are executed before a write to a specific location in storage.

The locations to be matched are given by an access path, such as a contract variable, array index, or a slot number. See [Access paths](#access-paths) below for information on the available access paths.

A load pattern contains the keyword `Sload`, followed by the type and name of a variable that will hold the loaded value, followed by an access path indicating the location that is read.

For example, here is a load hook that will execute whenever a contract reads the value of `C.owner`:

hook Sload address o C.owner { ... }

Inside the body of this hook, the variable `o` will be bound to the value that was read.

A store pattern contains the keyword `Sstore`, followed by an access path indicating the location that is being written to, followed by the type and name of a variable to hold the value that is being stored. Optionally, the pattern may also include the type and name of a variable to store the previous value that is being overwritten.

For example, here is a store hook that will execute whenever a contract writes the value of `C.totalSupply`:

hook Sstore C.totalSupply uint ts (uint old\_ts) { ... }

Inside the body of this hook, the variable `ts` will be bound to the value that is being written to the `totalSupply` variable, while `old_ts` is bound to the value that was stored there previously.

If you do not need to refer to the old value, you can omit the variable declaration for it. For example, the following hook only binds the new value of `C.totalSupply`:

hook Sstore C.totalSupply uint ts { ... }

Note

A hook will **not** be triggered by CVL code, including access to solidity storage from CVL.

### [Access paths](#id9)[](#access-paths "Link to this heading")

The patterns for load and store hooks are fine-grained; they allow you to hook on accesses to specific contract variables or specific array, struct, or mapping accesses.

Storage locations are designated by “access paths”. An access path starts with either the name of a contract field, or a [slot number](https://docs.soliditylang.org/en/v0.8.17/internals/layout_in_storage.html).

Contract fields may be qualified by the contract that defines them (e.g. `Contract.field`). If the contract name is omitted, it defaults to `currentContract`. The contract name may either be an instance variable introduced by a \[using statement\]\[using\] or the name of a contract on the [scene](../user-guide/glossary.html#term-scene).

Note

If a contract inherits multiple variables with the same name, you cannot use that variable as an access path.

If the indicated location holds a struct, you can refer to a specific field of the struct by appending `.<field-name>` to the path. For example, the following hook will execute on every store to the `balance` field of the struct `C.owner`:

hook Sstore C.owner.balance uint b { ... }

If the indicated location holds an array, you can refer to an arbitrary element of the array by appending `[INDEX uint <variable>]`. This pattern will match any store to an element of the array, and will bind the named variable to the index of the access. For example, the following hook will execute on any write to the array `C.entries` and will update the corresponding entry of the ghost mapping `_entries` to match:

hook Sstore C.entries\[INDEX uint i\] uint e {
    \_entries\[i\] \= e;
}

Additionally, if the indicated location holds a _dynamic_ array, you can refer to accesses to the length of the array with `.length`:

hook Sstore C.entries.length uint len {
    \_len \= len; // updates a ghost variable \`\_len\` with the length is written to storage.
}

Similarly, if the indicated location holds a mapping, you can refer to an arbitrary entry by appending `[KEY <type> <variable>]`. This pattern will match any write to the mapping, and will bind the named variable to the key. For example, the following hook will execute on any write to the mapping `C.balances`, and will update the `_balances` ghost accordingly:

hook Sstore C.balances\[KEY address user\] uint balance {
    \_balances\[user\] \= balance;
}

Finally, there is the low-level access pattern `<base>.(offset <n>)` for matching loads and stores that are a specific number of bytes from the base. For example, the following hook will match writes to the third or fourth byte of slot 1 (these two bytes are matched because the type of the variable is `uint16`:

hook Sstore (slot 1).(offset 2) uint16 b { ... }

These different kinds of paths can be combined, subject to restrictions listed below. For example, the following hook will execute whenever the contract writes to the `balance` field of a struct in the `users` mapping of contract `C`:

hook C.users\[KEY address user\].balance uint v (uint old\_value) { ... }

Inside the body of the hook, the variable `user` will refer to the address that was used as the key into the mapping `C.users`; the variable `v` will contain the value that is written, and the variable `old_value` will contain the value that was previously stored there.

There are a few restrictions on the available combinations of low-level and high-level access paths:

*   You cannot access struct fields on access paths that contain `slot` or `offset` components, because the struct type is not known.
    
*   You can only use `KEY` and `INDEX` patterns on word-aligned access paths (i.e. any `offset` components must be multiples of 32).
    

Note

The only available access paths for `solc` versions 5.17 and older are `slot` and `offset` paths.

### [Access path caveats](#id10)[](#access-path-caveats "Link to this heading")

In order to apply hooks correctly, the Prover must analyze the contract’s use of storage. In some cases, especially in the presence of inline assembly `sload` and `sstore` instructions, the analysis will fail. In this case, hooks may not be applied.

If storage analysis fails, you will see a message indicating the failure in the global problems view of the rule report. See [Analysis of EVM storage and EVM memory](../prover/techniques/index.html#storage-and-memory-analysis) for more details.

### [Hooking on all storage loads or stores](#id11)[](#hooking-on-all-storage-loads-or-stores "Link to this heading")

Load and store hooks apply to reads and writes to specific storage locations. In some cases, it is useful to instrument every load or store, regardless of the location.

The `ALL_SLOAD` and `ALL_SSTORE` opcode hooks are used for this purpose; they will be executed on every load and store instruction (in all contracts) respectively. See [EVM opcode hooks](#opcode-hooks) below for the general syntax of opcode hooks.

The `ALL_SLOAD` opcode hook takes one input `uint` argument containing the slot number of the load instruction, and has one `uint` output containing the value that is loaded from the slot. For example:

hook ALL\_SLOAD(uint slot) uint val { ... }

The `ALL_SSTORE` opcode hook takes two input `uint` arguments; the first is the slot number of the store instruction, and the second is the value being stored. For example:

hook ALL\_SSTORE(uint slot, uint val) { ... }

Note

The storage splitting optimization must be disabled using the [enableStorageSplitting](../prover/cli/options.html#enablestoragesplitting) option in order to use the `ALL_SLOAD` or `ALL_SSTORE` hooks.

If a load instruction matches an `Sload` hook pattern and there is also an `ALL_SLOAD` hook, then both hooks will be executed; the `Sload` hook will apply first, and then the `ALL_SLOAD` hook.

Similarly, if a store would trigger both an `Sstore` pattern and an `ALL_SSTORE` pattern, the `Sstore` hook would be executed, followed by the `ALL_SSTORE` hook.

Note

Just like the usual opcode hooks, the raw storage hooks are applied on all contracts. This means that a storage access on _any_ contract will trigger the hook. Therefore, in a rule that models calls to multiple contracts, if two contracts are accessing the same slot the same hook code will be called with the same slot number.

#### [Notes on transient storage.](#id12)[](#notes-on-transient-storage "Link to this heading")

In a similar vein to `ALL_SLOAD` and `ALL_SSTORE` hooks, CVL also allows to hook on `TLOAD` and `TSTORE` instructions for updating the transient storage, using the `ALL_TLOAD` and `ALL_TSTORE` hooks. The hooks for transient storage access share the syntax of their regular storage counterparts.

Given that Solidity only allows (as of v0.8.25) access to transient storage via inline-assembly, and has no notion of structure to transient storage, Prover currently does not expose access-path based access like for the regular storage hooks.

For more information on how the Prover models transient storage, please refer to the relevant section on [transient storage](transient.html#transient-storage).

## [EVM opcode hooks](#id13)[](#evm-opcode-hooks "Link to this heading")

Opcode hooks are executed just after[\[1\]](#before-hooks) a contract executes a specific [EVM opcode](https://ethereum.org/en/developers/docs/evm/opcodes/). An opcode hook pattern consists of the name of the opcode, followed by the inputs to the opcode (if any), followed by the type and variable name for the output (if any).

For example, the following hook will execute immediately after any contract executes the `EXTCODESIZE` instruction:

hook EXTCODESIZE(address addr) uint v { ... }

Within the body of the hook, the `addr` variable will be bound to the address argument to the opcode, and the variable `v` will be bound to the returned value of the opcode.

Note

Opcode hooks are applied to _all_ contracts, not just the main contract under verification.

Opcode hooks have the same names, arguments, and return values as the corresponding [EVM opcodes](https://ethereum.org/en/developers/docs/evm/opcodes/), with the exception of the `CREATE1` hook, which corresponds to the `CREATE` opcode.

Below is the set of supported opcode hook patterns:

hook ADDRESS address v

hook BALANCE(address addr) uint v

hook ORIGIN address v

hook CALLER address v

hook CALLVALUE uint v

hook CODESIZE uint v

hook CODECOPY(uint destOffset, uint offset, uint length)

hook GASPRICE uint v

hook EXTCODESIZE(address addr) uint v

hook EXTCODECOPY(address b, uint destOffset, uint offset, uint length)

hook EXTCODEHASH(address a) bytes32 hash

hook BLOCKHASH(uint n) bytes32 hash

hook COINBASE address v

hook TIMESTAMP uint v

hook NUMBER uint v

hook DIFFICULTY uint v

hook GASLIMIT uint v

hook CHAINID uint v

hook SELFBALANCE uint v

hook BASEFEE uint v

hook MSIZE uint v

hook GAS uint v

hook LOG0(uint offset, uint length)

hook LOG1(uint offset, uint length, bytes32 t1)

hook LOG2(uint offset, uint length, bytes32 t1, bytes32 t2)

hook LOG3(uint offset, uint length, bytes32 t1, bytes32 t2, bytes32 t3)

hook LOG4(uint offset, uint length, bytes32 t1, bytes32 t2, bytes32 t3, bytes32 t4)

hook CREATE1(uint value, uint offset, uint length) address v

hook CREATE2(uint value, uint offset, uint length, bytes32 salt) address v

hook CALL(uint g, address addr, uint value, uint argsOffset, uint argsLength, uint retOffset, uint retLength) uint rc

hook CALLCODE(uint g, address addr, uint value, uint argsOffset, uint argsLength, uint retOffset, uint retLength) uint rc

hook DELEGATECALL(uint g, address addr, uint argsOffset, uint argsLength, uint retOffset, uint retLength) uint rc

hook STATICCALL(uint g, address addr, uint argsOffset, uint argsLength, uint retOffset, uint retLength) uint rc

hook REVERT(uint offset, uint size)

hook BLOBHASH(uint n) bytes32 hash

hook SELFDESTRUCT(address a)

### [Call hooks](#id14)[](#call-hooks "Link to this heading")

We provide hooks for all four call opcodes: [`CALL`](https://www.evm.codes/#f1), [`CALLCODE`](https://www.evm.codes/#f2), [`DELEGATECALL`](https://www.evm.codes/#f4), and [`STATICCALL`](https://www.evm.codes/#fa). The hook parameters match the stack inputs of the respective opcodes.

The arguments for the call arguments and return values (`argsOffset`, `argsSize`, `retOffset`, and `retSize`) mostly exist for future use. Their values can be consumed, but currently they cannot be used to read data stored in memory before or after the call.

These hooks can be very useful to establish sensible security invariants. As an example, suppose no external code should gain write access to the data of the current contract. Both `CALLCODE` and `DELEGATECALL` have the potential to do exactly that by calling into another contract but keeping the current context. Note that there are exceptions to this property whenever _trusted_ 3rd party libraries are used. It might also be necessary to restrict these checks to the contract of interest.

hook CALLCODE(uint g, address addr, uint value, uint argsOffset, uint argsLength, uint retOffset, uint retLength) uint rc {
    // using CALLCODE is generally not expected in this contract
    assert (executingContract != currentContract, "we should not use \`callcode\`");
}
hook DELEGATECALL(uint g, address addr, uint argsOffset, uint argsLength, uint retOffset, uint retLength) uint rc {
    // DELEGATECALL is used in this contract, but it only ever calls into itself
    assert (executingContract != currentContract || addr \== currentContract,
        "we should only \`delegatecall\` into ourselves"
    );
}

### [Known inter-dependencies and common pitfalls](#id15)[](#known-inter-dependencies-and-common-pitfalls "Link to this heading")

Hooks are instrumented for every appearance of the matching instruction in the bytecode, as generated by a high-level source compiler such as the Solidity compiler. The behavior of the bytecode may sometimes be surprising. For example, every Solidity method call to a non-payable function will contain an early call to `CALLVALUE` to check that it is 0. This means that every time a non-payable Solidity function is invoked, the `CALLVALUE` hook will be triggered.

## [Hook bodies](#id16)[](#hook-bodies "Link to this heading")

The body of a hook may contain almost any CVL code, including calls to other Solidity functions. The only exception is that hooks may not contain parametric method calls. Expressions in hook bodies may reference variables bound by the hook pattern.

### [Keywords available in hook bodies](#id17)[](#keywords-available-in-hook-bodies "Link to this heading")

Hook bodies may refer to the special CVL variable `executingContract`, which contains the address of the contract whose code triggered the hook.

The call opcodes (`CALL`, `CALLCODE`, `STATICCALL` and `DELEGATECALL`) may also refer to a special CVL variable `selector`, which can be used to compare to a specific method signature selector. For example, here the hook body will assert that the native token value passed in the call is 0, only for a `transfer(address,uint)` call:

hook CALL(uint g, address addr, uint value, uint argsOffs, uint argLength, uint retOffset, uint retLength) uint rc {
    if(selector \== sig:transfer(address, uint).selector) {
      assert value \== 0;
    }
}

Note

In case the size of the input to the call is less than 4 bytes, the value of `selector` is 0. This can be checked by comparing the `argLength` argument of the hook to 4.

### [Reentrant hooks](#id18)[](#reentrant-hooks "Link to this heading")

While a hook is executing, additional hooks are skipped. If a hook calls a contract function which would normally cause another hook to execute, the inner hook will not execute.

For example, suppose the contract function `updateX()` always assigned to the value `x`, and consider the following hook (see [`HookReentrancy.spec` here](https://github.com/Certora/Examples/tree/28cb774fc03767e8aeaf00ea1bb0fac7db595fc9/ReferenceManual/HookReentrancy) for the complete example):

 1/// the number of stores to \`x\`
 2ghost mathint xStoreCount;
 3
 4/// increment xStoreCount and recursively update \`x\`
 5hook Sstore x uint v {
 6    xStoreCount \= xStoreCount + 1;
 7    if (xStoreCount < 5) {
 8        updateX();
 9    }
10}
11
12/// This rule will pass because hooks are not recursively applied
13rule checkStoreCount {
14    require xStoreCount \== 0;
15    updateX();
16    assert xStoreCount \== 1;
17}

In this example, the rule `checkStoreCount` calls `updateX` on line 15, which updates `x`, triggering the hook on line 5. The hook then calls `updateX` again on line 8. The recursive call to `updateX` will then update `x` a second time.

At this point, you may expect that the hook will be triggered a second time, but because there is already a hook executing, this second update to `x` will not trigger the hook. Therefore the `xStoreCount` ghost will _not_ be updated a second time, so its final value will be `1`.

* * *
