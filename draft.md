%%%
title           = "Authenticated transport encryption for authoritative DNS"
abbrev          = "ADoT"
workgroup       = "DNS Privacy"
area            = "Internet"
submissiontype  = "IETF"
ipr             = "trust200902"
date            = 2020-11-10T18:00:59Z
keyword         = [
    "DNS",
    "TLS",
]

[seriesInfo]
name            = "Internet-Draft"
value           = "draft-fanf-dprive-adot-00"
status          = "standard"

[[author]]
initials        = "T."
surname         = "Finch"
fullname        = "Tony Finch"
organization    = "University of Cambridge"
 [author.address]
 email          = "dot@dotat.at"
  [author.address.postal]
  streets       = [
    "University Information Services",
    "Roger Needham Building",
    "7 JJ Thomson Avenue",
  ]
  city          = "Cambridge"
  country       = "England"
  code          = "CB3 0RB"

%%%

.# Abstract

This note describes how a DNS resolver can establish an encrypted
DNS-over-TLS connection to an authoritative DNS server, authenticated
using TLSA records.


{mainmatter}


# Introduction

The DNS has two weaknesses that can be fixed by encrypting connections
from DNS resolvers to authoritative servers:

  * Cleartext DNS has a privacy leak: the contents of queries and
    responses are visible to on-path passive attackers;

  * DNS messages are vulnerable to alteration by on-path active
    attackers.

DNSSEC protects the integrity and authenticity of most DNS data, but
it does not protect delegation NS records or glue, nor does it protect
other parts of a DNS message such as its header and OPT record.


## Overview

A zone's apex NS RRset lists the hostnames of the zone's authoritative
nameservers, for example,

        myzone.example.    NS    ns1.hoster.example.
        myzone.example.    NS    nsa.2ndary.example.

TLSA records [@!RFC6698] can indicate that a host supports TLS for a
particular service. DNS-over-TLS uses TCP port 853, so an
authoritative server can advertise support for encryption like

        _853._tcp.ns1.hoster.example.    TLSA    0 1 1 ( ... )

