---
title: "Anonymous Bot Authentication: Authorization and Rate Limiting for Web Agents"
abbrev: "Anonymous Bot Agents"
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

Automated agents ("bots") represent a large fraction of the traffic to
many Web sites. In some cases, this traffic is desired, in others
undesired, and in yet others, desired as long as it remains within
certain rate limits. This memo describes Anonymous Bot Agents (ABA), a
system that allows Web site operators to distinguish wanted from
unwanted traffic, while not tying a given request to a specific
sender.


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

The traditional way that websites discriminate between clients is with client
authentication.  Sites can use behavioral analysis to determine when traffic
from a given endpoint appears bot-like (e.g., is high volume, has low latency
between requests, appears to be retrieving the entire site, etc.) and restrict
access by those endpoints unless they authenticate and the resulting identity is
acceptable to the site. {{?I-D.meunier-webbotauth-registry}} describes one such
architecture, where identities are rooted in the DNS and bots use digital
signatures to tie their activitity to a given identity.

While directly identifying each bot allows the site to precisely
monitor bots' behavior and restrict or block bots that it believes
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

In many use cases, it is possible to resolve this tension using anonymous
credentials.  When a bot presents an anonymous credential, the Web site learns
that the bot is part of some set of authorized bots -- say those whose operators
have agreed to good behavior -- and not the specific identity of the bot or the
bot operator.

This document describes an architecture for using anonymous credential to
authenticate bots to websites.  In addition to laying out how the system works,
we describe the trade-offs between anonymity and fine-grained abuse mitigation.


# Achitectural Overview {#architectural-overview}

The overall structure of ABA is shown in {{architectural-overview}}.

~~~
Alice         Bob           Attester             Site

Register --------------------->
<----------- Attestation[Alice]
              Register ------->
         <---- Attestation[Bob]

Request + Attestation ---------------------------->
                             Site only knows Issuer,
                                   not Alice or Bob
<------------------------------------------Response
~~~
{: #fig-architecture-overview title="Architectural Overview" }

Prior to contacting the site, Alice and Bob both register with a
credential Attester. They each are issued an anonymous credential, which
they can use to authenticate to the server.  What makes the credential
anonymous is that the site only learns that someone with a credential
from the Attester is authenticating, not whether it's Alice or Bob. This
prevents the site from discriminating _between_ customers of the
same attester, although it can discriminate between attesters.

As a result, while it is possible to distinguish authenticated from
unauthenticated bots, it is not possible to use the authentication
method to either (1) selectively block individual bots or (2)
determine which bots are accessing which resources.  It may still be
possible to identify bots via other mechanisms such as IP address or
fingerprinting.

In some cases it may be sufficient merely to identify the attester, for
instance if the attester performs some vetting to ensure policy
compliance. However, in some cases this may be insufficient, as
discussed below.

## Rate Limiting

In some cases it may be desirable to limit any individual agent to a
specific number of requests. For example:

* A given attester may have a large number of subscribers but only
  a few may access a given site. In this case, overall rate limits
  for an attester will not be effective, but individual rate limits
  are.

* A given attester may have multiple tiers of subscribers that have
  undergone different amounts of vetting and therefore should be
  allowed to offer different amounts of load. Individual rate limits
  permit this while maintaining a large anonymity set.

{::comment}
## Misbehavior Reporting

TODO
{:/comment}


## Attester Hiding

Conversely, in some cases, it is desirable to conceal which of a set
of attesters issued a credential. As a concrete example, the IETF
PrivacyPass WG is designing anonymous credentials that can be issued to
individuals indicating that they have passed some set of checks
indicating that they represent a human. As discussed below, it may be
possible to use the same type of credential for PrivacyPass and for
anonymous bot authentication, even if the attesters for those
credentials are different. A site which accepts both credentials
for users and bots does not necessarily need to know which type
of user a given request comes from--as long as there is some
rate limiting to prevent individual credentials from being used
for large scale bot activity--in which case it may be desirable
to instead show that a user has a valid credential from _either_
the attester for user or the attester for bots without revealing which.


# Concrete Implementation With Privacy Pass and ARC

This section describes how to implement ABA using Privacy Pass {{!RFC9576}}, and
Anonymous Rate-Limited Credentials
{{!I-D.yun-privacypass-crypto-arc}}. While this
is not the only possible implementation approach, it leverages existing
deployed and proposed IETF technologies and thus avoids duplicating
existing and fits well into existing deployments.

We make use of the "Joint Origin and Issuer Deployment Model" from
{{Section 4.3 of RFC9576}}, reproduced below in
{{fig-privacy-pass-model}}:

~~~
                                     +----------------------------.
   +--------+          +----------+  |  +--------+     +--------+  |
   | Client |          | Attester |  |  | Issuer |     | Origin |  |
   +---+----+          +-----+----+  |  +----+---+     +---+----+  |
       |                     |        `------|-------------|------'
       |<-------------------------------- TokenChallenge --+
       |                     |               |             |
       |<=== Attestation ===>|               |             |
       |                     |               |             |
       +------------ TokenRequest ---------->|             |
       |<---------- TokenResponse -----------+             |
       |                                                   |
       +--------------------- Token ----------------------->
       |                                                   |
~~~
{: #fig-privacy-pass-model title="Privacy Pass Model (Joint Origin and Issuer)" }

In this model, the client interacts with an Attester, which is
responsible for determining whether the client conforms to the
required policy. The Attester provides an Attestation which can then
be presented to the site (the Issuer in the diagram above), which
provides a Token.  The bot can then use the Token to get services from
the site (the Origin in the diagram above). The Issuer and the Origin
are operated by the same entity (this is a technical constraint of
ARC).

In ABA, the Attestation is performed with a general zero-knowledge
proof system such as {{!I-D.google-cfrg-libzk}} and the Token is
an ARC token, as shown in {{fig-aba-with-privacy-pass}}.
~~~
                                     +----------------------------.
   +--------+          +----------+  |  +--------+     +--------+  |
   | Client |          | Attester |  |  | Issuer |     | Origin |  |
   +---+----+          +-----+----+  |  +----+---+     +---+----+  |
       |                     |        `------|-------------|------'
       |                     |               |             |
       +---- Cred-Request--->|               |             |
       |<--------CWT---------+               |             |
       |                     |               |             |

                             [Later]

       |<-------------------------------- TokenChallenge --+
       |                     |               |             |
       +----TokenRequest + ZKP(CWT))-------->|             |
       |<--TokenResponse[ARC(Limit=XXX)]-----+             |
       |                                                   |
       +---------------- Request + ARC Proof-------------->|
