---
title: "Best practices for TLS Downgrade"
abbrev: TLS-Downgrade
docname: draft-richsalz-sample-latest
category: bcp
workgroup: TLSWG
ipr: trust200902

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: B. Sniffen
    name: Brian Sniffen
    org: Akamai Technologies
    email: bsniffen@akamai.com
 -
    ins: M. Bishop
    name: Mike Bishop
    org: Akamai Technologies
    email: mbishop@akamai.com
 -
    ins: E. Nygren
    name: Erik Nygren
    org: Akamai Technologies
    email: nygren@akamai.com
 -
    ins: R. Salz
    name: Rich Salz
    org: Akamai Technologies
    email: rsalz@akamai.com

--- abstract

Content providers delivering content via CDNs will sometimes deliver content
over HTTPS (or both HTTPS and HTTP) but configure the CDN to pull from the
origin over clear-text and unauthenticated HTTP.  From the perspective of a
client, it appears that their requests and associated responses are delivered
over HTTPS, while in reality their requests are being sent across the network
in-the-clear and responses are delivered unauthenticated.  This exposes user
request data to pervasive monitoring {{!RFC7258}}; it also means response data
may be tampered with by active adversaries.  Terminating TLS
connections on a load balancer and contacting a backend over cleartext has
long been common within data centers, but doing this TLS termination and
downgrade to HTTP at a CDN introduces additional risk when the unprotected
traffic is sent over the general Internet, sometimes across national
boundaries.

While it would be nice to say "never do this," customer demand, content
provider use-cases, and market forces today make it impossible for CDNs to not
support downgrade.  However, following a set of best practices can provide
visibility into when this is happening and can reduce some of the risks.

--- middle

# Background and Motivation

Browsers are helping drive a push to universal HTTPS through a variety of
mechanisms, including:

- Show HTTP as "not secure"

- Showing mixed-content warnings when images or advertisements are HTTP on an
HTTPS base page

- Evolving to evolution to not make powerful web features available for HTTP

On mobile, app stores sometimes require HTTPS for acceptance.

These factors have pushed many content providers to quickly enable HTTPS, even
when their origin infrastructure is not ready or not perceived as being ready.
Being able to use a CDN to convert HTTPS to HTTP has been looked at as a fast
path for getting onto HTTPS quickly.  Doing this has value in protecting
requests and responses over the last mile, but admittedly does not address or
worsens some other types of attacks (such as pervasive monitoring, or
corruption and manipulation of content crossing national boundaries).

Delivering content over HTTPS but fetching it insecurely over HTTP is done for
a variety of reasons, some of which have historic motivations with better
alternatives today, but where content providers are resistant to change. This
include:

* Lack of HTTPS support in origin infrastructure, such as due to using load
balancing hardware that does not support HTTPS, has bad performance
characteristics with HTTPS, or which only supports SSLv3.

* A perception that HTTPS is more expensive to deliver.  In some cases content
providers may have origin infrastructure using old hardware where this is a
real challenge and they lack the budget to upgrade to servers or load
balancers that can handle HTTPS well.

* A perception that using HTTPS introduces performance issues, such as due to
the additional round trips required to establish connections.  This can be a
real issue for origins that lack persistent connection or session ticket
support.

* Challenges in managing origin certificates, or a perception that it is hard
to get TLS certificates.  Automation with providers such as LetsEncrypt help
here, but some content provider origins may be using software or hardware
elements that don't yet integrate well with Auto-DV or may have organizational
policies against using DV certificates.

* Delivering the same library of content to end-users over both HTTP and
HTTPS, but wanting the CDN to cache any given object only once.  This can be
better addressed by always fetching content via HTTPS and storing in a cache
accessible for both HTTP and HTTPS requests, but this faces challenges for
transitioning from an entirely HTTP-fetched-and-served content library to one
that is served over a mixture of HTTP and HTTPS.

* A perception that there is no risk to their users or brand reputation,
sometimes due to thinking of pervasive monitoring and content manipulation as
esoteric threats that don't apply in their case.  For example, content
providers delivering on-demand streaming movies may not see a threat from
using HTTP and may view DRM as addressing most of their immediate concerns.

