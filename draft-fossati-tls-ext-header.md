---
title: Extension Header for TLS and DTLS
docname: draft-fossati-tls-ext-header-latest
date: 2018-01-17

# stand_alone: true

ipr: trust200902
area: Security
wg: TLS Working Group
kw: Internet-Draft
cat: std

coding: us-ascii
pi:    # can use array (if all yes) or hash here
  toc: yes
  sortrefs:   # defaults to yes
  symrefs: yes

author:
      -
        ins: T. Fossati
        name: Thomas Fossati
        org: Nokia
        email: thomas.fossati@nokia.org
      -
        ins: N. Mavrogiannopoulos
        name: Nikos Mavrogiannopoulos
        org: RedHat
        email: nmav@redhat.com

normative:
  RFC2119:
  RFC5246:
  RFC6066:

informative:
  I-D.ietf-tls-dtls-connection-id:

entity:
        SELF: "[RFCXXXX]"

--- abstract

This document proposes a mechanism to add extension headers to TLS and DTLS.

In the process, it changes the (D)TLS header in the following way: the length field is trimmed to 14 bits, the length's top bit is given the "extension header indicator" semantics, the other bit is reserved for future use.

--- middle

Introduction
============

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in {{RFC2119}}.

Length Redefined
================

{{RFC5246}} requires the size of TLS record payloads to not exceed 2^14, which means that the two top level bits in the length field of the TLS record header are unused.

The proposal here is to shorten the length field to 14 bits and:

- Use the top bit to signify the presence / absence of an extension header;
- Reserve the other bit for future use.

In the reminder, the top bit is called the "extension header indicator".

Reserved Bit Considerations
===========================

The reserved bit MUST NOT be set by a sender.

Extension Header
=======================

Format {#ext-header-format}
------

If the extension header indicator is asserted, then an extension header is appended to the regular header with the following format:

~~~
+-+-+-+-+-+-+-+------------+------------
|M|   Type    | Length ... | Value ... |
+-+-+-+-+-+-+-+------------+-----------+
~~~

where:

- M has the same semantic as the extension header indicator in the regular header - i.e.: if it is asserted, another extension header follows this one;
- Type is a fixed length (7-bits) field that defines the way Value has to be interpreted;
- Length is the minimal amount of bytes needed to encode the length of a legit Value for this Type;
- Value is the extension itself.

Negotiation {#ext-header-nego}
-----------

An extension header is allowed only if it has been negotiated via a companion TLS extension.

An endpoint MUST NOT send an extension header if it hasn't been successfully negotiated with the receiver.

An endpoint that receives an unexpected extension header MUST abort the session.

Extension headers MUST NOT be sent during the initial handshake phase.

Backwards Compatibility {#ext-header-backwards-compat}
-----------------------

A legacy endpoint that receives an extension header will interpret it as an invalid length field {{RFC5246}} and abort the session accordingly.

Note that this is equivalent to the behaviour of an endpoint implementing this spec which receives a non-negotiated extension header.

Use with Connection ID {#ext-header-and-cid}
----------------------

A plausible use of this mechanism is in relation with the CID extension defined in {{I-D.ietf-tls-dtls-connection-id}}.

In such case, the companion extension header might be defined as follows:

- Type: 0x01
- Length: 1-byte unsigned int
- Value: the CID itself

Note that, compared to all other possible ways to express presence/absence of a CID field within the constrains of the current header format (e.g., bumping the Version field, assigning new ContentType's, using an invalid length), an ad hoc xtension header provides a cleaner approach that can be used with any TLS version at a reasonable cost (2 bytes per record).

Security Considerations
=======================

- TODO discuss on-path active attacker - trying to modify an existing extension header or insert a new one

Privacy Considerations
======================

- TODO discuss metadata insertion - privacy implications must be discussed on a per extension basis

IANA Considerations
===================

This document defines a new IANA registry that, for each extension header, shall define:

- the Type codepoint;
- the Length size in bytes.

Acknowledgements
================

TODO

--- back
