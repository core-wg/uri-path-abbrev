---
title: "URI-Path abbreviation in CoAP"
abbrev: "Uri-Path-Abbrev"
category: std

docname: draft-ietf-core-uri-path-abbrev-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
workgroup: CoRE
keyword:
 - coap
venue:
  group: Constrained RESTful Environments
  mail: core@ietf.org
  github: core-wg/uri-path-abbrev

author:
-
  ins: C. Amsüss
  name: Christian Amsüss
  country: Austria
  email: christian@amsuess.com

- ins: M. Richardson
  name: Michael Richardson
  org: Sandelman Software Works
  email: mcr+ietf@sandelman.ca

normative:

informative:

...

--- abstract

Applications built on CoAP face a conflict between the technical need for short message sizes
and the interoperability requirement of following BCP190
and thus registering (relatively verbose) well-known URI paths.
This document introduces an option that allows expressing well-known paths in as little as two bytes.

--- middle

# Introduction

When building application components on CoAP ({{!RFC7252}}),
sending small messages is a general goal in the ecosystem
(i.e., constrained environments, where data rates are limited and large packets can lead to packet loss, see {{?RFC7228}}).
While CoAP can operate with a wide range of URIs,
short path names are therefore favored.

Those short path names need to be discovered, and {{RFC7252}} and {{?RFC6690}} provide mechanisms for that.
Applications that can not discover such paths because they precede a discovery step,
such as the discovery itself, setting up a security context ({{?RFC9528}}) or establishing an initial identity ({{?RFC9148}})
can not rely on discovered short paths,
and need to use well-known paths.
The best practice established in {{?BCP190}}
requires applications to use the prefix ".well-known" for their paths,
making the combined paths easily longer than the rest of the CoAP message.

This document establishes a CoAP option
that allows abbreviating the path component of the request URI through a numeric registry.

## Motivating example

The design criteria for {{?RFC9528}} described in {{Section 2.11 of ?I-D.ietf-lake-reqs-04}}
give a fragmentation limit of 47 bytes CoAP message payload for 6TiSCH and 51 bytes for some parameters (and implementations) of LoRaWAN,
and high performance penalties of not fitting in those frames.
An EDHOC message 1 on its own carries a minimum of 37 bytes.
The 18 bytes of an encoded "/.well-known/edhoc" path push the size over either limit,
whereas an equivalent Uri-Path-Abbrev stays well below the limit.

## Conventions and Definitions

{::boilerplate bcp14-tagged}

This document assumes basic familiarity with CoAP ({{!RFC7252}}),
in particular its Uri-\* options.

# The Uri-Path-Abbrev option

The Uri-Path-Abbrev option (short for "URI path, abbreviated") expresses a request's URI path in a more compact form.

The Uri-Path-Abbrev value represents a particular path,
and is thus equivalent to any number of Uri-Path options.
Those paths are typically in a "/.well-known" location as described in {{?RFC8615}}.
The numeric option values are coordinated by IANA in the Uri-Path-Abbrev registry established in this document in {{iana-reg}}.


| No.    | C | U | N | R | Name           | Format        | Len. | Default |
|--------+---+---+---+---+----------------+---------------+------+---------|
| CPA13  | x |   |   |   | Uri-Path-Abbrev | uint           | 0-4  | (none)  |
{:#option-table title="Uri-Path-Abbrev Option Summary (C = Critical, U = Unsafe, N = NoCacheKey, R = Repeatable)"}

[^cpa]

[^cpa]: RFC-Editor: This document uses the CPA (code point allocation)
      convention described in {{?I-D.bormann-cbor-draft-numbers}}.  For
      each usage of the term "CPA", please remove the prefix "CPA"
      from the indicated value and replace the residue with the value
      assigned by IANA; perform an analogous substitution for all other
      occurrences of the prefix "CPA" in the document.  Finally,
      please remove this note.

The option is a critical, safe-to-forward, and part of the cache key,
and used in CoAP requests.
{{option-table}} summarizes these, extending Table 4 of {{RFC7252}}).
Its OSCORE treatment is as Class E ({{?RFC8613}}).

The option has an integer value
from the registry established in {{iana-reg}}.

Apart from the format and repeatability,
the option's properties only deviate from the Uri-Path (for which it stands in)
in that this option is safe to forward.
This has consequences for the interactions with the Proxy-URI option,
but is generally desirable:
It allows the option to be used with proxies that do not implement the option.

## Server processing

A server receiving this option process it like the equivalent sequence of Uri-Path options.

A server that supports a Uri-Path-Abbrev value
MUST also support the equivalent request composed of Uri-Path components.