There is also a closely-related issue where content providers push content to
a storage service over an insecure protocol such as FTP that is subsequently
delivered to end-users via HTTPS.

# Recommended alternatives

The right thing to do is to use modern cryptography for secrecy and
authentication of the forward connection, both the request and the response:
HTTPS for pull-through caches, SCP or SFTP for pushed data.  FTP-over-TLS with
TLS client certificate authentication would be fine, but implementations are
scarce.

Origin sites that avoided TLS for fear of a performance hit should collect
data on the actual costs with modern implementations and modern crypto-support
hardware.  These are expected to be under 2% CPU overhead.

# Potential risk mitigations

An intermediate cache can take several actions to reduce the risk of
unpleasant consequences from using TLS downgrade -- though these practices do
not eliminate that risk.  They take two general strategies:

1. Informing the endpoints that this downgrade is in place.  End points have
more information about the details of the connection, and can expose details
to human controllers. For example, returning a response header such
as `Protocol-To-Origin: HTTP` and preventing customers from removing it.

2. Restricting the sort of data in transit. Examples of this include
    - Limiting to GET methods.  This prevents unauthenticated writes to the origin.
    - Refusing to downgrade requests for `/` , `/index/`, or `/index.html`.   This prevents accidental delivery of the entire site.  The goal is to rapidly detect a misconfiguration with too much downgrading by breaking the site.
    - Limiting the content types or file extensions (e.g., to streaming media).
    - Stripping outgoing request headers containing potential identifiers (Cookie, etc)
    - Stripping query strings

In practice, stripping query strings breaks an enormous amount of Web traffic:
searches, beacons, and the selection apparatus of streaming media clients.
Mechanisms that rely on lists of what is allowed (file extensions) or what is
banned (such as "Cookie" headers) rely on an implausibly detailed and
up-to-date models of Web use.

Other headers we may want to strip include `Origin`, `Referer`, `Cookie`,
`Cookie2`, and those starting with `Sec-` or `Proxy-`.

We recommend that CDNs do the following:

* Returning a `Protocol-To-Origin: HTTP` response header (this could be a
comma-separated list of protocols or not).

* Limiting to GET

* Refusing requests for `/` , `/index/`,or `/index.html`.

Other possible things to do:

* Use a VPN or IPSEC or other secure channel between the CDN and the origin.

* Validate asymmetric signatures of content at the CDN before serving, such
as for software downloads.

# Security Considerations

## Risks of doing downgrade

Downgrades allows end-users to protect last-mile connections, but it makes it
easier for adversaries who control the network between CDN caches and origin
(such as at national boundaries) to poison caches or perform surveillance (as
correlation attacks are possible, even if ostensible PII information is
stripped at the CDN)

> TODO: It is easy to say that everyone should use TLS everywhere (cite). Let's support that position by explaining which adversaries can win which games here -- including some where the adversary has an advantage in an HTTP-to-HTTPS connection _over a connection with no TLS at all_.

## Control of the network between the cache and the origin

ISPs on the HTTP path, including nation states at their borders, can surveil
traffic.  They can expect to get end-user IP information from
`X-Forwarded-For` or similar. In some circumstances, they can learn more from
correlated timing and sizes.  This is principally a risk to _secrecy_.

> TODO: Talk about cases such as software downloads or javascript getting corrupted and used to compromise clients or web pages...

## Confused-deputy issues at the browser

HTTP clients make different decisions based on whether they are using HTTPS or
HTTP -- for example, they send Secure cookies (cite), they enable certain Web
features (high-resolution location, Service Workers).   This is principally a
risk to _authentication_.

This attack is only available with downgrade.  A related attack is available
in the case of HTTP _upgrade_, in which a server makes a similar decision
based on seeing HTTPS on its end of the connection.  Because the CDN or proxy
cache is typically under server control, we speculate in the face of all
evidence that the origin operators can control this complexity and prevent the
complementary attack.

--- back

# Acknowledgements

I did it myself.

> TODO: See about Nick Sullivan, Mark Nottingham, ekr as co-authors
