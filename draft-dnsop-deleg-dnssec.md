---
title: Extensible Delegation for DNS -- trust anchor side-loading
abbrev: DELEG
docname: draft-dnsop-deleg-dnssec-latest
date: {DATE}
category: std

ipr: trust200902
workgroup: dnsop
area: Internet
category: info
keyword: Internet-Draft

stand_alone: yes
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

This document describes how to use new delegation mechanism DELEG in AliasMode {{?I-D.draft-dnsop-deleg}} to offload keys into a different DNS subtree. This creates chain of trust which goes "sideways", enables sharing keys among multiple zones, and allows DNS operators to manage keys without involving parent zones.

--- middle

# Introduction

Traditionally DNSSEC {{?RFC4033}} uses top-down chain of trust, which requires individual DS RRs in the parent zone and this must match DNSKEY in the child zone. This document describes how to use new delegation mechanism DELEG in AliasMode to offload keys into a different subtree, which has several implications.

The DELEG record leverages the Service Binding (SVCB) record format defined in {{?RFC9460}}, using a subset of the already defined service parameters as well as new parameters described here.

By using an AliasMode inherited from SVCB, DELEG also allows a level of indirection to ease the operational maintenance of multiple zones by the same operator.  For example, an operator can have numerous customer domains sharing the same DNSSEC public keys. These keys can updated without intervention from the customers.  Most notably, we expect that this provides a method for addressing the long-standing problem operators have with maintaining DS records on behalf of their customers.

## Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
"OPTIONAL" in this document are to be interpreted as described in \\
BCP 14 {{?RFC2119}} {{?RFC8174}} when, and only when, they appear in
all capitals, as shown here.

Terminology regarding the Domain Name System comes from {{?BCP219}}, with addition terms defined here:

* [tbd]
* [tbd]

## Reasoning for introducing new form of chain of trust

* DNS operators don’t have a formal role in the current system. Consequently, registries/registrars generally don’t talk to operators and users must act as go-between and make technical changes they don’t understand.
    * Asking domain owners to add or modify DS records when rolling DNSSEC keys etc. This is very inflexible when an operator needs to make technical changes, and it's assumed that it might be one of reasons why DNSSEC deployment on the authoritative side is low.


## Introductory Examples

In this example, the parent zone is using DELEG in AliasMode as a level of indirection. Essentially the parent points DELEG-supporting resolvers to different subtree where actual parameters are stored, instead of storing all parameters directly in the parent zone.

    example.com.  86400  IN DELEG 0     config.operator.example.
    example.com.  86400  IN NS          ns1.operator.example.

The NS record serves as compatibility shim for old clients.
The above DELEG record indicates to the new client that the actual configuration for the example.com zone can be found at config.operator.example.

The operator.example authoritative server may return the following SVCB records in response to a query as directed by the above records.

    config.operator.example. 3600    IN SVCB ns1.operator.example. (
                    ipv4hint=192.0.2.54,192.0.2.56
                    ipv6hint=2001:db8:2423::3,2001:db8:2423::4
                    sharedds="12345 13 1 DSFDSAFDSAFVCFSGFDSGDSAGFSA" )


Later sections of this document will go into more detail on the resolution process using these records.

### Multiple Service Providers
TODO:

Some zone owners may wish to use multiple providers to serve their zone, in which case multiple DELEG AliasMode records can be used. In the event that multiple DELEG AliasMode records are encountered, the resolver SHOULD treat those as a union the same way this is done with NS records, picking one at random for the first lookup and eventually discovering the others. How exactly DNS questions are directed and split between configuration sets is implementation specific:

    example.com.    86400    IN  DELEG     0   config1.example.net.
    example.com.    86400    IN  DELEG     0   config1.example.org.

DRAFT NOTE: SVCB says that there "SHOULD only have a single RR". This ignores that but keep the randomization part. Section 2.4.2 of SVCB

#### "sharedds"

The "sharedds" SvcParamKey is a delegation parameter representing a modified DS RRset.  For usage notes, see {{dnssec}}.

The SvcParamValue is a non-empty value-list.  The presentation and wire format of each value is the same as the presentation and wire format described for the DS record as defined in {{?RFC4034}}, sections 5.3 and 5.1 respectively.  When computing the digest, the DNSKEY Owner Name is always set to "." (i.e., the root), because this DS record approves the use of the specified DNSKEY on any zone that is delegated to this endpoint.

If the "sharedds" SvcParamKey is omitted, the applicable DS RRset is the one that is present at the zone cut, if any.

As a special case, the presentation value "disabled", corresponding to an empty value in wire format, indicates that there is no DS RRset for resolution to this delegated endpoint, even if a DS RRset is present at the zone cut.

This SvcParam is "automatically mandatory" (i.e. non-ignorable) in DELEG for validating resolvers.

## Deployment Considerations

The DELEG and SVCB records intends to replace the NS record while also adding additional functionality in order to support additional transports for the DNS. Below are discussions of considerations for deployment.

### Rollout

DNSSEC chain of trust provided by the "sharedds" parameter will be taken into account only by supporting resolvers validators. If the legacy DS record is not present in the parent zone the zone will be in DNSSEC insecure state for legacy resolvers and validators. In some cases, this may be a desired quality, e.g. because introducing DS into the parent zone is operationally too complicated, but it should be noted by zone owners and operators.

# DNSSEC and DELEG {#dnssec}

TODO: Should DS at parent serve as fallback if the SVCB does not have sharedds=? What if there is DS at the parent side and sharedds= in the SVCB?
If there are any DS records on the same name as a DELEG record, ...

When using the "sharedds" SvcParamKey, each DELEG record MAY indicate different DS contents.  This allows delegation of a zone to multiple signers with different DNSKEYs, and allows those configurations to change independently.  Note that a zone is only as secure as its least secure "sharedds" SvcParam.

# Privacy Considerations

All of the information handled or transmitted by this protocol is public information published in the DNS.

# Security Considerations

TODO: Fill this section out

## Resolution procedure

# IANA Considerations

## New SvcParamKey Values

This document defines new SvcParamKey values in the "Service Binding (SVCB) Parameter Registry".

| SvcParamKey | NAME       | Meaning    | Reference       |
|-------------|------------|------------|-----------------|
| TBD2        | sharedds   | DS RRset   | (This Document) |

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
