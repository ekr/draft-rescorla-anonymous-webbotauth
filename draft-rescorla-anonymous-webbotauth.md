---
title: "Anonymous Bot Authentication: Authorization and Rate Limiting for Web Agents"
abbrev: "Anonymous Bot Authentication"
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
 -
    fullname: "Richard L. Barnes"
    organization: Cisco
    email: "rlb@ipv.sx"

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

...

--- abstract

Automated agents ("bots") represent a large fraction of the traffic to
many Web sites. In some cases, this traffic is desired, in others
undesired, and in yet others, desired as long as it remains within
certain rate limits. This memo describes Anonymous Bot Authentication (ABA), a
system that allows Web site operators to distinguish wanted from
unwanted traffic, while not tying a given request to a specific
sender.


--- middle

# Introduction

DISCLAIMER: This is a work-in-progress draft and has not yet
seen significant security analysis. It is being published
at an early stage for discussion purposes.


Automated agents ("bots") represent a large fraction of the traffic
to many Web sites. For example, Wikimedia reports that about 35% of
its traffic is from bots {{PCMag-Wikipedia}} and Cloudflare reports
that over 50% of traffic is automated {{Cloudflare-2025}}. High
traffic loads--even when from otherwise desired bots--can have
negative impacts on sites in terms of performance, stability,
and operational costs.

In addition, there may be certain types of bot traffic that sites wish
to heavily restrict or block entirely. For example:

- Volumetric traffic intended to create a DoS
  attack on the site.
- Non-volumetric traffic intended to attack the site.
- Scraping for specific purposes such as training AI agents.

{{?I-D.nottingham-webbotauth-use-cases}} provides a more complete
list of potential use cases.

In response to this approach, there have been proposals to authenticate
automated clients. Sites can use behavioral analysis to determine when traffic
from a given endpoint appears bot-like (e.g., is high volume, has low latency
between requests, appears to be retrieving the entire site, etc.) and restrict
access by those endpoints unless they authenticate and the resulting identity is
acceptable to the site. {{?I-D.meunier-webbotauth-httpsig-protocol}} describes one such
architecture, where identities are rooted in the DNS and bots use digital
signatures to tie their activity to a given identity.

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

This document describes an approach for using anonymous credentials to
authenticate bots to websites.  In addition to laying out how the system works,
we describe the trade-offs between anonymity and fine-grained abuse mitigation.


# Architectural Overview {#architectural-overview}

The overall structure of ABA is shown in {{fig-architecture-overview}}.

~~~
Alice         Bob           Anchor             Site

Register --------------------->
<----------- Endorsement[Alice]
              Register ------->
         <---- Endorsement[Bob]

Request + Endorsement ---------------------------->
                             Site only knows Anchor
                                   not Alice or Bob
