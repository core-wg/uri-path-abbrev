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
This document intrduces an option that allows expressing well-known paths in as little as two bytes.

--- middle

# Introduction

\[ This is a -00, please read the abstract. \]

## Conventions and Definitions

{::boilerplate bcp14-tagged}

# The Short-Uri-Path option

The Short-Uri-Path option expresses a request's URI path in a more compact form.

The Short-Uri-Path option is a critical and safe-to-forward option
for use in CoAP requests.
It uses an opaque value.
Its OSCORE treatment is as Class E ({{?RFC8613}}).
These properties deviate from the Uri-Path for which it stands in
in that this option is safe to forward.
This has unfortunate consequences for the interactions with the Proxy-URI option,
but in the (wide-spread) absence of that option,
it is desirable for this option to traverse proxies unhindered.

The option is mutually exclusive with the Uri-Path option.
receiving both in a single request is to be treated like the presence of a critical request option that could not be processed
(that option being either the Short-Uri-Path option or the conflicting option).

The Short-Uri-Path option SHOULD NOT be used in combination with the the Proxy-Uri option or the Proxy-CRI option (of {{?ietf-core-href}}),
but since a proxy unaware of this option might compose other Uri-\* options into or decompose them out of the Proxy-CIR-like options,
it is can occur together with them, provided those options' path is empty.
In this case, the Short-Uri-Path splices into the URI expressed in the other option.

Any value of the Short-Uri-Path option represents a particular path,
typically in the /.well-known/ hierarchy.
The values are managed in the Short-Uri-Path registry established in this document.

## Repeated use

TBD: Later values expand as specified in the concrete registered item.

## Choice of the option number

TBD: Rephrase this to either be useful for readers of the final document
who can thus learn how the option number namespaced is managed,
or remove before publication.

> It's already 1+1 -- we generally do try to keep even the 1+1 high so
> that later option typically paired with a low option (like EDHOC
> paired with OSCORE) can use the small delta. In this case, there's a
> good reason (being ordered before Uri-Query) though, and I don't
> expect that any other option would need this particular property
> (expecially given that this option on its own has an extensible value
> range).

# Initial options

This document registers values for the following well-known URIs:

* `/.well-known/core`
* `/.well-known/rd` (see {{?RFC9175}})

TBD: Ask BRSKI for a description

For none of these, the repeated use of the option is specified;
note that both are commonly used with Uri-Query options.

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
  e.g. the path component of a CRI {{?ietf-core-href}}.
  Note that those CBOR strucutres can only use the major types 4 to 7 for the top-level item,
  but that includes all containers (arrays, maps and tags).

  Senders and recipients of this option do not need to concern themselves with that extension mechanism
  unless they implement it:
  As the first value is an opaque value compared to known registry entries,
  any CBOR item contained in it will simply not match any known value.
  Should the working group decide not to use that exension point,
  the registry's policy can be relaxed to also allow values with that leading bit set.

# Security Considerations

TODO Security

# Open questions

* Do we want to enable the use of Uri-Query with this option?

  If so, we need option number 13,
  or put what the author regards as unreasonable requirements on recipients.

  In particular, the .well-known/core resource that is attractive for compression is commonly used with Uri-Query options,
  and it also works well for /.well-known/rd.

  The alternative is to use a higher number (still 1+1 but less precious), eg. 267.

* This document might incentivise users to send more traffic through /.well-known/ paths,
  rather than go through discovery.
  It is up to WG discussion to decide whether this is desirable;
  to not make this document depend on that outcome,
  the registration policy is currently "IETF Review",
  which is extremely strict and can be relaxed in a later document if the WG decides so.

* Do we want to add /.well-known/edhoc here, or rather fix it by updating the EDHOC option to also work without an OSCOORE option?

  (The author prefers the latter).

# IANA Considerations

## CoAP option: Short-Uri-Path

IANA is requested to enter an one option into the CoAP Option Numbers registry in the CoRE Parameters group:

* Number: TBD13
* Name: Short-Uri-Path
* Reference: this document

## Short-Uri-Path registry

IANA is requested to establish a new registry in the CoRE parameters group.

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
focusing on applications that are expected to be useful in different constrained ecossystems.

The expanded path (or paths) are expected to be well-known paths at the time of writing,
but it is up to the reviewers to exceptionally also admit paths that are not well-known.

### Initial values

First option value | Simple expanded path | Reference
-----------------------------------------------------
(empty)            | /.well-known/core    | this document
00                 | /.well-known/rd      | this document, {{?RFC9176}}

--- back

# Acknowledgments
{:numbered="false"}

This document was created out of discussion with Esko Dijk and Michael Richardson.
Care to become authors?
