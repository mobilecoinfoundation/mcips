```
  MCIP: ?
  Layer: Peer Services
  Title: Versatile SCP
  Authors: koe <ukoe@protonmail.com>
  Status: Draft (DO NOT USE)
  Type: Standards Track
  Created: ?
```

## Abstract

The standard [SCP protocol](https://www.stellar.org/papers/stellar-consensus-protocol) (Stellar Consensus Protocol) has limited capabilities for constructing slots. This proposal adds nuance to that protocol in three ways, without affecting safety or liveness guarantees for any node.

1. Nodes may express opinions about the desirability of statements by choosing not the vote for them during the nomination phase. Only when a large part of the network considers a statement desirable will it pass through the SCP process and be externalized.
1. It is currently feasible for SCP ballots to contain statements with different types (e.g. if an SCP nomination statement is defined as a < type, value > pair). In this proposal, SCP implementers may define rules about the composition of ballots. For example, they could mandate that a subset of statement types be represented in all ballots.
1. SCP implementers may define statement types that are equivalent to 'statement clumps' - multiple statements represented by a single statement. In the context of quorums and blocking sets, these will be interpretted as collections of distinct statements instead of one single statement.



## Motivation

The SCP protocol used in MobileCoin does not support three things. Note that MobileCoin only has one statement type at this time (transactions). However, the SCP protocol would not support the below capabilities, which we assume are desirable.

1. If the network votes on a parameter change with SCP (e.g. a block version change), then the only way to 'reject' that change is to consider it invalid. However, if enough of the network decides to make the change, then nodes that rejected it will get stuck, unable to make progress. It isn't possible to express preference about parameter changes without risking liveness (even simple judgements like the 'timing' of a parameter change can't be expressed). There is a similar problem for expressing opinions about normal statements. For example, the network cannot censor transactions with low fees (e.g. to mitigate a DDOS attack) without risking liveness.
1. If a 'supporting' statement type appears in the network, but no other statements appear, then it is possible for a slot to be externalized that contains just that 'supporting' statement (e.g. a new block version, or a timestamp). It isn't possible to wait until a slot has enough 'content' before trying to externalize it.
1. During the nomination phase, statements are treated as singular, even if they might be considered 'clumps' of statements. This means it is not easy or efficient to consense on, for example, timestamps, which can be considered large collections of 'timepoint' statements.



## Versatile SCP

SCP can be made more versatile with three changes to the nomination phase. These are 'nomination filtering', 'optional ballot creation', and support for 'statement clumps'.


### Nomination filtering

**Current behavior**: During the nomination phase, nodes vote to nominate statements that have been proposed locally. If a different node has accepted or voted to nominate a statement, then the local node will 'mirror' the other node by issuing its own vote to nominate the same statement.

**New behavior**: During the nomination phase, nodes look at statements proposed locally and statements accepted or voted for by other nodes. They use locally-defined rules to assess those statements. A node only votes to nominate a statement if the statement passes that test. This is not the same as statement validity - for our discussion we assume all statements are valid.

#### Implementation

Nomination filtering can be added to an SCP implementation with an undefined virtual function called `nominate_fn()`. Concrete instantiations of the SCP protocol implementation must define that function. It takes an SCP statement (or 'value') as input, and outputs an SCP statement that the node should vote to nominate.

It turns out standard SCP already contains nomination filtering, in the form of 'federated leader selection'. In brief, during the nomination phase a node will only propose new statements if it is randomly selected as a 'leader' (based on an algorithm that uses hash functions), and will only mirror statements of other randomly selected 'leaders'.

To retain that behavior and allow nomination filtering to be more generic, the `nominate_fn()` may have the following signature. The point is to move 'filtering' behavior related to federated leader selection from in-line in SCP implementations into the concrete instantiation. This allows implementers to decide when federated leader information should be used, or ignored.

```
nominate_fn(StatementType statement, bool is_priority_peer, bool statement_proposer_is_self) -> Option<StatementType>;
```

- The function returns an SCP statement instead of `bool`. If the input SCP statement is a 'statement clump', then the node can decide to vote to nominate a subset of that clump. We use a Rust-like `Option<>` so the function can return `None()` to represent 'don't vote for anything'.
- The input `is_priority peer` is `true` if the statement is proposed by, or sourced from, a 'priority peer' according to federated leader selection (note that if the local node is selected as 'leader', this will be `true`).


### Optional ballot creation

**Current behavior**: During nomination, when a node sees at least one statement is confirmed nominated, it will create a ballot directly from the set of confirmed nominated statements. It will also stop proposing new statements to be nominated, and stop issuing new votes to nominate statements.

**New behavior**: During nomination, when a node sees at least one statement is confirmed nominated, it will check if the set of confirmed nominated statements can be combined into a ballot. It will only create a ballot if they can be combined. When the node can create a ballot from confirmed nominated statements, then it will stop proposing new statements to be nominated, and stop issuing new votes to nominate statements. When a ballot is received from another node, a node will check that it is well formed.

