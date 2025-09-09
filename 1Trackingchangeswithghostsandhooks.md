Sometimes it is useful for a rule to observe changes to a particular part of storage, or otherwise track the behavior of a contract while it is executing. For example, while verifying an ERC20 contract, we may want to observe each update to any user balance so that we can keep our own running tally of the total supply.

Ghosts and hooks are designed for this purpose. Ghosts are additional variables that you can add to use during verification. They are similar to contract storage variables — they are rolled back when a contract function reverts or when the storage is reset.

Hooks are blocks of CVL code that get executed when a contract performs a certain instruction. For example, a store hook will be run whenever the contract updates a storage variable, while a call hook will be run whenever the contract makes an external call.

Together, hooks and ghosts let you keep track of what the contract does during execution.

Ghosts are a way of defining additional variables for use during verification. These variables are often used to

*   communicate information between [Rules](rules.html#rules-main) and [Hooks](hooks.html#hooks).
    
*   define deterministic [expression summaries](methods.html#expression-summary).
    

Ghosts can be seen as an ‘extension’ to the state of the contracts under verification. This means that in case a call reverts, the ghost values will revert to their pre-state. Additionally, if an unresolved call is handled by a havoc, the ghost values will havoc as well. Ghosts are regarded as part of the state of the contracts, and when calls are invoked with `at storageVar` statements (see [The storage type](types.html#storage-type)), they are restored to their state as saved in `storageVar`. An exception to this rule are ghosts marked _persistent_. Persistent ghosts are **never** havoced, and **never** reverted. See [Ghosts vs. persistent ghosts](#persistent-ghosts) below for more details and examples.

## [Syntax](#id6)[](#syntax "Link to this heading")

The syntax for ghost declarations is given by the following [EBNF grammar](overview.html#ebnf-syntax):

ghost ::= "ghost" type id                             (";" | "{" axioms "}")
        | "ghost" id "(" cvl\_types ")" "returns" type (";" | "{" axioms "}")

persistent\_ghost ::=  "persistent" "ghost" type id                             (";" | "{" axioms "}")
                    | "persistent" "ghost" id "(" cvl\_types ")" "returns" type (";" | "{" axioms "}")

type ::= basic\_type
       | "mapping" "(" cvl\_type "=>" type ")"

axioms ::= axiom \[ axioms \]

axiom ::= \[ "init\_state" \] "axiom" expression ";"

See [Types](types.html) for the `basic_type` and `cvl_type` productions, and [Expressions](expr.html) for the `expression` syntax.

## [Declaring ghost variables](#id7)[](#declaring-ghost-variables "Link to this heading")

Ghost variables must be declared at the top level of a specification file. A ghost variable declaration includes the keyword `ghost` followed by the type and name of the ghost variable.

The type of a ghost variable may be either a [CVL type](types.html) or a `mapping` type. Mapping types are similar to solidity mapping types. They must have CVL types as keys, but may contain either CVL types or mapping types as values.

For example, the following are valid ghost declarations:

ghost uint x;
ghost mapping(address \=> mathint) balances;
ghost mapping(uint \=> mapping(uint \=> mathint)) delegations;

while the following are invalid:

ghost (uint, uint) x;                              // tuples are not CVL types
ghost mapping(mapping(uint \=> uint) \=> address) y; // mappings cannot be keys

*   [A simple `ghost` variable example](https://github.com/Certora/Examples/blob/f1be39c8ac49e5af9b2d450673dde5c1bc6257f2/DEFI/LiquidityPool/certora/specs/Full.spec#L46)

```
/***
This example is a full spec for LiquidityPool.
To run this use Certora cli with the conf file runFullPoll.conf
Example of a run: https://prover.certora.com/output/1512/b84b2123fc1f447ba6cff06d8e07552c?anonymousKey=9917501bc57d897a7ec341a2521b30d92237f95d
UnsatCores: https://prover.certora.com/output/1512/ce180e9d91464a3a9271cb5bf7119125/UnsatCoreVisualisation.html?anonymousKey=88059d4e9f56250f609546f0b77ebc3ed819509d
Mutation test for this spec: https://mutation-testing.certora.com/?id=66c71fdd-9a1d-44e4-b084-d8d4c3de9e61&anonymousKey=e157a2be-ed9d-4d30-90bb-06b6bee05daf
See https://docs.certora.com for a complete guide.
***/

using Asset as underlying;
using TrivialReceiver as _TrivialReceiver;

methods
{
    function balanceOf(address)                      external returns(uint256) envfree;
    function totalSupply()                           external returns(uint256) envfree;
    function transfer(address, uint256)              external returns(bool);
    function transferFrom(address, address, uint256) external returns(bool);
    function amountToShares(uint256)                 external returns(uint256) envfree;
    function sharesToAmount(uint256)                 external returns(uint256) envfree;
    function depositedAmount()                       external returns(uint256) envfree;
    function deposit(uint256)                        external returns(uint256);
    function withdraw(uint256)                       external returns(uint256);
    function calcPremium(uint256)                    external returns (uint256) envfree;

    function _.executeOperation(uint256,uint256,address) external => DISPATCHER(true);
    function flashLoan(address, uint256)                 external;

    function underlying.balanceOf(address)               external returns(uint256) envfree;
    function underlying.allowance(address, address)      external returns(uint256) envfree;
    function underlying.totalSupply()                    external returns(uint256) envfree;
}

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Reentrancy ghost and hook                                                                                           │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/
definition lock_on() returns bool = ghostReentrancyStatus == 2;
definition poll_functions(method f) returns bool = f.selector == sig:withdraw(uint256).selector ||
                                      f.selector == sig:deposit(uint256).selector ||
                                      f.selector == sig:flashLoan(address, uint256).selector;


ghost uint256 ghostReentrancyStatus;
ghost bool lock_status_on_call;

hook Sload uint256 status currentContract._status {
    require ghostReentrancyStatus == status;
}

hook Sstore currentContract._status uint256 status {
    ghostReentrancyStatus = status;
}

// we are hooking here on "CALL" opcodes in order to simulate reentrancy to a non-view function and check that the function reverts
hook CALL(uint g, address addr, uint value, uint argsOffset, uint argsLength, uint retOffset, uint retLength) uint rc {
    lock_status_on_call = lock_on(); 
}

// this rule prove the assumption e.msg.sender != currentContract;
rule reentrancyCheck(env e, method f, calldataarg args) filtered{f -> poll_functions(f)}{
    bool lockBefore = lock_on();
    
    f(e, args);
    
    bool lockAfter = lock_on();
    
    assert !lockBefore && !lockAfter;
    assert lock_status_on_call;
}

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Deposite                                                                                                            │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/
rule depositIntegrity(env e){

    require e.msg.sender != currentContract; // this assumption must hold to avoid shares dilute attack

    uint256 amount;
    uint256 clientBalanceBefore = underlying.balanceOf(e.msg.sender);
    uint256 clientSharesBefore = balanceOf(e.msg.sender);

    uint256 depositedShares = deposit(e, amount);

    uint256 clientBalanceAfter = underlying.balanceOf(e.msg.sender);
    uint256 clientSharesAfter = balanceOf(e.msg.sender);

    assert (amount == 0) => (depositedShares == 0) && (clientBalanceBefore == clientBalanceAfter) && (clientSharesBefore == clientSharesAfter);
    assert (amount > 0) => (clientBalanceBefore - amount == clientBalanceAfter) && (clientSharesBefore + depositedShares == clientSharesAfter);
}

rule depositRevertConditions(env e){

    require e.msg.sender != currentContract; // this assumption must hold to avoid shares dilute attack

    uint256 amount;
    uint256 clientBalanceBefore = underlying.balanceOf(e.msg.sender);
    uint256 clientSharesBefore = balanceOf(e.msg.sender);
    
    bool underFlow = clientBalanceBefore - amount < 0;
    bool emptyPool = totalSupply() == 0 || depositedAmount() == 0;
    bool clientSharesOverflow = (clientSharesBefore + amount > max_uint256 && emptyPool) || clientSharesBefore + amountToShares(amount) > max_uint256;
    bool totalSharesOverflow = totalSupply() + amountToShares(amount) > max_uint256;
    bool contractUnderlyingOverflow = underlying.balanceOf(currentContract) + amount > max_uint256 || depositedAmount() + amount > max_uint256;
    bool overflow =  clientSharesOverflow || totalSharesOverflow || contractUnderlyingOverflow;
    bool payable = e.msg.value != 0;
    bool reentrancy = lock_on();
    bool notEnoughAllowance = underlying.allowance(e.msg.sender, currentContract) < amount;
    bool zeroShares = amountToShares(amount) == 0 && !emptyPool;
    bool expectedRevert = underFlow || overflow || payable || reentrancy || notEnoughAllowance || zeroShares;

    deposit@withrevert(e, amount);

    assert lastReverted <=> expectedRevert;
}

rule depositGreaterThanZeroWithMinted(env e) {
    uint256 amount;
    require amount > 0;
    uint256 amountMinted = deposit(e, amount);
    
    assert amount > 0 <=> amountMinted > 0;
}

rule splitDepositFavoursTheContract(env e){
    uint256 wholeAmount;
    uint256 amountA; 
    uint256 amountB;
    require amountA + amountB == wholeAmount;
    requireInvariant totalSharesIsZeroWithUnderlyingDeposited();

    storage init = lastStorage;

    uint256 wholeShares = deposit(e, wholeAmount);

    uint256 sharesA = deposit(e, amountA) at init;
    uint256 sharesB = deposit(e, amountB);

    assert wholeShares >= sharesA + sharesB;
}

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Withdraw                                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/
rule withdrawIntegrity(env e){

    require e.msg.sender != currentContract; // this assumption must hold to avoid shares dilute attack

    uint256 shares;
    uint256 clientBalanceBefore = underlying.balanceOf(e.msg.sender);
    uint256 clientSharesBefore = balanceOf(e.msg.sender);

    uint256 withdrawAmount = withdraw(e, shares);

    uint256 clientBalanceAfter = underlying.balanceOf(e.msg.sender);
    uint256 clientSharesAfter = balanceOf(e.msg.sender);


    assert (shares == 0) => (withdrawAmount == 0) && (clientBalanceBefore == clientBalanceAfter) && (clientSharesBefore == clientSharesAfter);
    assert (shares > 0) => (clientBalanceBefore + withdrawAmount == clientBalanceAfter) && (clientSharesBefore - shares == clientSharesAfter);
}

rule withdrawRevertConditions(env e){

    require e.msg.sender != currentContract; // this assumption must hold to avoid shares dilute attack

    uint256 amount;
    uint256 clientBalanceBefore = underlying.balanceOf(e.msg.sender);
    uint256 clientSharesBefore = balanceOf(e.msg.sender);
    
    bool clientBalanceUnderflow = clientSharesBefore - amount < 0;
    bool poolUnderflow = underlying.balanceOf(currentContract) - sharesToAmount(amount) < 0 || totalSupply() - amount < 0;
    bool underflow = clientBalanceUnderflow || poolUnderflow;
    bool overflow = clientBalanceBefore + sharesToAmount(amount) > max_uint256;
    bool payable = e.msg.value != 0;
    bool reentrancy = lock_on();
    bool notEnoughAllowance = underlying.allowance(currentContract, currentContract) < sharesToAmount(amount);
    bool zeroAmount = sharesToAmount(amount) == 0;
    bool poolIsEmpty = underlying.balanceOf(currentContract) == 0;
    bool expectedRevert = poolIsEmpty || underflow || overflow || payable || reentrancy || notEnoughAllowance || zeroAmount;

    withdraw@withrevert(e, amount);

    assert lastReverted <=> expectedRevert;
}

rule splitWithdrawFavoursTheContract(env e){
    uint256 wholeShares;
    uint256 sharesA; 
    uint256 sharesB;
    require sharesA + sharesB == wholeShares;

    storage init = lastStorage;

    uint256 wholeAmount = withdraw(e, wholeShares);

    uint256 amountA = withdraw(e, sharesA) at init;
    uint256 amountB = withdraw(e, sharesB);

    assert wholeAmount >= amountA + amountB;
}


/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ FlashLoan, need to ask Nurit for interesting rules, pretty much depended on implementation                          │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/
rule flashLoanIntegrity(env e){
    require e.msg.sender != currentContract; // this assumption must hold to avoid shares dilute attack

    address receiver;
    uint256 amount;

    uint256 contractUnderlyingBalanceBefore  = underlying.balanceOf(currentContract);
    uint256 contractSharesBefore = balanceOf(currentContract);

    flashLoan(e, receiver, amount);

    uint256 contractUnderlyingBalanceAfter = underlying.balanceOf(currentContract);
    uint256 contractSharesAfter = balanceOf(currentContract);

    assert (amount == 0) => contractUnderlyingBalanceBefore == contractUnderlyingBalanceAfter;
    assert (amount > 0) => contractUnderlyingBalanceBefore < contractUnderlyingBalanceAfter;
    assert contractSharesBefore == contractSharesAfter;
}

rule flashLoanRevertConditions(env e){
    require e.msg.sender != currentContract; // this assumption must hold to avoid shares dilute attack

    address receiver;
    uint256 amount;
    
    bool noPremium = calcPremium(amount) == 0;
    bool receiverIsNotIFlashloanAddress = receiver != _TrivialReceiver;\
    bool payable = e.msg.value != 0;
    bool reentrancy = lock_on();
    bool clientUnderflow = underlying.balanceOf(e.msg.sender) - calcPremium(amount) < 0;
    bool poolUnderflow = underlying.balanceOf(currentContract) - amount < 0 || depositedAmount() - amount < 0;
    bool underflow = clientUnderflow || poolUnderflow;
    bool poolBlanceOverflow = underlying.balanceOf(currentContract) + calcPremium(amount) > max_uint256 || depositedAmount() + calcPremium(amount) > max_uint256;
    bool clientBalanceOverflow = underlying.balanceOf(e.msg.sender) + amount > max_uint256;
    bool overflow = poolBlanceOverflow || clientBalanceOverflow;
    bool notEnoughAllowance = underlying.allowance(e.msg.sender, currentContract) < calcPremium(amount) + amount || underlying.allowance(currentContract, currentContract) < amount;
    bool isExpectedToRevert = notEnoughAllowance || overflow || underflow || noPremium || receiverIsNotIFlashloanAddress || payable || reentrancy;

    flashLoan@withrevert(e, receiver, amount);

    assert isExpectedToRevert <=> lastReverted;
}

/// Validates that a flash loan generates yield by increase the value of each share (modulo rounding)
rule flashLoansGenerateYield(address receiver, uint amount) {
    env e;
    require e.msg.sender != currentContract;
    
    uint assetsBefore = depositedAmount();
    uint sharesBefore = totalSupply();
    flashLoan(e, receiver, amount);
    uint assetsAfter = depositedAmount();
    uint sharesAfter = totalSupply();

    // The total assets held by the contract must increase, while the number of shares remains constant.
    // The yield might not be reflected in sharesToAmount due to the rounding.

    assert assetsAfter > assetsBefore;
    assert sharesAfter == sharesBefore;
}

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Find and show a path for each method.                                                                               │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/
rule reachability(method f)
{
	env e;
	calldataarg args;
	f(e,args);
	satisfy true;
}

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ The total shares supply of the system is less than equal the underlying asset holding of the system                │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/
invariant totalSharesLessThanUnderlyingBalance()
    totalSupply() <= underlying.balanceOf(currentContract)
    {
        preserved with(env e) {
            require e.msg.sender != currentContract;
            requireInvariant totalSharesLessThanDepositedAmount();
            requireInvariant depositedAmountLessThanContractUnderlyingAsset();
        }
    }

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ The total shares supply of the system is less than equal the deposit amount                                         │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/
invariant totalSharesLessThanDepositedAmount()
    totalSupply() <= depositedAmount();

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ The deposited Amount of the system is less than equal the contract underlying asset                                 │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/
invariant depositedAmountLessThanContractUnderlyingAsset()
    depositedAmount() <= underlying.balanceOf(currentContract)
    {
        preserved with(env e) {
            require e.msg.sender != currentContract;
        }
    }

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ The total shares supply of the system is zero if and only if the underlying asset holding of the system is zero    │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/
invariant totalSharesIsZeroWithUnderlyingDeposited()
		totalSupply() == 0 <=> depositedAmount() == 0;

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Sum of shares equal total shares supply                                                                             │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/

ghost mathint sumOfShares {
    init_state axiom sumOfShares == 0;
}

hook Sstore _balanceOf[KEY address user] uint256 newSharesBalance (uint256 oldSharesBalance)
{
    sumOfShares = sumOfShares + newSharesBalance - oldSharesBalance;
}

invariant totalSharesEqualSumOfShares()
		totalSupply() == sumOfShares;


/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Sum of underlying equal total supply                                                                             │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/

ghost mathint sumBalances {
    init_state axiom sumBalances == 0;
}

hook Sstore underlying._balanceOf[KEY address user] uint256 newBalance (uint256 oldBalance) {
    sumBalances = sumBalances + newBalance - oldBalance;
}

invariant totalIsSumBalances()
    underlying.totalSupply() == sumBalances;

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Client shares and balance anti-monotonicity ((increase and decrease) or (decrease and increase))                    │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/
rule sharesAndBalanceConsistency(env e, method f) filtered {
    f -> f.selector != sig:transfer(address,uint256).selector &&
    f.selector != sig:transferFrom(address,address,uint256).selector
    } {
    require e.msg.sender != currentContract; // this assumption must hold to avoid shares dilute attack

    uint256 UnderlyingBalanceBefore = underlying.balanceOf(e.msg.sender);
    uint256 SharesBefore = balanceOf(e.msg.sender);
    
    calldataarg args;
    f(e, args);
    
    uint256 UnderlyingBalanceAfter = underlying.balanceOf(e.msg.sender);
    uint256 SharesAfter = balanceOf(e.msg.sender);
    
    assert UnderlyingBalanceBefore < UnderlyingBalanceAfter <=> SharesBefore > SharesAfter;
}

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ More shares leads to bigger withdraw                                                                                │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/
rule moreSharesMoreWithdraw(env e) {
    uint256 sharesX;
    uint256 sharesY;
    uint256 amountX;
    uint256 amountY;

    storage init = lastStorage;
    
    amountX = withdraw(e, sharesX);
    amountY = withdraw(e, sharesY) at init;
    
    assert sharesX > sharesY => amountX >= amountY;
}

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Rounding favours the house                                                                                          │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/
rule amountRoundingTripFavoursContract(env e) {
    requireInvariant totalSharesIsZeroWithUnderlyingDeposited();

    uint256 clientAmountBefore = underlying.balanceOf(e.msg.sender);
    uint256 contractAmountBefore = underlying.balanceOf(currentContract);

    uint256 clientShares = deposit(e, clientAmountBefore);
    uint256 clientAmountAfter = withdraw(e, clientShares);
    uint256 contractAmountAfter = underlying.balanceOf(currentContract);

    assert clientAmountBefore >= clientAmountAfter;
    assert contractAmountBefore <= contractAmountAfter;
}

/*
Bug detected, if contract deploy with balance it can be drained.
Bug fixed by calculating deposit amount and use in in the conversions instead of the contract underlying asset.
*/
rule sharesRoundingTripFavoursContract(env e) {
    uint256 clientSharesBefore = balanceOf(e.msg.sender);
    uint256 contractSharesBefore = balanceOf(currentContract);

    requireInvariant totalSharesLessThanDepositedAmount();
    require e.msg.sender != currentContract; // this assumption must hold to avoid shares dilute attack

    uint256 depositedAmount = depositedAmount();

    uint256 clientAmount = withdraw(e, clientSharesBefore);
    uint256 clientSharesAfter = deposit(e, clientAmount);
    uint256 contractSharesAfter = balanceOf(currentContract);

    /* 
    if client is last and first depositor he will get more shares
    but still he wont be able to withdraw more underlying asset than deposited amount (which in that case its only his deposit) 
    as proved in noClientHasSharesWithMoreValueThanDepositedAmount invariant.
    */ 
    assert (clientAmount == depositedAmount) => clientSharesBefore <= clientSharesAfter; 
    
    // all other states
    assert (clientAmount < depositedAmount) => clientSharesBefore >= clientSharesAfter;
    assert contractSharesBefore <= contractSharesAfter;
}

/*
  prove bug fix
*/
invariant noClientHasSharesWithMoreValueThanDepositedAmount(address a)
        totalSupply() == 0 || sharesToAmount(balanceOf(a)) <= depositedAmount()
		{
			preserved with(env e) {
				require balanceOf(a) + balanceOf(e.msg.sender) < totalSupply();
			}
            preserved transferFrom(address sender, address recipient, uint256 amount) with (env e) {
                require balanceOf(sender) + balanceOf(e.msg.sender) + balanceOf(recipient) < totalSupply();
            }
		}
/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Conversions are monotonic increasing functions.                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/

rule amountToSharesConversion(env e){
    uint256 amountA;
    uint256 amountB;
    assert amountA <= amountB => amountToShares(amountA) <= amountToShares(amountB);
}

rule sharesToAmountConversion(env e){
    uint256 sharesA;
    uint256 sharesB;
    assert sharesA <= sharesB => sharesToAmount(sharesA) <= sharesToAmount(sharesB);
}

rule calculatePremium(env e){
    uint256 amountA;
    uint256 amountB;
    assert amountA <= amountB => calcPremium(amountA) <= calcPremium(amountB);
}

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ ThirdParty not affected.                                                                                            │                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/
rule thirdPartyNotAffected(env e, method f, calldataarg args) filtered {
    f -> f.selector != sig:transfer(address,uint256).selector &&
    f.selector != sig:transferFrom(address,address,uint256).selector
    }{
    address thirdParty;

    require thirdParty != currentContract && thirdParty != e.msg.sender; 

    uint256 thirdPartyBalanceBefore = underlying.balanceOf(thirdParty);
    uint256 thirdPartySharesBefore = balanceOf(thirdParty);

    f(e, args);

    uint256 thirdPartyBalanceAfter = underlying.balanceOf(thirdParty);
    uint256 thirdPartySharesAfter = balanceOf(thirdParty);

    assert (thirdPartyBalanceAfter == thirdPartyBalanceBefore);
    assert (thirdPartySharesAfter == thirdPartySharesBefore);
}
```

    
*   [This example](https://github.com/Certora/Examples/blob/f1be39c8ac49e5af9b2d450673dde5c1bc6257f2/DEFI/ERC20/certora/specs/ERC20Fixed.spec#L99) has an `init_state` axiom


```
/***
 * # ERC20 Example
 *
 * This is an example specification for a generic ERC20 contract.
 */

methods {
    function balanceOf(address)         external returns(uint) envfree;
    function allowance(address,address) external returns(uint) envfree;
    function totalSupply()              external returns(uint) envfree;
    function add(uint256 x, uint256 y)  external returns(uint256) envfree;          
}

//// ## Part 1: Basic Rules ////////////////////////////////////////////////////

/// Transfer must move `amount` tokens from the caller's account to `recipient`
rule transferSpec {
    address sender; address recip; uint amount;

    env e;
    require e.msg.sender == sender;

    mathint balance_sender_before = balanceOf(sender);
    mathint balance_recip_before = balanceOf(recip);

    transfer(e, recip, amount);

    mathint balance_sender_after = balanceOf(sender);
    mathint balance_recip_after = balanceOf(recip);

    require sender != recip;

    assert balance_sender_after == balance_sender_before - amount,
        "transfer must decrease sender's balance by amount";

    assert balance_recip_after == balance_recip_before + amount,
        "transfer must increase recipient's balance by amount";
}



/// Transfer must revert if the sender's balance is too small
rule transferReverts {
    env e; address recip; uint amount;

    require balanceOf(e.msg.sender) < amount;

    transfer@withrevert(e, recip, amount);

    assert lastReverted,
        "transfer(recip,amount) must revert if sender's balance is less than `amount`";
}


/// Transfer shouldn't revert unless
///  the sender doesn't have enough funds,
///  or the message value is nonzero,
///  or the recipient's balance would overflow,
///  or the message sender is 0,
///  or the recipient is 0
///
/// @title Transfer doesn't revert
rule transferDoesntRevert {
    env e; address recipient; uint amount;

    require balanceOf(e.msg.sender) > amount;
    require e.msg.value == 0;
    require balanceOf(recipient) + amount < max_uint;
    require e.msg.sender != 0;
    require recipient != 0;

    transfer@withrevert(e, recipient, amount);
    assert !lastReverted;
}

//// ## Part 2: Parametric Rules ///////////////////////////////////////////////

/// If `approve` changes a holder's allowance, then it was called by the holder
rule onlyHolderCanChangeAllowance {
    address holder; address spender;

    mathint allowance_before = allowance(holder, spender);
    method f; env e; calldataarg args; // was: env e; uint256 amount;
    f(e, args);                        // was: approve(e, spender, amount);

    mathint allowance_after = allowance(holder, spender);

    assert allowance_after > allowance_before => e.msg.sender == holder,
        "approve must only change the sender's allowance";

    assert allowance_after > allowance_before =>
        (f.selector == sig:approve(address,uint).selector || f.selector == sig:increaseAllowance(address,uint).selector),
        "only approve and increaseAllowance can increase allowances";
}

//// ## Part 3: Ghosts and Hooks ///////////////////////////////////////////////

persistent ghost mathint sum_of_balances {
    init_state axiom sum_of_balances == 0;
}

hook Sstore _balances[KEY address a] uint new_value (uint old_value) {
    // when balance changes, update ghost
    sum_of_balances = sum_of_balances + new_value - old_value;
}

// This `sload` makes `sum_of_balances >= balance` hold at the beginning of each rule.
hook Sload uint256 balance _balances[KEY address a]  {
  require sum_of_balances >= balance;
}

//// ## Part 4: Invariants

/** `totalSupply()` returns the sum of `balanceOf(u)` over all users `u`. */
invariant totalSupplyIsSumOfBalances()
    totalSupply() == sum_of_balances;

// satisfy examples
// Generate an example trace for a first deposit operation that succeeds.
rule satisfyFirstDepositSucceeds(){
    env e;
    require totalSupply() == 0;
    deposit(e);
    satisfy totalSupply() == e.msg.value;
}

// Generate an example trace for a withdraw that results totalSupply == 0.
rule satisfyLastWithdrawSucceeds() {
    env e;
    uint256 amount;
    requireInvariant totalSupplyIsSumOfBalances();
    require totalSupply() > 0;
    withdraw(e, amount);
    satisfy totalSupply() == 0;
}

// A witness with several function calls.
rule satisfyWithManyOps(){
    env e; env e1; env e2; env e3;
    address recipient; uint amount;

    requireInvariant totalSupplyIsSumOfBalances();
    // The following two requirement are to avoid overflow exmaples.
    require balanceOf(e.msg.sender) > e.msg.value + 10 * amount;
    require balanceOf(recipient) + amount < max_uint;
    require e.msg.sender != 0;
    require recipient != 0;
    deposit(e1);
    depositTo(e2, recipient, amount);
    transfer(e3, recipient, amount);
    assert totalSupply() > 0;  
}



// A non-vacuous example where transfer() does not revert.
rule satisfyVacuityCorrection {
    env e; address recip; uint amount;

    require balanceOf(e.msg.sender) > 0;

    transfer(e, recip, amount);

    satisfy balanceOf(e.msg.sender) == 0;
}
```

    
*   [A `ghost mapping` example](https://github.com/Certora/Examples/blob/f1be39c8ac49e5af9b2d450673dde5c1bc6257f2/CVLByExample/Types/Structs/BankAccounts/certora/specs/structs.spec#L119)


```
/**
 * @title Structs Example
 *
 * This is an example reasoning about structs.
 * The spec contains examples for:
 * 1. Referencing a struct and its fields.
 * 2. method block including methods passing structs as arguments and returning structs.
 * 3. method block entry for a default getter.
 * 4. method block entry returning a struct as a tuple.
 * 5. structs in cvl functions - passing and returning.
 * 6. struct as a parameter of preserved function.
 */
 

using Bank as bank;  // bank is the same as currentContract.

methods {
     /// Definition of a user-defined solidity method returning a struct
    function getCustomer(address a) external returns(BankAccountRecord.Customer) envfree;
    /// Definition of a function with struct as an argument 
    function addCustomer(BankAccountRecord.Customer) external envfree;

    function balanceOf(address)        external returns(uint) envfree;
    function balanceOfAccount(address, uint) external returns(uint) envfree;
    function totalSupply()             external returns(uint) envfree;
    function getNumberOfAccounts(address) external returns (uint256) envfree;
    function isCustomer(address) external returns (bool) envfree;
}

/** 
 Comparison of full structs is not supported. Each field should be compared instead.
 Here only the id field is compared because arrays (accounts field) cannot be compared.
 */
function integrityOfCustomerInsertion(BankAccountRecord.Customer c1) returns bool {
    addCustomer(c1);
    BankAccountRecord.Customer c = getCustomer(c1.id);
    return (c.id == c1.id);
}

/**
 Calling a solidity method returning a struct.
 @param a - customer's address
 @param accountId - account number
 */
 function getAccount(address a, uint256 accountInd) returns BankAccountRecord.BankAccount {
    BankAccountRecord.Customer c = bank.getCustomer(a);
    return c.accounts[accountInd];
}

/// returning a struct as a tuple.
function getAccountNumberAndBalance(address a, uint256 accountInd) returns (uint256, uint256) {
    env e;
    BankAccountRecord.Customer c = getCustomer(a);
    BankAccountRecord.BankAccount account = getAccount(e.msg.sender, accountInd);
    return (account.accountNumber, account.accountBalance)  ;
}

/**
 You can define rule parameters of a user defined type.
 */
rule correctCustomerInsertion(BankAccountRecord.Customer c1){
    bool correct = integrityOfCustomerInsertion(c1);
    assert (correct, "Bad customer insertion");
}

/// Example for assigning to a tuple.
rule updateOfBlacklist() {
    env e;
    address user;
    address user1;
    uint256 account;
    uint256 account1;

    uint256 ind = addToBlackList(e, user, account);
    
    user1 = currentContract.blackList[ind].id;
    account1 = currentContract.blackList[ind].account;

    assert (user == user1 && account == account1, "Customer in black list is not the one added.");
}

/// Example for struct parameter and  nested struct member reference
rule witnessForIntegrityOfTransferFromCustomerAccount(BankAccountRecord.Customer c) {
    env e;
    uint256 accountNum;
    address to;
    uint256 toAccount;

    require c.accounts[accountNum].accountBalance > 0;
    transfer(e, to, assert_uint256(c.accounts[accountNum].accountBalance/2), accountNum, toAccount);
    satisfy c.accounts[accountNum].accountBalance < balanceOfAccount(to, toAccount);
}

/// Assignment to a struct. 
/// The term getCustomer(a).id is not supported yet.
rule integrityOfCustomerKeyRule(address a, method f) {
    env e;
    calldataarg args;
    BankAccountRecord.Customer c = getCustomer(a);  
    require c.id == a || c.id == 0;
    f(e,args);
    assert c.id == a || c.id == 0;
}


/// Represent the sum of all accounts of all users
/// sum _customers[a].accounts[i].accountBalance 
persistent ghost mathint sumBalances {
    init_state axiom sumBalances == 0;
}

/// Mirror on a struct _customers[a].accounts[i].accountBalance
persistent ghost mapping(address => mapping(uint256 => uint256)) accountBalanceMirror {
    init_state axiom forall address a. forall uint256 i. accountBalanceMirror[a][i] == 0;
}


/// Number of accounts per user 
ghost mapping(address => uint256) numOfAccounts {
    // assumption: it's infeasible to grow the list to these many elements.
    axiom forall address a. numOfAccounts[a] < max_uint256;
    init_state axiom forall address a. numOfAccounts[a] == 0;
}

/// Store hook to synchronize numOfAccounts with the length of the customers[KEY address a].accounts array.
hook Sstore _customers[KEY address user].accounts.length uint256 newLength {
    if (newLength > numOfAccounts[user])
        require accountBalanceMirror[user][require_uint256(newLength-1)] == 0 ;   
    numOfAccounts[user] = newLength;
}


/// This Sload is required in order to eliminate adding unintializaed account balance to sumBalances.
hook Sload uint256 length _customers[KEY address user].accounts.length {
    require numOfAccounts[user] == length; 
}

/// hook on a complex data structure, a mapping to a struct with a dynamic array
hook Sstore _customers[KEY address a].accounts[INDEX uint256 i].accountBalance uint256 new_value (uint old_value) {
    require  old_value == accountBalanceMirror[a][i]; // Need this inorder to sync on insert of new element  
    sumBalances =  sumBalances + new_value - old_value ;
    accountBalanceMirror[a][i] = new_value;
}

/// Sload on a struct field.
hook Sload uint256 value  _customers[KEY address a].accounts[INDEX uint256 i].accountBalance   {
    // when balance load, safely assume it is less than the sum of all values
    require value <= sumBalances;
    require i <= numOfAccounts[a]-1;
}

/// Non-customers have no account.
invariant emptyAccount(address user) 
     !isCustomer(user) => ( 
        getNumberOfAccounts(user) == 0 &&
         (forall uint256 i. accountBalanceMirror[user][i] == 0 )) ; 

/// struct as a parameter of preserved function.
invariant totalSupplyEqSumBalances()
    totalSupply() == sumBalances 
    {
        preserved addCustomer(BankAccountRecord.Customer c) 
        {
            requireInvariant emptyAccount(c.id);
        }
    }

/// Comparing nativeBalances of current contract.
invariant solvency()
    totalSupply() <= nativeBalances[currentContract] {
        // safely assume that Bank doesn't call itself
        preserved with (env e){ 
            require e.msg.sender != currentContract;
        }
    }
```

    
*   [A nested `ghost mapping` example](https://github.com/Certora/Examples/blob/f1be39c8ac49e5af9b2d450673dde5c1bc6257f2/CVLByExample/Types/Structs/BankAccounts/certora/specs/structs.spec#L113)
    
```
/**
 * @title Structs Example
 *
 * This is an example reasoning about structs.
 * The spec contains examples for:
 * 1. Referencing a struct and its fields.
 * 2. method block including methods passing structs as arguments and returning structs.
 * 3. method block entry for a default getter.
 * 4. method block entry returning a struct as a tuple.
 * 5. structs in cvl functions - passing and returning.
 * 6. struct as a parameter of preserved function.
 */
 

using Bank as bank;  // bank is the same as currentContract.

methods {
     /// Definition of a user-defined solidity method returning a struct
    function getCustomer(address a) external returns(BankAccountRecord.Customer) envfree;
    /// Definition of a function with struct as an argument 
    function addCustomer(BankAccountRecord.Customer) external envfree;

    function balanceOf(address)        external returns(uint) envfree;
    function balanceOfAccount(address, uint) external returns(uint) envfree;
    function totalSupply()             external returns(uint) envfree;
    function getNumberOfAccounts(address) external returns (uint256) envfree;
    function isCustomer(address) external returns (bool) envfree;
}

/** 
 Comparison of full structs is not supported. Each field should be compared instead.
 Here only the id field is compared because arrays (accounts field) cannot be compared.
 */
function integrityOfCustomerInsertion(BankAccountRecord.Customer c1) returns bool {
    addCustomer(c1);
    BankAccountRecord.Customer c = getCustomer(c1.id);
    return (c.id == c1.id);
}

/**
 Calling a solidity method returning a struct.
 @param a - customer's address
 @param accountId - account number
 */
 function getAccount(address a, uint256 accountInd) returns BankAccountRecord.BankAccount {
    BankAccountRecord.Customer c = bank.getCustomer(a);
    return c.accounts[accountInd];
}

/// returning a struct as a tuple.
function getAccountNumberAndBalance(address a, uint256 accountInd) returns (uint256, uint256) {
    env e;
    BankAccountRecord.Customer c = getCustomer(a);
    BankAccountRecord.BankAccount account = getAccount(e.msg.sender, accountInd);
    return (account.accountNumber, account.accountBalance)  ;
}

/**
 You can define rule parameters of a user defined type.
 */
rule correctCustomerInsertion(BankAccountRecord.Customer c1){
    bool correct = integrityOfCustomerInsertion(c1);
    assert (correct, "Bad customer insertion");
}

/// Example for assigning to a tuple.
rule updateOfBlacklist() {
    env e;
    address user;
    address user1;
    uint256 account;
    uint256 account1;

    uint256 ind = addToBlackList(e, user, account);
    
    user1 = currentContract.blackList[ind].id;
    account1 = currentContract.blackList[ind].account;

    assert (user == user1 && account == account1, "Customer in black list is not the one added.");
}

/// Example for struct parameter and  nested struct member reference
rule witnessForIntegrityOfTransferFromCustomerAccount(BankAccountRecord.Customer c) {
    env e;
    uint256 accountNum;
    address to;
    uint256 toAccount;

    require c.accounts[accountNum].accountBalance > 0;
    transfer(e, to, assert_uint256(c.accounts[accountNum].accountBalance/2), accountNum, toAccount);
    satisfy c.accounts[accountNum].accountBalance < balanceOfAccount(to, toAccount);
}

/// Assignment to a struct. 
/// The term getCustomer(a).id is not supported yet.
rule integrityOfCustomerKeyRule(address a, method f) {
    env e;
    calldataarg args;
    BankAccountRecord.Customer c = getCustomer(a);  
    require c.id == a || c.id == 0;
    f(e,args);
    assert c.id == a || c.id == 0;
}


/// Represent the sum of all accounts of all users
/// sum _customers[a].accounts[i].accountBalance 
persistent ghost mathint sumBalances {
    init_state axiom sumBalances == 0;
}

/// Mirror on a struct _customers[a].accounts[i].accountBalance
persistent ghost mapping(address => mapping(uint256 => uint256)) accountBalanceMirror {
    init_state axiom forall address a. forall uint256 i. accountBalanceMirror[a][i] == 0;
}


/// Number of accounts per user 
ghost mapping(address => uint256) numOfAccounts {
    // assumption: it's infeasible to grow the list to these many elements.
    axiom forall address a. numOfAccounts[a] < max_uint256;
    init_state axiom forall address a. numOfAccounts[a] == 0;
}

/// Store hook to synchronize numOfAccounts with the length of the customers[KEY address a].accounts array.
hook Sstore _customers[KEY address user].accounts.length uint256 newLength {
    if (newLength > numOfAccounts[user])
        require accountBalanceMirror[user][require_uint256(newLength-1)] == 0 ;   
    numOfAccounts[user] = newLength;
}


/// This Sload is required in order to eliminate adding unintializaed account balance to sumBalances.
hook Sload uint256 length _customers[KEY address user].accounts.length {
    require numOfAccounts[user] == length; 
}

/// hook on a complex data structure, a mapping to a struct with a dynamic array
hook Sstore _customers[KEY address a].accounts[INDEX uint256 i].accountBalance uint256 new_value (uint old_value) {
    require  old_value == accountBalanceMirror[a][i]; // Need this inorder to sync on insert of new element  
    sumBalances =  sumBalances + new_value - old_value ;
    accountBalanceMirror[a][i] = new_value;
}

/// Sload on a struct field.
hook Sload uint256 value  _customers[KEY address a].accounts[INDEX uint256 i].accountBalance   {
    // when balance load, safely assume it is less than the sum of all values
    require value <= sumBalances;
    require i <= numOfAccounts[a]-1;
}

/// Non-customers have no account.
invariant emptyAccount(address user) 
     !isCustomer(user) => ( 
        getNumberOfAccounts(user) == 0 &&
         (forall uint256 i. accountBalanceMirror[user][i] == 0 )) ; 

/// struct as a parameter of preserved function.
invariant totalSupplyEqSumBalances()
    totalSupply() == sumBalances 
    {
        preserved addCustomer(BankAccountRecord.Customer c) 
        {
            requireInvariant emptyAccount(c.id);
        }
    }

/// Comparing nativeBalances of current contract.
invariant solvency()
    totalSupply() <= nativeBalances[currentContract] {
        // safely assume that Bank doesn't call itself
        preserved with (env e){ 
            require e.msg.sender != currentContract;
        }
    }
```

## [Ghost Functions](#id8)[](#ghost-functions "Link to this heading")

CVL also has support for “ghost functions”. These serve a different purpose from ghost variables, although they can be used in similar ways.

Ghost functions must be declared at the top level of a specification file. A ghost function declaration includes the keyword `ghost` followed by the name and signature of the ghost function. Ghost functions should be used either:

*   when there are no updates to the ghost as the deterministic behavior and axioms are the only properties of the ghost
    
*   when updating the ghost - more than one entry is updated and then the havoc assuming statement is used.
    
    *   [`ghost` function example](https://github.com/Certora/Examples/blob/14668d39a6ddc67af349bc5b82f73db73349ef18/CVLByExample/QuantifierExamples/DoublyLinkedList/certora/spec/dll-linkedcorrectly.spec#L24)
        
```
methods {
    function getValueOf(address) external returns (uint256) envfree;
    function getHead() external returns (address) envfree;
    function getTail() external returns (address) envfree;
    function getNext(address) external returns (address) envfree;
    function getPrev(address) external returns (address) envfree;
    function remove(address) external envfree;
    function insertSorted(address, uint256, uint256) external envfree;
}

// GHOST COPIES
ghost mapping(address => address) ghostNext {
    init_state axiom forall address x. ghostNext[x] == 0;
}
ghost mapping(address => address) ghostPrev {
    init_state axiom forall address x. ghostPrev[x] == 0;
}
ghost mapping(address => uint256) ghostValue {
    init_state axiom forall address x. ghostValue[x] == 0;
}
ghost address ghostHead;
ghost address ghostTail;
ghost uint256 ghostLength;
ghost nextstar(address, address) returns bool {
    init_state axiom forall address x. forall address y. nextstar(x, y) == (x == y);
}
ghost prevstar(address, address) returns bool {
    init_state axiom forall address x. forall address y. prevstar(x, y) == (x == y);
}

// HOOKS

hook Sstore currentContract.dll.head address newHead STORAGE {
    ghostHead = newHead;
}
hook Sstore currentContract.dll.tail address newTail STORAGE {
    ghostTail = newTail;
}

hook Sstore currentContract.dll.accounts[KEY address key].next address newNext STORAGE {
    ghostNext[key] = newNext;
}

hook Sstore currentContract.dll.accounts[KEY address key].prev address newPrev STORAGE {
    ghostPrev[key] = newPrev;
}
hook Sstore currentContract.dll.accounts[KEY address key].value uint256 newValue STORAGE {
    ghostValue[key] = newValue;
}

hook Sload address head currentContract.dll.head STORAGE {
    require ghostHead == head;
}
hook Sload address tail currentContract.dll.tail STORAGE {
    require ghostTail == tail;
}
hook Sload address next currentContract.dll.accounts[KEY address key].next STORAGE {
    require ghostNext[key] == next;
}
hook Sload address prev currentContract.dll.accounts[KEY address key].prev STORAGE {
    require ghostPrev[key] == prev;
}
hook Sload uint256 value currentContract.dll.accounts[KEY address key].value STORAGE {
    require ghostValue[key] == value;
}

// INVARIANTS

invariant nextPrevMatch()
    // either list is empty, and both head and tail are 0,
    ((ghostHead == 0 && ghostTail == 0)
    // or both head and tail are set and their prev resp. next points to 0.
    || (ghostHead != 0 && ghostTail != 0 && ghostNext[ghostTail] == 0 && ghostPrev[ghostHead] == 0
        && ghostValue[ghostHead] != 0 && ghostValue[ghostTail] != 0))
    // for all addresses:
    && (forall address a.
           // either the address is not part of the list and every field is 0.
           (ghostNext[a] == 0 && ghostPrev[a] == 0 && ghostValue[a] == 0)
           // or the address is part of the list, address is non-zero, value is non-zero,
           // and prev and next pointer are linked correctly.
        || (a != 0 && ghostValue[a] != 0
            && ((a == ghostHead && ghostPrev[a] == 0) || ghostNext[ghostPrev[a]] == a)
            && ((a == ghostTail && ghostNext[a] == 0) || ghostPrev[ghostNext[a]] == a)));


invariant inList()
    (ghostHead != 0 => ghostValue[ghostHead] != 0)
    && (ghostTail != 0 => ghostValue[ghostTail] != 0)
    && (forall address a.  ghostNext[a] != 0 => ghostValue[ghostNext[a]] != 0)
    && (forall address a.  ghostPrev[a] != 0 => ghostValue[ghostPrev[a]] != 0)
    {
        preserved {
            requireInvariant nextPrevMatch();
        }
    }

rule insert_preserves_old {
    address newElem;
    address oldElem;
    uint256 newValue;
    uint256 maxIter;
    bool oldInList = ghostValue[oldElem] != 0;

    require oldElem != newElem;

    insertSorted(newElem, newValue, maxIter);

    assert oldInList == (ghostValue[oldElem] != 0);
}

rule insert_adds_new() {
    address newElem;
    uint256 newValue;
    uint256 maxIter;

    insertSorted(newElem, newValue, maxIter);

    assert ghostValue[newElem] != 0;
    assert ghostValue[newElem] == newValue;
}

rule insert_does_not_revert() {
    address newElem;
    uint256 newValue;
    uint256 maxIter;

    require newElem != 0;
    require ghostValue[newElem] == 0;
    require newValue != 0;

    insertSorted@withrevert(newElem, newValue, maxIter);

    assert !lastReverted;
}

rule remove_preserves_old {
    address elem;
    address oldElem;
    bool oldInList = ghostValue[oldElem] != 0;

    require oldElem != elem;

    remove(elem);

    assert oldInList == (ghostValue[oldElem] != 0);
}

rule remove_deletes() {
    address elem;

    remove(elem);

    assert ghostValue[elem] == 0;
}

rule remove_does_not_revert() {
    address elem;

    require elem != 0;
    require ghostValue[elem] != 0;

    remove@withrevert(elem);

    assert !lastReverted;
}
```

## [Restrictions on ghost definitions](#id9)[](#restrictions-on-ghost-definitions "Link to this heading")

*   A user-defined type, such as struct, array or interface is not allowed as the key or the output type of a `ghost mapping`.
    

## [Using ghost variables](#id10)[](#using-ghost-variables "Link to this heading")

While verifying a rule or invariant, the Prover considers every possible initial value of a ghost variable (subject to its [Ghost axioms](#ghost-axioms), see below).

Within CVL, you can read or write ghosts using the normal variable syntax. For example:

ghost mapping(address \=> mathint) balances;

function example(address user) {
    balances\[user\] \= x;
}

You can also use ghost variables in a [`sum` or `usum` expression](expr.html#ghost-mapping-sums) to calculate the total of numeric values in a ghost mapping.

The most common reason to use a ghost is to communicate information from a hook back to the rule that triggered it. For example, the following CVL checks that a call to the contract method `do_update(user)` changes the contract variable `userInfo[user]` and does not change `userInfo[other]` for any other user:

ghost mapping(address \=> bool) updated;

hook Sstore userInfo\[KEY address u\] uint i {
    updated\[u\] \= true;
}

rule update\_changes\_user(address user) {
    updated\[user\] \= false;

    do\_update(user);

    assert updated\[user\] \== true, "do\_update(user) should affect user";
}

rule update\_changes\_no\_other(address user, address other) {
    require user != other;
    require updated\[other\] \== false;

    do\_update(user);

    assert updated\[other\] \== false;
}

Here the `updated` ghost is used to communicate information from the `userInfo` hook back to the `updated_changes_user` and `updated_changes_no_other` rules.

## [Ghost axioms](#id11)[](#ghost-axioms "Link to this heading")

Ghost axioms are properties that the Prover assumes whenever it makes use of a ghost.

### [Global axioms](#id12)[](#global-axioms "Link to this heading")

Sometimes we might want to constrain the behavior of a ghost in some particular way. In CVL this is achieved by writing axioms. Axioms are simply CVL expressions that the tool will then assume are true about the ghost. For example:

ghost bar(uint256) returns uint256 {
    axiom forall uint256 x. bar(x) \> 10;
}

In any rule that uses bar, no application of bar could ever evaluate to a number less than or equal to 10.

*   [`axiom` example](https://github.com/Certora/Examples/blob/14668d39a6ddc67af349bc5b82f73db73349ef18/CVLByExample/structs/BankAccounts/certora/specs/Bank.spec#L119)
    
```
/**
 * @title Structs Example
 * This is an example of reasoning about structs.
 * The spec contains examples for:
 * - referencing a struct and its fields.
 * - method block including methods passing structs as arguments and returning structs.
 * - method block entry for a default getter.
 * - method block entry returning a struct as a tuple.
 * - structs in cvl functions - passing and returning.
 * - struct as a parameter of preserved function.
 */
 

using Bank as bank;  // bank is the same as currentContract.

methods {
     /// Definition of a user-defined solidity method returning a struct
    function getCustomer(address a) external returns(BankAccountRecord.Customer) envfree;
    /// Definition of a compiler-generated method returning a struct as a tuple 
    function blackList(uint256) external returns (address, uint) envfree;
    /// Definition of a function with struct as an argument 
    function addCustomer(BankAccountRecord.Customer) external envfree;

    function balanceOf(address)        external returns(uint) envfree;
    function balanceOfAccount(address, uint) external returns(uint) envfree;
    function totalSupply()             external returns(uint) envfree;
    function getNumberOfAccounts(address) external returns (uint256) envfree;
    function isCustomer(address) external returns (bool) envfree;
}

/** 
 Comparison of full structs is not supported. Instead, each field should be compared.
 Here, only the id field is compared because arrays (`accounts` field) cannot be compared.
 */
function integrityOfCustomerInsertion(BankAccountRecord.Customer c1) returns bool {
    addCustomer(c1);
    BankAccountRecord.Customer c = getCustomer(c1.id);
    return (c.id == c1.id);
}

/**
 Calling a solidity method returning a struct.
 @param a - customer's address
 @param accountId - account number
 */
 function getAccount(address a, uint256 accountInd) returns BankAccountRecord.BankAccount {
    BankAccountRecord.Customer c = getCustomer(a);
    return c.accounts[accountInd];
}

/// returning a struct as a tuple.
function getAccountNumberAndBalance(address a, uint256 accountInd) returns (uint256, uint256) {
    env e;
    BankAccountRecord.Customer c = getCustomer(a);
    BankAccountRecord.BankAccount account = getAccount(e.msg.sender, accountInd);
    return (account.accountNumber, account.accountBalance)  ;
}

/**
 You can define rule parameters of a user defined type.
 */
rule correctCustomerInsertion(BankAccountRecord.Customer c1){
    bool correct = integrityOfCustomerInsertion(c1);
    assert (correct, "Bad customer insertion");
}

/// Example for assigning to a tuple.
rule updateOfBlacklist() {
    env e;
    address user;
    address user1;
    uint256 account;
    uint256 account1;

    uint256 ind = addToBlackList(e, user, account);
    user1, account1 = blackList(ind);
    assert (user == user1 && account == account1, "Customer in black list is not the one added.");
}

/// Example for struct parameter and  nested struct member reference
rule witnessForIntegrityOfTransferFromCustomerAccount(BankAccountRecord.Customer c) {
    env e;
    uint256 accountNum;
    address to;
    uint256 toAccount;

    require c.accounts[accountNum].accountBalance > 0;
    transfer(e, to, assert_uint256(c.accounts[accountNum].accountBalance/2), accountNum, toAccount);
    satisfy c.accounts[accountNum].accountBalance < balanceOfAccount(to, toAccount);
}

/// Assignment to a struct. 
/// The term getCustomer(a).id is not supported yet.
rule integrityOfCustomerKeyRule(address a, method f) {
    env e;
    calldataarg args;
    BankAccountRecord.Customer c = getCustomer(a);  
    require c.id == a || c.id == 0;
    f(e,args);
    assert c.id == a || c.id == 0;
}


/// Represent the sum of all accounts of all users
/// sum _customers[a].accounts[i].accountBalance 
ghost mathint sumBalances {
    init_state axiom sumBalances == 0;
}

/// Mirror on a struct _customers[a].accounts[i].accountBalance
ghost mapping(address => mapping(uint256 => uint256)) accountBalanceMirror {
    init_state axiom forall address a. forall uint256 i. accountBalanceMirror[a][i] == 0;
}


/// Number of accounts per user 
ghost mapping(address => uint256) numOfAccounts {
    // assumption: it's infeasible to grow the list to these many elements.
    axiom forall address a. numOfAccounts[a] < max_uint256;
    init_state axiom forall address a. numOfAccounts[a] == 0;
}

/// Store hook to synchronize numOfAccounts with the length of the customers[KEY address a].accounts array.
/// We need to use (offset 32) here, as there is no keyword yet to access the length.
hook Sstore _customers[KEY address user].(offset 32) uint256 newLength STORAGE {
    if (newLength > numOfAccounts[user])
        require accountBalanceMirror[user][require_uint256(newLength-1)] == 0 ;   
    numOfAccounts[user] = newLength;
}

/**
 An internal step check to verify that our ghost works as expected, it should mirror the number of accounts.
 Note: Once this rule is proven, it is safe to have this as a require on the sload .
 Once the sload is defined, this invariant becomes a tautology  
 */
invariant checkNumOfAccounts(address user) 
    numOfAccounts[user] == getNumberOfAccounts(user);

/// This Sload is required in order to eliminate adding unintializaed account balance to sumBlanaces.
/// (offset 32) is the location of the size of the mapping. It is used because the field `size` is not yet supported in cvl.  
hook Sload uint256 length _customers[KEY address user].(offset 32) STORAGE {
    require numOfAccounts[user] == length; 
}

/// hook on a complex data structure, a mapping to a struct with a dynamic array
hook Sstore _customers[KEY address a].accounts[INDEX uint256 i].accountBalance uint256 new_value (uint old_value) STORAGE {
    require  old_value == accountBalanceMirror[a][i]; // Need this inorder to sync on insert of new element  
    sumBalances =  sumBalances + new_value - old_value ;
    accountBalanceMirror[a][i] = new_value;
}

// Sload on a struct field.
hook Sload uint256 value  _customers[KEY address a].accounts[INDEX uint256 i].accountBalance   STORAGE {
    // when balance load, safely assume it is less than the sum of all values
    require to_mathint(value) <= sumBalances;
    require to_mathint(i) <= to_mathint(numOfAccounts[a]-1);
}

/// Non-customers have no account.
invariant emptyAccount(address user) 
     !isCustomer(user) => ( 
        getNumberOfAccounts(user) == 0 &&
         (forall uint256 i. accountBalanceMirror[user][i] == 0 )) ; 

// struct as a parameter of preserved function.
invariant totalSupplyEqSumBalances()
    to_mathint(totalSupply()) == sumBalances 
    {
        preserved addCustomer(BankAccountRecord.Customer c) 
        {
            requireInvariant emptyAccount(c.id);
        }
    }

/// Comparing nativeBalances of current contract.
invariant solvency()
    totalSupply() <= nativeBalances[currentContract] {
        // safely assume that Bank doesn't call itself
        preserved with (env e){ 
            require e.msg.sender != currentContract;
        }
    }
```

### [Initial state axioms](#id13)[](#initial-state-axioms "Link to this heading")

When writing invariants, initial axioms are a way to express the “constructor state” of a ghost function or variable. They are used only when checking the base step of invariants (see [Invariants Overview](invariants.html#invariant-overview) for how invariants are checked). Before checking the initial state of an invariant, the Certora Prover adds a `require` for each `init_state` axiom. `init_state` axioms are not used in rules or the preservation check for invariants.

ghost mathint sumBalances{
    // assuming value zero at the initial state before constructor
    init\_state axiom sumBalances \== 0;
}

*   [initial state axiom example](https://github.com/Certora/Examples/blob/14668d39a6ddc67af349bc5b82f73db73349ef18/CVLByExample/ConstantProductPool/certora/spec/ConstantProductPool.spec#L207)
    
```
/***
This example explains the many features of the Certora Verification Language. 
See https://docs.certora.com for a complete guide.
***/

// reference from the spec to additional contracts used in the verification 
using DummyERC20A as _token0;
using DummyERC20B as _token1;


/*
    Declaration of methods that are used in the rules. `envfree` indicates that
    the method is not dependent on the environment (`msg.value`, `msg.sender`).
    Methods that are not declared here are assumed to be dependent on the
    environment.
*/

methods{
    function token0() external returns (address) envfree;
    function token1() external returns (address) envfree;
    function allowance(address,address) external returns (uint256) envfree;
    function balanceOf(address) external returns (uint256)  envfree; 
    function totalSupply() external returns (uint256)  envfree;
    function getReserve0() external returns (uint256) envfree;
    function getReserve1() external returns (uint256) envfree;
    function swap(address tokenIn, address recipient) external returns (uint256) envfree;
    
    //calls to external contracts  
    function _token0.balanceOf(address account) external returns (uint256) envfree;
    function _token1.balanceOf(address account) external returns (uint256) envfree;
    function _token0.transfer(address, uint) external;
    function _token1.transfer(address, uint) external;

    //external calls to be resolved by dispatcher - taking into account all available implementations 
    function _.transferFrom(address sender, address recipient, uint256 amount) external => DISPATCHER(true);
    function _.balanceOf(address) external => DISPATCHER(true);
    
}

// a cvl function for precondition assumptions 
function setup(env e){
    address zero_address = 0;
    uint256 MINIMUM_LIQUIDITY = 1000;
    require totalSupply() == 0 || balanceOf(zero_address) == MINIMUM_LIQUIDITY;
    require balanceOf(zero_address) + balanceOf(e.msg.sender) <= to_mathint(totalSupply());
    require _token0 == token0();
    require _token1 == token1();
}


/*
Property: For all possible scenarios of swapping token1 for token0, the balance of the recipient is updated as expected. 

This property is implemented as a unit-test style rule. It checks one method but on all possible scenarios.
Note:
 It also takes into account if the recipient is the contract itself, in which case this property does not hold since the balance is unchanged.
As a result, we add a requirement that the recipient is not the currentContract.

This property catches a bug in which there is a switch between the token and the recipient:
        transfer( recipient, tokenOut, amountOut);

Formula:
        { b = _token0.balanceOf(recipient) }
            amountOut := swap(_token1, recipient);
        { _token0.balanceOf(recipient) = b + amountOut }
*/

rule integrityOfSwap(address recipient) {
    env e; /* represents global solidity variables such as msg.sender, block.timestamp */
    setup(e);
    require recipient != currentContract; /* currentContract is a CVL keyword, assigned the main contract under test */  
    uint256 balanceBefore = _token0.balanceOf(recipient);
    uint256 amountOut = swap(_token1, recipient);
    uint256 balanceAfter = _token0.balanceOf(recipient);
    assert to_mathint(balanceAfter) == balanceBefore + amountOut; 
}

/*
Property: Only the user  or an allowed spender can decrease the user's LP balance.

This property is implemented as a parametric rule - it checks all public/external methods of the contract.

This property catches a bug when there is a switch between the token and the recipient in burnSingle:
        transfer( recipient, tokenOut, amountOut);

Formula:
        { b = balanceOf(account), allowance = allowance(account, e.msg.sender) }
            op by e.msg.sender;
        { balanceOf(account) < b =>  (e.msg.sender == account  ||  allowance >= (before-balanceOf(account)) }
*/

rule noDecreaseByOther(method f, address account) {
    env e;
    setup(e);
    require e.msg.sender != account;
    require account != currentContract; 
    uint256 allowance = allowance(account, e.msg.sender); 
    
    uint256 before = balanceOf(account);
    calldataarg args;
    f(e,args); /* check on all possible arguments */
    uint256 after = balanceOf(account);
    /* logic implication : true when: (a) the left hand side is false or (b) right hand side is true  */
    assert after < before =>  (e.msg.sender == account  ||  to_mathint(allowance) >= (before-after))  ;
}


/*
Property: For both token0 and token1 the balance of the system is at least as much as the reserves.

This property is implemented as an invariant. 
Invariants are defined as a specification of a condition that should always be true once an operation is concluded.
In addition, the invariant also checks that it holds immediately after the constructor of the code runs.

This invariant also catches the bug in which there is a switch between the token and the recipient in burnSingle:
        transfer( recipient, tokenOut, amountOut);

Formula:
    getReserve0() <= _token0.balanceOf(currentContract) &&
    getReserve1() <= _token1.balanceOf(currentContract)
*/

invariant balanceGreaterThanReserve()
    (getReserve0() <= _token0.balanceOf(currentContract))&&
    (getReserve1() <= _token1.balanceOf(currentContract))
    {
        preserved with (env e){
         setup(e);
        }
    }


/*
Property: Integrity of totalSupply with respect to the amount of reserves. 

This is a high level property of the system - the ability to pay back liquidity providers.
If there are any LP tokens (the totalSupply is greater than 0), then neither reserves0 nor reserves1 should ever be zero (otherwise the pool could not produce the underlying tokens).

This invariant catches the original bug in Trident where the amount to receive is computed as a function of the balances and not the reserves.

Formula:
    (totalSupply() == 0 <=> getReserve0() == 0) &&
    (totalSupply() == 0 <=> getReserve1() == 0)
*/

invariant integrityOfTotalSupply()
    
    (totalSupply() == 0 <=> getReserve0() == 0) &&
    (totalSupply() == 0 <=> getReserve1() == 0)
    {
        preserved with (env e){
            requireInvariant balanceGreaterThanReserve();
            setup(e);
        }
    }


/*
Property: Monotonicity of mint.

The more tokens a user transfers to the system the more LP tokens that user should receive. 
This property is implemented as a relational property - it compares two different executions on the same state.

This invariant catches a bug in mint where the LP tokens of the first depositor are not computed correctly and the less he transfers the more LP tokens he receives. 

Formula:
    { x > y }
        _token0.transfer(currentContract, x); mint(recipient);
        ~ 
        _token0.transfer(currentContract, y); mint(recipient);
    { balanceOf(recipient) at 1  >=  balanceOf(recipient) at 2  }
*/

rule monotonicityOfMint(uint256 x, uint256 y, address recipient) {
    env eT0;
    env eM;
    setup(eM);
    requireInvariant integrityOfTotalSupply();
    storage init = lastStorage;
    require recipient != currentContract;
    require x > y ;
    _token0.transfer(eT0, currentContract, x);
    uint256 amountOut0 = mint(eM,recipient);
    uint256 balanceAfter1 = balanceOf(recipient);
    
    _token0.transfer(eT0, currentContract, y) at init;
    uint256 amountOut2 = mint(eM,recipient);
    uint256 balanceAfter2 = balanceOf(recipient); 
    assert balanceAfter1 >= balanceAfter2; 
}

/*
Property: Sum of balances

The sum of all balances is equal to the total supply.

This property is implemented with a ghost, an additional variable that tracks changes to the balance mapping.

Formula:
    
    sum(balanceOf(u) for all address u) = totalSupply()

*/

ghost mathint sumBalances{
    // assuming value zero at the initial state before constructor 
	init_state axiom sumBalances == 0; 
}


/* here we state when and how the ghost is updated */
hook Sstore _balances[KEY address a] uint256 new_balance
// the old value that balances[a] holds before the store
    (uint256 old_balance) STORAGE {
  sumBalances = sumBalances + new_balance - old_balance;
}

invariant sumFunds() 
	sumBalances == to_mathint(totalSupply());


/*
Property: Full withdraw example

Give an example demonstrating a case where the user's deposit (transfer or mint) can be fully refunded by burning the liquidity provided.

This property uses the `satisfy` command, which causes the Prover to produce successful examples of expected properties rather than counterexamples.  In particular, this rule does not prove that every deposit can be fully withdrawn.

*/
rule possibleToFullyWithdraw(address sender, uint256 amount) {
    env eT0;
    env eM;
    setup(eM);
    uint256 balanceBefore = _token0.balanceOf(sender);
    
    require eM.msg.sender == sender;
    require eT0.msg.sender == sender;
    require amount > 0;
    _token0.transfer(eT0, currentContract, amount);
    uint256 amountOut0 = mint(eM,sender);
    // immediately withdraw 
    burnSingle(eM, _token0, amountOut0, sender);
    satisfy (balanceBefore == _token0.balanceOf(sender));
}



/*
Property: Zero withdraw has no effect

Withdraw (burn) of zero liquidity provides nothing - all the storage of all the contracts (including the ERC20s) stays the same. 

This property is implemented by saving the storage state before the transaction and comparing  that after the transaction the storage is the same.

*/

rule zeroWithdrawNoEffect(address to) {
    env e;
    setup(e);
    // The assumption is  no skimming 
    require getReserve0() == _token0.balanceOf(currentContract) && getReserve1() == _token1.balanceOf(currentContract);
    storage before = lastStorage;
    burnSingle(e, _token0, 0, to);
    storage after = lastStorage;
    assert before == after;
}
```

## [Restrictions on ghost axioms](#id14)[](#restrictions-on-ghost-axioms "Link to this heading")

*   A ghost axiom cannot refer to Solidity or CVL functions or to other ghosts. It can refer to the ghost itself.
    
*   Since the signature of a ghost contains just parameter types without names, it cannot refer to its parameters. `forall` can be used in order to refer the storage referred to by the parameters. [Example](https://github.com/Certora/Examples/blob/61ac29b1128c68aff7e8d1e77bc80bfcbd3528d6/CVLByExample/summary/ghost-summary/ghost-mapping/certora/specs/WithGhostSummary.spec#L12).
    

## [Ghosts vs. persistent ghosts](#id15)[](#ghosts-vs-persistent-ghosts "Link to this heading")

A `persistent ghost` is a `ghost` that will never be [havoc](../user-guide/glossary.html#glossary). The value of a non-persistent `ghost` will be `havoc'ed` when the Prover `havoc`s the storage, a `persistent ghost` however will keep its value when storage is havoced.

In most cases, non-persistent ghosts are the natural choice for a specification that requires extra tracking of information.

We present two examples where persistent ghosts are useful.

### [Persistent ghosts that survive havocs](#id16)[](#persistent-ghosts-that-survive-havocs "Link to this heading")

In the first example, we want to track the occurrence of a potential reentrant call[\[1\]](#reentrancy):

persistent ghost bool reentrancy\_happened {
    init\_state axiom !reentrancy\_happened;
}

hook CALL(uint g, address addr, uint value, uint argsOffset, uint argsLength, 
          uint retOffset, uint retLength) uint rc {
    if (addr \== currentContract) {
        reentrancy\_happened \= reentrancy\_happened 
                                || executingContract \== currentContract;
    }
}

invariant no\_reentrant\_calls !reentrancy\_happened;

To see why a persistent ghost must be used here for the variable `reentrancy_happened`, consider the following contract:

contract NotReentrant {
    function transfer1Token(IERC20 a) external {
        require (address(a) != address(this));
        a.transfer(msg.sender, 1);
    }
}

If we do not apply any linking or dispatching for the call done on the target `a`, the call to `transfer` would havoc. During a havoc operation, the Prover conservatively assumes that almost any possible behavior can occur. In particular, it must assume that during the execution of the `a.transfer` call, non-persistent ghosts can be updated arbitrarily (e.g. by other contracts), and thus (assuming `reentrancy_happened` were not marked as persistent), the Prover considers the case where `reentrancy_happened` is set to `true` due to the havoc. Thus, when the `CALL` hook executes immediately after, it does so where the `reentrancy_happened` value is already `true`, and thus the value after the hook will remain `true`.

In the lower-level view of the tool, the sequence of events is as follows:

1.  A call to `a.transfer` which cannot be resolved and results in a [havoc](../user-guide/glossary.html#glossary) operation. Non-persistent ghosts are havoced, in particular `reentrancy_happened` if it were not marked as such.
    
2.  A `CALL` hook executes, updating `reentrancy_happened` based on its havoced value, meaning it can turn to true.
    

Therefore even if the addresses of `a` and `NotReentrant` are distinct, we could still falsely detect a reentrant call as `reentrancy_happened` was set to true due to non-determinism. The call trace would show `reentrancy_happened` as being determined to be true due to a havoc in the “Ghosts State” view under “Global State”.

### [Persistent ghosts that survive reverts](#id17)[](#persistent-ghosts-that-survive-reverts "Link to this heading")

In this example, we use persistent ghosts to determine if a revert happened with user-provided data or not. This can help distinguishing between compiler-generated reverts and user-specified reverts (but only in [Solidity versions prior to 0.8.x](https://soliditylang.org/blog/2020/10/28/solidity-0.8.x-preview/)). The idea is to set a ghost to true if a `REVERT` opcode is encountered with a positive buffer size. As in early Solidity versions the panic errors would compile to reverts with empty buffers, as well as user-provided `require`s with no message specified.

persistent ghost bool saw\_user\_defined\_revert\_msg;

hook REVERT(uint offset, uint size) {
    if (size \> 0) {
        saw\_user\_defined\_revert\_msg \= true;
    }
}

rule mark\_methods\_that\_have\_user\_defined\_reverts(method f, env e, calldataarg args) {
    require !saw\_user\_defined\_revert\_msg;

    f@withrevert(e, args);

    satisfy saw\_user\_defined\_revert\_msg;
}

To see why a regular ghost cannot be used to implement this rule, let’s consider the following trivial contract:

contract Reverting {
	function noUserDefinedRevertFlows(uint a, uint b) external {
		uint c \= a/b; // will see a potential division-by-zero revert
		uint\[\] memory arr \= new uint\[\](1);
		uint d \= arr\[a+b\]; // will see a potential out-of-bounds access revert;
	}

	function userDefinedRequireMsg(uint a) external {
		require(a != 0, "a != 0");
	}

	function emptyRequire(uint a) external {
		require(a != 0);
	}
}

It is expected for the method `userDefinedRequireMsg` to satisfy the rule, and it should be the only method to satisfy it. Assuming `saw_user_defined_revert_msg` was defined as a regular, non-persistent ghost, the rule would not be satisfied for `userDefinedRequireMsg`: in case the input argument `a` is equal to 0, the contract reverts, and the value of `saw_user_defined_revert_msg` is reset to its value before the call, which must be `false` (because the rule required it before the call). In this case, after the call `saw_user_defined_revert_msg` cannot be set to true and thus the `satisfy` fails. Applying the same reasoning, it is clear that the same behavior happens for reverting behaviors of `noUserDefinedRevertFlows` and `emptyRequire`, which do not have user-defined revert messages. This means that if `saw_user_defined_revert_msg` is not marked persistent, the rule cannot distinguishing between methods that may revert with user-defined messages and methods that may not.

* * *

