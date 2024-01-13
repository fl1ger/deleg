---
title: Extensible Delegation for DNS
abbrev: DELEG
docname: draft-dnsop-deleg-latest
date: {DATE}
category: std
updates: 1035

ipr: trust200902
submissiontype: IETF
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
    name: Roy Arends
    organization: ICANN
    email: roy.arends@icann.org
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
A delegation in the Domain Name System (DNS) is a mechanism that enables efficient and distributed management of the DNS namespace. It involves delegating authority over subdomains to specific DNS servers via NS records, allowing for a hierarchical structure and distributing the responsibility for maintaining DNS records.

An NS record contains the hostname of the nameserver for the delegated namespace. Any facilities of that nameserver must be discovered through other mechanisms. This draft proposes a new extensible DNS record type, DELEG, which contains additional information about the delegated namespace and the capabilities of authoritative nameservers for the delegated namespace.
--- middle

# Introduction

In the Domain Name System {{!STD13}}, subdomains within the domain name hierarchy are indicated by delegations to servers which are authoritative for their portion of the namespace.  The DNS records that do this, called NS records, can only represent the name of a nameserver.  Clients can expect nothing out of this delegated server other than that it will answer DNS requests on UDP port 53.

As the DNS has evolved over the past four decades, this has proven to be a barrier for the efficient introduction of new DNS technology.  Many features that have been conceived come with additional overhead as they are constrained by the least common denominator of nameserver functionality.

The proposed DELEG record type remedies this problem by providing extensible parameters to indicate capabilities that a resolver may use for the delegated authority, for example that it should be contacted using a different transport mechanism than the default udp/53.

The DELEG record is served with the existing NS records and any DS records used to secure the delegation with DNSSEC, with all three types provided in the Authority section of DNS responses.  Legacy DNS resolvers will ignore the unknown DELEG type and continue to use the NS and DS records, per the testing described in Appendix A.  Resolvers that understand DELEG and its associated parameters can efficiently switch to the new mechanism.

The DELEG record leverages the Service Binding (SVCB) record format defined in {{?RFC9460}}, using a subset of the already defined service parameters.

DELEG can use AliasMode, inherited from SVCB, to insert a level of indirection to ease the operational maintenance of multiple zones by the same servers.  For example, an operator can have numerous customer domains all aliased to nameserver sets whose operational characteristics can be easily updated without intervention from the customers.  Most notably, this provides a method for addressing the long-standing problem operators have with maintaining DS records on behalf of their customers. This facility will be handled in a separate draft.

## Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
"OPTIONAL" in this document are to be interpreted as described in \\
BCP 14 {{?RFC2119}} {{?RFC8174}} when, and only when, they appear in
all capitals, as shown here.

Terminology regarding the Domain Name System comes from {{?BCP219}}, with addition terms defined here:

* \[tbd\]
* \[tbd\]

## Reasoning for changing delegation

* The DNS protocol has no method to indicate methods of secure transport between auth and resolver - e.g. authenticated DNS-over-TLS from resolver to auth.
* Delegation point NS records and glue address records are not DNSSEC signed. This presents a leap of faith. Spoofed delegation point NS records can be detected eventually if the domain is signed, but only after traffic is (potentially) sent to the spoofed endpoint.
* There is no secure way to signal what capabilities authoritative servers support. The client cannot rely on any new behavior and must resort to trial-and-error with EDNS, which can not be secured.
* The Registry, Registrar, Registrant (RRR) model has no formally defined role for  DNS operators. Consequently, registrants are the channel between DNS operators and registriesi/registrars on purely operational elements, such as adding NS records, modify DS records when rolling keys, etc. Deleg's AliasMode allows the registrants to delegate these facilities to a DNS Operator.

## Introductory Examples

To introduce DELEG record, this example shows the authority section of a DNS response that delegates a subdomain to another nameserver.


    example.com.  86400  IN DELEG  1 ns1.example.com. (
                    ipv4hint=192.0.2.1 ipv6hint=2001:DB8::1 )
    example.com.  86400  IN NS     ns1.example.com.
    ns1.example.com.    86400   IN  A  192.0.2.1
    ns1.example.com     86400   IN  AAAA    2001:DB8::1