~~~
{: #fig-aba-with-privacy-pass title="ABA with Privacy Pass)" }

When a new Client is deployed, it first must register with some set
of Attesters. These Attesters will require the bot to demonstrate
that it complies with their policies, for instance that it is a
registered corporation, holds a domain name, an IP address block,
etc. Once the Attester is satisfied, it issues a Credential to
the Client in the form of a CWT {{!RFC8392}} signed by the Attester. This
Credential can be used to authenticate to an arbitrary number of Issuers.

When a client contacts a new site for which it does not yet
have an ARC token, the client uses the Credential to authenticate
to the Issuer and request an ARC token. This authentication is
performed anonymously using a zero-knowledge proof as described
in {{auth-issuer}}, so that the Issuer only learns the following
information:

1. This Client has been authenticated by the Attester.
1. This Client has not authenticated to the Issuer previously
   using this Credential (potentially within a given time window).

Assuming that the Client's proof verifies correctly and the
Attester is acceptable, the Issuer issues an ARC token with
a rate limit appropriate for the Attester. For instance, if
the Attester has a policy designed for high traffic bots, the
Issuer might use one rate limit, whereas if the policy is
designed for low traffic bots, the Issuer might use a lower
rate limit. Note that the choice of rate limit is entirely up
to the Issuer, but because all Clients authenticated by a given
Attester are within the same anonymity set, it cannot provide
different per-Client rate limits to Clients attested to by
the same Attester.

Once the Client has an ARC token, it can use it to authenticate
to the Origin repeatedly up to the number of authentications in
in rate limit associated with the token. These authentications
are unlinkable provided that the Client does not exceed the rate
limit; authentications beyond the rate limit are linkable.
Because any Credential can only be used once for a given
Issuer within a given time window, the total rate limit for a given Client/Issuer pair
is bounded by the limit associated with the ARC token.

## Providing Attestation to the Issuer {#auth-issuer}

The protocol for Attestation to the Issuer is designed to meet
the following requirements:

* A Credential can be used with an arbitrary number of Issuers.
* A Credential can only be used once with a single Issuer within
  a given time window
* Credential presentations are unlinkable, both between
  Issuer and Attester and between Issuers.

These requirements can be met by using a signed credential (in this
case, a CWT {{!RFC8392}}), along with a generic circuit-based
zero-knowledge proof system (in this case Longfellow-ZK
{{!I-D.google-cfrg-libzk}}.  The interaction between the Attester and
the Client just yields an ordinary CWT and then the Client proves in
zero-knowledge that they have a valid Credential from a given Attester.

In order to prevent replay, the proof also includes an Issuer-specific
nullifier tied to the Issuer's domain name. While the proof itself
varies between presentations, the nullifier remains the same and
therefore can be used to detect replay.


# Issuance Models

Anonymous credentials are compatible with a variety of issuance
models, as discussed in this section.

## Independent Attesters

Probably the most natural approach is to have one or more independent
attesters, each of which publishes the policy that it uses to issue
credentials (e.g., verifying corporate existence, subject pays $100,
etc.). Sites can then select which attesters have policies they are
willing to accept. It is also possible to have multiple attesters
who conform to a common set of policies, as in the WebPKI, where
each CA has to meet the same requirements, but site operators
have the choice of which CA to use.


## Server as Attester

It is also possible for servers to act as their own attester. This is
not likely to be practical for small sites, as bots will simply
opt not to authenticate to those sites at all. However, a large
CDN which hosts many sites might opt to operate its own attester,
and it could be practical for bots to register with such an
attester.


## Number of Attesters

In general, it is desirable to have for bots to be able to acquire
a relatively small number of credentials and have high confidence that
those credentials will be compatible with most if not all of the
sites that the bot wishes to contact. This can be most straightforwardly
accomplished if there is only a small number of attesters, but it can
also work if there are a larger number of attesters but sites converge
on a relatively small number of policies (expressed as which attesters
they support) such that a bot can acquire a set of credentials that
covers all of those policies. The least desirable outcome is if
bots routinely are prompted to provide credentials for a new attester.


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
for hiding which attester out of a set of attesters was associated
with a given credential.

The ideal scenario would be to be able to use compatible tokens
for users and bots, differing only in the issuance policies,
the attesters, and the rate limits.

# Use Case Analysis

{{I-D.nottingham-webbotauth-use-cases}} describes a set of
use cases for web bot authentication. The architecture described
in this document address some but not all of these use cases.
This is intentional rather than a deficiency; the objective is
to address use cases which are compatible with limiting the
negative impact of bots while avoiding making it trivial for
sites to discriminate against individual bots. The remainder
of this section addresses each use case individually.

## Mitigating Volumetric Abuse by Bots

This document directly addresses the topic of volumetric abuse,
because bots can be authenticated and authenticated bots can be
restricted to specific bandwidth limits. Once a bot has exceeded
its limit, it can be blocked.

## Controlling Access by Bots

{{I-D.nottingham-webbotauth-use-cases}}
provides the following example applications of controlling
access:

>   *  Only allow access by bots on an allow list;
>
>   *  Disallow access to bots on an explicit deny list;
>
>   *  Condition access upon meeting some criteria (e.g., non-profit,
>      certification by a third party);
>
>   *  Condition access upon participation in some scheme or protocol
>      (e.g., payment for access);

As noted by the draft:

>   Note that the first two imply some notion of bots being tied to a
>   real-world identity, whereas the remaining do not necessarily require
>   it.

{::comment}
RLB - I would just say that anonymous credentials are consistent with all of these
cases.  E.g., if you had an attester / bit in the credential that says "the holder
of this credential is on the allow list".

RLB - It might be good to call out above (say arch-overview) the general
approach of shifting semantics from the verifier to the attester.
{:/comment}

In general, the architecture in this document can potentially used for the
second two use cases and can be used for some versions of the first two
use cases. Specifically, because allow and deny lists are enforced at
the attester, any given allow or deny list needs to be fairly widely
used--or at least used at a big site--in order to be practical. For
instance, an attester could have the policy not to issue to any bot which
was illegal to do business with in a given jurisdiction, because many
sites might be interested in such a policy, but a policy where a
site doesn't want to allow access by a direct competitor is more difficult
to execute.

## Providing Different Content to Bots

The architecture in this document may be usable to provide different
content to bots generally than humans depending on the structure of attesters
(e.g., does a given attester issue to both bots and to humans) and
whether techniques are used to conceal which attester is in use.
However, they are not generally useful to provide different ocntent
to specific bots.


## Auditing Bot Behavior

This use case is not addressed by this document.

## Classifying Traffic

Because this use case does not depend on determining which bot is which,
but only which traffic is human versus bot, the architecture in this
document may be able to address this use case, depending on the
ultimate deployment model, in particular whether bots and humans
use different attesters and whether the attester is concealed.

## Authenticating Site Services

This use case is not addressed by this document.


# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Security Considerations

The precise security and privacy details of a system of this type
depend on the cryptographic mechanism being deployed. However, it
is possible to make some general observations.

## Anonymity Set

The anonymity set for a given transaction is the set of credentials
associated with a given attester, or, if attester hiding is used, the set
of credentials associated with the set of attesters. However, it is
still possible to learn information about the client by manipulating
the attester set. For example, a site acting as an attester could use
different keys for each user or a site could use different attester
subsets to identify which of a set of attesters was in use.
Transparency/consistency mechansims like
{{?I-D.ietf-privacypass-key-consistency}} may be useful in detecting
this form of attack.


## Credential Misuse

Because clients are anonymous, some forms of misuse are harder to
manage. For example:

* Two clients can collude to exceed rate limits if they are
  interacting with disjoint sites.

* If registration standards are low and registration is cheap
  a bot can obtain multiple credentials.

* Patterns of misuse (e.g., credential stuffing) become harder
  to detect.

Note that existing mechanisms, such as IP address, will continue
to be usable, but the techniques described in this document
may not add to the server's ability to address these issues.


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
