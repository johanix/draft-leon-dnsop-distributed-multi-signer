- [Introduction](#introduction)
  - [Requirements Notation](#requirements-notation)
- [Terminology](#terminology)
- [Requirements](#requirements)
- [Multi-Signer Use Cases](#multi-signer-use-cases)
  - [Primary Use Case](#primary-use-case)
  - [Secondary Use Case](#secondary-use-case)
- [The Distributed Multi-Signer Model](#the-distributed-multi-signer-model)
  - [Multi-Signer Agent: Signer vs Sidecar](#multi-signer-agent-signer-vs-sidecar)
  - [Source of Truth](#source-of-truth)
    - [The COMBINER](#the-combiner)
- [Communication Between MSAs](#communication-between-msas)
  - [MSA Communication via REST API](#msa-communication-via-rest-api)
  - [MSA Communication via DNS](#msa-communication-via-dns)
- [Identifying the Designated Signers](#identifying-the-designated-signers)
- [Locating Remote Multi-Signer Agents](#locating-remote-multi-signer-agents)
  - [Locating a Remote API-Method Multi-Signer Agent](#locating-a-remote-api-method-multi-signer-agent)
  - [Locating a Remote DNS-Method Multi-Signer Agent](#locating-a-remote-dns-method-multi-signer-agent)
- [Synchronization of Changes Between MSAs](#synchronization-of-changes-between-msas)
  - [Leader/Follower Mode](#leaderfollower-mode)
  - [Peer Mode](#peer-mode)
- [Security Considerations](#security-considerations)
- [IANA Considerations.](#iana-considerations)
- [Change History (to be removed before publication)](#change-history-to-be-removed-before-publication)
---
title: "Distributed DNSSEC Multi-Signer"
abbrev: "Distributed Multi-Signer"
docname: draft-leon-distributed-multi-signer-00
date: {DATE}
category: std

ipr: trust200902
area: Internet
workgroup: DNSOP Working Group
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: L. Fernandez
    name: Leon Fernandez
    organization: The Swedish Internet Foundation
    country: Sweden
    email: leon.fernandez@internetstiftelsen.se
 -
    ins: E. Bergström
    name: Erik Bergström
    organization: The Swedish Internet Foundation
    country: Sweden
    email: erik.bergstrom@internetstiftelsen.se
 -
    ins: J. Stenstam
    name: Johan Stenstam
    organization: The Swedish Internet Foundation
    country: Sweden
    email: johan.stenstam@internetstiftelsen.se

normative:

informative:

--- abstract

This document presents an architecture for a distributed DNS
multi-signer model. It describes two models for multi-signer process
traversal: “leader/follower mode” and “peer mode”. It also discusses
two alternatives for secure communication between the individual
multi-signer agents: a RESTful API secured by TLS and “pure DNS”
communication secured by DNS SIG(0) signatures on each message.

The scope of the document is only the distributed aspect of DNS
multi-signer. The so-called “multi-signer processes” are the same as
described in {{!RFC8901}}.

TO BE REMOVED: This document is being collaborated on in Github at:
[https://github.com/johanix/draft-leon-dnsop-distributed-multi-signer](https://github.com/johanix/draft-leon-dnsop-distributed-multi-signer).
The most recent working version of the document, open issues, etc, should all be
available there.  The authors (gratefully) accept pull requests.

--- middle

# Introduction

The issue of how to eliminate so-called "single points of failure"
from systems to make them more robust is a recurring theme in systems
design and so also for DNS. In the DNS case redundancy is addressed by
having multiple name servers for the same zone. However, when the zone
is DNSSEC-signed there is traditionally an additional single point of
failure: the so-called "signer".

Multi-signer ({{!RFC8901}}) describes a process by which it is
possible to use more than one signer, by having the signers (or their
agents) communicate and exchange data that should be signed by the
other signer. The most obvious example is that each signer's
Key-Signing Key must sign a DNSKEY RRset that contains the
Zone-Signing Keys for all signers.

The communication between signers has two parts: first it is necessary
to find out what data each signer has for a zone. Once all data has
been collected it is possible to compute what changes are needed to
the zone data at each signer. That triggers the second phase where the
zone data for the individual signers is changed to get them in sync
with each other.

Knowledge of DNS NOTIFY {{!RFC1996}} and DNS Dynamic Updates
{{!RFC2136}} and {{!RFC3007}} is assumed. DNS SIG(0) transaction
signatures are documented in {{!RFC2931}}.

## Requirements Notation

The key words "**MUST**", "**MUST NOT**", "**REQUIRED**", "**SHALL**",
"**SHALL NOT**", "**SHOULD**", "**SHOULD NOT**", "**RECOMMENDED**",
"**NOT RECOMMENDED**", "**MAY**", and "**OPTIONAL**" in this document
are to be interpreted as described in BCP 14 {{!RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

# Terminology 

...

# Requirements

The requirements for an architecture for distributed multi-signer are
defined as follows:

 * Assuming all zone transfers are correctly set up, a zone owner MUST
   be able to signal to the individual multi-signer providers
   information sufficient for the providers to identify each other and
   establish secure communication.
 * The zone owner MUST be able to signal the intent to onboard an
   additional multi-signer provider. This must automatically initiate
   the multi-signer “add signer” process, as described in RFC nnnn.
 * The zone owner MUST be able to signal the intent to offboard an
   existing multi-signer provider. This MUST automatically initiate
   the multi-signer “remove signer” process, as described in RFC nnnn.
 * All signalling from zone owner to multi-signer providers SHOULD be
   carried out via data in the served zone, to ensure that all
   providers get the same configuration information at (almost) the
   same time.
 * By engaging a set of multi-signer providers (one or more), the zone
   owner MUST give up control over the following records:
   * All DNSSEC related records in the zone
   * The NS RRset
   * Any CDS and/or CSYNC RRsets

# Multi-Signer Use Cases

## Primary Use Case

The primary use case for the proposed multi-signer architecture is the
following scenario: A zone owner needs to remove the single point of
failure that the DNSSEC signer constitutes. For this reason it
contracts with two or more “multi-signer capable” service
providers. Each such service provider provides the following service:

 * Receive an unsigned zone via zone transfer.
 * Locate all active signers via the MSIGNER RRset as published by the
   zone owner. Establish secure communication with all remote signers
   (or their agents).
 * Update the NS, DNSKEY, CDS and CSYNC RRsets as needed, based on
   synchronization with the remote signers (or their agents).
 * Sign the zone, using own DNSKEYs, but with a published DNSKEY RRset
   that includes the DNSKEYs of other signers.
 * Distribute the signed zone to a set of downstream authoritative
   nameservers.

## Secondary Use Case

A slightly different use case is where a zone owner has a desire to
migrate from one DNSSEC provider to another. In the first step it
onboards the new provider by adding an MSIGNER RR with MSIGNER
State=“ON” identifying the new provider to the existing MSIGNER
RRset. This informs both the present providers and the incoming
provider about the addition of a new provider and the onboarding
process is automatically initiated.

Once the onboarding operation is completed it signals the pending
removal of another provider by changing the MSIGNER State flag for the
outgoing signer to “OFF”. This informs all the present providers about
the pending removal and the offboarding process is automatically
initiated.

# The Distributed Multi-Signer Model

The primary difference between monolithic and distributed multi-signer
is that the former has a central “controller” while the latter
doesn’t. But there is still an absolute need for synchronization
between the different participants in the distributed multi-signer
setup.

There are three immediate aspects for the design of a distributed
multi-signer architecture:

 * The first is “synchronization”: who decides what changes are needed.
 * The second is “method”: how to communicate between the individual
   instances in a multi-signer system.
 * The third is source of truth for different types of zone
   data. The zone owner is the source of truth for all unsigned zone
   data, except DNSSEC data and the NS RRset. The signer is the source
   of truth for all DNSSEC data in the zone. In a distributed
   multi-signer architecture the source of truth is

## Multi-Signer Agent: Signer vs Sidecar

In a distributed setup there must be a service located with each
multi-signer “signer” that manages communication with other
signers. This is referred to as the multi-signer agent, or MSA.

It is possible to implement support for the synchronization and
communication needs directly into each “signer” (i.e. typically an
authoritative nameserver with the ability to do online DNSSEC
signing). In this case the signer implements the MSA functionality.

However, it is also possible to separate the multi-signer complexity
into a separate service, which is sometimes referred to as
“sidecar”. This term is used as this service sits next to the signer,
and is under the same administrative control, but is a separate piece
of software. When using this design each signer has a sidecar attached
next to it. The sidecars configured as a “secondary nameserver” that
receives the (signed) zone from the actual signer.

The “sidecar” design has the major advantage of leaving the signer
almost entirely out of the multi-signer complexity. The requirements
are only that the “signer” treats the “sidecar” as a normal secondary
(sends NOTIFY messages and responds to zone transfer requests) and
that the “sidecar” has a configuration that allows it to make changes
to zones that the “signer” serves (most commonly via TSIG-signed DNS
UPDATEs, but other models are possible).

In this document the design using a separate MSA (i.e. a “sidecar”) is
used, while pointing out that it is possible to integrate this into a
future “signer” that implements both DNSSEC signing and the MSA
requirements.

## Source of Truth

A common design for DNSSEC signing (regardless of multi-signer)
is to use a separate, bump-on-the-wire signer. This is a signer that
receives the unsigned zone via an incoming zone transfer, signs the
zone, and publishes the signed zone via an outbound zone transfer. In
such a design the source of truth has been split up between the “zone
owner” (source of truth for all unsigned zone data), and the signer
(source of truth for all DNSSEC data in the zone).

In a distributed multi-signer architecture the source of truth is
further split up into three participants:

 * The zone owner is the source of truth for all unsigned zone data,
   except DNSSEC data and the NS RRset.
 * The signer is the source of truth for all data generated via DNSSEC
   signing: own DNSKEYs, NSEC/NSEC3 RRs, RRSIGs, etc.
 * The MSA is the source of truth for the RRsets that must be kept in
   sync across all the signers for the zone. This includes the zone NS
   RRset, DNSKEYs from other signers, CDS and CSYNC RRsets.

To be able to keep the signer as simple as possible the changes to the
NS, DNSKEY, CDS and CSYNC RRsets must be introduced into the unsigned
zone before the zone reaches the signer. Likewise, to keep the zone
owner as simple as possible (i.e. not involved in the details of the
multi-signer automation) these changes must be introduced into the
unsigned zone after the zone leaves the zone owner.

### The COMBINER

The consequence is that the NS, DNSKEY, CDS and CSYNC RRsets are
maintained via a separate piece of software inserted between the zone
owner and the signer. This is referred to as the multi-signer
COMBINER.

The COMBINER has the following features:

 * It supports inbound zone transfer of the unsigned zone from the
   zone owner.
 * It receives updates for the NS, DNSKEY, CDS and CSYNC
   RRsets from the MSA. Typically the mechanism used is DNS UPDATE
   with a TSIG signature, as this is easy to configure in a local
   context. However, other mechanisms, including APIs, are possible.
 * It stores all data received from the MSA separate from
   the zone data received from the zone owner.
 * Whenever it receives a new unsigned zone from the zone
   owner it COMBINES zone data from the zone owner (the majority of the
   zone) with specific zone data under control of the MSA: four
   specific RRsets, all in the apex of the zone: the DNSKEY, NS, CDS
   and CSYNC RRsets.
 * It does not sign the zone.
* It provides outbound zone transfer of the combined zone to the
   signer.

Example setup with two signers showing the logical flow of zone data
between the zone owner, the COMBINER, the signer and the MSA:

~~~
                            +--------------+
                            |     owner    |
               xfr          +-+---------+--+    xfr
            /----------------/           \--------------------\
           /                                                   \
    +-----+----+    DNS  +-----+  DNS/API  +-----+  DNS    +----+-----+
    | combiner +<--------+ msa +-----------+ msa +-------->+ combiner |
    +-----+----+  UPDATE +--+--+           +--+--+ UPDATE  +----+-----+
          |                 ^                 ^                 |
          v xfr             |                 |                 v xfr
    +-----+----+     xfr    |                 |   xfr      +----+-----+
    |  signer  +------------+                 +------------+  signer  |
    +-----+----+                                           +----+-----+
          |                                                     |
          v                                                     v
       +--+--+                                               +--+--+
       | NS  |--+                                            | NS  |+
       +-----+  |--+                                         +-----+|-+
          +-----+  |                                            +---+ |
             +-----+                                              +---+
~~~


# Communication Between MSAs

Also in the communication case there are two identified
alternatives. The first is to use a REST API between the MSAs and the
second is to use pure DNS communication.

## MSA Communication via REST API

REST APIs are well-known and a natural fit for many distributed
systems. The challenge is mostly in the initial setup of secure
communication. The certificates need to be validated, preferably
without a requirement on trusting a third party CA. The API endpoints
for each MSA need to be located. Once secure communication has been
established, using a REST API for MSA communication is
straight-forward.

## MSA Communication via DNS

This alternative is based on the observation that all the
communication needs between MSAs can be expressed via DNS
messages. Notifications are sent as DNS NOTIFY messages. In
Leader/Follower mode requests for changes to a zone are sent as a DNS
UPDATE from the Leader to the Follower. The sole remaining
communication requirement is for how to communicate information about
the current state between MSAs in an ongoing multi-signer
process. This is done via a dedicated EDNS(0) opcode specifically for
communicating multi-signer state. This model is based on
{{!draft-berra…}} that solves a similar problem for delegation
synchronization between child and parent.


# Identifying the Designated Signers 

It is the responsibility of the zone owner to choose a set of
“signers”, either internal or external to the zone owners
organization. These signers must be clearly and uniquely designated
via publication in the MSIGNER RRset, located at the apex of the zone
and consisting of one MSIGNER record for each signer.

The MSIGNER RRset must be added, by the zone owner, to the, typically
unsigned, zone that the zone owner maintains so that this RRset is
visible to the downstream signers and their multi-signer agents.
## The MSIGNER RRset Yada, yada.

The MSIGNER RR has the zone name that publishes the MSIGNER RRset as
the owner name and the two fields State  and Identity as RDATA.

zone.    MSIGNER State Identity

State
: Unsigned 8-bit. Defined values are 1=ON and 2=OFF. The value 0 is an error.
Values 3-127 are presently undefined. Values 128-255 are reserved for private 
use. The presentation format allows either as integers (1 or 2) or as tokens (“ON” 
or “OFF”).

Identity
: Domain name. Used to uniquely identify the Multi-Signer Agent.

Example:

zone.example.   MSIGNER ON msa.example.


# Locating Remote Multi-Signer Agents

When an MSA receives a zone via zone transfer from the signer it will
analyze the zone to see whether it contains an MSIGNER RRset. If there
is no MSIGNER RRset the zone must be ignored by the MSA from the
point-of-view of multi-signer synchronization.

If, however, the zone does contain an MSIGNER RRset then the MSA must
analyze this RRset to identify the other MSAs for the zone via their
target names in each MSIGNER record. If any of the other MSAs listed
in the MSIGNER RRset is previously unknown to this MSA then secure
communication with this other MSA must be established. 

Secure communication can be achieved via various transports and it is up to the
MSAs in the zone's MSIGNER records to determine amongst themselves. In this
document we propose two transports: DNS and API. We also establish DNS as a
baseline that MSAs MUST support to be compliant.

In the following two subsections we detail how an MSA can locate a remote MSA
and establish secure DNS-based and API-based communications, respectively.

## Locating a Remote DNS-Method Multi-Signer Agent

Locating a remote MSA using the DNS mechanism consists of the
following steps:

 * Lookup and DNSSEC-validate the URI record of the MSIGNER
   identity. This provides the domain name and port to which DNS
   messages should be sent.
 * Lookup and DNSSEC-validate the KEY record of the URI record target
   name. This enables verification of the SIG(0) public key of the
   remote MSA once communication starts.

Example: given the following MSIGNER record for a remote MSA:

zone.example.     IN MSIGNER ON  DNS msa.provider.com.

The local MSA will look up the URI record for msa.provider.com:

_dns._tcp.msa.provider.com.  IN  URI  10 10 “dns://ns.msa.provider.com:5399/”
_dns._tcp.msa.provider.com.  IN  RRSIG URI …

which triggers a lookup for ns.msa.provider.com. SVCB to get the IPv4
and IPv6 addresses as ipv4hints and ipv6hints in the response to the
SVCB query:

ns.msa.provider.com.   IN  SVCB  1 ipv4hint=5.6.7.8 ipv6hint=2001::53
ns.msa.provider.com.   IN RRSIG SVCB …

and also a look up for the KEY record for ns.msa.provider.com, which
may look like this:

ns.msa.provider.com.  IN KEY …
ns.msa.provider.com.  IN RRSIG KEY …

Once all the DNS lookups and DNSSEC-validation of the returned data
has been done, the local MSA is able to initiate communication with
the remote MSA and verify the identity of the responding party via the
validated KEY record for the remote MSAs SIG(0) public key.


## Locating a Remote API-Method Multi-Signer Agent

Locating a remote MSA with the identity “msa.example.” using the API
mechanism consists of the following steps:

* Lookup and DNSSEC-validate the URI record for
  “_https._tcp.msa.example.”. This provides the base URL that will be
  used to construct the individual API endpoints for the REST API. It
  also provides the port to use.
* Lookup and DNSSEC-validate the SVCB record for the domain name
  returned in the response to the URI query
  (eg. “api.msa.example.”). This provides the IP-addresses (and
  possibly the port, but we already have that) to use for
  communication with the MSA.
* Lookup and DNSSEC-validate the TLSA record for
  “_{port}._tcp.api.msa.example. This will enable verification of the
  certificate of the remote MSA once communication starts.

Example: given the following MSIGNER record for a remote MSA:

zone.example.     IN MSIGNER ON  API msa.provider.com.

the local MSA will look up the URI record for msa.provider.com:

_https._tcp.msa.provider.com.  IN  URI  10 10 “https://api.msa.provider.com:443/api/v2/”
_https._tcp.msa.provider.com.  IN  RRSIG URI …

which triggers a lookup for api.msa.provider.com IPv4 and IPv6
addresses as hints in an SVCB RR:

api.msa.provider.com.   IN  SVCB 1 ipv4hint=1.2.3.4 ipv6hint=2001::bad:cafe:443
api.msa.provider.com.   IN  RRSIG SVCB …

Now we know the IP-address and the port as well as the base URL to
use. Now look up the TLSA record for _443._tcp.api.msa.provider.com,
which may look like this:

  _443._tcp.api.msa.provider.com.  IN  TLSA 3 1 1 ….
  _443._tcp.api.msa.provider.com.  IN  RRSIG TLSA …

Once all the DNS lookups and DNSSEC-validation of the returned data
has been done, the local MSA is able to initiate communication with
the remote MSA and verify the identity of the responding party via the
TLSA record for the remote MSAs certificate.

## Multi-Signer EDNS(0) Option Format 

This document uses an Extended Mechanism for DNS (EDNS0) {{!RFC6891}}
option to include Key State information in DNS messages. The option is 
structured as follows: 

~~~
                                               1   1   1   1   1   1 
       0   1   2   3   4   5   6   7   8   9   0   1   2   3   4   5 
     +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
 0:  |                            OPTION-CODE                        |
     +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
 2:  |                           OPTION-LENGTH                       |
     +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
 4:  |           OPERATION           |           TRANSPORT           |
     +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
 8:  |        SYNCHRONIZATION        |                               /
     +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+ 
 10: / OPERATION-BODY                                                /
     +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
~~~

Field definition details: 

OPTION-CODE: 
    2 octets / 16 bits (defined in {{!RFC6891}}) contains the value TBD
    for KeyState.

OPTION-LENGTH: 
    2 octets / 16 bits (defined in {{!RFC6891}}) contains the length of 
    the payload (everything after OPTION-LENGTH) in octets and should 
    be 3 plus the length of the EXTRA-TEXT field (which may be zero 
    octets long). 

OPERATION:
    8 bits. Signals the type of operation the message performs. Currently,
    the operations HELLO and HEARTBET exist.

TRANSPORT:
    8 bits. Encodes the

OPERATION-BODY:
    variable-length. Used to carry operation-specific parameters.

### Encoding Transport Capabilities in the Multi-Signer EDNS(0) Option
An MSA signals its transport capabilities by setting the corresponding bits to
1.

0: DNS transport supported (baseline, MUST be supported by all MSAs)

1: API transport supported

2: <unused>

3: <unused>

4: <unused>

5: <unused>

6: <unused>

7: <unused>

### Encoding Synchronization Capabilities in the Multi-Signer EDNS(0) Option
An MSA signals its synchronization capabilities by setting the corresponding
bits to 1.

0: Leader/Follower synchronization supported (baseline, MUST be supported by all MSAs)

1: Peer-to-Peer synchronization supported

2: <unused>

3: <unused>

4: <unused>

5: <unused>

6: <unused>

7: <unused>

# Sequence Diagram Example of Establishing Secure Comms - "The Hello Phase"
The procedure of locating another MSA and establishing a secure communication,
referred to as "The Hello Phase" is examplified in the sequence diagram below.

The procedure is as follows:

1. The multisigner agents retrieve a zone that is to be signed, the become
   aware of this from observing the MSIGNER RRset.

2. They start querying any unrecognized MSIGNER identities. Here we only
   illustrate the baseline case where DNS-based communications is to be used
   in the following phase.

3. Once an MSA has received the desired responses (SVCB and KEY records in the
   baseline case) it send a NOTIFY message with a dedicated Multi-Signer OPT
   code with a "hello" signal. The sender uses this OPT field to signal its
   transport and synchronization capabilities. Similarly, the responder signals
   its capabilities using the same field.

4. When an MSA either gets a NOERROR response to its NOTIFY OPT(hello) message
   or responds with a NOERROR, it transitions out of "The Hello Phase" with
   the exchanging party and they transition to the next phaste where they start
   sending NOTIFY OPT(heartbeat) signals instead.

In case one MSA is too fast for the other, perhaps the zone transfer was
delayed for one of them, the slower one can simply respond in the negative to
any NOTIFY OPT(hello) it receives (perhaps the other party is polling it). Once
it is ready, it simply sends a NOTIFY OPT(hello) of its own and will hopefully
get a NOERROR or, it responds positively to the other party's message.


~~~
+----------+                 +----------+                        +----------+
|  Owner   |                 |  MSA A   |                        |  MSA B   |
+----------+                 +----------+                        +----------+
     |                            |                                    |
     |      AXFR(sign-me.se)      |                                    |
     |--------------------------->|                                    |
     |      AXFR(sign-me.se)      |                                    |
     |---------------------------------------------------------------->|
     |                            |                                    |
     |                            |                                    |
     |                            |       QUERY dns.msa-b.se SVCB?     |
     |                            |----------------------------------->|
     |                            |       QUERY dns.msa-b.se KEY?      |
     |                            |----------------------------------->|
     |                            |                                    |
     |                            |                                    |
     |                            |   NOTIFY sign-me.se OPT(hello)     |
     |                            |----------------------------------->|
     |                            |   NOERROR sign-me.se OPT(hello)    |
     |                            |<-----------------------------------|
     |                            |                                    |
     |                            |                                    |
     |                            |   NOTIFY sign-me.se OPT(heartbeat) |
     |                            |----------------------------------->|
     |                            |                                    |
     |                            |                                    |
     |                            |   NOTIFY sign-me.se OPT(heartbeat) |
     |                            |<-----------------------------------|
     |                            |                                    |
     |                            |                                    |
     |                            |                                    |

~~~
# Synchronization of Changes Between MSAs

There are two defined models for synchronization. The first
(Leader/Follower) has the advantage of more clearly mapping to the
original multi-signer model, with a single controller. The second
model has the advantage of less total communication between MSAs
(including no elections) but the potential disadvantage of more fine
grained communication during the execution of a multi-signer process.

At this stage it is not clear that one model is superior to the other.

## Leader/Follower Mode

In a leader/follower deployment, a designated multi-signer agent
assumes the role of a leader, directing other agents, or followers,
through the multi-signer process state transitions. In this mode it is
necessary to conduct “elections” where one of the MSAs is chosen as
the Leader before initiating a new multi-signer process. Once the
Leader has been chosen, this model is mostly equivalent to the
original multi-signer “model 2”, with a single controller. The other
MSAs (the followers) essentially become proxies between the controller
(the Leader) and the signers.

## Peer Mode

In peer mode, the MSAs still need to locate each other, but instead of
relying on trust in each other, each multi-signer agent operates
independently as a peer. I.e. each MSA executes each step in the
multi-signer process on its own. The communication is essentially
reduced to a notification mechanism (“I am now in state N”), although
authenticated to avoid having the contents of this communication
become an attack vector for an adversary.


# Security Considerations

Multi-signer is a complex system with a number of components and a
significant amount of automation. The authors believe that the only
way to make a multi-signer architecture useful in practice is via
automation. However, automation is a double-edged sword. It can both
make the system more robust and more vulnerable.

From a vulnerability point-of-view this architecture introduces several
new components into the zone signing and publication process. In
particular the COMBINER and the MSAs are new components that need
to be secure. The COMBINER has the advantage of not having to announce
its location to the outside world, as it only needs to communicate with
internal components (the zone owner, the signer and the MSA).

The MSAs are more vulnerable. They need to be discoverable by other MSAs
and hence they are also discoverable by an adversary. On the other hand,
the MSAs are not needed for a new zone to signed and published, they are
only needed when there are changes that require the MSAs to synchronize,
which is an infrequent event. Furthermore, should an MSA be unable to
fulfill its role during the execution of a multi-signer process, the
multi-signer process will simply stop where it is. Regardless of where
the stop (or rather pause) occurs, the zone will be fully functional
and once the MSA is able to resume its role, the multi-signer process
will continue from where it left off.

# IANA Considerations.
## New Multi-Signer EDNS Option

This document defines a new EDNS(0) option, entitled "Multi-Signer",
assigned a value of TBD "DNS EDNS0 Option Codes (OPT)" registry

TO BE REMOVED UPON PUBLICATION: 
[https://www.iana.org/assignments/dns-parameters/dns-parameters.xhtml#dns-parameters-11](foo)

   +-------+--------------------+----------+----------------------+
   | Value | Name               | Status   | Reference            |
   +-------+--------------------+----------+----------------------+
   | TBD   | Multi-Signer       | Standard | ( This document )    |
   +-------+--------------------+----------+----------------------+

## A New Registry for EDNS Option Multi-Signer Codes {: #keystate-code-registry}

The KeyState option also defines a 16-bit state field, for which IANA is
requested to create and mainain a new registry entitled "Multi-Signer Operations",
used by the Multi-Signer option. Initial values for the "Multi-Signer Operations"
registry are given below; future assignments in  in the 8-127 range are to be
made through Specification Required review {{?BCP26}}.

a link is [here](#new-keystate-edns-option)


+-----------+---------------------------------------------+-------------------+
| OPERATION | Mnemonic                                    | Reference         |
+-----------+---------------------------------------------+-------------------+
| 0         | <forbidden>                                 | ( This document ) |
+-----------+---------------------------------------------+-------------------+
| 1         | HELLO                                       | ( This document ) |
+-----------+---------------------------------------------+-------------------+
| 2         | HEARTBEAT                                   | ( This document ) |
+-----------+---------------------------------------------+-------------------+
| ??-??     | Unassigned                                  | ( This document ) |
+-----------+---------------------------------------------+-------------------+
| ??-??     | Private Use                                 | ( This document ) |
+-----------+---------------------------------------------+-------------------+


--- back

# Change History (to be removed before publication)

* draft-leon-distributed-multi-signer-00

> Initial public draft.
