---
title: "Key Directory over HTTP"
category: info

docname: draft-darling-key-directory-over-http-latest
submissiontype: IETF
consensus: true
v: 3
area: ""
workgroup: "???"
keyword:
 - Internet Draft
 - Publickey
 - Key directory
 - Key rotation
 - Key cache
 - Key ID
venue:
  group: "???"
  type: ""
  mail: "httpapi@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/httpapi/"
  github: "thibmeu/draft-darling-key-directory-over-http"
  latest: "https://thibmeu.github.io/draft-darling-key-directory-over-http/draft-darling-key-directory-over-http.html"

author:
 -
    fullname: Fisher Darling
    organization: Cloudflare Inc.
    email: fisher@darling.dev
 -
    fullname: Thibault Meunier
    organization: Cloudflare Inc.
    email: ot-ietf@thibault.uk
 -
    fullname: Simon Newton
    organization: Cloudflare Inc.
    email: rfc@simonnewton.com

normative:
  CONSISTENCY: I-D.ietf-privacypass-consistency-mirror-01
  COSE: RFC8152
  DAP: I-D.draft-ietf-ppm-dap-14
  HTTP: RFC9110
  HTTP-CACHE: RFC9111
  JOSE: RFC7517
  OHTTP: RFC9458
  PRIVACYPASS: RFC9578

informative:


--- abstract

This document defines recommendations for protocols that expose public keys over
HTTP.


--- middle

# Introduction

Multiple Internet protocols rely on public key cryptography. They require keys
to be distributed by origins to clients. This can be done via certificates,
software releases, or HTTP. This document focuses on this last mechanism. It
aims to set recommendations on how to design a key directory that is served
over HTTP.

Distribution via HTTP allows for a more dynamic use of public keys, for
rotation, or caching on intermediate mirrors or clients.
This document specifies which cache directives origin should set, how clients
and mirrors should consume cache directive set by origins, how origins should
expose their key directory, and rotate them.
The document does not cover a specific directory format, as these needs might
vary from one protocol to the next.

# Motivation

{{Section 5 of JOSE}} and
{{Section 7 of COSE}} both define ways to structure key sets, but no way to serve them.
This creates issues when serving these keys over HTTP because caching is not
taken into account, and there is no standard way to derive an ID from a key.
{{Section 4 of PRIVACYPASS}}, {{Section 3 of OHTTP}}, and
{{Section 4.7.1 of DAP}} are also defining their own public key
directory, and are faced with similar issues. While Privacy Pass seems to have
been the most thorough, even considering {{CONSISTENCY}} for instance, these seem to
be duplicated efforts that would benefit from being consolidated into one
specification.

# Presentation Language

This document uses the TLS presentation language {{!RFC8446}} to describe the
structure of protocol messages.  In addition to the base syntax, it uses two
additional features: the ability for fields to be optional and the ability for
vectors to have variable-size length headers.

## Variable-Size Vector Length Headers

In the TLS presentation language, vectors are encoded as a sequence of encoded
elements prefixed with a length.  The length field has a fixed size set by
specifying the minimum and maximum lengths of the encoded sequence of elements.

In this document, there are several vectors whose sizes vary over significant
ranges.  So instead of using a fixed-size length field, it uses a variable-size
length using a variable-length integer encoding based on the one described in
{{Section 16 of ?RFC9000}}. They differ only in that the one here requires a
minimum-size encoding. Instead of presenting min and max values, the vector
description simply includes a `V`. For example:

~~~ tls-presentation
struct {
    uint32 fixed<0..255>;
    opaque variable<V>;
} StructWithVectors;
~~~

# Conventions and Definitions

{::boilerplate bcp14-tagged}

The following terms are used throughout this document:

**Client:**
: An entity using public key material.

**Origin:**
: An entity exposing public key material via HTTP.

**Mirror:**
: An intermediary entity between client and origin. May cache data, and act as a
  privacy proxy.

**Key metadata:**
: Public data associated to a public key.

**Key Directory:**
: Set of public keys.

