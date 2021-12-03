```
  MCIP: ?
  Title: Ring Decoy Selection as Clumps
  Authors: koe <ukoe@protonmail.com>
  Status: Draft (DO NOT USE)
  Type: Informational
  Created: ?
```

## Abstract

Recommends selecting the decoy ring members for input signatures randomly from outputs near the real output being spent.



## Motivation

In the initial version of MobileCoin, it is standard for decoy ring members to be selected at random from the set of outputs existing in the chain. This method permits the 'guess-newest' heuristic, which says the most 'recent' output in a ring is likely to be the true signer [about 80\% of the time](http://arxiv.org/abs/1704.04299).

Since MobileCoin transactions are validated in secure enclaves, an analyst of the MobileCoin blockchain can only use that heuristic if they break into a secure enclave participating in the MobileCoin network. However, MobileCoin's security model does not require secure enclaves to function as advertised. Ring signatures can be considered a 'second layer of defense' below secure enclaves. It is therefore beneficial for ring signatures to obfuscate the origin of funds as optimally as possible.

To that end, a more reliable ring member selection algorithm than pure random selection is useful.



## Ring member selection

Ring members for an input are selected randomly in a window of size `2 x RING_SELECTION_RADIUS + 1` that overlaps with the true spend.


### Algorithm

1. Define constants and inputs.

```
// constants
#define RING_SELECTION_RADIUS = [[[TODO]]];
#define RING_SIZE = 11;  // exists in v0 of protocol

// inputs
uint64 real_index: index of real spend
uint64 min_index: minimum index of ring members
uint64 max_index: maximum index of ring members

// invariants
assert(RING_SIZE > 0);
assert(2xRING_SELECTION_RADIUS + 1 >= RING_SIZE);
assert(max_index > min_index);
assert(max_index - min_index >= 2xRING_SELECTION_RADIUS);
assert(real_index >= min_index && real_index <= max_index);
```

2. Select the clump locus.

```
// get clump window randomly around real index
int128 locus_min = real_index - RING_SELECTION_RADIUS;
int128 locus_max = real_index + RING_SELECTION_RADIUS;
int128 clump_locus_temp = rand<int128>(Range{locus_min, locus_max});  // randomly from range [a, b]

// if clump window is out of range, adjust it
if (clump_locus_temp - RING_SELECTION_RADIUS < min_index)
	clump_locus_temp = min_index + RING_SELECTION_RADIUS;
else if (clump_locus_temp + RING_SELECTION_RADIUS > max_index)
	clump_locus_temp = max_index - RING_SELECTION_RADIUS;

assert(clump_locus_temp + RING_SELECTION_RADIUS >= 0);

uint64 clump_locus = static_cast<uint64>(clump_locus_temp);
```

3. Select ring members.

```
array<uint64, RING_SIZE> ring_members;
Range clump_range_normalized = Range{0, 2 x RING_SELECTION_RADIUS};
uint64 real_index_normalized = real_index - (clump_locus - RING_SELECTION_RADIUS);
uint64 clump_size = 2 x RING_SELECTION_RADIUS + 1;

// randomly select normalized ring members
for (int index{0}; index < RING_SIZE; index++)
{
	bool try_again{true};

	// prevent duplicates
	while(try_again)
	{
		ring_members[index] = rand(clump_range_normalized);
		try_again = false;

		for (int earlier_member{0}; earlier_member < index; earlier_member++)
		{
			if (ring_members[index] == ring_members[earlier_member])
			{
				try_again = true;
				break;
			}
		}
	}
}

// find difference between first ring member and normalized real index
uint64 offset = mod_subtract<uint64>(real_index_normalized, ring_members[0], clump_size);  // a - b mod c

// add offset to all ring members
// - maps first ring member onto real index
for (auto &member : ring_members)
	member = mod_add<uint64>(member, offset, clump_size);  // a + b mod c

// map ring members back into real index space (de-normalize them)
for (auto &member : ring_members)
	member += (clump_locus - RING_SELECTION_RADIUS);

// return the result (sorted)
return sort(ring_members);
```



## Rationale

#### What is the purpose of using an offset to map a random member onto the real index?

Using an offset ensures the real index is embedded in a random distribution instead of composed on top of a random distribution. This reduces the possibilities for statistical analysis.

#### What are potential drawbacks to this proposal?

If an individual spams many transactions in a row, then the blocks containing those transactions will have a high density of outputs created by that individual. If another individual owns an output in those blocks and spends it, then the set of decoys in the corresponding ring signature will contain many outputs created by the first individual (assuming decoy selection uses this proposal). If that individual was malicious, and intentionally sent himself all those outputs, then the second individual's ring signature will be 'poisoned'. The malicious person will be able to deduce with probability `> 1/RING_SIZE` which output was the true spend in the ring signature.

#### What are the alternatives to this proposal?

1. Pure random selection over the on-chain output set. This is vulnerable to the guess-newest heuristic.

2. Selection from a gamma distribution over the on-chain output set, where the gamma distribution is designed to mimic the true spend distribution. The true distribution can only be approximated, gamma distributions are non-trivial to implement, and using the Fog service to obtain membership proofs that are spread across the output set does not scale optimally at the point where Fog must shard the Merkle tree between multiple servers.


## Backward compatibility

This proposal only recommends a new pattern for selecting ring members, so it is backward compatible with existing transaction-builder implementations.



## Reference implementation

Not completed yet.



## References

- [An Empirical Analysis of Traceability in the Monero Blockchain](http://arxiv.org/abs/1704.04299)
