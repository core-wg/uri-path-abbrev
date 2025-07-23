---
title: "Short Paths in CoAP"
abbrev: "ShoPinC"
category: info

docname: draft-amsuess-core-shopinc-latest
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
  github: chrysn/shopinc

author:
-
  ins: C. Amsüss
  name: Christian Amsüss
  country: Austria
  email: christian@amsuess.com

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

\[ This is a -00, please read the abstract. \]

## Conventions and Definitions

{::boilerplate bcp14-tagged}

This document assumes basic familiarity with CoAP ({{!RFC7252}}),
in particular its Uri-\* options.

# The Short-Uri-Path option

The Short-Uri-Path option expresses a request's URI path in a more compact form.

The Short-Uri-Path option represents a particular path,
and is thus equivalent to any number of Uri-Path options.
Those paths are typically in a "/.well-known" location as described in {{?RFC8615}}.
The option values are coordinated by IANA in the Short-Uri-Path registry established in this document.

A client may use the option instead of the Uri-Host option if there is a suitable value that can express the requested path.
Unless the client can be assured that the server supports it
(e.g. because the specification describing the interaction mandates support for the option in the server)
it SHOULD fall back to sending the path explicitly if it receives an error indicating that the option was not understood
(otherwise, it would have limited interoperability).

A server receiving the option with an unknown value MUST treat it as an unprocessable critical option,
returning 4.02 Bad Option
and MUST NOT return a 4.04 Not Found response,
because the equivalent path may be present on the server.
A server that supports a Short-Uri-Path value
MUST also support the equivalent request composed of Uri-Path components.


~~~~~~~~~~
+--------+---+---+---+---+----------------+--------+------+---------+
| No.    | C | U | N | R | Name           | Format | Len. | Default |
+--------+---+---+---+---+----------------+--------+------+---------+
| CPA13  | x |   |   | x | Short-Uri-Path | opaque | any  | (none)  |
+--------+---+---+---+---+----------------+--------+------+---------+

      C = Critical, U = Unsafe, N = NoCacheKey, R = Repeatable
~~~~~~~~~~
{: #option-table title="Short-Uri-Path Option Summary" artwork-align="center"}

[^cpa]

[^cpa]: RFC-Editor: This document uses the CPA (code point allocation)
      convention described in [I-D.bormann-cbor-draft-numbers].  For
      each usage of the term "CPA", please remove the prefix "CPA"
      from the indicated value and replace the residue with the value
      assigned by IANA; perform an analogous substitution for all other
      occurrences of the prefix "CPA" in the document.  Finally,
      please remove this note.

The Short-Uri-Path option
has an opaque value.
It is a critical and safe-to-forward option that is part of the cache key,
used in CoAP requests.
{{option-table}} summarizes these, extending Table 4 of {{RFC7252}}).
Its OSCORE treatment is as Class E ({{?RFC8613}}).

These properties only deviate from the Uri-Path (for which it stands in)
in that this option is safe to forward.
This has unfortunate consequences for the interactions with the Proxy-URI option,
but is generally desirable:
It allows the option to be used with proxies that do not implement the option.

A proxy MAY expand or introduce a Short-Uri-Path when forwarding a request,
in particular for serving cached responses,
as long as this introduces no new errors to the client.

## Interaction with other options {#interactions}

The option is mutually exclusive with the Uri-Path option.
Receiving both options in a single request MUST treated like the presence of a critical request option that could not be processed
(that option being either the Short-Uri-Path option or the conflicting option).

The Short-Uri-Path option MUST NOT be used in combination with the Proxy-Uri option (or the similar Proxy-CRI option (of {{?I-D.ietf-core-href}})) by clients,
and proxies that convert Uri-\* options into Proxy-Path MUST expand any Short-Uri-Path if they know the value.
By the (de)composition rules of Proxy-Uri and Short-Uri-Path being safe-to-forward,
a proxy is allowed to combine the option with Proxy-Uri (or Proxy-CRI) when it combines the Uri-\* options.
In such a combined message, the Uri-Path segments to which the Short-Uri-Path corresponds are appended to the path as if all components were present as individual options in the request without conflicting.
Servers that support both Short-Uri-Path and Proxy-URI/-CRI SHOULD process requests accordingly.
(This is not a strict requirement, as there are no known implementations of proxies that actually ).

## Repeated use

If the document defining the registered value of the first Short-Uri-Path option allows it,
further Short-Uri-Path options may be added after that.
Their value is not expanded through the Short-Uri-Path IANA registry,
but according to rules set up in that particular registration.
To be implementable on a wide variety of platforms,
those rules should allow expansion into Uri-Path options in an iterative way
(i.e., any added Short-Uri-Path option corresponds only to appended Uri-Path options,
or cause a 4.02 Bad Option error).

