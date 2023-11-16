---
title: Extensible Delegation for DNS
abbrev: DELEG
docname: draft-dnsop-deleg-transport-latest
date: {DATE}
category: std

ipr: trust200902
workgroup: dnsop
area: Internet
category: info
keyword: Internet-Draft

stand_alone: no
pi: [toc, sortrefs, symrefs]

author:
-
    ins: T. April
    name: Tim April
    organization:
    email: ietf@tapril.net
-
    ins: P.Špaček
    name: Petr Špaček
    organization: ISC
    email: pspacek@isc.org
-
    ins: R.Weber
    name: Ralf Weber
    organization: Akamai Technologies
    email: rweber@akamai.com
-
    ins: D.Lawrence
    name: David C Lawrence
    organization: Salesforce
    email: tale@dd.org

contributor:
-
    name: Christian Elmerot
    organization: Cloudflare
    email: christian@elmerot.se
-
    name: Edward Lewis
    organization: ICANN
    email: edward.lewis@icann.org
-
    name: Shumon Huque
    organization: Salesforce
    email: shuque@gmail.com
-
    name: Klaus Darilion
    organization: nic.at
    email: klaus.darilion@nic.at
-
    name: Libor Peltan
    organization: CZ.nic
    email: libor.peltan@nic.cz
-
    name: Vladimír Čunát
    organization: CZ.nic
    email: vladimir.cunat@nic.cz
-
    name: Shane Kerr
    organization: NS1
    email: shane@time-travellers.org
-
    name: David Blacka
    organization: Verisign
    email: davidb@verisign.com
-
    name: George Michaelson
    organization: APNIC
    email: ggm@algebras.org
-
    name: Ben Schwartz
    organization: Meta
    email: bemasc@meta.com
-
    name: Jan Včelák
    organization: NS1
    email: jvcelak@ns1.com
-
    name: Peter van Dijk
    organization: PowerDNS
    email: peter.van.dijk@powerdns.com
-
    name: Philip Homburg
    organization: NLnet Labs
    email: philip@nlnetlabs.nl
-
    name: Erik Nygren
    organization: Akamai Technologies
    email: erik+ietf@nygren.org
-
    name: Vandan Adhvaryu
    organization: Team Internet
    email: vandan@adhvaryu.uk

--- abstract

This document extends the DNS DELEG record, and SVCB records pointed to by the DELEG record, as defined in {{?I-D.draft-dnsop-deleg}}, with the ability to specify transport protocols and authentication parameters supported by nameservers.

--- middle

# Introduction

The new Domain Name System delegation mechanism based on the DELEG record type allows an authoritative nameserver to specify attributes a resolver can use when talking to another delegated authority. This document introduces parameters specific to different transport mechanisms than the default DNS protocol transports of udp and tcp to port 53.

Legacy DNS resolvers that are unaware of the DELEG mechanism would continue to use the NS and DS records, while resolvers that understand DELEG and its associated parameters can efficiently switch to new transports.

## Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
"OPTIONAL" in this document are to be interpreted as described in
BCP 14 {{?RFC2119}} {{?RFC8174}} when, and only when, they appear in
all capitals, as shown here.

Terminology regarding the Domain Name System comes from {{?BCP219}}, with addition terms defined here:

* transport parameters	  The set of attributes that define how to reach the nameserver specified in a ServiceMode DELEG record.
* [tbd]

## Introductory Examples

To use a DELEG record to point a resolver to an authority that supports DNS-over-TLS {{?RFC7858}}, this example shows a possible response from an authoritative server in the Authority section of the DNS response:

    example.com. 86400  IN DELEG 1 ns2.example.com. (
                    alpn=dot
                    tlsa="2 0 1 ABC..."
                    ipv4hint=192.0.2.54
                    ipv6hint=2001:db8:2423::3 )
    example.com. 86400  IN NS ns1.example.com.

