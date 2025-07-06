# Producing Positive Examples — Certora Prover Documentation 0.0 documentation
Sometimes it is useful to produce examples of an expected behavior instead of counterexamples that demonstrate unexpected behavior. You can do this by writing a rule that uses satisfy statements instead of the `assert` command. For each `satisfy` command in a rule, the Prover will produce an example that makes the condition true, or report an error.

The purpose of the `satisfy` statement is to produce examples that demonstrate some execution of the code. Not every example is interesting — users should inspect the example to ensure that it demonstrates the expected behavior.

For example, we may be interested in showing that it is possible for someone to deposit some assets into a pool and then immediately withdraw them. The following rule demonstrates this scenario:

Positive example

```
rule possibleToFullyWithdraw(address sender, uint256 amount) {
    env eT0;
    env eM;
    setup(eM);
    address token;
    require token == _token0 || token == _token1;
    uint256 balanceBefore = token.balanceOf(eT0,sender);
    
    require eM.msg.sender == sender;
    require eT0.msg.sender == sender;
    require amount > 0;
    token.transfer(eT0, currentContract, amount);
    uint256 amountOut0 = mint(eM,sender);
    // immediately withdraw 
    burnSingle(eM, _token0, amountOut0, sender);
    satisfy (balanceBefore == token.balanceOf(eT0, sender));
}

```


The Prover will produce an example that satisfies this condition. Sometimes the example will be uninteresting, such as having `amount == 0` in the example for `possibleToFullyWithdraw`. In such cases we need to strengthen the conditions in order to produce more interesting examples. In `possibleToFullyWithdraw` we added a `require amount > 0;` statement to prevent such a case.

Alternatively, we could have strengthened the `satisfy` condition by adding

```
    satisfy (amount > 0) && ...

```
