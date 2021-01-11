# Graffiti Protocol: Secure Async Communication over Untrusted Public Databases (pre-alpha extremely rough draft)

by lucash_dev@bitcoinhackers.org

## Motivation

In the past decade, usage of the internet concentrated around a few popular service providers for performing
all sorts of communication between individuals and organizations.
The business model these services evolved is widely recognized as being based on amassing and selling surveillance
data on users, and providing opportunities to influence user behaviour (ads, boosted posts, disinformation campaigns etc).
Meanwhile data breaches are becoming increasingly common, which means even if the amassed surveillance data isn't
intended to be used in any malicious way, the likelihood of that data becoming publinc approaches 1 with time.
Existing decentralized solutions still either require a trusted server (even if it can be run individually) to
manage user identity (and must hold secrets), such as Mastodon or Matrix, and often rely on existing trusted
infrastructure (such as DNS, PKI certificates) for connecting to them, as connecting to the wrong server can be cathastrophic. 
Other proposed alternatives rely entirey on a friend-to-friend connectivity graph or other less resilient assumptions.
Thus it is imperative that research keeps moving in the direction of protocols that can easily allow for trust-minimized
communication between any two typical clients reachable through TCP/IP.
The rest of this paper explains a suggested protocol for asynchronous communication satisfying those conditions -- by relying
on a set of non-trusted database servers.

## Assumptions

1 - Trusted third-parties are security holes.

The protocol should minimize the amount of information any third party can access, and reliance on any third-party for
it's functioning.

2 - Every piece of data transmitted outside of devices physically controlled by users will eventually become public.

All data shared with a third party is assumed to eventually become public without compromising the continuation or secrecy of the communication.
Of course this assumes decryption of sensitive information within devices controlled by users.

3 - Unsolicited messages are DoS attacks.

Perhaps not needed to be explicitly cited, ads and spam bots are attacks on user interaction.

4 - Both ends of a communication can reach at least one common point in the network.

This protocol will require one or more servers willing to perform a "best effort" to deliver messages -- but not trusted in any other way.
Multiple servers can be used and switching between them should be trivial. Please note that this assumption is essentially the same as
required for TCP/IP communication.

## Goal

The goal of this protocol is to allow bi-directional transfer of information between any two users that is end-to-end ecrypted and is trust minimized,
without requiring continuously running a trusted server.

Extensions to allow for multi-cast, public in-boxes, and public broadcast will be briefly discussed.

## General architecture

TODO: Insert diagram here

Elements of the architecture:

1 - Database Server(s)

2 - Clients

## The Database

In this document we will refer to "the database" as dealing with just one server makes explanation simpler, but it will become obvious
that a number of database servers can be used as long as both clients access at least one of them. Access to the database is assumed
to be made through HTTP(S) requests.

The database used for this protocol is a very simplified data store:

1 - No permission model for reading (all users have access to all data stored there, regardless of authentication). However, writing data
should be restricted as explained below.

2 - Only one numerical index which must allow both exact quries and range queries.

The database must implement those two primitives:

- STORE:
A STORE request shares with the server a piece of data -- a PARCEL -- that is supposed to be stored within the database, to be queried in the future.
A PARCEL is composed of:

 a - A *program*: a piece of executable script that validates that a user is allowed to modify this parcel. The concept is similar to output
    scripts in Bitcoin.

 b - A *payload*: a piece of data that can be picked up by other users.

 c - A *witness*: a piece of data that cryptographically *proves* that this request was authorized.

 d - TTL: a requested time stamp for this PARCEL to be retained until, in a best effort basis. 

The simplest form of a program is a public verifying key (ECDSA). In that case the *witness* will be a signature for the hash of the hashes
of the (program,payload,TTL). All programs will be assumed to be an ECDSA public key in the remainder of this document. Extensions to include
more complex forms of validations can be explored later.

Parcels should have a static maximum size (say 64KB)

The database server MUST validate that the *program*, *witness*, and *payload* match, that is, that the signature provided is valid.

- LOAD:

A LOAD request retrieves one or more PARCELs from the server.
The data is queried by a numeric range (min, max) pair, and is matched agains an index of all hashes of *programs* for all parcels.