**Directory Metadata:**
: Public data associated to a key directory. This is protocol specific.


# Architecture

An Origin is exposing a key directory for Clients to fetch. Clients MAY fetch the
directory from a Mirror, either to protect its privacy, or because the Origin
wants to leverage a content delivery network. The endpoint and request pattern
MUST be the same as if the fetch was to an Origin.

This document focuses on the below interaction, which is triggered when the
Client does not have valid key for the Origin. This can be because the Client is
new, its cache is expired, or the Origin refuses requests with the current key set.

~~~aasvg
+--------+                +--------+               +--------+
| Client |                | Mirror |               | Origin |
+---+----+                +---+----+               +----+---+
    |                         |                         |
    +--- GET Key Directory -->|                         |
    |                         +--- GET Key Directory -->|
    |                         |<---- Key Directory -----+
    |                         +---.                     |
    |                         |    | cache              |
    |                         |<--'                     |
    |<---- Key Directory -----+                         |
    +---.                     |                         |
    |    | cache              |                         +---.
    |<--'                     |                         |    | rotate
    |                         |                         |<--'
    |                         |                         |
~~~

## Key ID {#key-id}

Each key in the directory MUST be associated with a unique Key ID.

Key ID MUST be derived from key material that is shared publicly.
Protocols SHOULD provide the following blob of data:

~~~tls
struct {
  opaque ProtocolBlob<V>;
} PublicKeyMaterial;
~~~

PublicKeyMaterial MAY be composed of both cryptographic material and metadata.

Key ID is defined has follow:

~~~
key_id = encode(H(PublicKeyMaterial))
~~~

where

* `PublicKeyMaterial` is a length-prefix-encoded blob of data
* `H` is a hash function
* `encode` is some encoding function

**TODO Open questions about H**

* Should the draft provide specific H?
* Should the draft define an IANA registry and require protocols to register
  their H?

## Key Selection

The following is a deterministic algorithm for determining which Key a Client
SHOULD use to fulfill their cryptographic needs. By using a deterministic
algorithm, Origins can more easily predict the effects of a Key rotation and
implement grace periods, soak times, etc. Protocols MAY place additional
restrictions, or push these decision details to deployments.

### Algorithm

1. **Filter invalid keys**: Exclude keys that:
   * Have a `not-after` field in the past.
   * Have a {{not-before}} field in the future.
   * Do not meet required cryptographic properties.
2. **Set missing activation times**: If a key does not have a {{not-before}} field, set it to either:
   * The `Last-Modified` header from the request as defined in {{Section 8.8.2 of HTTP}}, if available.
   * The Date header from the request as defined in {{Section 6.6.1 of HTTP}}, if available.
   * The client’s local time.
3. **Sort by activation time**: If a {{not-before}} field exists, sort the
   remaining keys in **descending** order based on {{not-before}}.
4. **Select the first key**: Choose the first key from the Key Directory, as
   ordered in the Key Directory format.

Clients SHOULD implement the Key Selection Algorithm. Origins SHOULD present
the newest Keys first.

For protocols which define a {{not-before}} field, the above algorithm minimizes the
chance that the Client uses a key that has expired between fetching the directory
from the origin and its usage as part of the protocol.

For protocols without a {{not-before}} field, using the first key allows Origin
to present their key directory so that the newest is always first, and
the soon-to-be-removed key is last. This minimizes the chance of a client using
an expired key.

Expired key MAY be presented for completion, only if the protocol defines a
`not-after` field.

## Rotation {#rotation}

Clients and Origins SHOULD NOT assume a key directory is fixed. Origins SHOULD
rotate keys on a schedule. Clients SHOULD fetch keys upon an immediate rotation
for security reasons. This section goes over how Origins SHOULD rotate their
keys, and how that interacts with scheduled and immediate rotations.

### Algorithm

We approach a public key generation by the following function

~~~
Generate(params, RAND) -> (publickey, privatekey, metadata)
~~~

At any point in time, all keys in the directory MUST have a unique key id as defined in {{key-id}}.
When adding a key in the directory, that key MUST have a unique key id.