The alpn attribute specifies DNS-over-TLS, with the tlsa attribute
providing the necessary cryptographic material to establish the
connection.  The ipv4hint and ipv6hint serves the traditional role of
glue, since the ns2 server lives in the example.com zone.

The Additional section would continue to serve glue records for the
nameservers within example.com as well, for the benefit of legacy
resolvers.  Note that these could potentially be the same addresses as
are used in the hints of a DELEG record; NS and DELEG are orthogonal.

    ns1.example.com. 86400   IN  A      192.0.2.1
    ns1.example.com  86400   IN  AAAA   2001:DB8::1

As with SVCB, DELEG also offer the ability to use AliasMode for a
delegation. The example below shows that example.com is
being delegated with a DELEG AliasMode record which can then be
further resolved using SVCB to locate the actual parameters.

    example.com.  86400  IN DELEG 0 config2.example.net.
    example.com.  86400  IN NS      ns1.example.net.

The example.net authoritative server's SVCB record could look like this:

    config2.example.net 3600    IN SVCB ns2.example.net. (
                        alpn=dot
                        tlsa="2 0 1 ABC..."
                        ipv4hint=192.0.2.54
                        ipv6hint=2001:db8:2423::3 )

This document defines in detail the resolution process using these records.

## SvcParams

All SvcParamKeys for the "dns" scheme from SVCB-DNS {{?RFC9461}} apply as specified.  These are the "transport parameters", attributes describing how to reach an endpoint.

The "alpn" transport parameter is OPTIONAL to include, unlike in SVCB-DNS, where it is generally required.  If the "alpn" SvcParamKey is omitted, the only available transport is presumed to be unencrypted DNS over UDP/TCP port 53.  Endpoints can indicate that insecure transport is not available by specifying "mandatory=alpn". [Why do we not require do53 to be explicit?]

When TargetName is below the zone cut, DELEG ServiceMode records MUST include "ipv4hint" or "ipv6hint" SvcParamKeys.  These keys provide the address glue, which enables the initial connection to this endpoint.

The following additional SVCB-DNS parameters are defined for the DELEG and SVCB ServiceModes.

### tlsa

The "tlsa" SvcParamKey is a transport parameter representing a TLSA RRset {{?RFC6698}} to be used when connecting to TargetName using a TLS-based transport. If present, this SvcParam SHOULD match the TLSA records whose base domain ({{?RFC6698, Section 3}}) is TargetName. Due to bootstrapping concerns, this SvcParamKey has been added to the DELEG record as the TLSA records might only be resolveable after the initial connection to the delegated nameserver was established. When this field is not present, certificate validation should be performed by either DANE or by traditional TLS certification validation using trusted root certification authorities.

The SvcParamValue is a non-empty value-list.  The presentation and wire format of each value is the same as the presentation and wire format described for the TLSA record as defined in {{?RFC6698}}, sections 2.1 and 2.2 respectively.  To avoid wasting resources in the parent zone, parents MAY reject RRSets containing "tlsa" SvcParams that use matching type 0 (exact match).

As a special case, the presentation value "disabled", corresponding to an empty value in wire format, indicates that DANE MUST NOT be used with this record.

To avoid a circular dependency, "tlsa" MUST appear in any DELEG record that is used to serve its own TargetName.

Resolvers that support TLS-based transports MUST adopt one of the following behaviors:

1. Use DANE for authentication, and treat any endpoints lacking DANE support as incompatible.
2. Use PKI for authentication, and treat any DANE-only endpoint as incompatible.  In this behavior, compatibility with `tlsa=disabled` endpoints is REQUIRED.
3. Support both DANE and PKI for authentication, preferring DANE if it is available for each endpoint.

This SvcParamKey MAY be used in any SVCB context where TLSA usage is defined.

~~~
;; The server is only authenticatable via DANE.  Any resolver that
;; lacks DANE support will treat this record as incompatible.
alpn=doq tlsa="..." mandatory=alpn