<------------------------------------------Response
~~~
{: #fig-architecture-overview title="Architectural Overview" }

Note: we are borrowing terms here from MoLE {{!I-D.jms-mole-architecture}}.

Prior to contacting the site, Alice and Bob both register with a
credential Anchor, which is responsible for evaluating
whether they comply with the Anchor's policies.
They each are issued an anonymous credential, which
they can use to authenticate to the server.  What makes the credential
anonymous is that the site only learns that someone with a credential
from the Anchor is authenticating, not whether it's Alice or Bob. This
prevents the site from discriminating _between_ customers of the
same anchor, although it can discriminate between anchors.

As a result, while it is possible to distinguish authenticated from
unauthenticated bots, it is not possible to use the authentication
method to either (1) selectively block individual bots(2)
determine which bots are accessing which resources or (3) link
multiple visits by the same bot. It may still be
possible to identify bots via other mechanisms such as IP address or
fingerprinting.

In some cases it may be sufficient merely to identify the anchor, for
instance if the anchor performs some vetting to ensure policy
compliance. However, in some cases this may be insufficient, as
discussed below.

## Trade-offs

Assuring the anonymity of bot requests means imposes some limits on how this
authentication mechanism can be used to counter abuse.  These trade-offs are
discussed in detail in {{use-case-analysis}}, and the below bullets provide a
summary:

* ABA supports:
    * Rate-limiting of requests by a a single bot
    * Providing separate content to bots vs. humans
    * Conditioning access based on a bot meeting certain criteria
    * Conditioning access based on participation in some scheme or protocol
    * Classifying human vs. bot traffic
    * IP address mobility and sharing of IP addresses
    * Unlinkability between authenticated requests by the same client
* ABA does not support:
    * Allow lists / deny lists
    * Auditing bot behavior
    * Authenticating site services

Essentially, abuse controls that rely on knowing the identity of the abusive
party are incompatible with anonymous authentication.

## Rate Limiting

In some cases it may be desirable to limit any individual agent to a
specific number of requests. For example, a given Anchor may have a
large number of subscribers but only a few may access a given site. In
this case, overall rate limits for an anchor will not be effective,
but individual rate limits are.

{::comment}
## Misbehavior Reporting

TODO




## Anchor Hiding

Conversely, in some cases, it is desirable to conceal which of a set
of Anchors issued a credential. As a concrete example, the W3C Anti-Fraud CG is designing anonymous credentials that can be issued
to individuals indicating that they have passed some set of checks
indicating that they represent a human. As discussed below, it may be
possible to use the same type of credential for
ABA, even if the Anchors for those credentials are different. A site
which accepts both credentials for users and bots does not necessarily
need to know which type of user a given request comes from--as long as
there is some rate limiting to prevent individual credentials from
being used for large scale bot activity--in which case it may be
desirable to instead show that a user has a valid credential from
_either_ the anchor for user or the anchor for bots without
revealing which.

{:/comment}

# Concrete Implementation With MoLE

This section describes how to implement ABA using mechanisms from MoLE
{{I-D.jms-mole-architecture}}. The MoLE architecture has three
entities, as shown in {{fig-mole-model}}, reproduced from Section 4
of {{!I-D.jms-mole-architecture}}.

~~~

   +--------+            +--------+              +-----------+
   | Anchor |            | Client |              | Moderator |
   +---+----+            +---+----+              +-----+-----+
       |                     |                         |
       |<~~~ Interaction ~~~>|                         |
       |                     |                         |
       +==== Endorsement ===>|                         |
       |                     |                         |
    ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
       |                     |                         |
       |                     |<~~~~~ Interaction ~~~~~>|
       |                     |                         |
       |                +---- (If needed) -----------------.
       |                |    |                         |    |
       |                |    +------ Endorsement ----->|    |
       |                |    |                         |    |
       |                |    |<----- Credential -------+    |
       |                |    |                         |    |
       |                 `---------------------------------'
       |                     |                         |
       |                     +====== Presentation ====>|
       |                     |                         |
~~~
{: #fig-mole-model title="MoLE Architecture" }

In this model, the Client interacts with an Anchor, which is
responsible for determining whether the client conforms to the
required policy. The Anchor provides an Endorsement which can then
be redeemed by the client with the Moderator for a Credential, which
can then be presented to the Moderator at a future time in order
to provide anonymous authentication. The Moderator can be
operated directly by the Site or can be separate from the
Site but cooperating with it.

MoLE supports a number of different types of Endorsement
and Credential. For concreteness, this document describes the
use of Longfellow-based Endorsements and ACT-based Credentials
(Sections 4.2 and 5.1 of {{!I-D.jms-mole-protocols}} respectively).
Other Endorsement and Credentials may have somewhat different
operational properties, and are covered briefly in {{other-endorsement-structures}}.

When a new Client is deployed, it first must register with some set of
Anchors. These Anchors will require the bot to demonstrate that it
complies with their policies, for instance that it is a registered
corporation, holds a domain name or an IP address block, etc.
Once the Anchor is satisfied, it issues a Endorsement to
the Client in the form of a CWT {{!RFC8392}} signed by the Anchor. This
Endorsement can be used to authenticate to an arbitrary number of Moderators.

When a client contacts a new site for which it does not yet
have a Credential, the client uses the Endorsement to authenticate
to the Moderator and request a Credential. This authentication is
performed anonymously using a zero-knowledge proof that it has
a valid Endorsement  so that the Moderator only learns the following
information:

1. This Client has been authenticated by the Anchor.
1. This Client has not authenticated to the Moderator previously
   using this Endorsement within a given time window.

Assuming that the Client's proof verifies correctly and the Anchor is
acceptable, the Moderator issues a Credential. Each credential is
associated with a usage limit, which may be Anchor-dependent. For
instance, if the Anchor has a policy designed for high traffic bots,
the Moderator might use usage rate limit, whereas if the policy is
designed for low traffic bots, the Moderator might use a lower usage
limit. Note that the choice of usage limit is entirely up to the
Moderator, but because all Clients authenticated by a given Anchor are
within the same anonymity set, it cannot use different per-Client
usage limits to multiple Clients attested to by the same Anchor.

Once the Client has a Credential it can use it to authenticate
to the Moderator repeatedly.

## Providing Endorsement to the Moderator {#auth-Moderator}

The protocol for Endorsement to the Moderator is designed to meet
the following requirements:

* A Endorsement can be used with an arbitrary number of Moderators.
* A Endorsement can only be used once with a single Moderator within
  a given time window
* Endorsement presentations are unlinkable, both between
  Moderator and Anchor and between Moderators.

These requirements can be met by using a signed credential (in this
case, a CWT {{!RFC8392}}), along with a generic circuit-based
zero-knowledge proof system (in this case Longfellow-ZK
{{!I-D.google-cfrg-libzk}}).  The interaction between the Anchor and
the Client just yields an ordinary CWT and then the Client proves in
zero-knowledge that they have a valid Endorsement from a given Anchor.

In order to prevent replay, the proof also includes an Moderator-specific
nullifier tied to the Moderator's domain name and the current time
window. While the proof itself varies between presentations, the
nullifier remains the same and therefore can be used to detect replay.

Because we are using a generic ZK system it should also be
possible to use it to conceal the  Anchor if desired: for
example if there are multiple Anchors who follow equivalent
vetting procedures, then the Moderator does not need to know which
Anchor the Client used; the ZKP can be designed to prove
that the Client has a Endorsement from one of a set of Anchors
within the same equivalence class; of course, the Moderator will
need to use the same usage limit for all such Anchors.

Note: The Longfellow nullifier mechanism in MoLE does not appear
to include the Moderator's identity, thus potentially precluding
Endorsement reuse.
[[https://github.com/Moderation-of-unLinkable-Endorsements/internet-drafts/issues/28]]


## Alternate Cryptographic Approaches

This section considers a number of alternate cryptographic
approaches and explains the choice of primitives in this
section.

### Only Generic Zero-Knowledge Proofs

In principle, it is possible to omit the use of Credentials
and instead have the Client authenticate each transaction
directly to the Site. However, even efficient generic ZKP systems
like Longfellow-ZK have far higher computational and bandwidth
costs than more limited systems such as Credential mechanisms
provided in MoLE.
Using a generic ZKP system to demonstrate ownership of
an attested Endorsement to the Moderator and then using that
single interaction to obtain a Credential makes it possible
to amortize the generic proof over a large number of subsequent
interactions.

### Other Endorsement Structures

MoLE also includes an Endorsement mechanism called Issuer-Hiding Anonymous
Token (IHAT). IHAT uses single-show Endorsements, so the Client needs a
fresh Endorsement for each Moderator. This is more
computationally efficient than a generic ZKP and allows the Anchor
to have more tight control of the rate at which the Client makes
requests, but may be challenging in settings where the bot has to
contact many sites, as with crawlers. Endorsement mechanisms that support late
binding of the Endorsement to a specific Moderator allow the Client to use one
Anchor-issued Endorsement across multiple Moderators, while still letting each
Moderator enforce its own redemption and rate-limit policy.

# Issuance Models

Anonymous credentials are compatible with a variety of issuance
models, as discussed in this section.

## Independent Anchors

Much like millions of websites rely on a much smaller number of independent
WebPKI certificate authorities, it is possible to have a system of independent
Anchors that are broadly relied upon by many websites.  Each anchor could
publish the policy that it uses to issue credentials (e.g., verifying corporate
existence, subject pays $100, etc.). Sites can then select which anchors have
policies they are willing to accept. It is also possible to have multiple
Anchors who conform to a common set of policies, as in the WebPKI, where each
CA has to meet the same requirements, but site operators have the choice of
which CA to use.

## Server as Anchor

It is also possible for a server to act as their own Anchor. This is
not likely to be practical for small sites, as bots will simply opt
not to authenticate to those sites at all. However, a large CDN which
hosts many sites might opt to operate its own Anchor, and it could
be practical for bots to register with such an Anchor. Note that
this approach makes it possible for the Anchor to refuse
service to individual Clients, but not to selectively do so for
individual customers unless it uses separate Moderators for each
of those customers.

The ZKP element of ABA ensures that clients remain anonymous even in this case,
since the client's redemption of an Endorsement is not linkable to its issuance.

## Number of Anchors

In general, it is desirable for bots to be able to acquire
a relatively small number of credentials and have high confidence that
those credentials will be compatible with most if not all of the
sites that the bot wishes to contact. This can be most straightforwardly
accomplished if there is only a small number of anchors, but it can
also work if there are a larger number of anchors but sites converge
on a relatively small number of policies (expressed as which anchors
they support) such that a bot can acquire a set of credentials that
covers all of those policies. The least desirable outcome is if
bots routinely are prompted to provide credentials for a new anchor.

# Relationship to Browser Authentication

There is significant overlap with existing work on anonymous
authentication happening in the PrivacyPass WG and in the W3C
AntiFraud Community Group. While that work is directed towards
access control for users, the same cryptographic techniques
can be used to authenticate bots. A good description of
the vision and requirements can be found at {{PACT-Issue}},
and the MoLE effort that has grown out of it.

# Use Case Analysis

{{I-D.nottingham-webbotauth-use-cases}} describes a set of
use cases for web bot authentication. The architecture described
in this document address some but not all of these use cases.
This is intentional rather than a deficiency; the objective is
to address use cases which are compatible with limiting the
negative impact of bots while avoiding making it trivial for
sites to discriminate against individual bots. The remainder
of this section addresses each use case individually.

## Site Use Cases

### Mitigating Volumetric Abuse by Bots

This document directly addresses the topic of volumetric abuse,
because bots can be authenticated and authenticated bots can be
restricted to specific bandwidth limits. Once a bot has exceeded
its limit, it can be blocked.

### Controlling Access by Bots

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
>
>   Note that the first two imply some notion of bots being tied to a
>   real-world identity, whereas the remaining do not necessarily require
>   it.

ABA can be used for the second two use cases and can be used for some
versions of the first two use cases. However, the more fine-grained
controls are used (e.g., by having many Anchors with different
policies) the worse the privacy and scalability properties of the
system become.n

### Providing Different Content to Bots

The architecture in this document may be usable to provide different
content to bots generally than humans depending on the structure of anchors
(e.g., does a given anchor issue to both bots and to humans) and
whether techniques are used to conceal which anchor is in use.
However, they are not generally useful to provide different content
to specific bots.


### Auditing Bot Behavior

This use case is not addressed by this document.

### Classifying Traffic

Because this use case does not depend on determining which bot is which,
but only which traffic is human versus bot, the architecture in this
document may be able to address this use case, depending on the
ultimate deployment model.

### Authenticating Site Services

This use case is not addressed by this document.

## Bot Use Cases

### IP Address Mobility

ABA provides authentication for bots independent of IP address.
Because ABA authentication is at the Anchor rather than the Client
level, it is not possible to use ABA to build bot-specific reputations
based on observed bot activity; instead the Anchor is responsible
for assessing bots and then providing that information to sites.

### Sharing IP Addresses

Because ABA provides authentication independent of IP address, it
allows sites to discriminate between unauthenticated users of
an IP address and those which are ABA-authenticated, thus reducing
the negative reputational side effects of misuse by unauthenticated
users sharing the same IP address.

### Robots.txt Alignment

This use case is not addressed by this document.

### Conveying Contextual Information

Because different Anchors can have different policies and Anchors
can have multiple policies, ABA allows for limited conveyance of
contextual information. While in principle this information can
be arbitrarily fine-grained, coarse-grained information ensures
a larger anonymity set (see {{number-of-anchors}}).

# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Security Considerations

The precise security and privacy details of a system of this type
depend on the detailed cryptographic mechanism being deployed. However, it
is possible to make some general observations.

## Anonymity Set

The anonymity set for a given transaction is the set of credentials
associated with a given Anchor, or, if Anchor hiding is used, the set
of credentials associated with the set of anchors. However, it is
still possible to learn information about the client by manipulating
the Anchor set. For example, a site acting as an Anchor could use
different keys for each user or a site could use different Anchor
subsets to identify which of a set of Anchors was in use.
Transparency/consistency mechanisms like
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

This document benefited from helpful comments from Dennis Jackson
and Thibault Meunier.

