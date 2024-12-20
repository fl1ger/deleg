# DELEG and basic usage

Based on conversation about DELEG, I think there are two views of how DELEG in "ServiceMode" should be used.  DELEG is either:

1. A very SVCB like record that is really just metadata attached to each nameserver.  There would as many DELEG service mode records as NS records. Or,
2. A bundle of all information needed to follow a referral to a *set of nameservers*.

Just to be explicit, DELEG *must* be able to satisfactorily represent every delegation scenario that NS+DS does.  Obviously it should be able to represent *more* delegation scenarios, but we shouldn't make operators have to use NS because they can't represent what they want with DELEG.

## A simple in-domain example

In this example, the delegation has all "in-domain" nameservers.  In this case, the glue is required.

```text
# simple in-domain example using the idea from #1
# although the "ipv?hint" fields aren't hints, they are *required glue* in this case
example.com. IN DELEG 1 ns1.example.com. ipv4hint=192.168.10.7 ipv6hint=dead:beef:cafe:f00d
example.com. IN DELEG 1 ns2.example.com. ipv4hint=192.168.240.22 ipv6hint=...
# so maybe this should be:
example.com. IN DELEG 1 ns1.example.com. gluev4=192.168.10.7 gluev6=dead:beef:cafe:f00d
example.com. IN DELEG 1 ns2.example.com. gluev4=192.168.240.22 gluev6=...

# simple in-domain example using the concept in #2:
example.com. IN DELEG 1 . gluev4=192.168.10.7,192.168.240.22 gluev6=dead:beef:cafe:f00d,...
```

In #2, the "TargetName" doesn't have an obvious meaning.  I've set it to "." which means something in SVCB (It means that the TargetName is the owner name).  With DoT, presumably it would be the certificate name.

In #1, the "TargetName" service the same purpose as the NS target name -- is is the name of a nameserver, and you look up the A and AAAA records for it (and maybe TLSA records, etc.)

## A simple NON in-domain example

When the nameservers are not in-domain, then the glue is not required, and maybe even not desired.

```zone
# simple non-in-domain example using the idea from #1
# Imagine that with some other DELEG extensions we have other service parameters, like "alpn", but at the base it is like this.
example.com. IN DELEG 1 ns1.example.net. ds=1234 13 2 ....
example.com. IN DELEG 1 ns2.example.org. ds=1234 13 2 ....
example.com. IN RRSIG DELEG ...

# using the idea from #2
example.com. IN DELEG 1 . gluev4=<ip of ns1.example.net>,<ip of ns2.example.org> ds=1234 13 2
example.com. IN RRSIG DELEG ...
```

Concept #2 has some advantages: we don't need to repeat the DS information, for one.  However, it likely has a serious issue.  The owner of "example.com." may not have any control over `ns1.example.net` (maybe them both being `example` doesn't help here, but imagine `example.com.`, `example.net.`, and `example.org` being mangaged by different entities, only loosely coupled.)  So how would `ns1.example.net.` be renumbered?

So, try again?

```zone
# simple non-in-domain example using #2 and domain names
example.com. IN DELEG 1 . ns=ns1.example.net,ns2.example.org ds=1234 13 2 ...
```

This maybe solves the management problem: `ns1.example.net` can renumber without the owner of example.com having to update their DELEG.  However, if we had only IP addresses in the DELEG, we could have made the resolution process more efficient.  If we only have IP addresses in the DELEG when they are in-domain, then we are as efficient as NS.

## Resolver Efficiency

What do we mean by resolver efficiency?  Roughly, when a resolver follows a delegation, it potentially has to do a bunch of extra work.  In particular, if we have NS targets without glue, resolvers need to look up the address records at that name.

DELEG's AliasMode adds more work to the resolver.  It needs to first resolve the `<targetName>` SVCB record RRset.  To be clear, this is not at all very different than resolving NS targets.

If the resolver need to resolve the AliasMode DELEG targetNames *and* still resolve all of the NS targets (really SVCB targetNames), then we've added more work to the resolver without taking any away.

If, however, you could embrace the idea in #2, then (maybe) while adding work to resolve AliasMode targets, we can eliminate work to resolve NS targets.

Making resolvers a little less efficient is far from a deal-breaker, though.  Resolvers are already doing a lot of work, what is one more set of lookups?

## Data Organization

I think the main difference between #1 and #2 is how the data is organized.

Since #1 appears to be essentially per-NS, we have properties, like DS, that likely go with multiple nameservers.  If that set of nameservers is all controlled by the same entity, maybe this is no big deal -- just all the set of IP address to the one targetName, and lo! you have a group of nameservers.

However, if the nameservers are controlled by different entities, then maybe I can't just add them to my one targetName.  Instead, I need to let the owners of those nameservers change the IP addresses without my knowledge.

Most of the time that we've been talking about scenarios like this, we've been talking about AliasMode.  But consider for an instant a simpler scenario.  We just have normal DNS secondaries, but run by other organizations.  My own domain is like this:

```zone
blacka.com. IN NS keilir.ogud.com.   ; run by Olafur
blacka.com. IN NS typhoon.kahlerlarson.org.  ; run by Matt Larson
blacka.com. IN NS ns1.blacka.com.  ; run by me, and the primary
```

How would this look with DELEG?

```zone
blacka.com. IN DELEG 1 keilir.ogud.com. ds=14551 15 2 FC2459...   ; resolvers have to look up this name to get the IP
blacka.com. IN DELEG 1 typhoon.kahlerlarson.org. ds=14551 15 2 FC2459...  ; and this one
blacka.com. IN DELEG 1 ns1.blacka.com ipv4hint=70.164.19.155 ds=14551 15 2 FC2459...  ; but this one is in-domain, so we get the required "hint"
```

Note that the DS must be repeated with each DELEG.  Is that OK?

Theoretically, we could say "just use AliasMode" for this, but is that right?  What if Olafur or Matt doesn't want to publish an SVCB record that I can point to?

Idea #2 is not as well thought out at this point.  Do we add domain names to the SvcParams?  Do we allow for a set of targetNames?  If we add domain names to the SvcParams, is there a more compact way to encode them?

```zone
blacka.com. IN DELEG 1 ns1.blacka.com ns=keilir.ogud.com,typhoon.kahlerlarson.org gluev4=70.164.19.155 ds=14551 15 2 FC2459...
```