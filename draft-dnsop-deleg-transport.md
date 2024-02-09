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

This document extends DELEG record, and SVCB records pointed to by DELEG record, as defined in `{{?I-D.draft-dnsop-deleg}}`, with ability to specify transport protocols and authentication parameters supported by name servers.

--- middle

# Introduction

The new delegation mechanism based on DELEG record type allows to specify attributes a resolver can use when talking to a delegated authority. This document introduces parameters specific to different transport mechanism than the default udp/53 protocol.  Transport selection and configuration follows the SVCB mapping for DNS {{!RFC9461}}, with some modifications described in this document.

Legacy DNS resolvers unaware of DELEG mechanism would continue to use the NS and DS records, while resolvers that understand DELEG and its associated parameters can efficiently switch to new transports.

## Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
"OPTIONAL" in this document are to be interpreted as described in \\
BCP 14 {{?RFC2119}} {{?RFC8174}} when, and only when, they appear in
all capitals, as shown here.

Terminology regarding the Domain Name System comes from {{?BCP219}}, with addition terms defined here:

* (tbd)
* (tbd)

## Introductory Examples

To introduce DELEG record with DNS-over-TLS {{?RFC7858}}, this example shows a possible response from an authoritative in the authority section of the DNS response when delegating to another nameserver.


    example.com.  86400  IN DELEG  1 ns2.example.com. (
                    alpn=dot
                    tlsa="2 0 1 ABC..."
                    ipv4hint=192.0.2.54
                    ipv6hint=2001:db8:2423::3 )
    example.com.  86400  IN NS     ns1.example.com.
    ns1.example.com.    86400   IN  A  192.0.2.1
    ns1.example.com     86400   IN  AAAA    2001:DB8::1

In this example, the authoritative nameserver is delegating using DNS-over-TLS.
Like in SVCB, DELEG also offer the ability to use the AliasMode delegation. The example below shows an example where example.com is being delegated with a DELEG AliasMode record which can then be further resolved using standard SVCB to locate the actual parameters.

    example.com.  86400  IN DELEG 0   config2.example.net.
    example.com.  86400  IN NS     ns1.example.net.

The example.net authoritative server may return the following SVCB records in response to a query as directed by the above records.

    config2.example.net 3600    IN SVCB ns2.example.net. (
                    alpn=dot
                    tlsa="2 0 1 ABC..."
                    ipv4hint=192.0.2.54
                    ipv6hint=2001:db8:2423::3 )

The above records indicate to the client that the actual configuration for the example.com zone can be found at config2.example.net.

Later sections of this document will go into more detail on the resolution process using these records.

## Goal of the DELEG record

The primary goal of transport specification in DELEG records is to provide zone owners a way to signal to new clients how to connect to servers serving a child domain that can coexist with NS records in the same zone, and do not break software that does not support new transports.

# Use of SvcParams

All SvcParamKeys for the "dns" scheme {{!RFC9461}} apply as specified.  These are the "transport parameters", describing how to reach an endpoint.

The "alpn" transport parameter is OPTIONAL to include (unlike in SVCB-DNS, where it is generally required).  If the "alpn" SvcParamKey is omitted, the only available transport is presumed to be unencrypted DNS over UDP/TCP port 53.  Endpoints can indicate that insecure transport is not available by specifying "mandatory=alpn".

When TargetName is below the zone cut, DELEG ServiceMode records MUST include "ipv4hint" or "ipv6hint" SvcParamKeys.  These keys provide the address glue, which enables the initial connection to this endpoint.

## New transport parameter: "tlsa"

The "tlsa" SvcParamKey is a transport parameter representing a TLSA RRset {{?RFC6698}} to be used when connecting to TargetName using a TLS-based transport. If present, this SvcParam MUST contain all the TLSA records whose owner name ({{?RFC6698, Section 3}}) can be inferred from TargetName and the other SvcParams (e.g., "alpn", "port"). Like "ipv4hint"/"ipv6hint", this SvcParam serves two purposes in DELEG:

* When a nameserver serves its own TargetName, the "tlsa" SvcParam MUST be included, because any TLSA queries would encounter a circular dependency.
* Otherwise, the TLSA query cannot generally be performed until after the SvcParams are received, potentially delaying connection establishment.  The "tlsa" SvcParam might save time in this case.

When this SvcParam is not present, certificate validation MAY be performed either using DANE (if TLSA records are present) or by PKI-based TLS certification validation using trusted root certification authorities.

The SvcParamValue is a non-empty value-list.  The presentation and wire format of each value is the same as the presentation and wire format described for the TLSA record as defined in {{?RFC6698}}, sections 2.1 and 2.2 respectively.  To avoid wasting space in the parent zone, parents MAY reject RRsets containing "tlsa" SvcParams that use matching type 0 (exact match).

