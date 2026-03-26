---
title: "Anonymous Authorization and Rate Limiting for Web Agents"
abbrev: "Anonymous Web Agents"
category: info

docname: draft-rescorla-anonymous-webbotauth-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
# area: AREA
# workgroup: WG Working Group
keyword:
 - next generation
 - unicorn
 - sparkling distributed ledger
venue:
#  group: WG
#  type: Working Group
#  mail: WG@example.com
#  arch: https://example.com/WG
  github: "ekr/draft-rescorla-anonymous-webbotauth"
  latest: "https://ekr.github.io/draft-rescorla-anonymous-webbotauth/draft-rescorla-anonymous-webbotauth.html"

author:
 -
    fullname: "Eric Rescorla"
    organization: Independent
    email: "ekr@rtfm.com"

normative:

informative:
  PCMag-Wikipedia:
    title: "Wikipedia Faces Flood of AI Bots That Are 'Eating Bandwidth,' Raising Costs"
    author:
      -
        ins: "L. Kan"
        name: "Lawrence Kan"
        org: "PCMag"
    date: 2024-02-14
    target: "https://www.pcmag.com/news/wikipedia-faces-flood-of-ai-bots-that-are-eating-bandwidth-raising-costs"
  Cloudflare-2025:
    title: "Cloudflare 2025 Year in Review"
    author:
      -
        org: "Cloudflare"
    date: 2025
    target: "https://radar.cloudflare.com/year-in-review/2025"
  PACT-Issue:
    title: "Private Access Control Tokens"
    author:
      -
        ins: "D. Jackson"
        name: "Dennis Jackson"
    date: 2025-12-02
    target: "https://github.com/antifraudcg/proposals/issues/22"
  IACR-2025-2080:
    title: "Issuer Hiding for BBS-Based Anonymous Credentials"
    author:
      -
        ins: "J. Katz"
        name: "Jonathan Katz"
        org: "Google"
      -
        ins: "M. Sefranek"
        name: "Marek Sefranek"
        org: "TU Wien"
    date: 2025-11-10
    target: "https://eprint.iacr.org/2025/2080"

...

--- abstract

Automated agents ("bots") represent a large fraction of the traffic
to many Web sites. In some cases, this traffic is desired, in others
undesired, and in yet others, desired as long as it remains within
certain rate limits. This memo describes an overall architecture
for authorizing bot traffic while protecting anonymous access.


--- middle

# Introduction

Automated agents ("bots") represent a large fraction of the traffic
to many Web sites. For example, Wikimedia reports that about 35% of
its traffic is from bots {{PCMag-Wikipedia}} and Cloudflare reports
that over 50% of traffic is automated {{Cloudflare-2025}}. High
traffic loads--even when from otherwise desired bots--can have
negative impacts on sites in terms of performance, stability,
and operational costs.

In addition, there may be certain types of bot traffic that sites which
to heavily restrict or block entirely. For example:

- Volumetric traffic intended to create a DoS
  attack on the site.
- Non-volumetric traffic intended to attack the site.
- Scraping for specific purposes such as training AI agents.

{{?I-D.nottingham-webbotauth-use-cases}} provides a more complete
list of potential use cases.

The obvious way to address undesired bot traffic is to require
bots to authenticate to the server. The overall idea here is
that sites can use behavioral analysis to determine when traffic
from a given endpoint appears bot-like (e.g., is high volume,
has low latency between requests, appears to be retrieving the
entire site, etc.) and restrict access by those endpoints unless
they authenticate and the resulting identity is acceptable to
the site. {{?I-D.meunier-webbotauth-registry}} describes
one such architecture, where identities are rooted in the
DNS and bots use digital signatures to tie their activitity
to a given identity.

While directly identifying each bot allows the site to precisely
monitor bots behavior and restrict or block bots that it believes
are misbehaving, it also has potential negative effects on the
open Web, including:

- Allowing sites to precisely discriminate against specific
  bots, even when those bots are acting in the public interest.
  For example:
  * Government sites might block bots which are tracking
    law enforcement activity.
  * Employment or housing sites might block bots which are
    monitoring for discriminatory behavior.
  * Shopping sites might block bots used for price comparison.

- Preventing users and groups from being able to anonymously
  retrieve Web content using automated tools.

This document describes an alternative architecture that allows
for anonymous authorization and rate limiting of bots while
avoiding directly identifying individual bots, thus making it
more difficult for sites to precisely select the list of acceptable
bots.


# Architectural Overview

The overall idea, is shown in {{arch-overview}}:

~~~
Alice         Bob           Issuer             Site

