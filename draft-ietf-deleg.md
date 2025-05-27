---
title: Extensible Delegation for DNS
abbrev: DELEG
docname: draft-ietf-deleg-latest
date: {DATE}
category: std
updates: 1035

ipr: trust200902
submissiontype: IETF
area: Internet
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
-
    ins: T. April
    name: Tim April
    organization: Google, LLC
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
-
    name: Bob Halley
    organization: Cloudflare
    email: bhalley@cloudflare.com

--- abstract
A delegation in the Domain Name System (DNS) is a mechanism that enables efficient and distributed management of the DNS namespace. It involves delegating authority over subdomains to specific DNS servers via NS records, allowing for a hierarchical structure and distributing the responsibility for maintaining DNS records.

An NS record contains the hostname of the nameserver for the delegated namespace. Any facilities of that nameserver must be discovered through other mechanisms. This document proposes a new extensible DNS record type, DELEG, for delegation of the authority for a domain. Future documents then can use this mechanism to use additional information about the delegated namespace and the capabilities of authoritative nameservers for the delegated namespace.
--- middle

# Introduction

In the Domain Name System {{!STD13}}, subdomains within the domain name hierarchy are indicated by delegations to servers which are authoritative for their portion of the namespace.  The DNS records that do this, called NS records, contain hostnames of nameservers, which resolve to addresses.  No other information is available to the resolver. It is limited to connect to the authoritative servers over UDP and TCP port 53. This limitation is a barrier for efficient introduction of new DNS technology. 

The proposed DELEG record type remedies this problem by providing extensible parameters to indicate capabilities and additional information, such as glue that a resolver may use for the delegated authority. It is authoritative and thus signed in the parent side of the delegation making it possible to validate all delegation parameters (names and glue records) with DNSSEC.

This document only shows how DELEG can be used instead of or along side a NS record to create a delegation. Future documents can use the extensible mechanism for more advanced features like connecting to a name server with an encrypted transport.

## Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
"OPTIONAL" in this document are to be interpreted as described in \\
BCP 14 {{?RFC2119}} {{?RFC8174}} when, and only when, they appear in
all capitals, as shown here.

Terminology regarding the Domain Name System comes from {{?BCP219}}, with addition terms defined here:

* legacy name servers: An authoritative server that does not support the DELEG record.
* legacy resolvers: A resolver that does not support the DELEG record.

# DELEG Record Type

The DELEG record uses a new resource record type, whose contents are identical to the SVCB record defined in {{?RFC9460}}. For extensions SVCB and DELEG use Service Parameter Keys (SvcParamKeys) and new SvcParamKeys that might be needed also will use the existing IANA Registry. 

## Differences from SVCB

* DELEG can only have two priorities 0 indicating INCLUDE and 1 indicating a DIRECT delegation. These terms MUST be used in the presentation format of the DELEG record.
* INCLUDE and DIRECT delegation can be mixed within an RRSet.
* The final INCLUDE target is an SVCB record, though there can be further indirection using CNAME or AliasMode SVCB records.
* There can be multiple INCLUDE DELEG records, but further indirections through SVCB records have to comply with {{?RFC9460}} in that there can be only one AliasMode SVCB record per name.
* In order to not allow unbounded indirection of DELEG records the maximum number of indirections, CNAME or AliasMode SVCB is 4.
* The SVCB IPv4hint and IPv6hint parameters keep their key values of 4 and 6, but the presentation format with DELEG MUST be Glue4 and Glue6.
* Glue4 and Glue6 records when present MUST be used to connect to the delegated name server.
* The target of a DELEG record MUST NOT be '.' and for INCLUDE must be outside of the delegated domain and for DIRECT in domain delegations below the zone cut.

## Use of DELEG record

The DELEG record creates a zone cut similar to the NS record so all data at or below the zone cut has to be resolved using the name servers defined in the DELEG record. 

A DELEG RRset MAY be present at a delegation point.  The DELEG RRset MAY contain multiple records. DELEG RRsets MUST NOT appear at a zone's apex.

A DELEG RRset MAY be present with or without NS or DS RRsets at the delegation point. 

An authoritative server that is DELEG aware MUST put all DELEG resource records for the delegation into the authority section when the resolver has signalled DELEG support. It SHOULD NOT supply DELEG records in the response when receiving a request without the DE bit.

If the delegation does not have DELEG records the authoritative server MUST send the NS records and, if the zone is DNSSEC signed, prove the absence of the DELEG RRSet.

A resolver that is DELEG aware MUST signal its support by sending the DE bit when iterating and MUST use the DELEG records in the referral response. 

## Signaling DELEG support

For a long time there will be both DELEG and NS needed for delegation. As both methods should be configured to get to a proper resolution it is not necessary to send both in a referral response. We therefore purpose an EDNS flag to be use similar to the DO Bit for DNSSEC to be used to signal that the sender understands DELEG and does not need NS or glue information in the referral.