The profile of TLSA for authoritative DNS servers is in (#tlsa).

Sometimes a resolver cannot get a nameserver's TLSA records before
querying the nameserver, for instance when the nameserver is
in-bailiwick and its zone's delegation requires glue. For example,

        hoster.example.    NS    ns1.hoster.example.
        hoster.example.    NS    ns2.hoster.example.

To allow a resolver to use an encrypted transport before it can get a
nameserver's TLSA records, we change the nameserver's hostname to
include a tag that indicates which transports it supports, as
described in (#hints), for example,

        hoster.example.    NS    dot--ns1.hoster.example.
        hoster.example.    NS    dot--ns2.hoster.example.

The reason for this ugly hack is discussed in (#rationale).


## Terminology

DNS-related terminology can be found in [@!RFC8499].

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**,
**SHALL NOT**, **SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **MAY**,
and **OPTIONAL** in this document are to be interpreted as described in
[@!RFC2119].


# TLSA records for name servers {#tlsa}


# Encrypted transport hints {#hints}


# Resolver algorithm {#algorithm}


# IANA considerations {#iana}


# Security considerations

Many security considerations are discussed in (#rationale).


# Rationale {#rationale}

This section discusses the constraints on how the DNS can be updated
to include transport encryption.


## Explicit encryption support signal

How can a resolver know an authoritative server supports encryption?
There are three basic alternatives:

 1. No explicit signal: the resolver tries to make an encrypted
    connection, and falls back to cleartext if it gets an ICMP
    unreachable error or connection timeout.

    This is problematic because connection timeouts can be very slow,
    especially if the resolver tries multiple encrypted transports.
    This is also is vulnerable to downgrade attacks.

    The working group consensus is that an explicit signal is
    required.

 2. Signal in an EDNS [@?RFC6891] or DSO [@?RFC8490] option: the
    resolver starts by connecting in the clear, and upgrades to an
    encrypted connection if the authoritative server supports it.

    This is vulnerable to downgrade attacks. The initial cleartext
    connection adds latency, and would need to be specified carefully
    to avoid privacy leaks.

 3. Signal in DNS records: the resolver makes extra queries during
    iterative resolution to find out whether an authoritative server
    supports encryption before the resolver connects.

    The extra queries add latency, though that can be mitigated by
    querying concurrently, and by placing the new records on the
    existing resolution path.

    DNSSEC can provide authentication and downgrade protection.

This specification takes the last option, since it is best for
security and not too bad for performance.


## Where can nameserver encryption records go?

Where can we put the new DNS records to signal that a nameserver
supports encryption? There are a few issues to consider:

  1. Performance: we don't want the extra queries to slow down
     resolution too much;

  2. Authentication: DNSSEC does not protect delegation NS records or
     glue address records;

  3. DNS data model: we ought to re-use existing RRtypes according to
     their intended purpose;

  4. DNS extensibility: make use of well-oiled upgrade points and
     avoid changes that have a track record of very slow deployment;

  5. EPP compatibility: a zone's delegation is usually managed via the
     Extensible Provisioning Protocol [@?RFC5730] [@?RFC5731]
     [@?RFC5732] so any changes need to work with EPP.

The following subsections discuss the possible locations, and explain
why most of them have been rejected.


## In the reverse DNS?

Given a nameserver's IP address, a resolver might make a query like

        _853._tcp.1.2.0.192.in-addr.arpa.    TLSA?

This possibility is rejected because:

  * It would be very slow: after receiving a referral, a resolver
    would have to iterate down the reverse DNS, instead of immediately
    following the referral.

  * At the moment the reverse DNS refers to the forward DNS for NS
    records; this would make the forward DNS refer to the reverse DNS
    for TLSA records. Dependency loops are best avoided.

  * It's often difficult to put arbitrary records in the reverse DNS.

  * Encryption would apply to the server as a whole, whereas the
    working group consensus is that it should be possible for
    different zones on the same server to use encrypted and
    unencrypted transports.


## A new kind of delegation?

In theory, DNSSEC provides a way to update the DNS data model, along
the lines of the way NSEC3 was introduced [@?RFC5155]. The rough idea
is to introduce new DNSSEC algorithm types to indicate that a zone can
include new types of records that need special validation logic.
Existing validators will be able to resolve names in the zone, but
will treat it as insecure.

We might use this upgrade strategy to introduce new delegation records
that indicate support for transport encryption. However, it would
require a very long deployment timeline. It would also require a
corresponding upgrade to EPP.

This is much too difficult.


## Non-delegation records in the parent zone?

It's extremely difficult to change which DNS records can appear in a
delegation: they are restricted to NS, DS, and glue addresses. But in
principle the parent zone could contain information about a delegation
in a separate subdomain, like

        zone.example.    NS    ns1.zone.example.
        zone.example.    NS    ns2.zone.example.
        _853._tcp.ns1.zone._dot.example.    TLSA (...)
        _853._tcp.ns2.zone._dot.example.    TLSA (...)

The `_dot` tag makes the TLSA records into authoritative data in the
parent zone, rather than non-authoritative glue. The parent zone's
nameservers could include these records as additional data in
referrals. The resolver could authenticate them with DNSSEC and
immediately use an encrypted connection to the child zone's nameservers.

Although this idea would be secure and fast, it is incompatible with
EPP, so it is not deployable.


## New DS record algorithm?

The basic idea is to introduce a special DNSSEC algorithm number that
can be used in DS records to indicate support for encryption. This
runs into a number of problems, ultimately because it's trying to
abuse DS records for an unintended purpose.

  * DS records are per-zone, whereas encryption support is per-server.
    Support for incremental deployment would need a hack like having a
    DS record per nameserver, with further hacks to make it possible
    for humans to understand encryption DS records.

  * DS records can be updated by the child zone's DNSSEC key manager
    using CDS and/or CDNSKEY records [@?RFC8078]; CDS implementations
    would need to be updated to avoid interfering with encryption DS
    records.

  * There is a similar problem with ensuring that these DS records can
    be updated through EPP. There are registries that generate DS
    records from DNSKEY records themselves, rather than using DS
    records submitted by the zone owner, so these encryption DS
    records would have to be specified as the hash of a special DNSKEY
    record.

  * There are orders of magnitude more zones than there are
    nameservers. If the encryption signal is per-zone like this idea,
    then it requires orders of magnitude more work to deploy.


## Special-use nameserver addresses?

Could we abuse special IP addresses (such as a new special-use IPv6
address block) to signal support for encryption? This terrible idea is
listed for completeness; it's bad because:

  * It will cause problems for resolvers and other software that
    doesn't understand the special IP addresses and tries to use them
    as normal IP addresses;

  * Glue addresses are not authenticated by DNSSEC so it's vulnerable
    to downgrade attacks if the connection to the parent zone's
    nameserver is insecure.


## TLSA records alongside nameserver addresses

This idea is to use TLSA records in a straightforward way:

        ns1.zone.example.    A     192.0.2.1
        ns1.zone.example.    AAAA  2001:db8::1
        _853._tcp.ns1.zone.example.    TLSA    ( ... )

The TLSA records only appear in the zone's authoritative data, so
there are no delegation-related complications. They are signed with
DNSSEC, which protects against downgrade attacks.

It does not add any significant delay compared to a resolver that
validates nameserver addresses: when the resolver queries for the
nameserver's A and AAAA records, and the zone's DNSKEY records, it can
also concurrently request the nameserver's TLSA records.

There is a clear framework for supporting other transports such as
QUIC, by adding more TLSA records. (With the caveat that each new
transport requires another query because the TLSA owner name varies
per transport.)

The main problem with this setup is that it needs something like glue:
in many cases a resolver will need to know a nameserver's TLSA records
in order to securely fetch the nameserver's TLSA records.


## TLSA records without glue

The DNS needs glue addresses when a resolver follows a referral to a
zone whose nameservers are under the zone cut. When the resolver wants
to make an encrypted connection to these nameservers, it also needs
the TLSA records, which are also under the zone cut.

The resolver can make an unencrypted query for the TLSA records, then
upgrade to an encrypted connection. This leaks the zone name to
on-path passive attackers. The extra query is also slower.

This might be acceptable in limited circumstances: If the zone
containing the nameserver records is not used for other purposes, then
a passive snoop already knows the zone name from the IP address of the
nameserver. Queries for other domains hosted on the same nameserver
can remain private.


## Encryption hint in nameserver hostname

The last place where we might signal support for encryption is in the
nameserver hostname itself. There are a couple of ways this might be
done:

  * Just a hint that encrypted connections are possible;

  * Pack an authenticator into the hostname. (There is room for more
    than 300 bits in a domain name label.)

Nameserver hostnames appear both in the zone's apex NS RRset, which
can be authenticated by DNSSEC, and in the delegation NS RRset, which
is not authenticated. So referrals are vulnerable to
encryption-stripping downgrade attacks if the connection to the parent
zone's nameserver is insecure.



{backmatter}


# Acknowledgments