In this example, the authoritative nameserver is delegating using the same parameters as regular DNS, but the delegation as well as the glues can be signed.

Like in SVCB, DELEG also offer the ability to use the Alias form delegation. The example below shows an example where example.com is being delegated with a DELEG AliasMode record which can then be further resolved using standard SVCB to locate the actual parameters.

    example.com.  86400  IN DELEG 0   config2.example.net.
    example.com.  86400  IN NS     ns2.example.net.

The example.net authoritative server may return the following SVCB records in response to a query as directed by the above records.

    config2.example.net 3600    IN SVCB . (
                    ipv4hint=192.0.2.54,192.0.2.56
                    ipv6hint=2001:db8:2423::3,2001:db8:2423::4 )

The above records indicate to the client that the actual configuration for the example.com zone can be found at config2.example.net

Later sections of this document will go into more detail on the resolution process using these records.

## Goal of the DELEG record

The primary goal of the DELEG records is to provide zone owners a method to signal capabilities to clients how to connect and validate a subdomain. This method coexists with NS records in the same zone. 

The DELEG record is authoritative in the parent zone and if signed has to be signed with the key of the parent zone. The target of an alias record is an SVCB record that exists and can be signed in the zone it is pointed at, including the child zone.

## Facilities 

The DELEG record is extensible in such a way that future innovations in the domain name system, such as new methods of secure transport, message encoding, error reporting, etc, does not depend on a re-design of the DNS. 

# DELEG Record Type

The SVCB record allows for two types of records, the AliasMode and the ServiceMode. The DELEG record takes advantage of both and each will be described below in depth. The wire format of and the registry for the DELEG record is the same as SVCB record defined in  {{?RFC9460}}

## Difference between the records

This document uses two different resource record types. Both records have the same functionality, with the difference between them being that the DELEG record MUST only be used at a delegation point, while the SVCB is used as a normal resource record and does not indicate that the label is being delegated. For example, take the following DELEG record:

    Zone com.:
    example.com.  86400  IN DELEG 1   config2.example.net.

When a client receives the above record, the resolver should send queries for any name under example.com to the nameserver at config2.example.net unless further delegated. By contrast, when presented with the records below:

    Zone com.:
    example.com.  86400  IN DELEG 0   config3.example.org.

    Zone example.org.:
    config3.example.org.  86400  IN SVCB 1 . ( ipv4hint=192.0.2.54,192.0.2.56
                    ipv6hint=2001:db8:2423::3,2001:db8:2423::4 )

A resolver trying to resolve a name under example.com would get the first record above from the parent authoritative server, .COM, indicating that the SVCB records found at config3.example.org should be used to locate the authoritative nameservers of example.com, and other parameters.

The primary difference between the two records is that the DELEG record means that anything under the record label should be queried at the delegated server while the SVCB record is just for redirection purposes, and any names under the record's label should still be resolved using the same server unless otherwise delegated.

It should be noted that both DELEG and SVCB records may exist for the same label, but they will be in different zones. Below is an example of this:

    Zone com.:
    example.com.  86400  IN DELEG 0   c1.example.org.

    Zone example.org.:
    c1.example.org.  86400  IN DELEG  1   config3.example.net. (
                                ipv6hint=2001:db8:2423::3 )
    c1.example.org.  86400  IN NS test.c1.example.org.
    test.c1.example.org. 600 IN A 192.0.2.1

    Zone c1.example.org:
    c1.example.org.  86400  IN SVCB 1   config2.example.net. (
                        ipv6hint=2001:db8:4567::4  )
    c1.example.org.  86400  IN NS test.c1.example.org.
    test.c1.example.org. 600 IN A 192.0.2.1

In the above case, the DELEG record for c1.example.org would only be used when trying to resolve names at or below c1.example.org. This is why when an AliasMode DELEG or SVCB record is encountered, the resolver MUST query for the SVCB record associated with the given name.


## AliasMode Record Type