Register --------------------->
<------------------ Cred[Alice]
              Register ------->
                <---- Cred[Bob]

Request + Cred ----------------------------------->
                             Site only knows Issuer,
                                   not Alice or Bob
<------------------------------------------Response
~~~
{: #arch-overview title="Architectural Overview" }

Prior to contacting the site, Alice and Bob both register with a
credential Issuer. They each are issued an anonymous credential, which
they can use to authenticate to the server.  What makes the credential
anonymous is that the site only learns that someone with a credential
from the Issuer is authenticating, not whether it's Alice or Bob. This
prevents the site from discriminating _between_ customers of the
same issuer, although it can discriminate between issuers.

As a result, while it is possible to distinguish authenticated from
unauthenticated bots, it is not possible to use the authentication
method to either (1) selectively block individual bots or (2)
determine which bots are accessing which resources.  It may still be
possible to identify bots via other mechanisms such as IP address or
fingerprinting.

In some cases it may be sufficient merely to identify the issuer, for
instance if the issuer performs some vetting to ensure policy
compliance. However, in some cases this may be insufficient, as
discussed below.


## Rate Limiting

In some cases it may be desirable to limit any individual agent to a
specific number of requests. For example:

* A given issuer may have a large number of subscribers but only
  a few may access a given site. In this case, overall rate limits
  for an issuer will not be effective, but individual rate limits
  are.

* A given issuer may have multiple tiers of subscribers that have
  undergone different amounts of vetting and therefore should be
  allowed to offer different amounts of load. Individual rate limits
  permit this while maintaining a large anonymity set.

{::comment}
## Misbehavior Reporting

TODO
{:/comment}


## Issuer Hiding

Conversely, in some cases, it is desirable to conceal which of a set
of issuers issued a credential. As a concrete example, the IETF
PrivacyPass WG is designing anonymous credentials that can be issued to
individuals indicating that they have passed some set of checks
indicating that they represent a human. As discussed below, it may be
possible to use the same type of credential for PrivacyPass and for
anonymous bot authentication, even if the issuers for those
credentials are different. A site which accepts both credentials
for users and bots does not necessarily need to know which type
of user a given request comes from--as long as there is some
rate limiting to prevent individual credentials from being used
for large scale bot activity--in which case it may be desirable
to instead show that a user has a valid credential from _either_
the issuer for user or the issuer for bots without revealing which.


# Issuance Models

Anonymous credentials are compatible with a variety of issuance
models, as discussed in this section.

## Independent Issuers

Probably the most natural approach is to have one or more independent
issuers, each of which publishes the policy that it uses to issue
credentials (e.g., verifying corporate existence, subject pays $100,
etc.). Sites can then select which issuers have policies they are
willing to accept. It is also possible to have multiple issuers
who conform to a common set of policies, as in the WebPKI, where
each CA has to meet the same requirements, but site operators
have the choice of which CA to use.


## Server as Issuer

It is also possible for servers to act as their own issuer. This is
not likely to be practical for small sites, as bots will simply
opt not to authenticate to those sites at all. However, a large
CDN which hosts many sites might opt to operate its own issuer,
and it could be practical for bots to register with such an
issuer.


## Number of Issuers

In general, it is desirable to have for bots to be able to acquire
a relatively small number of credentials and have high confidence that
those credentials will be compatible with most if not all of the
sites that the bot wishes to contact. This can be most straightforwardly
accomplished if there is only a small number of issuers, but it can
also work if there are a larger number of issuers but sites converge
on a relatively small number of policies (expressed as which issuers
they support) such that a bot can acquire a set of credentials that
covers all of those policies. The least desirable outcome is if
bots routinely are prompted to provide credentials for a new issuer.


# Relationship to Existing Technologies

There is significant overlap with existing work on anonymous
authentication happening in the PrivacyPass WG and in the W3C
AntiFraud Community Group. While that work is directed towards
access control for users, the same cryptographic techniques
can be used to authenticate bots. A good description of
the vision and requirements can be found at {{PACT-Issue}}.
There are a number of potential cryptographic techniques
that can be used to issue anonymous rate-limited credentials
including Anonymous Credit Tokens (ACT) {{?I-D.schlesinger-cfrg-act}}
and Anonymous Rate Limited Credentials (ARC) {{?I-D.yun-cfrg-arc}}.
Katz and Sefranek {{IACR-2025-2080}} have published techniques
for hiding which issuer out of a set of issuers was associated
with a given credential.

The ideal scenario would be to be able to use compatible tokens
for users and bots, differing only in the issuance policies,
the issuers, and the rate limits.

# Use Case Analysis



# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