Generation looks as follows

~~~
do
  (publickey, privatekey, metadata) <- Generate(params, RAND)
  public_key_material <- (publickey | metadata)
  key_id <- encode(H(public_key_material))
while (key_id is not unique)
~~~

### Scheduled rotation

Scheduled rotation happens at a time known to Origins, Clients, and Mirrors.
This MAY be a regular interval (monthly, weekly, daily), or an ad-hoc schedule
agreed between all parties.

Scheduled rotations MUST be communicated in one of the two mode below

**Passive:**
: Origins rely on cache headers to inform Clients about key expiry. They stop
  advertising the key at time `t`, and delete it at time `t_expiry=t+maxage`.
  Origins MAY have to take intermediate mirrors into considerations, if they
  are aware these mirrors don't respect their cache headers.

**Active:**
: Origins keep serving the key and add a `not-after` field. This field MUST be
  at least `t+maxage`.

With both modes, an Origin SHOULD signal a key is not supported by sending a
response with status code 400.
It is RECOMMENDED to use key ID as defined in {{key-id}}.

### Immediate {#immediate}

Origins MIGHT have to rotate keys immediately. Existing keys MAY
have to be invalidated and/or new keys be provisioned. Immediate key rotation
can happen in the event of a key compromise, loss, or other imperious reason.

Immediate key rotation will cause some client requests to the server to fail
until the client and mirrors retrieve a new version of the directory. The key
directory endpoint is going to be placed under a higher load.

1. Origins MAY introduce a random backoff to spread the load of key distribution
   over time. See {{cache-behaviour}}
2. Clients on a scheduled rotation MAY be configured to distrust rotation outside
   a fixed window. Protocols SHOULD define such policies.

## Cache behaviour {#cache-behaviour}

Caching the Key Directory lowers latency and reduces resource usage on
Mirrors and the Origin. An optimal caching strategy should minimize resource
usage for both the Client and Origin while preventing the client from using an
invalid key.

These two requirements, minimizing resource usage and never using an invalid
key, are at odds with each other. In the event of an {{immediate}} key rotation, a
Client might use an invalid key. However, if a Client fetches an Origin key
directory for every request, it would waste time and network resources.

### `not-before` fields {#not-before}

Protocols SHOULD define a `not-before` field. Origins SHOULD add a `not-before`
field or equivalent to each Key in the Directory. The not-before field allows
Origins to signal to Clients when a Key is ready to use and reduces the
chance a Client uses a key which is not yet available. not-before fields SHOULD
be a Unix epoch timestamp in seconds.

### Cache directives

Origins SHOULD respond with cache directives {{HTTP-CACHE}} which
control when the Key Directory should be refreshed. Origins SHOULD provide a
`Cache-Control: max-age` header, or `Expires` header which is slightly less than
the grace period given for a key about to rotate. Clients SHOULD respect the
`max-age` cache directive and MAY respect other directives. If an Origin
provides a `max-age` header AND a Mirror is used, the Origin SHOULD provide a
`s-maxage` header that is equivalent to `max-age`.

To prevent Clients refreshing their Key Directories at the same time
(synchronization), Mirrors SHOULD provide to its clients a `max-age` cache
directive with duration in the range `[0, Origin s-maxage]`.

Origins SHOULD consider using `Date` ({{Section 6.6.1 of HTTP}}) and
`Last-modified` ({{Section 8.8.2 of HTTP}}) headers to ease {{rotation}}.

### Client cache refresh

The primary method a Client SHOULD use to determine when it refreshes its view
of the Key Directory is through the delta seconds described in the `max-age`
cache directive. The higher the delta, the less frequent a Client will update
its cache. The lower the delta, the quicker clients will respond to unplanned
key rotations.

`min-fresh` MAY also be sent by Origins as defined in {{HTTP-CACHE}}.

## Well known URL

It is RECOMMENDED protocol register a {{!WELL-KNOWN=RFC8615}} URL and associated
content-type.

A key directory server MUST support both GET and HEAD request on that endpoint.