A server receiving the option with an unknown value MUST treat it as an unprocessable critical option,
returning 4.02 Bad Option
(or reject the message[^ref52small])
and MUST NOT return a 4.04 Not Found response,
because the equivalent path may be present on the server.

[^ref52small]: This option may go away if <https://github.com/core-wg/corrclar/issues/52> is resolved before this document is published.

## Client processing

A client may use the option instead of the Uri-Path option if there is a suitable value that can express the requested path.

Unless the client can be assured that the server supports it
(e.g. because the specification describing the interaction mandates support for the option in the server)
it SHOULD fall back to sending the path explicitly if it receives an error indicating that the option was not understood
(otherwise, it would have limited interoperability).
As it is permissible for a server contacted with a a Non-confirmable message over UDP to not respond at all,
clients that use the option in Non-confirmable messages and expect to fall back need to exert extra caution to cover all indications.
As a concequence,
the option can not be used with multicast requests when fallback is expected.

There are four possible indications of the option not being supported:

* A 4.02 Bad Option response.
* A RST caused by handling of a Non-confirmable message.
* Not receiving a response to a Non-confirmable message.
* A 5.02 Bad Gateway caused by a proxy that received a RST or lack of response.

[^ref52]

[^ref52]: There is ongoing discussion about whether that behavior is desirable
        at <https://github.com/core-wg/corrclar/issues/52>.
        In the unlikely case that discussion concludes before this document,
        the 4.02 outcome might be shown as preferred in here and in server processing.

A generic client implementation SHOULD NOT apply this optimization
without explicit instructions from a higher layer or the known specification of the numeric value:
In general, it is too unlikely that the Uri-Path-Abbrev value is understood by any server,
and the message size savings in the successful case are dwarved by the almost doubling of resources needed to perform the fallback.

## Proxy processing

A proxy MAY expand or introduce a Uri-Path-Abbrev before consulting its cache.

It MAY expand a Uri-Path-Abbrev option before forwarding,
in particular if it has reason to assume that the option is not understood.
Like a generic client, it SHOULD NOT introduce an abbreviation without good reason;
and then, it MUST fall back to the expanded form, as to not introduce unexpected errors to the client.

A proxy that knows Uri-Path-Abbrev but not the concrete value
SHOULD forward it unmodified,
which is the behavior it would apply if it did not know the option.
A reason to reject the request instead is when the proxy is tasked with enforcing access control
(see {{seccons}}).

When cross-proxying to protocols that can not transport this option
(such as HTTP),
the proxy needs to expand the path.
<!-- No need to state anything about the inverse direction, as the 2nd paragraph applies. -->

## Interaction with other options {#interactions}

The option is mutually exclusive with the Uri-Path option.
Receiving both options in a single request MUST be treated like the presence of a critical request option that could not be processed
(that option being either the Uri-Path-Abbrev option or the conflicting option).

The Uri-Path-Abbrev option MUST NOT be used in combination with the Proxy-Uri option (or the similar Proxy-CRI option (of {{?I-D.ietf-core-href}})) by clients.
Proxies that understand Uri-Path-Abbrev and convert Uri-\* options into Proxy-Uri MUST expand any Uri-Path-Abbrev if they know the value.

By the (de)composition rules around Proxy-Uri, and because Uri-Path-Abbrev is safe-to-forward,
a proxy (being generally unaware of this specification) is allowed to combine the option with Proxy-Uri (or Proxy-CRI) when it combines the Uri-\* options.
In such a combined message, the Uri-Path segments to which the Uri-Path-Abbrev corresponds are appended to the path as if all components were present as individual options in the request without conflicting.
Servers that support both Uri-Path-Abbrev and Proxy-URI/-CRI SHOULD process requests accordingly.
(This is not a strict requirement, as there are no known implementations of proxies that actually compose a Proxy-URI/-CRI from individual options,
nor is there a reason known why they should).

## Future development

Future updates to this document
might extend the capabilities of the option to be repeated;
that document will need to specify how later occurrences of the option
extend the series of equivalent Uri-Path options from the first value.

Server implementations that treat repeated Uri-Path-Abbrev options
like any other critical unprocessable option (i.e., by responding with 4.02 Bad Option)
support the transition to such an extension.
<!-- It'd be great to state "this is already required in 7252", but it only implies that and doesn't make it explicit. -->

## Choice of the option number

TBD: Rephrase this to either be useful for readers of the final document
who can thus learn how the option number namespace is managed,
or remove before publication.

> It's already 1+1 -- we generally do try to keep even the 1+1 high so
> that later option typically paired with a low option (like EDHOC
> paired with OSCORE) can use the small delta. In this case, there's a
> good reason (being ordered before Uri-Query) though, and I don't
> expect that any other option would need this particular property
> (especially given that this option on its own has an extensible value
> range).

