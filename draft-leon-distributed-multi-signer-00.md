- [Abstract](#abstract)
- [Introduction](#introduction)
  - [Requirements Notation](#requirements-notation)
- [Terminology](#terminology)
- [Requirements](#requirements)
- [Multi-Signer Use Cases](#multi-signer-use-cases)
  - [Primary Use Case](#primary-use-case)
  - [Secondary Use Case](#secondary-use-case)
  - [Tertiary Use Case](#tertiary-use-case)
- [The Distributed Multi-Signer Model](#the-distributed-multi-signer-model)
  - [Multi-Signer Agent: Integrated Signer vs Separate Agent](#multi-signer-agent-integrated-signer-vs-separate-agent)
  - [Source of Truth](#source-of-truth)
    - [The COMBINER](#the-combiner)
- [Identifying the Designated Signers](#identifying-the-designated-signers)
- [The SIGNER RRset](#the-signer-rrset)
  - [Use of the SIGNER State Field](#use-of-the-signer-state-field)
  - [Semantics of the SIGNER NSMgmt Field](#semantics-of-the-signer-nsmgmt-field)
- [Communication Between MSAs](#communication-between-msas)
  - [MSA Communication via DNS](#msa-communication-via-dns)
  - [MSA Communication via REST API](#msa-communication-via-rest-api)
  - [Locating Remote Multi-Signer Agents](#locating-remote-multi-signer-agents)
    - [Locating a Remote DNS-Method Multi-Signer Agent](#locating-a-remote-dns-method-multi-signer-agent)
    - [Locating a Remote API-Method Multi-Signer Agent](#locating-a-remote-api-method-multi-signer-agent)
      - [Fallback to DNS-based Communication](#fallback-to-dns-based-communication)
  - [The Initial HELLO Phase](#the-initial-hello-phase)
    - [DNS-based HELLO Phase](#dns-based-hello-phase)
    - [API-based HELLO Phase](#api-based-hello-phase)
    - [Interpretation of the HELLO Responses](#interpretation-of-the-hello-responses)
  - [Multi-Signer EDNS(0) Option Format](#multi-signer-edns0-option-format)
    - [Encoding Transport Capabilities in the Multi-Signer EDNS(0) Option](#encoding-transport-capabilities-in-the-multi-signer-edns0-option)
    - [Encoding Synchronization Capabilities in the Multi-Signer EDNS(0) Option](#encoding-synchronization-capabilities-in-the-multi-signer-edns0-option)
- [Sequence Diagram Example of Establishing Secure Comms - "The Hello Phase"](#sequence-diagram-example-of-establishing-secure-comms---the-hello-phase)
- [Synchronization of Changes Between MSAs](#synchronization-of-changes-between-msas)
  - [Leader/Follower Mode](#leaderfollower-mode)
  - [Peer Mode](#peer-mode)
- [Migration from Single-Signer to Multi-Signer](#migration-from-single-signer-to-multi-signer)
  - [Adding a single SIGNER record to an already signed zone](#adding-a-single-signer-record-to-an-already-signed-zone)
  - [Changing the SIGNER NSMGMT Field from OWNER To MSA](#changing-the-signer-nsmgmt-field-from-owner-to-msa)
  - [Migrating from a Multi-Signer Architecture Back to Single-Signer.](#migrating-from-a-multi-signer-architecture-back-to-single-signer)
- [Rationale](#rationale)
  - [Separation of MSA and COMBINER](#separation-of-msa-and-combiner)
- [Security Considerations](#security-considerations)
- [IANA Considerations.](#iana-considerations)
  - [New Multi-Signer EDNS Option](#new-multi-signer-edns-option)
  - [A New Registry for EDNS Option Multi-Signer Operation Codes](#a-new-registry-for-edns-option-multi-signer-operation-codes)
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
 -
    ins: S. Crocker
    name: Steve Crocker
    organization: Edgemoor Research Institute
    country: United States
    email: steve@shinkuro.com

normative:

informative:

--- abstract

# Abstract

This document presents an architecture for a distributed DNS
multi-signer model. It defines two multi-signer specific entities:
the "multi-signer agent" (MSA) that is responsible for the multi-signer
process and the "combiner", which manages combination of unsigned zone
data from the zone owner with zone data under control of the MSA. It
introduces a new DNS RRtype, SIGNER, that is used by the zone
owner to designate the chosen multi-signer agents. Furthermore it
describes a mechanism for the MSAs to establish secure communication
with each other, either via “pure DNS” communication secured by DNS
SIG(0) signatures on each message or via a RESTful API secured by
TLS. Finally, the document describes two models for multi-signer
process synchronization: “leader/follower mode” and “peer mode” and
the mechanism by which a set of MSAs decide which model to use for a
given zone.

The scope of the document is only the distributed aspect of DNS
multi-signer up to the point where secure communication and synchronization
method between MSAs has been established. The “multi-signer processes” that
deal with the actual synchronization required for multi-signer operation
are the same as described in {{!RFC8901}}.

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
with each other. All of this is done automatically.

However, from a slightly different perspective, the multi-signer
alternative is the more general case of DNSSEC signing, with the (very
common) case of a single signer being a special case. 

From that point of view, this document proposes an architecture for a
completely automated, distributed multi-signer model together with a
seamless transition path from the current single-signer model to the
multi-signer model. From the zone owners point of view, the transition
is done through the addition of a new RRtype, SIGNER, that is used to
designate the chosen multi-signer agents.

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
   * Any CDS and/or CSYNC RRsets
   * The NS RRset (according to the zone owner's policy)

# Multi-Signer Use Cases

## Primary Use Case

The primary use case for the proposed multi-signer architecture is the
following scenario: A zone owner needs to remove the single point of
failure that the DNSSEC signer constitutes. For this reason it
contracts with two or more “multi-signer capable” service
providers. Each such service provider provides the following service:

 * Receive an unsigned zone via zone transfer.
 * Locate all active signers via the SIGNER RRset as published by the
   zone owner. Establish secure communication with all remote signers
   (or their agents).
 * Update the DNSKEY, CDS and CSYNC RRsets as needed, based on
   synchronization with the remote signers (or their agents).
 * Update the NS RRset if allowed by the zone owner, based on synchronization
   with the remote signers (or their agents).
 * Sign the zone, using own DNSKEYs, but with a published DNSKEY RRset
   that includes the DNSKEYs of other signers.
 * Distribute the signed zone to a set of downstream authoritative
   nameservers.

## Secondary Use Case

A slightly different use case is where a zone owner has a desire to
replace one DNSSEC provider with another. In the first step it
onboards the new provider by adding an SIGNER RR with SIGNER
State=“ON” identifying the new provider to the existing SIGNER
RRset. This informs both the present providers and the incoming
provider about the addition of a new provider and the onboarding
process is automatically initiated.

Once the onboarding operation is completed the zone owner may trigger the
pending removal of another provider by changing the SIGNER State flag for the
outgoing signer to “OFF”. This informs all the present providers about
the pending removal and the offboarding process is automatically
initiated.

## Tertiary Use Case

The third use case is where a zone owner wants to migrate from a
single-signer model to a multi-signer model, but as a first step only
wants to transition the existing signer to be designated via a single
SIGNER record. Once that is done the zone owner can continue the
transition to a full multi-signer model at a later time by adding more
SIGNER records.

# The Distributed Multi-Signer Model

The primary difference between monolithic and distributed multi-signer
is that the former has a central “controller” while the latter
doesn’t. But there is still an absolute need for synchronization
between the different participants in the distributed multi-signer
setup.

There are three immediate aspects for the design of a distributed
multi-signer architecture:

 * The first is “synchronization”: who decides what changes are needed.
 * The second is “transport”: how to communicate between the individual
   instances in a multi-signer system.
 * The third is source of truth for different types of zone
   data. The zone owner is the source of truth for all unsigned zone
   data, except DNSSEC data and the NS RRset. The signer is the source
   of truth for all DNSSEC data in the zone. In a distributed
   multi-signer architecture the source of truth is

## Multi-Signer Agent: Integrated Signer vs Separate Agent

In a distributed setup there must be a service located with each
multi-signer “signer” that manages communication with other
signers. This is referred to as the multi-signer agent, or MSA.

It is possible to implement support for the synchronization and
communication needs directly into each “signer” (i.e. typically an
authoritative nameserver with the ability to do online DNSSEC
signing). In this case the signer implements the MSA functionality.

However, it is also possible to separate the multi-signer functionality
into a separate agent. This agent sits next to the signer, and is
under the same administrative control, but is a separate piece of
software. When using this design each signer has an agent attached
next to it. Each agent is configured as a “secondary nameserver” to a
signer and receives the (signed) zone from this signer.

The “separate agent” design has the major advantage of leaving the signer
almost entirely out of the multi-signer complexity. The requirements
are only that the “signer” treats the “agent” as a normal secondary
(sends NOTIFY messages and responds to zone transfer requests) and
that the “agent” has a configuration that allows it to make changes
to zones that the “signer” serves (most commonly via TSIG-signed DNS
UPDATEs, but other models are possible).

In this document the design using a separate MSA is used, while pointing
out that it is possible to integrate this into a future “signer” that
implements both DNSSEC signing and the MSA functionality.

## Source of Truth

A common design for DNSSEC signing (regardless of multi-signer)
is to use a separate, bump-on-the-wire signer. This is a signer that
receives the unsigned zone via an incoming zone transfer, signs the
zone, and publishes the signed zone via an outbound zone transfer. In
such a design the source of truth has been split up between the “zone
owner” (source of truth for all non-DNSSEC zone data), and the signer
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
   zone) with specific zone data under control of the MSA: three
   specific RRsets, all in the apex of the zone: the DNSKEY,CDS
   and CSYNC RRsets.
 * If zone owner policy so allows, it will also update the NS RRset with
   the unified data from all MSAs.
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

# Identifying the Designated Signers 

It is the responsibility of the zone owner to choose a set of
“signers”, either internal or external to the zone owners
organization. These signers must be clearly and uniquely designated
via publication in the MSIGNER RRset, located at the apex of the zone
and consisting of one MSIGNER record for each signer.

The MSIGNER RRset must be added, by the zone owner, to the, typically
unsigned, zone that the zone owner maintains so that this RRset is
visible to the downstream signers and their multi-signer agents.


# The SIGNER RRset

The SIGNER RR has the zone name that publishes the SIGNER RRset as
the owner name (i.e. the SIGNER RRset must be located at the apex of
the zone). The RDATA consists of two fields "State" and "Identity":

zone.example.    SIGNER State NSMgmt Identity

State:
    Unsigned 8-bit. Defined values are 1=ON and 2=OFF. The value 0
    is an error.  Values 3-127 are presently undefined. Values 128-255
    are reserved for private use. The presentation format allows
    either as integers (1 or 2) or as tokens (“ON” or “OFF”).

NSMgmt:
    Unsigned 8-bit. Defined values are 1=Zone owner and 2=MSA. The value
    0 is an error. Values 3-255 are presently undefined (and not expected
    to be defined). The presentation format allows either as integers (1
    or 2) or as tokens (“OWNER” or “AGENT”).

Identity:
    Domain name. Used to uniquely identify the Multi-Signer Agent.

Example:

zone.example.   SIGNER ON AGENT msa.example.

## Use of the SIGNER State Field

The SIGNER State field is used to signal to all MSAs what the status of
each MSA is from the point-of-view of the zone owner. The two possible
values are "ON" and "OFF" where "ON" means that the MSA is a currently
designated signer for the zone and "OFF" means that the MSA is previously
designated signer for the zone that is in the process of being offboarded.

The reason for the "OFF" state is that the offboarding process involves
the remaining signers (hence the signalling) and it is important to know
which signer is being offboarded so that the correct data may be removed
in the correct order during the multi-signer "remove signer" process
(see {{!RFC8901}}).

Once the offboarding process is complete the SIGNER RR for the offboarded 
MSA may be removed from the zone at the zone owners discretion.

## Semantics of the SIGNER NSMgmt Field

The NSMgmt field is used to signal to the MSAs who is responsible for
the NS RRset for the zone. The two possible values are "OWNER" and "AGENT".

The value "OWNER" means that the zone owner is responsible for the NS
RRset and is responsible for updating the NS RRset with the unified data
from all MSAs. In this case the COMBINER MUST NOT in any way modify the NS
RRset as received from the zone owner.

The value "AGENT" means that the MSA is responsible for the contents of the NS
RRset. In this case the COMBINER MUST ensure that the NS RRset is updated
with the unified NS RRset data from all MSAs.

# Communication Between MSAs

For the communication between MSAs there are two choices that need to be
made among the designated MSAs for a zone. The first is what "transport"
to use for the communication. The second is what "synchronization" model
to use when executing future multi-signer processes.

The two defined transport alternatives are:

* DNS-based communication (mandatory to support)
* REST API-based communication

Each has pros and cons and at this point in time it is not clear that one
always is better than the other. To simplify the choice of transport DNS-based
communication is mandatory to support and the REST API-based communication
may only be used if all MSAs support it. Supported transports are signaled
in the Multi-Signer EDNS(0) Option (see section NNN below).

The two defined synchronization alternatives are:

* Leader/Follower synchronization (mandatory to support)
* Peer-to-Peer synchronization

Just as for transport, supported synchronization models are signaled in the
Multi-Signer EDNS(0) Option (see section NNN below).

 ## MSA Communication via DNS

This transport alternative is based on the observation that all the
communication needs between MSAs can be expressed via DNS
messages. Notifications are sent as DNS NOTIFY messages. Requests
for changes to a zone are sent as DNS UPDATE messages, etc. The
sole remaining communication requirement is for how to communicate
information about the current state between MSAs in an ongoing
multi-signer process. For this reason a dedicated EDNS(0) opcode
specifically for multi-signer synchronization is proposed.

This model is based on {{!draft-berra…}} that solves a similar
problem for delegation synchronization between child and parent,
which has already been implemented and shown to work.

## MSA Communication via REST API

REST APIs are well-known and a natural fit for many distributed
systems. The challenge is mostly in the initial setup of secure
communication. The certificates need to be validated, preferably
without a requirement on trusting a third party CA. The API endpoints
for each MSA need to be located. Once secure communication has been
established, using a REST API for MSA communication is
straight-forward.

## Locating Remote Multi-Signer Agents

When an MSA receives a zone via zone transfer from the signer it will
analyze the zone to see whether it contains an SIGNER RRset. If there
is no SIGNER RRset the zone MUST be ignored by the MSA from the
point-of-view of multi-signer synchronization.

If, however, the zone does contain an SIGNER RRset then the MSA must
analyze this RRset to identify the other MSAs for the zone via their
target names in each SIGNER record. If any of the other MSAs listed
in the SIGNER RRset is previously unknown to this MSA then secure
communication with this other MSA must be established. 

Secure communication can be achieved via various transports and it is up to the
MSAs in the zone's SIGNER records to determine amongst themselves. In this
document we propose two transports: DNS and API. We also establish DNS as a
baseline that MSAs MUST support to be compliant.

In the following two subsections we detail how an MSA can locate a remote MSA
and establish secure DNS-based and API-based communications, respectively.

### Locating a Remote DNS-Method Multi-Signer Agent

Locating a remote MSA using the DNS mechanism consists of the
following steps:

 * Lookup and DNSSEC-validate a URI record for the SIGNER identity.
   This provides the domain name and port to which DNS messages should be sent.
 * Lookup and DNSSEC-validate the SVCB record of the URI record target to
   get the IP addresses to use for communication with the remote MSA.
 * Lookup and DNSSEC-validate the KEY record of the URI record target name.
   This enables verification of the SIG(0) public key of the remote MSA
   once communication starts.

Example: given the following SIGNER record for a remote MSA:

zone.example.     IN SIGNER ON msa.provider.com.

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


### Locating a Remote API-Method Multi-Signer Agent

Locating a remote MSA using the API mechanism consists of the following steps:

* Lookup and DNSSEC-validate the URI record for for the HTTPS protocol for
  the SIGNER identity. This provides the base URL that will be
  used to construct the individual API endpoints for the REST API. It
  also provides the port to use.
* Lookup and DNSSEC-validate the SVCB record for the URI record target.
  This provides the IP-addresses to use for communication with the MSA.
* Lookup and DNSSEC-validate the TLSA record for the port and protocol
  specified in the URI record. This will enable verification of the
  certificate of the remote MSA once communication starts.

Example: given the following SIGNER record for a remote MSA:

zone.example.     IN SIGNER ON  msa.provider.com.

the local MSA will look up the URI record for msa.provider.com:

_https._tcp.msa.provider.com.  IN  URI  10 10 “https://api.msa.provider.com:443/api/v2/”
_https._tcp.msa.provider.com.  IN  RRSIG URI …

which triggers a lookup for api.msa.provider.com IPv4 and IPv6
addresses as hints in an SVCB RR:

api.msa.provider.com.   IN  SVCB 1 ipv4hint=1.2.3.4 ipv6hint=2001::bad:cafe:443
api.msa.provider.com.   IN  RRSIG SVCB …

Now we know the IP-address and the port as well as the base URL to
use. Finally the TLSA record for _443._tcp.api.msa.provider.com is looked up,
with a response that may look like this:

  _443._tcp.api.msa.provider.com.  IN  TLSA 3 1 1 ….
  _443._tcp.api.msa.provider.com.  IN  RRSIG TLSA …

Once all the DNS lookups and DNSSEC-validation of the returned data
has been done, the local MSA is able to initiate communication with
the remote MSA and verify the identity of the responding party via the
TLSA record for the remote MSAs certificate.

#### Fallback to DNS-based Communication

If the API-based communication fails, either because needed DNS records
are missing, the TLSA record fails to validate the remote MSAs certificate
or the remote MSA simply doesn't respond, the local MSA MUST fall back to
DNS-based communication.

## The Initial HELLO Phase

When two MSAs need to communicate with each other for the first time (because
they are both deisgnated signers for the same zone), they need to establish
secure communication. This is done in a "HELLO" phase where the MSAs
exchange information about their capabilities.

### DNS-based HELLO Phase

When using DNS-based communication the HELLO phase is done by sending a
NOTIFY(SOA) for the zone that triggered the need for communication. The
NOTIFY message MUST contain a Multi-Signer EDNS(0) Option (see
section NNN below). 

In the Multi-Signer EDNS(0) Option the OPERATION field MUST have the value
"HELLO" (1). Furthermore, the MSA signals its transport and synchronization
capabilities in the TRANSPORT and SYNCHRONIZATION fields. This message is
signed with the SIG(0) key for the local MSA for which the public key is
published as a KEY record for the MSA.

In the response to the NOTIFY, the remote MSA does the same and the two
MSAs can now verify each other's identity and are also aware of the other
MSAs transport and synchronization capabilities.

### API-based HELLO Phase

When using API-based communication the HELLO phase is done by sending a
REST API POST request to the remote MSA at the "/hello" endpoint. The request
MUST contain a JSON encoded object with the following fields:

* "transport": The transport capabilities of the local MSA.
* "synchronization": The synchronization capabilities of the local MSA.

The response MUST contain a JSON object with the following fields:

* "transport": The transport capabilities of the remote MSA.
* "synchronization": The synchronization capabilities of the remote MSA.

### Interpretation of the HELLO Responses

Once an MSA has received HELLO responses from all other MSAs that are designated
signers for the zone, it knows the capabilities of the MSAs as a group. It can
then use this information to determine which transport to use:

* If all MSAs support API-based communication, the MSAs will use API-based
  communication.
* If one or more MSAs only support DNS-based communication, the MSAs will use
  DNS-based communication for this zone.

Likewise, each MSA now knows the synchronization capabilities of the other
MSAs and can determine which synchronization model to use:

* If all MSAs support the Peer-to-Peer synchronization model, the MSAs
  will use the Peer-to-Peer synchronization model for this zone.
* If one or more MSAs only support the Leader/Follower synchronization
  model, the MSAs will use the Leader/Follower synchronization model for
  this zone.

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
    8 bits. Signals the type of operation the message performs. This document
    defines the two operations HELLO and HEARTBEAT. For a complete distributed
    multi-signer specification a number of additional operations will need to be
    specified, either in a revision to this document or in a subsequent document.

TRANSPORT:
    8 bits. Encodes the transport capabilities of the MSA. With 8 bits it is possible to
    define up to 8 different transports of which this document defines two: DNS and API.

SYNCHRONIZATION:
    8 bits. Encodes the synchronization capabilities of the MSA. With 8 bits it is
    possible to define up to 8 different synchronization models of which this document
    defines two: Leader/Follower and Peer-to-Peer.

OPERATION-BODY:
    Variable-length. Used to carry operation-specific parameters.

### Encoding Transport Capabilities in the Multi-Signer EDNS(0) Option

An MSA signals its transport capabilities by setting the corresponding bits to
1.

0: DNS transport supported (baseline, MUST be supported by all MSAs)

1: API transport supported

2: unused

3: unused

4: unused

5: unused

6: unused

7: unused

### Encoding Synchronization Capabilities in the Multi-Signer EDNS(0) Option

An MSA signals its synchronization capabilities by setting the corresponding
bits to 1.

0: Leader/Follower synchronization supported (baseline, MUST be supported by all MSAs)

1: Peer-to-Peer synchronization supported

2: unused

3: unused

4: unused

5: unused

6: unused

7: unused

# Sequence Diagram Example of Establishing Secure Comms - "The Hello Phase"

The procedure of locating another MSA and establishing a secure
communication, referred to as "The Hello Phase" is examplified in the
sequence diagram below.

The procedure is as follows:

1. The multisigner agents receive a zone via zone transfer. By
   analyzing the SIGNER RRset each MSA become aware of the identities
   of the other MSAs for the zone. I.e. each MSA knows which other
   MSAs it needs to communicate with.  Communication with each of
   these, previously unknown, remote MSAs is referred to as "NEEDED".

2. Each MSA starts aquiring the information needed to establish secure
   communications with any previously unknown MSAs. Here we only
   illustrate the baseline case where DNS-based communications is to
   be used in the following phase. Once all needed information has
   been collected the communication with this remote MSA is considered
   to be "KNOWN".

3. Once an MSA has received the required information (URI, SVCB and
   KEY records in the baseline case) it sends a NOTIFY message with a
   dedicated Multi-Signer OPT code with OPERATION="HELLO". The sender
   uses this OPT field to signal its transport and synchronization
   capabilities. Similarly, the responder signals its capabilities
   using the same field.

4. When an MSA either gets a NOERROR response to its NOTIFY OPT(hello)
   message or responds with a NOERROR, it transitions out of "The
   Hello Phase" with the exchanging party and they transition to the
   next phaste where they start sending NOTIFY OPT(heartbeat) signals
   instead. The communication with the remote MSA is now considered to
   be in the "OPERATIONAL" state.

In the case where one MSA is aware of the need to communicate with
another MSA, but the other is not (eg. the zone transfer was dealyed
for one of them), the slower one SHOULD respond with a RCODE=REFUSED
to any NOTIFY OPT(hello) it receives. Once it is ready, it will send
its own NOTIFY OPT(hello) which should be responded to with a
RCODE=NOERROR.

~~~
+----------+                 +----------+                        +----------+
|  Owner   |                 |  MSA A   |                        |  MSA B   |
+----------+                 +----------+                        +----------+
     |                            |                                    |
     |      AXFR(sign-me.se.)     |                                    |
     |--------------------------->|                                    |
     |      AXFR(sign-me.se.)     |                                    |
     |---------------------------------------------------------------->|
     |                            |                                    |
     |                            |                                    |
     |                            |  QUERY _dns._tcp.msa-b.se. URI?    |
     |                            |----------------------------------->|
     |                            |  QUERY ns.msa-b.se. SVCB?          |
     |                            |----------------------------------->|
     |                            |  QUERY ns.msa-b.se. KEY?           |
     |                            |----------------------------------->|
     |                            |                                    |
     |                            |                                    |
     |                            |  NOTIFY sign-me.se. OPT(hello)     |
     |                            |----------------------------------->|
     |                            |  NOERROR sign-me.se. OPT(hello)    |
     |                            |<-----------------------------------|
     |                            |                                    |
     |                            |                                    |
     |                            |  NOTIFY sign-me.se. OPT(heartbeat) |
     |                            |----------------------------------->|
     |                            |                                    |
     |                            |                                    |
     |                            |  NOTIFY sign-me.se. OPT(heartbeat) |
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

# Migration from Single-Signer to Multi-Signer

The migration from a single-signer to a multi-signer architecture is
done by adding the SIGNER RRset to the zone. However, this may be
done in several steps.

## Adding a single SIGNER record to an already signed zone

Adding a single SIGNER record to a zone that is already signed by
the DNS provider "provider.com" with NSMGMT=OWNER is a no-op that
does not change anything:

zone.example. IN SIGNER ON OWNER msa.provider.com.

The zone was already signed by the DNS provider "provider.com" and
the provider added any needed DNSSEC records, including DNSKEYs. The
zone NS RRset was managed by the zone owner. All of this is unchanged
by the addition of the SIGNER RRset.

## Changing the SIGNER NSMGMT Field from OWNER To MSA

In a multi-signer architecture each MSA publishes the data it contributes to
the zone under the domain name {zone}.{identity}. I.e. the zone DNSKEYs that
the MSA msa.provider. uses are published as

zone.example.msa.provider. DNSKEY ...
zone.example.msa.provider. DNSKEY ...

Likewise, the NS records for the zone are published as

zone.example.ns.msa.provider. NS ...
zone.example.ns.msa.provider. NS ...

To migrate from "owner maintained" NS RRset to "MSA maintained", the 
zone owner must verify that the NS RRset as published by the MSA is
correct and in sync with the NS RRset as published by the zone owner itself.
After this verification the zone owner changes the SIGNER NSMGMT field in the
existing SIGNER record from NSMGMT=OWNER to NSMGMT=MSA.

## Migrating from a Multi-Signer Architecture Back to Single-Signer.

If for some reason a zone owner wants to migrate back to a single-signer
architecture, the process is essentially the reverse of the migration from
single-signer to multi-signer:

1. The zone owner offboards all MSAs but one (the one that will be the single-signer)
2. The zone owner must verify that the NS RRset it publishes (in the unsigned zone) 
   is correct and in sync with the NS RRset as published by the remaining MSA.
3. The zone owner changes the SIGNER NSMGMT field in the SIGNER record from
   NSMGMT=MSA to NSMGMT=OWNER. 

The zone is now essentially back to a single-signer architecture.
The remaining SIGNER record may be removed from the zone.

TO BE REMOVED BEFORE PUBLICATION:
# Rationale

## Separation of MSA and COMBINER

It is possible to integrate all three multi-signer components (signer,
msa and combiner) into a single piece of software (or two pieces,
depending on the preferred way of slicing the functionality). However,
such a composite module would be a fairly complex piece of software.
This document aims to describe the functional separation of the different
components rather than make a judgement on software design alternatives.
Hence possible implementation choices are left to the implementer.

# Security Considerations

Multi-signer is a complex system with a number of components and a
significant amount of automation. The authors believe that the only
way to make a multi-signer architecture useful in practice is via
automation. However, automation is a double-edged sword. It can both
make the system more robust and more vulnerable.

While all communication between MSAs is authenticated (either via SIG(0)
signatures ore TLS), the signalling from the zone owner to the MSAs is
via the SIGNER RRset in an unsigned zone. This is a potential attack
vector. However, securing zone transfers from zone owner to DNS providers
is a well-known issue with lots of existing solutions (TSIG, zone transfer
via a secure channel, zone transfer-over-TLS, etc). Employing some of these
solutions is strongly recommended.

From a vulnerability point-of-view this architecture introduces
several new components into the zone signing and publication
process. In particular the COMBINER and the MSAs are new components
that need to be secure. The COMBINER has the advantage of not having
to announce its location to the outside world, as it only needs to
communicate with internal components (the zone owner, the signer and
the MSA).

The MSAs are more vulnerable. They need to be discoverable by other
MSAs and hence they are also discoverable by an adversary. On the
other hand, the MSAs are not needed for a new zone to signed and
published, they are only needed when there are changes that require
the MSAs to synchronize, which is an infrequent event. Furthermore,
should an MSA be unable to fulfill its role during the execution of a
multi-signer process, the multi-signer process will simply stop where
it is. Regardless of where the stop (or rather pause) occurs, the zone
will be fully functional and once the MSA is able to resume its role,
the multi-signer process will continue from where it left off.

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

## A New Registry for EDNS Option Multi-Signer Operation Codes

The Multi-Signer option also defines an 8-bit operation field, for
which IANA is requested to create and mainain a new registry entitled
"Multi-Signer Operations", used by the Multi-Signer option. Initial
values for the "Multi-Signer Operations" registry are given below;
future assignments in in the 3-127 range are to be made through
Specification Required review {{?BCP26}}.

+-----------+---------------------------------------------+-------------------+
| OPERATION | Mnemonic                                    | Reference         |
+-----------+---------------------------------------------+-------------------+
| 0         | forbidden                                   | ( This document ) |
+-----------+---------------------------------------------+-------------------+
| 1         | HELLO                                       | ( This document ) |
+-----------+---------------------------------------------+-------------------+
| 2         | HEARTBEAT                                   | ( This document ) |
+-----------+---------------------------------------------+-------------------+
| 3-127     | Unassigned                                  | ( This document ) |
+-----------+---------------------------------------------+-------------------+
| 128-255   | Private Use                                 | ( This document ) |
+-----------+---------------------------------------------+-------------------+


--- back

# Change History (to be removed before publication)

* draft-leon-distributed-multi-signer-00

> Initial public draft.
