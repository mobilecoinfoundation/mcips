```
  MCIP: ?
  Layer: Peer Services
  Title: Protocol Version Statement Type
  Authors: koe <ukoe@protonmail.com>
  Status: Draft (DO NOT USE)
  Type: Standards Track
  Created: ?
  Requires: [SCP statement types]
```

## Abstract

MCIP-[scp hard forks] requires a new MCIP-[statement types] statement type. This MCIP specifies a 'Protocol Version' statement type.



## Motivation

MCIP-[statement types] statement types can only be proposed with a Peer Services Standards Track MCIP. MCIP-[scp hard forks] is a Process MCIP, so it is not appropriate to propose a new statement type there. Moreover, a 'Protocol Version' statement type may be more broadly usable than MCIP-[scp hard forks], so it should not depend on that proposal to be implemented.



## MCIP-[statement types] statement type `0x01`

- Let type `0x01` be 'Protocol Version'.
  - serialize as a 4-byte unsigned integer
  - record in little-endian form in the least significant 4 bytes of the SCP statement value
- A `0x01` statement should be considered invalid by a node if that node does not believe the recorded version number is scheduled (see MCIP-[scp hard forks] for example).
  - In particular, a `0x01` statement is only valid if greater than the previous block's version.
- For MCIP-[Versatile SCP] 'optional ballot creation', version statements are not required to make a ballot.
- When combining confirmed nominated statements into a ballot, take the lowest `0x01` statement and discard all others.
- An MCIP-[Versatile SCP] 'simplify' function that takes a set of nomination statements and returns a minimized set of statements should not discard any `0x01` statements.



## Rationale

None at this time.



## Backward compatibility

N/A



## Reference implementation

Not completed



## References

- None