# Initial Uri-Path-Abbrev values {#initial}

This document registers values for the following well-known URIs:

* `/.well-known/core`
* `/.well-known/rd` (see {{?RFC9175}})
* `/.well-known/edhoc` (see {{?RFC9528}})
* For EST ({{?RFC9148}}):
  * `/.well-known/est/crts`
  * `/.well-known/est/sen`
  * `/.well-known/est/sren`
  * `/.well-known/est/skg`
  * `/.well-known/est/skc`
  * `/.well-known/est/att`

  EST does allow using other paths, such as different root resources or arbitrary labels;
  for those, no abbreviations are supported in this document.
* For {{?I-D.ietf-anima-constrained-voucher}}:
  * `/.well-known/brski/es`
  * `/.well-known/brski/rv`
  * `/.well-known/brski/vs`

For all those,
later occurrences of Uri-Path-Abbrev are interpreted as additional Uri-Path values.
While there are currently no resources under the CoRE and RD resource,
this behavior is useful in BRSKI and EST.

Note that the `core` and `rd` paths are commonly used with Uri-Query options.

# Implementation Status

{::boilerplate rfc7942}

* aiocoap <https://christian.amsuess.com/tools/aiocoap/>

  A general-purpose implementation of CoAP for unconstrained sytems,
  published under MIT License.

  In its current main branch,
  the implementation covers the server side of this specification,
  applying expansion automatically before looking up which resource to serve.
  For client, all it provides is the option field where to place a number if the application decides it is suitable,
  relying on the client application to perform the fallback.

  It implements version ietf-core-uri-path-abbrev-01.
  Implementation experience:
  Generally straightforward
  unless one tries to preserve the information whether Uri-Path-Abbrev was used for the server application
  (but that was probably just a bad idea in the first place).

  Contact information: Christian Amsüss (author), updated 2025-09-26

* libcoap (preliminary)

  A report on a yet unpublished implementaiton and related tests of ietf-core-uri-path-abbrev-01 in libcoap
  has been submitted at <https://github.com/core-wg/uri-path-abbrev/issues/25> on 2025-10-01.

# Security Considerations {#seccons}

Having alternative expressions for information that is input to policy decisions
can be problematic when the mechanism performing the check has a different interpretation of the presented data than the mechanism at time of use.
That concern is not new to this document:
Both the Proxy-Uri of {{RFC7252}} and the Proxy-Cri option of {{I-D.ietf-core-href}} have the same properties in that regard.
The appropriate behavior is for policy checkers to reject any request that contains critical options that is not understood;
the application protected by the checker may provide the checker with an allow-list of options that it will treat as unchecked input.

# IANA Considerations

## CoAP option: Uri-Path-Abbrev {#iana-option}

IANA is requested to enter an one option into the CoAP Option Numbers registry in the CoRE Parameters group:

* Number: CPA13
* Name: Uri-Path-Abbrev
* Reference: this document

## Uri-Path-Abbrev registry {#iana-reg}

IANA is requested to establish a new registry in the CoRE parameters group:
Values of the first Uri-Path-Abbrev option in a CoAP request correspond to a URI path according to this registry.

The policy for adding any value is IETF Review (as described in {{?RFC8126}}).
Change control for the registry follows this document's publication stream.
Initial values are given in {{initial-table}}.

Entry fields are:

* First option value.

  An non-negative integer that can be expressed in 32 bits,
  unique within this registry.

  All positive values whose most significant bit of the most significant byte is 1
  are reserved.

  The Python invocation
  `python3 -c 'print("reserved" if (250).bit_length() % 8 == 0 else "unreserved")'`
  can be used to quickly test this property for any positive number (250 in this example).

* Simple expanded path.

  This text is the URI path (starting with `/`) that the option,
  when present only once in a request,
  is expanded to.

  This field may be empty if the document describes that the option needs to be repeated when using this first value.

* Reference.

  A document that requested the allocation,
  and describes whether the option may be repeated after this first value,
  and how later values are expanded

Reviewer instructions:

The reviewer is instructed to be frugal with the 128 values that correspond to a single-vbyte value,
focusing on applications that are expected to be useful in different constrained ecosystems.

The expanded path is expected to be well-known paths at the time of writing,
but it is up to the reviewers to exceptionally also admit paths that are not well-known.