~~~
GET /.well-known/<your-protocol>

HTTP/1.1 200 OK
Cache-Control: max-age=<Client Cache TTL>, s-maxage=<Shared Cache TTL>
Content-Type: <your-protocol>
Last-Modified: <datestamp>
~~~

HEAD requests can be used by clients to cheaply determine if the directory has
changed. The Origin server SHOULD issue a Last-Modified header with the date
stamp of when the key directory resource was last modified.

If issuing a Last-Modified header, the Origin server SHOULD support the correct
response to a 'If-Modified-Since' HTTP GET or HEAD request, returning the
appropriate HTTP status codes {{HTTP-CACHE}}.

It is RECOMMENDED that Mirrors support Last-Modified and 'If-Modified-Since'
{{HTTP-CACHE}}.

## Future considerations

These considerations should be addressed in future drafts.

### Consistency

Consistency allows client to prevent themselves from split view attack. A
proposal that has been made for Privacy Pass is to use multiple mirrors
{{!CONSISTENCY}}. With a sufficiently high quorum, clients get more confident
that they are not singled out.
It presents scalability issues as you need multiple mirrors, and have one more
requests from client per mirror in the quorum.

### Key Transparency

Key Directory over HTTP should integrate with transparency, once the protocol has
been defined in {{!KEYTRANS=I-D.ietf-keytrans-protocol}}.
There are specific consideration as to what goes in the log: the full
directory, keys individually, privacy considerations.


# Deployment Considerations

Rotation schedule: fast?
Proxy improves client experience and shields key directory server


# Privacy Considerations

TODO Privacy

Clients fetching keys mean they reveal their IP, time, and other informations.
When the key directory is for an external service, Clients SHOULD consider
proxying their traffic through a mirror server. Mirrors SHOULD NOT collide with
the key server.


# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Test vectors

List how to test cache
List how to test rotation

# Use cases

See existing key directory on https://key-directory-over-http.research.cloudflare.com/

## DAP

HpkeConfigList [1]

~~~tls
HpkeConfig HpkeConfigList<0..2^16-1>;

struct {
  HpkeConfigId id;
  HpkeKemId kem_id;
  HpkeKdfId kdf_id;
  HpkeAeadId aead_id;
  HpkePublicKey public_key;
} HpkeConfig;

opaque HpkePublicKey<0..2^16-1>;
uint16 HpkeAeadId; /* Defined in [HPKE] */
uint16 HpkeKemId;  /* Defined in [HPKE] */
uint16 HpkeKdfId;  /* Defined in [HPKE] */
~~~

Partially informed comments:

* HpkeConfigId could be removed
* Need not-before to handle early capture

[1] https://datatracker.ietf.org/doc/html/draft-ietf-ppm-dap-13#section-4.5.1

## OHTTP

Key Configutation [1]

~~~tls
HPKE Symmetric Algorithms {
  HPKE KDF ID (16),
  HPKE AEAD ID (16),
}

Key Config {
  Key Identifier (8),
  HPKE KEM ID (16),
  HPKE Public Key (Npk * 8),
  HPKE Symmetric Algorithms Length (16) = 4..65532,
  HPKE Symmetric Algorithms (32) ...,
}
~~~

Partially informed comments:

* Key Identifier could be removed/be deterministic
* No mention of not-before
* No mention of HTTP Caching for rotation

[1] https://www.ietf.org/rfc/rfc9458.html#name-key-configuration

## Privacy Pass

Issuer directory [1]

~~~tls
 {
    "issuer-request-uri": "https://issuer.example.net/request",
    "token-keys": [
      {
        "token-type": 2,
        "token-key": "MI...AB",
        "not-before": 1686913811,
      },
      {
        "token-type": 2,
        "token-key": "MI...AQ",
      }
    ]
 }
~~~

Partially informed comments:
* Not as flexible as HPKE
* Has some protocol metadata (token-type, issuer-request-uri, rate-limit)

[1] https://www.rfc-editor.org/rfc/rfc9578#name-configuration

## Masque relay

..

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