All parcels matching the range of hashes should be returned within a limit (say, 20) -- ordered by the hash of the program.
 The whole (program, payload, witness) tuple must be returned.

Upon receiving the parcels the client MUST validate programs and witnesses for all, marking the server as compromised in case validation fails.

## Basic client-to-client communication

With these primitives a simple one-way communication can be obtained.

Assuming that:
a - The receiver knows a public verifying key that corresponds to a secret signing key the sender possesses.
b - The sender knows an encryption key corresponding to a decryption key known to the receiver.

The communication protocol goes like:

1 - The sender makes a STORE request to the server:
   - *program* is the public signing key known by the receiver.
   - *payload* is the data to be sent, encrypted using the receiver's encryption key.
   - *witness* is a signature generated using the sender's private signing key.
   - the operation might be repeated multiple times, and accross multiple servers for reliability.

2 - The receiver polls the server with LOAD requests:
   - the (min, max) range must include the hash of the senders public key (it's possible to query for exact match, ranges will have other uses
     discussed further down).
   - once a valid parcel is received that corresponds to the sender's public key, the payload is decrypted using the receiver's secret decryption
     key.

For a continued flow of data, all that is required is for a series of public keys be agreed -- this can be obtained using a master HD key, and
agreed upon sequence of derivation paths (see Bitcoin's xpub). Alternatively, each message can contain the public signing key to be used
in the next one in the chain. 

In this scenario, as long as public keys aren't reused, it isn't possible for any observer to link the different parcels as part of the same
communication.

However, the database server can (and we should assume will) keep logs of IP addresses and time of access to parcels, and can likely perform
network analysis to link users. Defeating that vector will require onion routing of requests (accessing the database server through Tor) or
parcel re-mailers as discussed further down.

## Bi-directional channels 

To establish a persistent bi-direction channel, users Alice and Bob must:

1 - Alice generates an *invite* consisting of (invite signing key, encryption key).

2 - Alice shares the *invite* with Bob using a side channel (the invite is assumed to be at least temporarily secure).

3 - Bob generates a master key to be shared with Alice in an *open channel* message.

4 - Bob publishes a parcel:
    - *program* is the invite's public verifying key
    - *payload* is the encrypted master public key from which the signing keys for future parcels will be derived.
    - *witness* is a signature generated using the invite's signing key.

5 - Alice polls for the *open channel* message, using the hash of the public key corresponding to the signing key in the invite. 
    Once it is received, starts polling for the subsequent parcels using the keys derived from the master key.

6 - Bob uses this Bob->Alice uni-directional channel to send Alice a new *invite* message for the corresponding Alice->Bob channel. Repeat
    steps 1-5 with Alice/Bob switching places.

It is clear that any outside actor with no access to the private keys in Alice's and Bob's devices can only disrupt this process
by obtaining the *invite* before step 5 is completed -- thus the *invite* message needs to be at least temporarily secured.

Furthemore an eavesdropper (including the database server) can only obtain further information linking both by network analysis. For many
applications that level of security is acceptable -- as most popular service providers suffer from the same limitation. It's possible
to mitigate that issue using futher techniques.

A database server can definitely deny service to an IP address, though this can also be mitigated by using multiple servers and onion routing.

It's clear that a man in the middle attack can't achieve more than a malicious database server.

## Public mailboxes

Using the above database primitives it's also possible to generate a sort of "public mailbox" that can be widely shared so as to allow
anonymous invites to be sent to an user.

Such a mailbox would be a tuple (encryption key, base hash, range).

For Alice's to send an invite to Bob's mailbox she must:

1 - Obtain (by multiple random attempts) a signing key such that it's hash lies between (base hash, base hash + range).

2 - Encrypt the invite using Bob's encryption key.

3 - Publish a parcel where:
    - *program* is the public key obtained in 1.
    - *payload* is the encrypted invite followed by the hash of the clear text invite.
    - *witness* is a signature obtained with the key found in 1.

To receive the invites Bob must:

1 - Poll for parcels in the range (base hash, base hash + range).

2 - For each parcel found the payload must be decrypted and the hash must match the clear text invite.

3 - Parcels whose hashes and clear text invite don't match should be ignored.


Note that *range* is an important number, as it defines how much work (key generation, hashing) must be done
to send a message to Bob's mail box.

A smaller number would prevent DoS more effectively, and reduce the cost of looking through parcels that randomly
match the hash range, while making it costlier for legitimate users to send messages.

To prevent older messages to crowd out new ones, hashes between (base hash, bashe hash + timestamp*range) could be used.
Bob would poll for parcels within a certain timestamp in the near past and another one in the near future.

To further simplify the sharing of mail box addresses, the base hash can be derived from the public encryption key, and the range
can be expressed as a power of two, or assumed to be a standard number.


## Parcel re-mailers

To mitigate network analisys risks, the idea of the "public mail box" can be harnessed to implement a "parcel remailer".

For Alice to send a parcel to Bob, through a remailer Ron, she must:

1 - Encrypt the parcel using Ron's mail box's encryption key.

2 - Send the encrypted parcel to Ron's mail box.

A remailer Ron would do as follows:

1 - Publicly anounce his "open mail box" as willing to receive re-mailing requests.

2 - When a message arrives at the mail box, decrypt it and publish the decrypted parcel to the server.

Multiple remailers can be chained in onion-routing manner, further twarting network analysis -- as long as any one re-mailer
in the chain is "honest".

Furthermore, it is very easy for any user to become a remailer, regardless of computing/network resources, by setting a mailbox
with low enough range. Known re-mailers can be gossiped around between users.

Users whishing to send a re-mailed parcel should keep a list of multiple re-mailers, which will lower the amount of work necessary
to find a good public key for one of them.

## Public broadcast messages

The same primitives can obviously be used for broadcasting a public "timeline" of messages, by publicly sharing a master public key, 
and then delivering multiple ordered parcels using the derived public keys.

It must be noted that it is trivial for a database server to "ban" a public "broadcaster" by dropping all parcels that correspond to any
of their keys.

Here is a tentative scheme for making censorship of broadcast flows costly -- with the downside of making "following" broadcasters costly too:

- Instead of just a master signing key, the broadcaster announces a range of hashes to look for his messages -- much in the same way as a
public in-box, except it's a public out-box -- the broadcaster's signature for each payload will need to be inserted in the parcel payload itself. 
- Anyone following that broadcaster must download *all* the parcels within the range, and figure out which ones correspond to the desired broadcaster.
- To give "temporary plausible deniability" and make preemptive censorship hard, the payload can be uploaded not in clear text, but encrypted
using an encryption key derived from Nth hash of the parcel's program.
- *N* must be large enough to delay figuring out which parcels correspond to which broadcaster before followers download the encrypted content.
- To preemptively censor the broadcaster, a database server would need to delay considerably a range of hashes, interfering with
activity by all users -- which would be at very least detectable, and most likely costly in terms of competitiveness vs other providers.

It must also be noted that sharing of messages signed by third parties through a friend-to-friend network of graffiti protocol channels would 
be hard to censor too.


## Funding Servers

The database servers function much in the same way as general TCP/IP infrastructure, transmitting parcels of data back and forth, and
if the protocol becomes popular paying for that service might become an issue.

It's likely that these services may be provided by ISPs or other paid providers.

For enhanced privacy, servers could allow for LN payments that generate "tokens" that can be used until a certain amount of traffic is reached.
By using Tor, multiple tokens, and re-mailers -- it might be possible to prevent linking payment identities to parcels and thus
avoid a new vector for network analysis.

Alternatively a large enough number of non-profit providers might split the domain of program hashes so that each only needs to deal with
a small part of the traffic -- with some level of redundancy.

Exactly who provides the server functionality is mostly irrelevant to the protocol, and new research can be done in collaborative protocols
for decentralizing this function (perhaps adpating existing ones) -- if needed.

## Applications

Besides messaging between users, it's clear this protocol could be used as a base layer of communication between multiple applications.

It's also possible that the *program* validation mechanism could be re-purposed to enforce a subset of business logic (perhaps partially
revealing user data), in a way similar to "smart contracts". It's unclear whether that could be useful or not.


## References

TODO
