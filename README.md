# StellarSecuredSmartWallet


**Secure self-custody solutions are essential for crypto inclusion and accessibility.** Currently, owning cryptocurrencies can be risky, and many people may prefer custodial services, even though cryptocurrency offers incomparable alternatives with multisignatures for security.

This project enhances the existing smart wallet implementation on Stellar by adding functionalities that allow:
- **Wallet Recovery** using an "inactivity time threshold and a social multisignature" mechanism.
- **Secure External Custody**, allowing an external entity to safeguard wallets by securing its own keys without having control over wallet funds. This feature allows for efficient signer updates in case of compromise, without needing to update each wallet individually.

As explained in the presentation, these tools provide an optimal balance for self-custody, achieving security levels comparable to a bank. They deliver what non-crypto users truly expect: a seamless and secure experience that can pave the way for insurance over crypto-asset ownership.

(Note that users could maintain multiple wallets, some without a securer, to function as full self-custody cash — free from censorship risk.)



## Project Presentation 
This project aims to close the gap between non-crypto users and crypto user by shifting 
how non-crypto users may access and use self-custody wallets.

Stellar already offers a number of tools such as recovery tool (with SEP-30: https://stellar.org/blog/developers/sep-30-recoverysigner-user-friendly-key-management) and smart wallets (passkey,multisig...)

Here we add functionnalities on the Smart Wallet (SW) and by taking a different approach on these solutions: 
- We offer a recovery system based on smart contract adding a time based "inactivity" parameter - ensuring inability to recover until enough inactivity has passed.
- We also add a simple multisignature scheme that is designed to be used by an external signers that acts as a "securer" of the account. It is an external entity that has censorship power but no control of the assets.
The difference between a simple multisignatures is that here we allow the securer to handle the securer's allowed signers by himself (through an external contract). That way, a securer can have a few valid addresses. In case of one being compromise, he can update the storage to adapt the public keys he can use. 
(This is because, in our case, the private key will be on servers for automatic multisig. Despite the ability to have an enclave, there is still risks involve. It would be beneficial to have a low risk multisig that can update the addresses in case one is compromise and we want to avoid to have to do this for each wallet!)

While the second functions may not be used (optional) the combinaison of both only rely on a "time" based compromise for controlling the assets, and have no potential breach of security when it comes 
to assets ownerships during the controlled time period (whereas previous works may rely on external factors which could be compromised at any given time).

This constitute a "small" compromise for users and mostly handled by the wallet/client itself. 
The external "securer" fonctionnality is done in a way that a "securer" can secure many accounts 
and potentially act quickly in case of attacks, ensuring funds safety. This is done in order to have potential insurance over assets against thiefs given a high security standards and the fact we have 
a double authentication system through multisigners.

Note that if both functionnality are well defined for a specific instance, a user will have low risk 
when any actor is acting badly: he could have priority over funds recovery thanks to a personnal backup for instance, ensuring any external entity attempt for censorship to be useless. 
And in case of thief of the user private key (if it is not a recovery key with only 1 threshold) then
the external entity will have power to censor every transactions so the user can recover the funds
and the thief cannot steal anything. This works even in the event of a thief actually getting access
to an enclave (from a fingerprint spoof or face spoof or just knowing the code).

On a technical level this project aims to add functionnalities to a smart wallet integration: 
https://developers.stellar.org/docs/build/apps/smart-wallets
https://docs.google.com/document/d/1c_Wom6eK1UpC3E7VuQZfOBCLc2d5lvqAhMN7VPieMBQ/edit?tab=t.0#heading=h.zhjcc9awntwu


### Time compromise handling

The main compromise we make is based on the recovery system which is time based.
(We basically have full ownership of the funds as long as we do one transaction during the inactivity period
and for those who wants an external securer, we keep full censorship resistance given this time-frame as well. So choosing this time is not always easy)

This is a main part of this solution, but it comes at a cost: it means the user needs to be active.
Well, it doesn't have to necessarilly.

Given a time of inactivity of X (for instance 2 weeks), if the users are using their account as a 
savings accounts, they may not use it quite frequently (although this is designed to be potentially 
used as a peer-to-peer payment system). Hence, they can pre-sign a few transactions of 0 XML and have 
their phone send these transactions automatically. That way, we can keep our inactivity period low but 
only have to do a transaction manually every month for instance (and sign maybe again other transactions).

In case of the phone being stolen, the transactions will not be sent, and after 2 weeks (X) we can recover
our phones. This lower the necessity of activity. This can also be achieve as a service from a service provider
which will ask you to authenticate to prevent the new transaction being sent to the network if you 
are authenticated and are asking for a stop because you lost your device.


### Risks and how securers are supposed to be used

The risk of using a securer is having the securer acting badly and steal your private key or run a bad software to sign unwanted transactions. Hence, if you choose a securer, you should definitively trust it and preferably use a wallet software that have no link with the actual securer.

You could potentially use several securer as well for saving account that you seldomly use (because this will increase the fee).

The concept of securer should be use as a way to improve security, obtain insurance... and not increase the risks. Thus, we suggest generally to have several accounts: 
- A smart wallet without securer (if you wish to) for small transactions - knowing the potential risks of scams.
- A smart wallet with one securer for your main account.
- A smart wallet with potentially more securers if that seems necessary, for saving account you rarely use.

## Contract

I forked the passkey-kit in order to add the functionnalities I want to add directly in it.
This is here added as a submodule to the project.

### Implementation specification

Remenber we have two main functionnalities: 
1. Recovery with last-active time and inactivity duration
2. A special kind of multisig that has: a contract with a list of allowed signers this list can be change by the external entity (no matter how many wallets are using it).

#### Recovery

Recovery functionnality is mentionned in the draft for the smart wallet standard : https://docs.google.com/document/d/1c_Wom6eK1UpC3E7VuQZfOBCLc2d5lvqAhMN7VPieMBQ/edit?tab=t.0#heading=h.zhjcc9awntwu

Here we can hence create a generic recovery: working both for time-based or not.
Basically we can specify a duration of 0 to ensure a standard social recovery system.

We can specify a list of recovery entities with threshold and time:
```rust
pub struct Recovery{
    pub signers: Vec<SignerPubKey>, // Maximum 255 signers because it doesn't make sense to have more
    pub conditions: Vec<Condition>
}

pub struct Condition {
    pub allowed_signers_index: Bytes, // Each byte is the index of the signer + Assuming if this is empty we use all the signers
    pub threshold: BytesN<1>,
    pub inactivity_time: u64
}
```

Example: 
Here there is a multisig with alice_backup, bob and companyX after 10 days of inactivity,
after 15 days only alice_backup and companyX can recover alone without a multisig.
```
signers: [alice_backup, bob, companyX],
conditions: [
{[0, 1, 2], threshold: 2, time: 10 days},
{[0, 2], threshold: 1, time: 15 days}
]
```

In case you don't want to use time based just do: 
```
signers: [alice, bob, adam],
conditions: [
{[], threshold: 2, time: 0},
]
```


It has been decided to have the recovery put in a policy. It indeed seems better to keep the Smart Wallet small. However we don't use the fact we could update the "last_active_timestamp" on each authentication almost for "free". It is "good" because we may not need to update it too regularly (it is lost of computation) but at the same time, it implies to have a good system in the wallet to regularly signs another transaction (or include the call in a transaction) just to update the last_active_timestamp.


#### Multisig from an external entity

This could be added as a simple signers in the list of signers. But actually, 
there is a problem if there is a leak or the signer is compromise. 
This is why it is better to have an external contract that will actually do the checks and 
controlled the different signers. The signers can be updated with an admin multisignature, ensuring good security (rather than adding a potential breach of security).

This is added as a policy as well.

Since this means we have two authentication (one for the policy and one for admins) we have to use the context in order to do the authentication.

Implementation : 

We have a policy contract

- Admin : A simple multsig with threshold
- Allowed signers (for the people using the policy) that use a simple multisig with threshold as well (but it will almost always be 1. It would lead to more cost to add more than one and it feel relatively useless. We only ensure several allowed signers in case of server failure. If a user wants several securers, he can add several policy contract from different securer).


## App

I didn't had the time yet to implement it in my wallet app. 

## Backend 

I didn't had the time yet to implement a simple backend handling a securer signature.