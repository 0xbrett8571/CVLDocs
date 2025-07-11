# Invariants — Certora Prover Documentation 0.0 documentation
An invariant is a property of the contract’s storage state that should hold between calls to the contract. For example, in some ERC20 contracts the balance of `address(0)` is always zero.

Below, we provide examples of using invariants. You can read more about invariants in Invariants (from The Certora Verification Language).

Teams example
-------------------------------------------------------

The ITeams interface, shown below, is an interface for managing teams consisting of two players and a team leader.

```
interface ITeams {

    /// @return The team id of the player
    /// @notice Return value of 0 means the player has not been assigned to a team
    function teamOf(address player) external view returns (uint8);

    /// @return The team leader
    /// @notice Return value of zero means the team has not been created
    function leaderOf(uint8 teamId) external view returns (address);

    /// @dev Players must not be assigned to any team
    /// @dev The team must not exist before the call
    function createTeam(
        address leader,
        address playerA,
        address playerB,
        uint8 teamId
    ) external;

    /// @notice Change the team's leader
    /// @dev Only the current leader may call this function
    /// @dev The new leader must be a member of the tem
    function changeLeader(address newLeader) external;
}

```


A contract implementing this interface should satisfy the following properties:

1.  Each team has three _different_ players, one of which is the team leader.
    
2.  Each player can belong only to one team.
    
3.  A team is identified by a unique id between 1 and 255.
    
4.  A player having team-id 0 indicates this player has no assigned team.
    
5.  Address zero cannot be part of a team.
    
6.  If a team has not been created yet, it will have `address(0)` as a team leader.
    

Next we translate these properties to CVL invariants. You can find the entire spec in The team of zero is zero.

*   To run the spec on a correct implementation of `ITeams` use correct.conf, which will use the implementation in Teams.sol.
    
*   To see the spec discover bugs, use buggy.conf. This will run the spec against the buggy implementation in TeamsBugs.sol.
    

Simple invariants
---------------------------------------------------------------

### No team for address zero

We can readily deduce from the properties that `teamOf(address(0))` must be zero. Here it is written as an invariant:

```
methods {
    function teamOf(address) external returns (uint8) envfree;
    function leaderOf(uint8) external returns (address) envfree;
}

/// @title Address zero is not in any team
invariant addressZeroNotPlayer()
    teamOf(0) == 0;

```


We declared the functions `teamOf` and `leaderOf` as `envfree` to remove the need for an `env` type argument.

### The leader is part of the team

Another invariant property is that the team-id of the leader of team \\(x\\) is \\(x\\). This only holds if \\(x\\) is not zero and the leader is not `address(0)`. Here is the property written as an invariant:

```
/// @title The team's leader is part of the team
invariant leaderBelongsToTeam(uint8 teamId)
    (teamId != 0 && leaderOf(teamId) != 0) => teamOf(leaderOf(teamId)) == teamId;

```


Using preserved blocks inside invariants
-------------------------------------------------------------------------------------------------------------

Sometimes additional conditions are needed to prove invariants. These additional conditions are given using preserved blocks, see Preserved blocks. Here are two examples using preserved blocks.

### A team not created has no players

Before team `i` is created, `leaderOf(i)` must be `address(0)`. In such a case, there should be no players in team `i`. We can write this condition as:

```
/** @title If a team does not exist it has not players
 *  This invariant cannot be proven without a preserved block.
 */
invariant nonExistTeamHasNoPlayers(uint8 teamId, address player)
    (teamId != 0 && leaderOf(teamId) == 0) => teamOf(player) != teamId;

```


Running this rule, the Prover will find the following violation, which you can see in this rule report nonExistTeamHasNoPlayers violation report. The function called is `changeLeader(address(0))`, changing the leader from address `a` (where `a` is not zero) to zero. Before the call `address(0)` is a member of team `i`, where `i > 0`. After the call the left hand side of the invariant condition holds true: `i != 0 && leaderOf(i) == 0`. But the right hand side is false for `player = a`, since `teamOf(a) = i`. The violation is expressed in the following table, showing the change in state.


|           |Pre call state|Post call state|
|-----------|--------------|---------------|
|leaderOf(i)|a             |0              |
|teamOf(a)  |i             |i              |
|teamOf(0)  |i             |i              |


In order for the invariant to be proved, we need to require that the team of `address(0)` is zero. We’ll do that using a preserved block. Since we already proved this in No team for address zero, we can simply require that the invariant `addressZeroNotPlayer` holds, like so:

```
/// @title If a team does not exist it has not players
invariant nonExistTeamHasNoPlayers(uint8 teamId, address player)
    (teamId != 0 && leaderOf(teamId) == 0) => teamOf(player) != teamId
    {
        preserved {
            requireInvariant addressZeroNotPlayer();
        }
    }

```


### A team has at most three players

Here is how we phrase this property:

> Let `a`, `b`, `c` and `d` be four different addresses, and suppose that `a`, `b` and `c` are all on the same non-zero team `i`. Then `d` does not belong to team `i`.

#### Helper functions

To enhance readability we’ll define two helper functions:

1.  A function checking that four addresses are different, called `fourDifferentAddresses`.
    
2.  A function checking that three addresses are on the same team, called `sameTeam`.
    

Their definitions are given below.

```
/// @return If the four addresses are all different from each other
function fourDifferentAddresses(
    address a,
    address b,
    address c,
    address d
) returns bool {
    return (
        a != b && a != c && a != d &&
        b != c && b != d &&
        c != d
    );
}

```


```
/// @return If all players are on the same team
function sameTeam(address a, address b, address c) returns bool {
    return teamOf(a) == teamOf(b) && teamOf(b) == teamOf(c);
}

```


#### The rule

Here is the complete rule.

```
/// @title A team has at most three players
invariant teamHasMaxThreePlayers(address a, address b, address c, address d)
    (
        teamOf(a) != 0 && 
        fourDifferentAddresses(a, b, c, d) &&
        sameTeam(a, b, c)
        => teamOf(d) != teamOf(a)
    )
    {
        preserved createTeam(
            address leader,
            address playerA,
            address playerB,
            uint8 teamId
        ) with (env e) {
            requireInvariant nonExistTeamHasNoPlayers(teamId, a);
            requireInvariant nonExistTeamHasNoPlayers(teamId, b);
            requireInvariant nonExistTeamHasNoPlayers(teamId, c);
            requireInvariant nonExistTeamHasNoPlayers(teamId, d);
        }
    }

```


As you can see, we used a different preserved block here. This preserved block adds a pre-condition only when verifying the invariant on the function `createTeam` using environment `env e`. Without this preserved block, the Prover may assume that the team had players _before it was created_.

See also

You can find out more about preserved blocks in Preserved blocks section.