Examples of rules are:

* Options after the first are treated exactly like Uri-Path options.

* There can be only one added Short-Uri-Path option,
  and its opaque value is looked up in a table shaped like the Short-Uri-Path IANA registry.

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

# Initial Short-Uri-Path values {#initial}

This document registers values for the following well-known URIs:

* `/.well-known/core`
* `/.well-known/rd` (see {{?RFC9175}})

TBD: Ask BRSKI for a description

For none of these, the repeated use of the option is specified;
note that both are commonly used with Uri-Query options.

# Security Considerations

Having alternative expressions for information that is input to policy decisions
can be problematic when the mechanism performing the check has a different interpretation of the presented data than the mechanism at time of use.
That concern is not new to this document:
Both the Proxy-Uri of {{RFC7252}} and the Proxy-Cri option of {{I-D.ietf-core-href}} have the same properties in that regard.
The appropriate behavior is for policy checkers to reject any request that contains critical options that is not understood;
the application protected by the checker may provide the checker with an allow-list of options that it will treat as unchecked input.

# IANA Considerations

## CoAP option: Short-Uri-Path

IANA is requested to enter an one option into the CoAP Option Numbers registry in the CoRE Parameters group:

* Number: CPA13
* Name: Short-Uri-Path
* Reference: this document

## Short-Uri-Path registry

IANA is requested to establish a new registry in the CoRE parameters group:
Values of the first Short-Uri-Path option in a CoAP request correspond to a URI path according to this registry.

The policy for adding any value is IETF Review (as described in {{?RFC8126}}).
Change control for the registry follows this document's publication stream.

Entry fields are:

* First option value.

  An arbitrary length byte sequence given in hexadecimal format;
  this value needs to be unique within this registry.

  The most significant bit of the first byte can not be 1.
  <!-- And the sequence may be empty, but that is a value we already register, do we really have to spell that out? -->

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

The reviewer is instructed to be frugal with the 128 single-byte values,
focusing on applications that are expected to be useful in different constrained ecosystems.

The expanded path (or paths) are expected to be well-known paths at the time of writing,
but it is up to the reviewers to exceptionally also admit paths that are not well-known.

If the registration foresees updates,
those should always just allow previously unacceptable values into new path segments,
and not alter the semantics of previously valid expansions.

### Initial values

First option value | Simple expanded path | Reference
-----------------------------------------------------
(empty)            | /.well-known/core    | {{initial}} of this document
00                 | /.well-known/rd      | {{initial}} of this document, and {{?RFC9176}}

<!-- We could also say in prose to take them from there and have the bytes there, but it is useful for later registrant to have a ready-made template in the document that sets things up. -->

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

* The registry for Short-Uri-Path values is set up such that first values can not have the most significant bit of the first byte set.

  This allows future documents to reuse the option for any CBOR expressions,
  e.g. the path component of a CRI {{?I-D.ietf-core-href}}.
  Note that those CBOR strucutres can only use the major types 4 to 7 for the top-level item,
  but that includes all containers (arrays, maps and tags).

  Senders and recipients of this option do not need to concern themselves with that extension mechanism
  unless they implement it:
  As the first value is an opaque value compared to known registry entries,
  any CBOR item contained in it will simply not match any known value.
  Should the working group decide not to use that exension point,
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

* Do we want to enable the use of Uri-Query with this option?

  If so, we need option number 13,
  or put what the author regards as unreasonable requirements on recipients.

  In particular, the .well-known/core resource that is attractive for compression is commonly used with Uri-Query options,
  and it also works well for /.well-known/rd.

  The alternative is to use a higher number (still 1+1 but less precious), eg. 267.

* Is the transformation of separate options to Proxy-URI even *legal* for proxies?

  If not, we can simplify the handling (and Uri-Path would *reall* not have needed to be proxy-unsafe).

* This document might incentivise users to send more traffic through /.well-known/ paths,
  rather than go through discovery.
  It is up to WG discussion to decide whether this is desirable;
  to not make this document depend on that outcome,
  the registration policy is currently "IETF Review",
  which is extremely strict and can be relaxed in a later document if the WG decides so.

* Do we want to add /.well-known/edhoc here, or rather fix it by updating the EDHOC option to also work without an OSCORE option?

  (The author prefers the latter).

# Acknowledgments
{:numbered="false"}

This document was created out of discussion with Esko Dijk and Michael Richardson.
Care to become authors?
