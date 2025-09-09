---
title: "Short Paths in CoAP"
abbrev: "ShoPinC"
category: std

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

\[ This is a -00, please read the abstract. \]

## Conventions and Definitions

{::boilerplate bcp14-tagged}

This document assumes basic familiarity with CoAP ({{!RFC7252}}),
in particular its Uri-\* options.

# The Uri-Path-Abbr option

The Uri-Path-Abbr option (short for "URI path, abbreviated") expresses a request's URI path in a more compact form.

The Uri-Path-Abbr value represents a particular path,
and is thus equivalent to any number of Uri-Path options.
Those paths are typically in a "/.well-known" location as described in {{?RFC8615}}.
The numeric option values are coordinated by IANA in the Uri-Path-Abbr registry established in this document in {{iana-reg}}.

A client may use the option instead of the Uri-Path option if there is a suitable value that can express the requested path.
Unless the client can be assured that the server supports it
(e.g. because the specification describing the interaction mandates support for the option in the server)
it SHOULD fall back to sending the path explicitly if it receives an error indicating that the option was not understood
(otherwise, it would have limited interoperability).

A server receiving the option with an unknown value MUST treat it as an unprocessable critical option,
returning 4.02 Bad Option
and MUST NOT return a 4.04 Not Found response,
because the equivalent path may be present on the server.

A server that supports a Uri-Path-Abbr value
MUST also support the equivalent request composed of Uri-Path components.