;; The server is only authenticatable via DANE, but also offers Do53
;; services.  All resolvers can access it, but only resolvers with
;; DANE support will use DoQ.
alpn=doq tlsa="..."
[... how does this signal that it also offers do53? Is the though that it is
  because only this draft specifies alpn and the core draft doesn't (currently)
  have a method to explicitly signal do53?  Maybe it should... ]

;; The server is authenticatable via PKI.  If TLSA records exist at
;; the SVCB-DANE owner names, it is also authenticatable via DANE.
;; This record cannot be used to serve the zone containing its TargetName.
alpn=doq

;; The server is only authenticatable via PKI.  Any resolver that
;; lacks PKI validation support will treat this record as incompatible.
alpn=doq tlsa=disabled mandatory=alpn
~~~
{: title="tlsa SvcParam Examples for DELEG"}

## Resolution procedure

As specified in {{?I-D.draft-dnsop-deleg}}, the ServiceMode DELEG records provide a weak signal from the operator about which transport mechanisms they would most prefer clients to use, but it is not mandatory that resolvers try the delegated servers in priority order.  It might not even be possible for a resolver to do, such as if the highest priority record specifies an alpn that the resolver does not implement, but a lower priority record has an alpn that it can use.

Thus, as with authority server selection for NS records, a resolver is free to use its own policy implementation to determine which servers to continue to use for resolution.

If a connection error occurs when a resolver attempts to access a nameserver delegated to by a DELEG or SVCB record, such as with a certificate mismatch or by being unreachable, the resolver SHOULD attempt to connect to the other nameservers in the DELEG/SVCB RRset to until either exhausting the list or the resolver's policy indicates that they should treat the resolution as failed.

Operators MAY return SERVFAIL when all attempts to use the DELEG/SVCB resolution process have failed.  This might have privacy benefits, especially if the DELEG records only offer encrypted transports.  More permissive resolvers can fall back to using the legacy DNS resolution process via NS records.

## Deployment Considerations

DELEG and SVCB records are intended to be the best method for modern resolvers to use delegation information for zones.

Realistically, however, legacy resolvers will continue to exist on the Internet for a long time to come.  If a zone operator removes all NS records, the availability of their zones will be impacted for the clients that are using non-supporting resolvers. In some cases, this may be a desired quality, but should be noted by zone owners and operators.

# Privacy Considerations

All of the information transmitted by this protocol is public information published in the DNS.

# Security Considerations

No new security risks are introduced by this specification.  [Too bold to say?]

The primary security considerations of DNS operators and domain owners are whether they should enable more secure transports for their domains, and how they want resolution to be handled if the secure transport is not available.

Similarly, resolvers should be considering whether they will use secure transport mechanisms, and if so how strongly they want to enforce their usage.

# IANA Considerations

## New SvcParamKey Values

This document defines new SvcParamKey values in the "Service Binding (SVCB) Parameter Registry".

| SvcParamKey | NAME | Meaning    | Reference       |
|-------------|------|------------|-----------------|
| TBD1        | tlsa | TLSA RRset | (This Document) |

--- back

# Acknowledgments

This draft is heavily based on past work (draft-tapril-ns2) done by Tim April and thus extends the thanks to the people helping on this which are:
John Levine, Erik Nygren, Jon Reed, Ben Kaduk, Mashooq Muhaimen, Jason Moreau, Jerrod Wiesman, Billy Tiemann, Gordon Marx and Brian Wellington.

# TODO

RFC EDITOR:
: PLEASE REMOVE THE THIS SECTION PRIOR TO PUBLICATION.

* Write a security considerations section
* Add the dohpath references RFC
* Define and give examples for DNSSEC svcparams
* worked out resolution example including AliasMode delegation
* DoH URI teamplte does not include post


# Change Log

RFC EDITOR:
: PLEASE REMOVE THE THIS SECTION PRIOR TO PUBLICATION.

~~~
01234567890123456789012345678901234567890123456789012345678901234567891
