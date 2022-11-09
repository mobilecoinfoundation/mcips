- Feature Name: `View Authenticate Sender Memo`
- Start Date: `2022-11-09`
- MCIP PR: [mobilecoinfoundation/mcips#0058](https://github.com/mobilecoinfoundation/mcips/pull/58)

# Summary
[summary]: #summary

[#4 Recoverable Transaction History (RTH)](https://github.com/mobilecoinfoundation/mcips/pull/4) introduced three different memo types for [encrypted memos](https://github.com/mobilecoinfoundation/mcips/pull/3). This proposal specifies three more memo types:

- View Authenticated Sender
- View Authenticated Sender With Payment Request Id Memo
- View Authenticated Sender With Payment Intent Id Memo

This feature aims to extend the SenderMemo to allow for senders to use their subaddress view private keys to generate Authenticated Sender Memos

# Motivation
[motivation]: #motivation

With the introduction of the ViewAccountKey and UnsignedTransactions, generating a SenderMemo for RTH became impossible because of the requirement of the Subaddress Spend Private Key during the creation of the TxOuts for the transaction.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

In a scenario where a user is building an unsigned transaction with a view account, instead of using SenderMemoCredential to generate an AuthenticatedSenderMemo (since this requires the subaddress spend private key which is unavailable), the user will use the subaddress view private key (which is available) to generate a SenderViewMemoCredential, used to create a ViewAuthenticatedSenderMemo.

# Drawbacks
[drawbacks]: #drawbacks

This may potentially reduce the security of the SenderMemo by requiring the view private key (which would potentially be available on an online machine) instead of the spend private key (which is only contained on an offline machine of hardware wallet). If the view private key were to ever be compromised, then the attacker could authenticate their payments using the ViewAuthenticatedSenderMemo of the compromised party, making it appear as if their sent transaction came from the compromised user.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

An alternative would be to generate the SenderMemoCredential during signing and modify the TxOut during the signing process. This would require more discovery as it is uncertain if this is possible without regenerating all of the contents of the TxOut and TxPrefix, which may not be desireable (especially if signing using a hardware wallet).

The rationale for adding the ViewAuthenticateSenderMemo is that it allows users to utilize RTH with an offline transaction signer or hardware wallet.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

> - Is it possible to modify just the encrypted memo portion of a TxOut without requiring the TxPrefix and OutputSecrets to be regenerated?

# Future possibilities
[future-possibilities]: #future-possibilities

None at this time
