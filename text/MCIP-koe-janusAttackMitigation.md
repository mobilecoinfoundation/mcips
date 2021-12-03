```
  MCIP: ?
  Layer: Application
  Title: Janus Attack Mitigation
  Authors: koe <ukoe@protonmail.com>
  Status: Draft (DO NOT USE)
  Type: Standards Track
  Created: ?
```

## Abstract

Adjust wallet-side transaction construction standards to mitigate the [Janus attack](https://web.getmonero.org/2019/10/18/subaddress-janus.html). Specifically, set the fog hint's ephemeral key equal to the txout public key's 'base key'.



## Motivation

The [Janus attack](https://web.getmonero.org/2019/10/18/subaddress-janus.html) allows a malicious transaction author to discern if two subaddresses are part of the same account.

1. Construct an output from components of two addresses.

```
address A: K^{v,A}, K^{s,A}
address B: K^{v,B}, K^{s,B}

sender-receiver shared secret: shared_secret = r_i K^{v,A}
one-time address: K^o = H(shared_secret) G + K^{s,B}
txout public key: r_i K^{s,A}
```

2. Output recipient identifies they own the output.

```
sender-receiver shared secret: shared_secret = k^v * r_j K^{s,A}
nominal spend key: K^s_nom = K^o - H(shared_secret) G

if: K^s_nom matches subaddress spend key B in [set of subaddress spend keys]
then: the output is owned by subaddress B
```

However, in this case the shared secret is based on address A!

3. If the output recipient notifies the sender that they got an output, then the sender will know that subaddresses A and B belong to the same account (i.e. were constructed from the same private key pair `k^v, k^s`).

There are currently no mitigations to this attack implemented in MobileCoin.



## Janus attack mitigation

### Transaction-builder standard

When constructing a transaction:

- Instead of generating separate values `r_i` and `r_fog` for the txout private key and fog hint ephemeral key, respectively, generate one value `r_i`. Let the fog hint ephemeral key equal `r_i G`.


### Output recovery

After identifying an output owned by the subaddress with index `i`, test for the Janus attack.

1. Compute `txout_public_key_nom = k^{s,i} * fog_hint.ephemeral_key`.
2. Expect: `txout_public_key_nom == output.public_key`.
3. If the test fails, then raise a 'Janus test failed' warning. This output is spendable, but may have been (or likely was) created by a malicious sender.

In other words, the user should be able to recompute the txout public key from the fog hint's ephemeral key and the private spend key of the address that supposedly owns this output. Doing so successfully means the output was not constructed by components of multiple addresses (except with negligible probability).



## Rationale

#### What are alternative mitigations for the Janus attack?

According to the discussions [here](https://github.com/monero-project/research-lab/issues/62) and [here](https://github.com/monero-project/monero/issues/6456), there are two good solutions for the Janus mitigation.

1. Let the txout private key `r_i` equal a hash of a 16-byte random nonce. Mask that nonce with the sender-receiver shared secret (similar to how amounts are masked/encoded), and store it in output data. Users can decode the nonce, recompute `r_i`, and test that `r_i K^{s,i}` equals the txout public key.

2. Use a specially designed [3-key address format](https://github.com/monero-project/research-lab/issues/62#issuecomment-870147617) instead of the current CryptoNote-style 2-key addresses.

This proposal does not use either of these solutions because they are more 'costly' (in terms of storage space and ecoysystem burden, respectively) than using the fog hint's ephemeral key. Unfortunately, in this proposal, testing for a Janus attack requires the private spend key, which means view-key-only wallets cannot detect Janus attacks. Since view-key-only wallets are only moderately useful, as full balance recovery requires the private spend key, this limitation is not of prohibitive concern.



## Backward compatibility

Output-recovery implementations need to know when to start performing the Janus test, so they don't erroneously flag outputs as being victims of the attack. To that end, this proposal is well-suited to being rolled out alongside a hard fork of some kind.

Output-recovery implementations that do not implement this proposal are compatible with transaction-builders that do implement it, because only the ephemeral fog hint is changed. However, transaction-builders that don't implement this will cause output-recovery implementations to think all outputs created by those builders have failed the Janus test.



## Reference implementation

Not completed yet.



## References

- [Advisory note for users making use of subaddresses](https://web.getmonero.org/2019/10/18/subaddress-janus.html)