As a special case, the presentation values "enabled" and "disabled" correspond to a single octet value of 1 and 0 in wire format.  A value of "enabled" indicates that TLSA records exist but are not embedded in this record and must be queried directly.  A value of "disabled" indicates that TLSA records do not exist, and DANE MUST NOT be used with this record.  A value of "enabled" SHOULD be accompanied by "mandatory=tlsa", while a value of "disabled" SHOULD NOT.  A value of "enabled" MUST NOT be used when the nameserver serves its own TargetName, as this creates a circular dependency.

Resolvers that support TLS-based transports MUST adopt one of the following behaviors:

1. Use DANE for authentication, and treat any endpoints lacking DANE support as incompatible.
1. Use PKI for authentication, and treat any DANE-only endpoint as incompatible.
1. Support both DANE and PKI for authentication, preferring DANE if it is available for each endpoint.

This SvcParamKey MAY be used in any SVCB context where TLSA usage is defined.

| SvcParams                            | DANE  | PKI |
|--------------------------------------|-------|-----|
| (no params)                          | No    | No  |
| alpn=doq                             | Maybe | Yes |
| alpn=doq tlsa="..."                  | Yes   | Yes |
| alpn=doq tlsa="..." mandatory=tlsa   | Yes   | No  |
| alpn=doq tlsa=enabled mandatory=tlsa | Yes   | No  |
| alpn=doq tlsa=disabled               | No    | Yes |
{: title="Representation of all combinations of DANE and PKI support"}

# Identifying the Server

In the SVCB mapping for DNS {{!RFC9461}}, the client is presumed to know an "authentication name" that is used to identify and authenticate the server in TLS.  When DANE is in use {{!I-D.ietf-dnsop-svcb-dane}}, the owner of the authentication name can use DNSSEC to authorize a different TLS server identity.  Otherwise, the DNS contents do not affect the authentication process, so the connection is secure even if DNS responses have been modified by an adversary.

In DELEG, the client (a resolver) does not initially know an "authentication name" for any of the desired authoritative servers.  The resolver only knows the apex name of the child zone.  This name is not used as the authentication name.  Instead, the TargetName of each initial DELEG record serves as the initial authentication name for connections relying on that record.  The subsequent connection setup (and SVCB queries, if the DELEG record is in AliasMode) are performed in accordance with {{!RFC9461}} (and {{!I-D.ietf-dnsop-svcb-dane}} if the resolver supports DANE).

If the initial DELEG record is resolved securely, this procedure is secure even if the child zone is not signed.  Otherwise, this procedure provides only opportunistic security, because an attacker could have replaced the DELEG record with one that authorizes the attacker's certificate.

## Examples

~~~Zone
example.com. DELEG 1 ns2.example.com. (
                alpn=dot
                tlsa="3 0 1 ABC..."
                ipv4hint=192.0.2.54
                ipv6hint=2001:db8:2423::3 )
~~~
{: title="SNI is 'ns2.example.com'"}

~~~Zone
example.com. DELEG 0 ns-1234.operator.example.

;; The operator.example. zone is presumed to be signed.
ns-1234.operator.example.          SVCB 1 x1234.opshost.example. alpn=dot

;; The opshost.example. zone is also presuemd to be signed.
_853._tcp.x1234.opshost.example. TLSA 2 0 1 ABC...
~~~
{: title="SNI is 'x1234.opshost.example' if the resolver supports DANE, otherwise 'ns-1234.operator.example'"}

~~~Zone
example.com. DELEG 1 ns-v4.example.com. (
                alpn=dot
                ipv4hint=192.0.2.54 )
example.com. DELEG 1 ns-v6.example.com. (
                alpn=dot
                ipv6hint=2001:db8:2423::3 )
~~~
{: title="SNI is 'ns-v4.example.com' for IPv4, and 'ns-v6.example.com' for IPv6"}

# Deployment Considerations

The DELEG and SVCB records intends to replace the NS record while also adding additional functionality in order to support additional transports for the DNS. Below are discussions of considerations for deployment.

## Availability

If a zone operator removes all NS records before DELEG and SVCB records are implemented by all clients, the availability of their zones will be impacted for the clients that are using non-supporting resolvers. In some cases, this may be a desired quality, but should be noted by zone owners and operators.

# Privacy Considerations

All of the information handled or transmitted by this protocol is public information published in the DNS.

# Security Considerations

TODO: Fill this section out

## Resolution procedure

TODO: Define how resolver selects which server connect to, especially when faced with selection of multiple transports. I's probably resolver's policy decision, but it should be said explicitly.

### Connection Failures

When a resolver attempts to access nameserver delegated by a DELEG or SVCB record, if a connection error occurs, such as a certificate mismatch or unreachable server, the resolver SHOULD attempt to connect to the other nameservers delegated to until either exhausting the list or the resolver's policy indicates that they should treat the resolution as failed.

The failure action when failing to resolve a name with DELEG/SVCB due to connection errors is dependent on the resolver operators policies. For resolvers which strongly favor privacy, the operators may wish to return a SERVFAIL when the DELEG/SVCB resolution process completes without successfully contacting a delegated nameserver(s) while opportunistic privacy resolvers may wish to attempt resolution using any NS records that may be present.

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