In order to take full advantage of the AliasMode of DELEG and SVCB, the parent, child and resolver must support these records. When supported, the use of the AliasMode will allow zone owners to delegate their zones to another operator with a single record in the parent. AliasMode SVCB records SHOULD appear in the child zone when used in the parent. If a resolver were to encounter an AliasMode DELEG or SVCB record, it would then resolve the name in the TargetName of the original record using SVCB RR type to receive either another AliasMode record or a ServiceMode SVCB record.

For example, if the name www.example.com was being resolved, the .com zone may issue a referral by returning the following record:

    example.com.    86400    IN  DELEG     0   config1.example.net.

The above record would indicate to the resolver that in order to obtain the authoritative nameserver records for example.com, the resolver should resolve the RR type SVCB for the name config1.example.net.

### Multiple Service Providers

Some zone owners may wish to use multiple providers to serve their zone, in which case multiple DELEG AliasMode records can be used. In the event that multiple DELEG AliasMode records are encountered, the resolver SHOULD treat those as a union the same way this is done with NS records, picking one at random for the first lookup and eventually discovering the others. How exactly DNS questions are directed and split between configuration sets is implementation specific:

    example.com.    86400    IN  DELEG     0   config1.example.net.
    example.com.    86400    IN  DELEG     0   config1.example.org.

DRAFT NOTE: SVCB says that there "SHOULD only have a single RR". This ignores that but keeps the randomization part. Section 2.4.2 of SVCB

### Loop Prevention

The TargetName of an SVCB or DELEG record MAY be the owner of a CNAME record. Resolvers MUST follow CNAMEs as well as further alias SVCB records as normal, but MUST not allow more then 4 total lookups per delegation, with the first one being the DELEG referral and then 3 SVCB/CNAME lookups maximal.

Special care should be taken by both the zone owner and the delegated zone operator to ensure that a lookup loop is not created by having two AliasMode records rely on each other to serve the zone. Doing so may result in a resolution loop, and likely a denial of service. The mechanism on following CNAME and SVCB alias above should prevent exhaustion of server resources. If a resolution can not be found after 4 lookups the server should reply with a SERVFAIL error code.

## Deployment Considerations

The DELEG and SVCB records intends to replace the NS record while also adding additional functionality in order to support additional transports for the DNS. Below are discussions of considerations for deployment.

### AliasMode and ServiceMode in the Parent

Both the AliasMode and ServiceMode records can be returned for the DELEG record from the parent. This is different from the SCVB  {{?RFC9460}} specification and only applies for the DELEG RRSet in the parent.


### Rollout

When introduced, the DELEG and SVCB record will likely not be supported by the Root or second level operators, and may not be for some time. In the interim, zone owners may place these records into their zones. However this is only useful for further delegations down the tree as an SVCB record at the zone apex alone does not indicate a new delegation type. The only way to discover new delegations is with the DELEG record at the parent.


### Availability

If a zone operator removes all NS records before DELEG and SVCB records are implemented by all clients, the availability of their zones will be impacted for the clients that are using non-supporting resolvers. In some cases, this may be a desired quality, but should be noted by zone owners and operators.

## Response Size Considerations

For latency-conscious zones, the overall packet size of the delegation records from a parent zone to child zone should be taken into account when configuring the NS, DELEG and SVCB records. Resolvers that wish to receive DELEG and SVCB records in response SHOULD advertise and support a buffer size that is as large as possible, to allow the authoritative server to respond without truncating whenever possible.

# Implementation

## Including DELEG RRs in a Zone

### Signing DELEG RRs

#### The DELEG DNSKEY Flag


## Authoritative Name Servers

### Including DELEG RRs in a Response

### Responding to Queries for Type DELEG

### Authenticated Denial of DELEG 


## Example DNSSEC responses


## Resolving with DELEG

### extending the resolving algorithm

See RFC1034, section 5.3.3 step 4b.

### Priority of DELEG over NS and Glue Address records


## Authenticating DELEG Responses  

### Status of the DELEG DNSKEY flag

### Verifying Authenticated Denial of DELEG

### Stub Resolver Support

# DNSSEC and DELEG

Unlike NS records, DELEG records are always signed by the parent, similar to DS records.
The SVCB records the DELEG record point to, are signed as normal record types in a signed child/leaf zone.

