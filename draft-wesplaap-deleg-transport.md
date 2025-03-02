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
-
    name: Manu Bretelle
    organization: Meta
    email: chantr4@gmail.com

--- abstract

This document extends DELEG record, and SVCB records pointed to by DELEG record, as defined in {{?I-D.draft-wesplaap-deleg}}, with ability to specify transport protocols and authentication parameters supported by name servers.

--- middle

# Introduction

The new delegation mechanism based on DELEG record type allows to specify attributes a resolver can use when talking to a delegated authority. This document introduces parameters specific to different transport mechanism than the default udp/53 protocol.

Legacy DNS resolvers unaware of DELEG mechanism would continue to use the NS and DS records, while resolvers that understand DELEG and its associated parameters can efficiently switch to new transports.

## Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
"OPTIONAL" in this document are to be interpreted as described in \\
BCP 14 {{?RFC2119}} {{?RFC8174}} when, and only when, they appear in
all capitals, as shown here.

Terminology regarding the Domain Name System comes from {{?BCP219}}, with addition terms defined here:

* legacy transport name servers: An authoritative name server that only supports unencrypted DNS via UDP/TCP port 53

# New DNS transports

There has been a lot of work defining new ways to transport DNS over new encrypted protocols. While most of this work was focusd on the client/stub resolver to recursive resolver connection the protocols will remain the same for an recursive to authoritative connections when iterating for a name. These are:

* DNS over TLS as defined in {{?RFC7858}}
* DNS over HTTPS as defined in {{?RFC8484}}
* DNS over Dedicated QUIC Connections {{{?RFC9250}}}

While the DNS over HTTPs recommends HTTP/2 {{?RFC9113}} as the minimum version there are no differences when using to over HTTP/3 {{?RFC9114}}.

## Selecting transport protocols

The selection of the transport protocol to use to connect to an authoritative server for a delegation is done by the ALPN parameter as defined in {{?RFC9460}}. 

If there is no ALPN parameter the connection MUST be established using unencrypted DNS over UDP/TCP on port 53.

If there is an ALPN parameter up to 4 protocols in the list MUST be tried to connect to the authoritative server.

The following ALPN are allowed to be used and may need the additional parameters as defined in this table:
| ALPN  | parameters |
|-------|------------|
| dot   |            |
| doq   |            |
| h2    | dohpath    |
| h3    | dohpath    |


## Authenticating transport protocols

As all defined transport protocols here rely on TLS the authentication for the authentication of them is identical. When no TLSA parameter is present authentication is MUST be done using the normal PKI infrastructure of the recursive resolver. The name used for the authentication is the target name of the DELEG or SVCB record. When a TLSA paramert is present the authentication MUST be done using the digests in that record

### "TLSA" parameter

The "TLSA" SvcParamKey is a transport parameter representing a TLSA RRset {{?RFC6698}} to be used when connecting to TargetName using a TLS-based transport.

The SvcParamValue is a non-empty value-list.  The presentation and wire format of each value is the same as the presentation and wire format described for the TLSA record as defined in {{?RFC6698}}, sections 2.1 and 2.2 respectively.  To avoid wasting resources in the parent zone parents MAY reject RRSets containing "tlsa" SvcParams that use matching type 0 (exact match).

### Authentication Failures

When a resolver attempts to access nameserver delegated by a DELEG or SVCB record, if a connection error occurs, such as a certificate mismatch or unreachable server, the resolver SHOULD attempt to connect to the other nameservers delegated to until either exhausting the list or the resolver's policy indicates that they should treat the resolution as failed.

# Privacy Considerations

All of the information handled or transmitted by this protocol is public information published in the DNS.

# Security Considerations

TODO: Fill this section out

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


# Change Log

RFC EDITOR:
: PLEASE REMOVE THE THIS SECTION PRIOR TO PUBLICATION.

~~~
01234567890123456789012345678901234567890123456789012345678901234567891