| No.    | C | U | N | R | Name           | Format        | Len. | Default |
|--------+---+---+---+---+----------------+---------------+------+---------|
| CPA13¹ | x |   |   |   | Uri-Path-Abbr | uint           | 0-4  | (none)  |
| CPA13² | x |   |   | x | Uri-Path-Abbr | uint / opaque  | any  | (none)  |
{:#option-table title="Uri-Path-Abbr Option Summary (C = Critical, U = Unsafe, N = NoCacheKey, R = Repeatable). In basic (¹) and extended (²) use"}

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

The (first occurrence of the) option has an integer value
from the registry established in {{iana-reg}}.
Unless that entry specifies "extended use",
the option is used in the simpler "basic use" mode.

Only applications that support entries with extended use need to implement the extensions of {{repeated}}.
All others can regard the option as non-repeatable and uint valued,
as long as they treat any additional Uri-Path-Abbr options as the unexpected critical option it is.

Apart from the format,
the option's properties only deviate from the Uri-Path (for which it stands in)
in that this option is safe to forward.
This has unfortunate consequences for the interactions with the Proxy-URI option,
but is generally desirable:
It allows the option to be used with proxies that do not implement the option.

## Proxy behavior

A proxy MAY expand or introduce a Uri-Path-Abbr when forwarding a request,
in particular for serving cached responses,
as long as this introduces no new errors to the client.

A proxy that knows Uri-Path-Abbr but not the concrete value
SHOULD forward it unmodified,
which is the behavior it would apply if it did not know the option.
A reason to reject the request instead is when the proxy is tasked with enforcing access control
(see {{seccons}}).

## Interaction with other options {#interactions}

The option is mutually exclusive with the Uri-Path option.
Receiving both options in a single request MUST be treated like the presence of a critical request option that could not be processed
(that option being either the Uri-Path-Abbr option or the conflicting option).

The Uri-Path-Abbr option MUST NOT be used in combination with the Proxy-Uri option (or the similar Proxy-CRI option (of {{?I-D.ietf-core-href}})) by clients.
Proxies that understand Uri-Path-Abbr and convert Uri-\* options into Proxy-Uri MUST expand any Uri-Path-Abbr if they know the value.

By the (de)composition rules around Proxy-Uri, and because Uri-Path-Abbr is safe-to-forward,
a proxy (being generally unaware of this specification) is allowed to combine the option with Proxy-Uri (or Proxy-CRI) when it combines the Uri-\* options.
In such a combined message, the Uri-Path segments to which the Uri-Path-Abbr corresponds are appended to the path as if all components were present as individual options in the request without conflicting.
Servers that support both Uri-Path-Abbr and Proxy-URI/-CRI SHOULD process requests accordingly.
(This is not a strict requirement, as there are no known implementations of proxies that actually compose a Proxy-URI/-CRI from individual options,
nor is there a reason known why they should).

## Extended use {#repeated}

[^maybedrop]

[^maybedrop]: This document should also work if we dropped this and references to it completely;
  it'd then be up to updating documents to "create" extended rules.
  All that would need to remain is a small note that implementations must properly respond with Bad Option on unexpected occurrences..

If a registered abbreviation describes an entry to apply "extended use",
the option can be repeated, and have non-uint values ({{option-table}}).

Their value of later options is not expanded through the Uri-Path-Abbr IANA registry,
but according to rules set up in that particular registration.

To be implementable on a wide variety of platforms,
those rules should allow expansion into Uri-Path options in an iterative way
(i.e., any added Uri-Path-Abbr option corresponds only to appended Uri-Path options,
or cause a 4.02 Bad Option error,
except if there are no resources at all with that prefix,
in which case 4.04 Not Found may be used instead).

### Examples of rules

Some rules anticipated to be used are:

* Options after the first are treated exactly like Uri-Path options.

* There can be only one added Uri-Path-Abbr option,
  and its opaque value is looked up in a table shaped like the Uri-Path-Abbr IANA registry.

## Considerations for choosing rules and prefixes {#rulecons}

As demonstrated with the initial options for {{?RFC9175}} in {{initial}},
several abbreviations can be registered for a single document;
those will often use numbers above 256 to be frugal with the namespace.
Even using two bytes for the number,
the path is encoded more efficiently than if extended use was specified:
The total option is 4 byte long, whereas a small number would produce a 3 byte long option followed by a 2 byte repeated option with value.

It is recommended <!-- not normative: doesn't affect interop, just applicant's later options -->
that the expansion of the first Uri-Path-Abbr option does not end with a trailing slash,
even when the entry is specified for basic use.

This enables updates to the entry that enable extended use,
as there is no empty path component that stands in the way of iterative expansion into Uri-Path options.
Such a change is compatible because clients that use the new behavior
will receive 4.02 errors from older servers,
and fall back to sending the path in Uri-Path.

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

# Initial Uri-Path-Abbr values {#initial}

This document registers values for the following well-known URIs:

* `/.well-known/core`
* `/.well-known/rd` (see {{?RFC9175}})
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
later occurrences of Uri-Path-Abbr are interpreted as additional Uri-Path values.
While there are currently no resources under the CoRE and RD resource,
this behavior is useful in BRSKI and EST.

Note that the `core` and `rd` paths are commonly used with Uri-Query options.

# Security Considerations {#seccons}

Having alternative expressions for information that is input to policy decisions
can be problematic when the mechanism performing the check has a different interpretation of the presented data than the mechanism at time of use.
That concern is not new to this document:
Both the Proxy-Uri of {{RFC7252}} and the Proxy-Cri option of {{I-D.ietf-core-href}} have the same properties in that regard.
The appropriate behavior is for policy checkers to reject any request that contains critical options that is not understood;
the application protected by the checker may provide the checker with an allow-list of options that it will treat as unchecked input.

# IANA Considerations

## CoAP option: Uri-Path-Abbr {#iana-option}

IANA is requested to enter an one option into the CoAP Option Numbers registry in the CoRE Parameters group:

* Number: CPA13
* Name: Uri-Path-Abbr
* Reference: this document

## Uri-Path-Abbr registry {#iana-reg}

IANA is requested to establish a new registry in the CoRE parameters group:
Values of the first Uri-Path-Abbr option in a CoAP request correspond to a URI path according to this registry.

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

* Extended use.

  Brief description of the mode of extended use, or empty (`-`).

  Examples: "Uri-Path string", "table lookup", "custom".

* Reference.

  A document that requested the allocation,
  and describes whether the option may be repeated after this first value,
  and how later values are expanded

Reviewer instructions:

The reviewer is instructed to be frugal with the 128 values that correspond to a single-vbyte value,
focusing on applications that are expected to be useful in different constrained ecosystems.

If the considerations of {{rulecons}} are not followed,
the reviewer is asked to verify with the applicant that they are deliberately deciding otherwise.

The expanded path (or paths) are expected to be well-known paths at the time of writing,
but it is up to the reviewers to exceptionally also admit paths that are not well-known.

If the registration foresees updates,
those should always just allow previously unacceptable values into new path segments,
and not alter the semantics of previously valid expansions.

| First option value | Simple expanded path | Ext |Reference |
|--------------------+----------------------+----------------|
| 0                  | /.well-known/core    | - | {{initial}} of this document                         |
| 1                  | /.well-known/rd      | - | {{initial}} of this document, and {{?RFC9176}}       |
| 301                | /.well-known/est/crts  | - | {{initial}} of this document, and {{?RFC9148}}    |
| 302                | /.well-known/est/sen   | - | {{initial}} of this document, and {{?RFC9148}}    |
| 303                | /.well-known/est/sren  | - | {{initial}} of this document, and {{?RFC9148}}    |
| 304                | /.well-known/est/skg   | - | {{initial}} of this document, and {{?RFC9148}}    |
| 305                | /.well-known/est/skc   | - | {{initial}} of this document, and {{?RFC9148}}    |
| 306                | /.well-known/est/att   | - | {{initial}} of this document, and {{?RFC9148}}    |
| 401                | /.well-known/brski/es  | - | {{initial}} of this document, and {{?I-D.ietf-anima-constrained-voucher}}       |
| 402                | /.well-known/brski/rv  | - | {{initial}} of this document, and {{?I-D.ietf-anima-constrained-voucher}}       |
| 403                | /.well-known/brski/vs  | - | {{initial}} of this document, and {{?I-D.ietf-anima-constrained-voucher}}       |
{:#initial-table title="Initial values for the Uri-Path-Abbr registry"}

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

* The registry for Uri-Path-Abbr values is set up such that first values can not have the most significant bit of the first byte set.

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

* This document might incentivise users to send more traffic through /.well-known/ paths,
  rather than go through discovery.
  It is up to WG discussion to decide whether this is desirable;
  to not make this document depend on that outcome,
  the registration policy is currently "IETF Review",
  which is extremely strict and can be relaxed in a later document if the WG decides so.

* Do we want to add /.well-known/edhoc here, or rather fix it by updating the EDHOC option to also work without an OSCORE option?

  (The author prefers the latter).

# Change log

Since -00:

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