#### Implementation

SCP implementations typically use something like an undefined virtual function '`combine_fn()`' to convert sets of confirmed nominated statements into ballots. This function is then implemented for concrete instantiations of SCP.

In the case of MobileCoin, that function has a Rust-like signature that looks something like this.

```
combine_fn(Vec<StatementType> statements) -> Result<Ballot, Error>;
```

To implement this MCIP, the function should have a Rust-like signature that looks like this.

```
combine_fn(Vec<StatementType> statements) -> Result<Option<Ballot>, Error>;
```

If the input set of statements can't be combined into a ballot, then the function should return `Ok(None())` (i.e. in Rust). Otherwise, the function's behavior remains unchanged.

To verify that a ballot `B` received from another node is well formed, use the following test.

```
bool ballot_well_formed = Ballot{combine_fn(B:values)} == B;
```


### Statement clumps

**Current behavior**: In the nomination phase, statements are assumed to be 'singular'. When looking for a quorum that votes to nominate a statement (and other quorum-related searches), implementations use equality tests to see if two nodes have voted to nominate the same statement.

**New behavior**: In the nomination phase, statements are assumed to represent 'clumps' of statements (with 'singular' statements being a degenerate case). When two statements are compared for quorum searches, implementations look for intersections between the clumps.

#### Implementation

Current SCP implementations can already use the concept of set intersections to do quorum searches. However, in the current SCP protocol those intersections can be implemented in-line by relying on equality tests between statements. To support statement clumps, set operations should be undefined virtual functions implemented by inheritors, who can decide how to handle their own arbitrary statement types.

Define three undefined virtual functions.

1. Given two sets of statements A and B, return those parts of B that don't intersect with A. If comparing two statement clumps X and Y, then the return set should include those parts of Y that don't intersect with X, assuming there is no other statement clump Q in A that intersects with the result.
```
statements_not_interesect_fn(Vec<StatementType> statements_A, Vec<StatementType> statements_B) -> Vec<StatementType>;
```

2. Given two sets of statements A and B, return their intersection. If comparing two statement clumps X and Y, then the return set should include the intersection of X and Y.
```
statements_intersect_fn(Vec<StatementType> statements_A, Vec<StatementType> statements_B) -> Vec<StatementType>;
```

3. Given a set of statements, minimize its length by combining statement clumps as much as possible.
```
simplify_statements_fn(Vec<StatementType> statements) -> Vec<StatementType>;
```

In every situation where `combine_fn` is used, replace it with a call to the following function. This avoids code duplication and improves robustness.

```
fn simplify_and_combine(Vec<StatementType statements) -> Result<Option<Ballot>, Error>
{
	return combine_fn(simplify_fn(statements));
}
```

Here is a pseudocode helper function for searching through a quorum set using 'predicates' to test agreement between nodes. It returns a set of statements.

```
fn search_quorum_set(Vec<StatementType> candidates,
	member_statement_filter_fn [function that gets statements from a quorum set member],
	Enum search_type [v-blocking set or quorum])
	-> Vec<StatementType>
{
	Vec<StatementType> results;

	// the first v-blocking set or quorum that intersects with the candidate set may only
	// cause the node to accept/confirm 'some' statements; other v-blocking sets or quorums could
	// cause it to accept/confirm more statements, so we loop until no more statements are added to results
	do
	{
		candidates = simplify_statements_fn(candidates);

		predicate = StatementSetPredicate {
			// set initial state of predicate
			values = candidates.clone();

			// test function of predicate
			// input: quorum set member
			// behavior:
			//  - get statements to test for intersection against from member of quorum set
			//  - they are either a member of a v-blocking set or quorum
			//  - the updated predicate will be applied to the next member of that set/quorum
			test_fn (Node quorum_set_member)
			{
				member_statements = member_statement_filter_fn(quorum_set_member);
				values = statements_intersect_fn(values, member_statements);
			}
		}
		if (search_type == v_blocking_set)
		{
			// use the predicate to find a v-blocking set that intersects with the candidate list
			find_blocking_set(predicate);
		}
		else if (search_type == quorum)
		{
			// use the predicate to find a quorum that intersects with the candidate list
			find_quorum(predicate);
		}

		Vec<StatementType> additional_results = predicate.values;

		// record statements
		results.extend(additional_results);

		// remove found statements from candidates
		candidates = statements_not_intersect_fn(additional_results, candidates);
	} while (!additional_results.empty() && !candidates.empty())

	return results;
}
```

Here is a pseudocode function for finding statements that a node accepts are nominated.