This bit is referred to as the "DELEG" (DE) bit.  In the context of the EDNS0 OPT meta-RR, the DE bit is the TBD of the "extended RCODE and flags" portion of the EDNS0 OPT meta-RR, structured as follows (to be updated when assigned):

                +0 (MSB)                +1 (LSB)
         +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
      0: |   EXTENDED-RCODE      |       VERSION         |
         +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
      2: |DO|CO|DE|              Z                       |
         +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+

Setting the DE bit to one in a query indicates to the server that the resolver is able to accept delegations using DELEG only. The DE bit cleared (set to zero) indicates the resolver is unprepared to handle DELEG and hence can only be served NS, DS and glue in a delegation response. The DE bit of the query MUST be copied in the response.


## DNSSEC

As the DELEG record is authoritative in the parent zone of a zone cut similar to DS it has to be signed in the parent zone. 

In order for the validator to understand that the delegation uses DELEG this draft introduces a new DNSKEY flag TBD. When this flag is set for the key that signs the DS or DELEG record, usually the zone signing key (ZSK), and the requester has signalled that it understands DELEG an authenticated denial of existence MUST be send with the referral response, so that a DELEG aware validator can prove the existence or absence of a DELEG record and detect a downgrade attack.

A Validating Stub Resolver that is DELEG aware has to use a Security-Aware Resolver that is DELEG aware and if it is behind a forwarder this has to be security and DELEG aware as well.

# IANA Considerations

IANA is requested to allocate the DELEG RR in the Resource Record (RR) TYPEs registry, with the meaning of "enchanced delegation information" and referencing this document.

IANA is requested to assign a new bit in the DNSKEY RR Flags registry ({{!RFC4034}}) for the DELEG bit (N), with the descripion "DELEG" and referencing this document.

IANA is requested to assign a bit from the EDNS Header Flags registry ({{!RFC6891}}), with the abbreviation DE, the description "DELEG enabled" and referencing this document.

For the RDATA parameters to a DELEG RR, the DNS Service Bindings (SVCB) registry ({{!RFC9460}}) is used.  This document requests no new assignments to that registry, though it is expected that future DELEG work will.  

--- back

#  Examples

The following example shows an excerpt from a signed root zone. It shows the delegation point for "example." and "test."

The "example." delegation has DELEG and NS records. The "test." delegation has DELEG but no NS records.

    example.   300 IN DELEG DIRECT a.example. Glue4=192.0.2.1 (
                            Glue6=2001:DB8::1 )
    example.   300 IN DELEG INCLUDE ns2.example.net.
    example.   300 IN DELEG INCLUDE ns3.example.org.
    example.   300 IN RRSIG DELEG 13 4 300 20250214164848 (
                            20250207134348 21261 . HyDHYVT5KcqWc7J..= )
    example.   300 IN NS    a.example.
    example.   300 IN NS    b.example.net.
    example.   300 IN NS    c.example.org.
    example.   300 IN DS    65163 13 2 5F86F2F3AE2B02... 
    example.   300 IN RRSIG DS 13 4 300 20250214164848 (
                            20250207134348 21261 . O0k558jHhyrC21J..= )
    example.   300 IN NSEC  a.example. NS DS RRSIG NSEC DELEG
    example.   300 IN RRSIG NSEC 13 4 300 20250214164848 (
                            20250207134348 21261 . 1Kl8vab96gG21Aa..= )
    a.example. 300 IN A     192.0.2.1
    a.example. 300 IN AAAA  2001:DB8::1

The "test." delegation point has a DELEG record and no NS record.
    
    test.      300 IN DELEG INCLUDE ns2.example.net
    test.      300 IN RRSIG DELEG 13 4 300 20250214164848 (
                            20250207134348 21261 . 98Aac9f7A1Ac26Q..= ) 
    test.      300 IN NSEC  a.test. RRSIG NSEC DELEG
    test.      300 IN RRSIG NSEC 13 4 300  20250214164848 (
                            20250207134348 21261 . kj7YY5tr9h7UqlK..= )
    
## Responses

The following sections show referral examples:

## DO bit clear, DE bit clear

### Query for foo.example
         
;; Header: QR RCODE=0  
;;
    
;; Question  
foo.example.  IN MX
    
;; Answer  
;; (empty)

;; Authority  
example.   300 IN NS    a.example.  
example.   300 IN NS    b.example.net.  
example.   300 IN NS    c.example.org.  

;; Additional   
a.example. 300 IN A     192.0.2.1  
a.example. 300 IN AAAA  2001:DB8::1  

### Query for foo.test

;; Header: QR AA RCODE=3
;;
    
;; Question  
foo.test.   IN MX 
    
;; Answer  
;; (empty)

;; Authority  
.   300 IN SOA ...

;; Additional   
;; (empty)    
    
     
## DO bit set, DE bit clear

### Query for foo.example


    ;; Header: QR DO RCODE=0  
    ;;
    
    ;; Question  
    foo.example.   IN MX
    
    ;; Answer  
    ;; (empty)
    
    ;; Authority  
        
    example.   300 IN NS    a.example.  
    example.   300 IN NS    b.example.net.  
    example.   300 IN NS    c.example.org.  
    example.   300 IN DS    65163 13 2 5F86F2F3AE2B02...  
    example.   300 IN RRSIG DS 13 4 300 20250214164848 (  
                            20250207134348 21261 . O0k558jHhyrC21J..= )  
    ;; Additional  
    a.example. 300 IN A     192.0.2.1  
    a.example. 300 IN AAAA  2001:DB8::1
    
    