TODO: As we already discovered a potential downgrade attack by just removing the DELEG refrerral we have to think about and come up with a way how to signal that the parent zone has DELEG.

# Privacy Considerations

All of the information handled or transmitted by this protocol is public information published in the DNS.

# Security Considerations

TODO: Fill this section out

## Availability of zones without NS

## Resolution procedure

There are three ways of having a fallback safe way of resolving a delegating using the new DELEG record. We need to find out which is the most deployable by doing some testing. All of them introduce some new part in the DNS query/response. They are:
* Use a new EDNS flag in the query to indicate that you want to receive the new DELEG record so that authoritative name servers that support them can include them and others don't. DELEG records would only be sent to resolvers using that EDNS flag putting it in the authority section. The failure case here would be the authoritative failing when getting the new EDNS flag.
* Keep queries the same, but when answering with a referral put the DELEG record and possible signatures in the authority part of the response with the NS and DS records
* Keep queries the same, but when answering put the DELEG records  and possible signatures in the additional section of the answer, while population authority only with NS and DS records

Here is an example of DNS interactions simplified for a full resolution after priming for www.example.com query type AAAA with the com and example.com authoritative servers supporting DELEG and the root not. The example is without qname minimization, but there is no difference when one would use that:

* Ask www.example.com qtype AAAA to a.root-servers.net the answer is:
    Answer section: (empty)
    Authority section:
        com.   172800 IN NS a.gtld-servers.net.
    Additional section:
        a.gtld-servers.net. 172800 IN AAAA 2001:db8:a83e::2:30
* Ask www.example.com qtype AAAA to a.gtld-servers.net the answer is:
        Answer section: (empty)
        Authority section:
            example.com.   172800 IN NS ns1.example.com.
            example.com.   172800 IN DELEG   1 config1.example.com.
                                        ( ipv6hint=2001:db8:440:1:1f::24 )
        Additional section:
            ns1.example.com. 172800 IN AAAA 2001:db8:322c::35:42
* Ask www.example.com qtype AAAA to config1.example.com (2001:db8:1:1f::24) the answer is:
            Answer section:
                www.example.com.  3600 IN AAAA 2001:db8:a0:322c::2
            Authority section: (empty)
            Additional section: (empty)

TODO: more resolution examples (e.g out of bailiwick)

### Failures when DELEG delegation is present

When a delegation using DELEG to a child is present, the resolver MUST use it and SERVFAIL if none of the configurations provided work.

# IANA Considerations

DELEG will use the SVCB IANA registry definitions in section 14.3 of {{!RFC9460}}.

--- back

# Appendix A Legacy Test Results {#Testing}

In December 2023, Roy Arends and Shumon Huque tested the core idea that enables this approach, that legacy resolvers would simply ignore the unknown DELEG record type from authoritative servers.  Tested both with DNSSEC and without against the most widely used DNS resolver implementations, the tests clearly supported the soundness of this method.  The tested legacy resolvers were all able to continue to successfully resolve the test domains via traditional DNS over port 53 even in the presence of DELEG records.

The tested legacy resolver installations included BIND, Unbound,
PowerDNS, and Knot.  In addition, the public resolver services run by Cloudflare (1.1.1.1), Google (8.8.8.8), and Packet Clearing House (9.9.9.9) were examined and provided similar results.

For more details about the specific testing methodology, please see test-plan.

# Acknowledgments {:unnumbered}

This draft is heavily based on past work done by Tim April in 
{{?I-D.tapril-ns2}} and thus extends the thanks to the people helping on this which are:
John Levine, Erik Nygren, Jon Reed, Ben Kaduk, Mashooq Muhaimen, Jason Moreau, Jerrod Wiesman, Billy Tiemann, Gordon Marx and Brian Wellington.

# TODO

RFC EDITOR:
: PLEASE REMOVE THE THIS SECTION PRIOR TO PUBLICATION.

* Write a security considerations section
* worked out resolution example including alias form delegation


# Change Log

RFC EDITOR:
: PLEASE REMOVE THE THIS SECTION PRIOR TO PUBLICATION.

~~~
01234567890123456789012345678901234567890123456789012345678901234567891