| First option value | Simple expanded path | Reference |
|--------------------+----------------------+-----------|
| 0                  | /.well-known/core    | {{initial}} of this document                         |
| 1                  | /.well-known/rd      | {{initial}} of this document, and {{?RFC9176}}       |
| 2                  | /.well-known/edhoc   | {{initial}} of this document, and {{?RFC9528}}       |
| 301                | /.well-known/est/crts  | {{initial}} of this document, and {{?RFC9148}}    |
| 302                | /.well-known/est/sen   | {{initial}} of this document, and {{?RFC9148}}    |
| 303                | /.well-known/est/sren  | {{initial}} of this document, and {{?RFC9148}}    |
| 304                | /.well-known/est/skg   | {{initial}} of this document, and {{?RFC9148}}    |
| 305                | /.well-known/est/skc   | {{initial}} of this document, and {{?RFC9148}}    |
| 306                | /.well-known/est/att   | {{initial}} of this document, and {{?RFC9148}}    |
| 401                | /.well-known/brski/es  | {{initial}} of this document, and {{?I-D.ietf-anima-constrained-voucher}}       |
| 402                | /.well-known/brski/rv  | {{initial}} of this document, and {{?I-D.ietf-anima-constrained-voucher}}       |
| 403                | /.well-known/brski/vs  | {{initial}} of this document, and {{?I-D.ietf-anima-constrained-voucher}}       |
{:#initial-table title="Initial values for the Uri-Path-Abbrev registry"}

<!-- We could also say in prose to take them from there and list the numbers there, but it is useful for later registrant to have a ready-made template in the document that sets things up. -->

--- back

# Further development

Several possible further directions are anticipated in this document,
but not specified at this point in time;
they are left for further documents:

* The mechanism of expanding one option into another option
  might be expressed using the terminology of SCHC.

  Such a generalization is not aimed for in this document;
  authors of any future document providing such a framework
  are encouraged to provide an equivalent but machine-readable explanation of the mechanism specified here.

* The registry for Uri-Path-Abbrev values is set up such that first values can not have the most significant bit of the first byte set.

  This allows future documents to reuse the option for any CBOR expressions,
  e.g. the path component of a CRI {{?I-D.ietf-core-href}}.
  Note that those CBOR structures can only use the major types 4 to 7 for the top-level item,
  but that includes all containers (arrays, maps and tags).

  Senders and recipients of this option do not need to concern themselves with that extension mechanism
  unless they implement it:
  As the first value is compared to known registry entries,
  any CBOR item contained in it will simply not match any known value.
  Should the working group decide not to use that extension point,
  the registry's policy can be relaxed to also allow values with that leading bit set.

* A future document may update this document
  to allow registering values that are allowed to use together with Uri-Path values
  (but at the time of writing, no examples are known by which such a design could be properly vetted).
  In particular, that update weakens the "MUST" in {{interactions}}.

* This option is designed to stand in for the Uri-Path option alone,
  not for any other option;
  this simplifies its interaction rules.

  In particular,
  application authors who seek to express Uri-Query options in a more concise or easier to process way
  are advised to avail themselves of the FETCH method introduced in {{?RFC8790}}.

# Open questions

This section will be gone by the time this document is published.

* Is the transformation of separate options to Proxy-URI even *legal* for proxies?

  If not, we can simplify the handling (and Uri-Path would *really* not have needed to be proxy-unsafe).

  Tracked at <https://github.com/core-wg/corrclar/issues/51>.

* This document might incentivise users to send more traffic through /.well-known/ paths,
  rather than go through discovery.
  It is up to WG discussion to decide whether this is desirable;
  to not make this document depend on that outcome,
  the registration policy is currently "IETF Review",
  which is extremely strict and can be relaxed in a later document if the WG decides so.

# Change log

Since ietf-core-uri-path-abbrev-00: Processing previous two interims.

* Rename option to Uri-Path-Abbrev.
* Allocate per-resource codes for EST and cBRSKI.
* Allocate code for EDHOC.
* Defer repeated use to future extensions.
* Rearrange content to have dedicated server, client and proxy subsections for option processing.
* Establish that generic clients SHOULD NOT use this without reason.
* More explicit language for proxies, including cross-proxies.
* Add introduction and motivating example.
* Add RFC7942 Implementation Status section.

Since draft-amsuess-core-shopinc-02:

Adopted into WG unmodified as I-D.ietf-core-uri-path-abbrev

Since draft-amsuess-core-shopinc-01: Processing 2025-08-27 interim.

* Document is standards track.
* Change name of the option from Short-Uri-Path to Uri-Path-Abbr.
* Close question of whether use of option 13 is justified (it is).
* Minor editorials.

Since draft-amsuess-core-shopinc-00:

* Switched option type from opaque to uint (retaining the lockout for values that look like CBOR arrays/maps).
* MCR joined as author.
* Added initial values for BRSKI and EST.
* Allow 4.04 responses.
* Add guidance for choosing prefixes and rules.
* Large editorial changes.

# Acknowledgments
{:numbered="false"}

This document was created out of discussion with Esko Dijk and Michael Richardson.
Carsten Bormann provided useful input on shaping the registry.