### Query for foo.test

    ;; Header: QR DO AA RCODE=3  
    ;;
        
    ;; Question  
    foo.test.      IN MX 
        
    ;; Answer  
    ;; (empty)

    ;; Authority  
    .          300 IN SOA ...
    .          300 IN RRSIG SOA ...
    .          300 IN NSEC  aaa NS SOA RRSIG NSEC DNSKEY ZONEMD
    .          300 IN RRSIG NSEC 13 4 300  
    test.      300 IN NSEC  a.test. RRSIG NSEC DELEG
    test.      300 IN RRSIG NSEC 13 4 300  20250214164848 (
                            20250207134348 21261 . aBFYask;djf7UqlK..= )

    ;; Additional   
    ;; (empty)    
    

## DO bit clear, DE bit set

### Query for foo.example


    ;; Header: QR DE RCODE=0  
    ;;

    ;; Question  
    foo.example.  IN MX

    ;; Answer  
    ;; (empty)

    ;; Authority  
    example.   300 IN DELEG DIRECT a.example. Glue4=192.0.2.1 (
                            Glue6=2001:DB8::1 )
    example.   300 IN DELEG INCLUDE ns2.example.net.
    example.   300 IN DELEG INCLUDE ns3.example.org.

    ;; Additional   
    ;; (empty)  
    
### Query for foo.test

    ;; Header: QR AA RCODE=0
    ;;
        
    ;; Question  
    foo.test.   IN MX 
        
    ;; Answer  
    ;; (empty)

    ;; Authority  
    test.      300 IN DELEG INCLUDE ns2.example.net

    ;; Additional   
    ;; (empty)    
        


## DO bit set, DE bit set

### Query for foo.example

    ;; Header: QR DO DE RCODE=0  
    ;;
    
    ;; Question  
    foo.example.  IN MX
    
    ;; Answer  
    ;; (empty)
    
    ;; Authority  
        
    example.   300 IN DELEG DIRECT a.example. Glue4=192.0.2.1 (
                            Glue6=2001:DB8::1 )
    example.   300 IN DELEG INCLUDE ns2.example.net.
    example.   300 IN DELEG INCLUDE ns3.example.org.
    example.   300 IN RRSIG DELEG 13 4 300 20250214164848 (
                            20250207134348 21261 . HyDHYVT5KcqWc7J..= )
    example.   300 IN DS    65163 13 2 5F86F2F3AE2B02...  
    example.   300 IN RRSIG DS 13 4 300 20250214164848 (  
                            20250207134348 21261 . O0k558jHhyrC21J..= )  

    ;; Additional  
    a.example. 300 IN A     192.0.2.1  
    a.example. 300 IN AAAA  2001:DB8::1  

### Query for foo.test
    
    
    ;; Header: QR DO DE AA RCODE=3  
    ;;
        
    ;; Question  
    foo.test.      IN MX 
        
    ;; Answer  
    ;; (empty)

    ;; Authority  
    .          300 IN SOA ...
    .          300 IN RRSIG SOA ...
    .          300 IN NSEC  aaa NS SOA RRSIG NSEC DNSKEY ZONEMD
    .          300 IN RRSIG NSEC 13 4 300  
    test.      300 IN NSEC  a.test. RRSIG NSEC DELEG
    test.      300 IN RRSIG NSEC 13 4 300  20250214164848 (
                            20250207134348 21261 . aBFYask;djf7UqlK..= )

    ;; Additional   
    ;; (empty)    
    
     
# Acknowledgments {:unnumbered}

This document is heavily based on past work done by Tim April in 
{{?I-D.tapril-ns2}} and thus extends the thanks to the people helping on this which are:
John Levine, Erik Nygren, Jon Reed, Ben Kaduk, Mashooq Muhaimen, Jason Moreau, Jerrod Wiesman, Billy Tiemann, Gordon Marx and Brian Wellington.

# TODO

RFC EDITOR:
: PLEASE REMOVE THE THIS SECTION PRIOR TO PUBLICATION.

* Write a security considerations section
* Change the parameters form temporary to permanent once IANA assigned. Temporary use:
  * DELEG QType code is 65432
  * DELEG EDNS Flag Bit is 3
  * DELEG DNSKEY Flag Bit is 0

# Change Log

RFC EDITOR:
: PLEASE REMOVE THE THIS SECTION PRIOR TO PUBLICATION.

## since draft-wesplaap-deleg-00

* Clarified SVCB priority behaviour

* Added section on differences to draft-homburg-deleg-incremental-deleg

## since draft-wesplaap-deleg-01

* Reorganised and streamlined the draft to the bare mininum for DELEG as an NS replacement
* Defined codepoints for temporary testing
* Added examples
