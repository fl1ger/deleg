# DELEG FAQ

This FAQ is a supplement to the full specification for DELEG, available [here](draft-dnsop-deleg.md).

## What is DELEG?

DELEG is a new DNS record type that is currently under development.  It provides additional information about each delegation that can improve the security and performance of the iterative DNS resolution process.

DELEG only exists at the parent side of a delegation.  The parent is authoritative for the record, and it is signed by the parent when the parent supports DNSSEC.

## Does DELEG encrypt DNS queries?

DELEG does not require any encryption, but it does allow nameservers to indicate support for encrypted transports.  These transports encrypt the queries and responses for zone contents, but do not encrypt the name of the nameserver itself.  The nameserver name is typically revealed in the TLS SNI, or in followup queries for out-of-bailiwick nameservers.

## How does DELEG balance privacy and reliability?

DELEG aims to enable downgrade-resistant authenticated encryption, but downgrade resistance is not required.  When using DELEG, nameservers can provide an indication of the protocols that they support, and resolvers can choose how to use this information.  Different resolver policies could result in authenticated encryption ("fail closed") or opportunistic encryption ("fail open").

We plan to make DELEG compatible with DNS Error Reporting, so that nameserver operators can monitor the frequency and impact of encrypted transport failures.

## Does DELEG make DNS resolution faster or slower?

We expect that DELEG will allow resolution with the same overall latency as today.  Some features enabled by DELEG, such as AliasMode and encrypted transport, will require a slight increase in resolution latency, but use of these features is optional for zone owners.

For some zones, adding DELEG will increase the size of responses beyond the UDP response size limit, resulting in additional delay due to DNS TCP fallback.

## Does DELEG make configuration changes harder or easier?

In it simplest usage, DELEG is used similarly to current NS records, and does not substantially impact the process of making configuration changes.  However, DELEG AliasMode allows operators to change DELEG metadata without requiring any action by their customers, which should make configuration changes easier.

## Why do we need a new RR type?

Currently, the only signals conveying information from parent to child are NS records and DS records.  Neither of these are suitable without changes to both parent servers and resolvers.

Given the necessity for protocol changes, the DELEG proposal attempts to "do it right", aiming for a purpose-built design with a minimum of legacy baggage which future protocol extensions will be able to build upon.

### Why can't we use NS records?

When delegating to a signed child zone, the parent-side NS record is unsigned, so it is subject to forgery by on-path attackers and cannot be used to bootstrap authenticated encryption.  Assuming the NS record cannot encode delegation metadata, this metadata would have to be added to the nameserver name, increasing the number of queries required for each delegation.

### Why can't we use DS records?

The DS record is signed, but it is often synthesized by the parent by hashing the client's DNSKEY, so there is no easy way to store additional information in it.  Any use of the DS record for this purpose would require some hacky reinterpretation of the DS RDATA and some significant parent-side changes.  Previous designs of this kind were not adopted in the IETF DPRIVE working group, for these reasons.

## How does the child instruct the parent to publish a DELEG record for its zone?

We expect that DELEG will often be provided to the parent by EPP (via a new EPP extension).  Similar to existing delegation and glue records, the parent is expected to apply its acceptance rules when deciding to publish or reject the provided DELEG records.

A child-to-parent signalling record like CDS, CDNSKEY, or CSYNC is possible but is not currently anticipated.

## How does the child indicate that it wants to outsource operation to multiple operators?

A child zone can publish a DELEG RRset containing multiple AliasMode records, each pointing to a different operator.  This allows each operator to manage its own delegation parameters independently, while ensuring that the child benefits from the combined availability of both operators.

## How soon can I use DELEG?

DELEG is an early-stage proposal, and has not yet started the process for formal standardization.  DELEG will take time to deploy, and may not be successful in deployment even if it is formally standardized.  However, DELEG is extensible, and we hope that DELEG will become more appealing to implementers over time as new extensions are defined.

If you would like to experiment with DELEG, you can try implementing it today using a private RR type.