```
// returns: set of statements accepted nominated
find_accepted_nominated() -> Vec<StatementType>
{
	// find statements accepted nominated due to v-blocking sets
	candidates = [statements accepted by any node in local quorum set];

	accepted_nominated = search_quorum_set(candidates,
		member_statements_accepted_nominated,
		v_blocking_set);

	// find statements accepted nominated due to a quorum that votes or accepts them
	candidates = [statements the local node voted to nominate, but has not yet accepted nominated];

	accepted_nominated = accepted_nominated.extend(search_quorum_set(candidates,
		member_statements_accepted_or_voted_nominated,
		quorum));

	return simplify_statements_fn(accepted_nominated);
}
```

Here is a pseudocode function for finding statements that a node confirms are nominated.

```
// returns: set of statements confirmed nominated
find_confirmed_nominated() -> Vec<StatementType>
{
	// find statements confirmed nominated due to a quorum that accepts them
	candidates = [statements the local node accepts are nominated];

	confirmed_nominated = search_quorum_set(candidates,
		member_statements_accepted_nominated,
		quorum);

	return simplify_statements_fn(confirmed_nominated);
}
```

**Note**: These functions should be optimized by removing statements already accepted or confirmed nominated from the initial candidates list (e.g. with `statements_not_interesect_fn`). Similar optimizations should be done to the equivalent ballot-searching functions.



## Open questions

In normal SCP, nomination timeouts related to selecting federated leaders begin rolling as soon as an SCP message is heard from another node, or as soon as the local node has a statement to propose. In this proposal, due to nomination filtering and optional ballot creation, it is conceivable that statements observed/proposed shouldn't cause timeouts for federated leader selection to start. Open questions:

- Should Versatile SCP keep using federated leader selection at all?
- Should Versatile SCP add rules about when nomination timeouts should start running?



## Rationale

#### Why does nomination filtering not affect SCP safety/liveness guarantees?

1. The nomination phase of SCP is designed so the network can collect a set of statements that it's willing to populate a slot with. This process begins with nodes casting votes to nominate statements. If a node sees that a statement has accumulated enough votes, then it will 'accept' the statement is nominated. Accepted statements progress deeper into nomination. Nominated statements that don't gather enough votes will be ignored. A statement will not get enough votes if it appears late during a 'slot round' after too many nodes have exited the nomination phase and have stopped issuing new votes. It will also not get enough votes if, due to network problems, the statement is not seen by large parts of the network. Moreover, due to federated leader selection nodes already actively use a form of nomination filtering. In other words, it is already considered safe for some nominated statements to get a few votes, but not enough to be accepted by any node. Adding more locally-defined rules around voting does not fundamentally change the system's design.
1. Even if a node chooses not to vote for a statement, it will still accept that statement if a `v`-blocking set accepts it. This means nodes can safely disagree with each other without getting locked out of being able to participate in consensing on a slot.
1. Nomination filtering could impact liveness if not implemented carefully. For example, if the 'nominatible' test always returns false, then the network can never make progress. More generally, implementations must take care that 'nominatible' rules, in combination with the process for creating statements, don't lead to situations where statements that are required in order to create a ballot don't gain votes, and 'waiting long enough' won't resolve the problem (e.g. the conditions for nominating required statements don't loosen over time, or 'alternative' required statements won't inevitably appear).

#### Why does optional ballot creation not affect SCP safety/liveness guarantees?

1. If all nodes implement the same rules for ballot creation, then there can be no impact on safety.
1. The network can lose liveness if ballot creation rules are too strict. For example, if the ballot creation function always returns false/no-ballot, then no ballots will ever be made, and the network will never complete a slot. More generally, implementers must take care to ensure the network can always automatically complete a slot 'given enough time' (i.e. without node operator intervention). If ballot creation requires a certain statement type to exist, then we should expect one of those statements to always appear as a matter of course.

#### Why is this MCIP in 'Standards Track' for 'Peer Services'?

This MCIP does not directly affect compatibility between validator nodes executing the SCP protocol. However, new network capabilities based on this MCIP (either at the Consensus or Peer Services layer; see [Motivation](#motivation)) cannot be easily/safely supported by nodes if they don't also support this MCIP.



## Backward compatibility

These changes can be implemented without affecting the existing behavior of the SCP protocol.


### Nomination filtering

In the original SCP protocol, there is no nomination filtering (aside from federated leader selection). A function that implements 'nominate this statement?' can always return true to retain the old behavior (it should implement current federated leader selection rules).


### Optional ballot creation

In the original SCP protocol, ballot creation is always implementation-defined. Suppose a function that implements 'ballot creation', given a set of confirmed-nominated statements. Before this change, that function would return either a 'Ballot' or 'error message'. After this change, it can additionally return 'No Ballot'. To retain old behavior, the function implementation should be designed to never return 'No Ballot' (i.e. it always succeeds in creating a ballot, or errors out).


### Statement clumps

Current SCP implementations support a degenerate case of statement clumps. Specifically, clumps with just one element in them ('singular' statements). A more generic handling of arbitrarily defined statement clumps will not affect old behavior.



## Reference implementation

Not completed yet.



## References

- [Stellar Consensus Protocol whitepaper](https://www.stellar.org/papers/stellar-consensus-protocol)
