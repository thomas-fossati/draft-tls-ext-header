---
title: Record Header Extensions for DTLS
docname: draft-fossati-tls-ext-header-latest
date: 2018-01-17

# stand_alone: true

ipr: trust200902
area: Security
wg: TLS Working Group
kw: Internet-Draft
cat: std
updates: 5246, 6347

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
        email: thomas.fossati@nokia.com
      -
        ins: N. Mavrogiannopoulos
        name: Nikos Mavrogiannopoulos
        org: RedHat
        email: nmav@redhat.com

normative:
  RFC2119:
  RFC5246:
  RFC6347:
  I-D.ietf-tls-tls13:
  I-D.ietf-tls-dtls13:

informative:
  I-D.ietf-tls-dtls-connection-id:

entity:
        # If the value of foo changes, fix the title and figures accordingly
        foo: "record header extension"
        Foo: "Record header extension"
        FOO: "Record Header Extension"

--- abstract

This document proposes a mechanism to extend the record header in DTLS.  To that aim, the DTLS header is modified as follows: the length field is trimmed to 15 bits, and the length's top bit is given the "{{&foo}} indicator" semantics, allowing a sender to signal that one or more {{&foo}}s have been added to this record.  We define the generic format of a {{&foo}} and the general rules associated with its handling.  Any details regarding syntax, semantics and negotiation of a specific {{&foo}}, are left to future documents.

--- middle

Introduction
============

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in {{RFC2119}}.

Length Redefined
================

DTLS ({{RFC6347}}, {{I-D.ietf-tls-dtls13}}) requires the size of record payloads to not exceed 2^14 bytes - plus a small amount that accounts for compression or AEAD expansion.  This means that the first bit in the length field of the DTLS record header is, in fact, unused.

The proposal ({{fig-length-redefined}}) is to shorten the length field to 15 bits and use the top bit (E) to signify the presence / absence of a {{&foo}}.

~~~
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+
|  ContentType  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|        ProtocolVersion        |             epoch             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        sequence_number                        |
+                               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                               |E|            length           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~             (zero or more) Extension Header(s)                ~
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         Payload (including optional MAC and padding)          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~
{: #fig-length-redefined title="Length redefined"}

Length counts the bytes of Payload and of all {{&foo}}s that are added to this record (possibly none).

In the reminder, the top bit is called the E-bit.

{{&FOO}}
=======================

Format {#ext-header-format}
------

If the E-bit is asserted, then a {{&foo}} is appended to the regular header with the following format:

~~~
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-----------+
|M| Type  |       Length        | Value ... |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-----------+
~~~

Where:

- M(ore) has the same semantics as the E-bit in the regular header - i.e.: if it is asserted then another extension header follows this one;
- Type is a fixed length (4-bits) field that defines the way Value has to be interpreted;
- Length is the size of Value in bytes.  It uses 11 bits, therefore allowing a theoretical maximum size of 2047 bytes for any {{&foo}};
- Value is the {{&foo}} itself.

The fact that Type only allows 16 {{&foo}} is a precise design choice: the allocation pool size is severely constrained so to raise the entry bar for any new {{&foo}}.

Negotiation {#ext-header-nego}
-----------

A {{&foo}} is allowed only if it has been negotiated via a companion DTLS extension.

An endpoint MUST NOT send a {{&foo}} that hasn't been successfully negotiated with the receiver.

An endpoint that receives an unexpected {{&foo}} MUST abort the session.

{{&Foo}}s MUST NOT be sent during the initial handshake phase.

Backwards Compatibility {#ext-header-backwards-compat}
-----------------------

A legacy endpoint that receives a {{&foo}} will interpret it as an invalid length field ({{RFC6347}}, {{I-D.ietf-tls-dtls13}}) and abort the session accordingly.

Note that this is equivalent to the behaviour of an endpoint implementing this spec which receives a non-negotiated {{&foo}}.

Use with Connection ID {#ext-header-and-cid}
----------------------

A plausible use of this mechanism is with the CID extension defined in {{I-D.ietf-tls-dtls-connection-id}}.

In that case, the companion {{&foo}} could be defined as follows:

- Type: 0x0 (i.e., CID {{&foo}});
- Value: the CID itself

A DTLS 1.2 record carrying a CID "AB" would be formatted as in {{fig-cid-example}}:

- E=1
- Type=0x0
- Length=0x002
- Value=0x4142

~~~
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5
+-+-+-+-+-+-+-+-+
|  ContentType  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|            Version            |             Epoch             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Sequence Number                        |
+                               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                               |1|            Length           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|0|  0x0  |       0x002         |            0x4142             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         Payload (including optional MAC and padding)          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~
{: #fig-cid-example title="CID header example"}

Note that, compared to all other possible ways to express presence/absence of a CID field within the constraints of the current header format (e.g., bumping the Version field, assigning new ContentType(s), using an invalid length), an ad hoc {{&foo}} provides a cleaner approach that can be used with any TLS version at a reasonable cost - an overhead of 2 bytes per record.

Security Considerations {#sec-cons}
=======================

An on-path active attacker could try and modify an existing {{&foo}}, insert a new {{&foo}} in an existing session, or alter the result of the negotiation in order to add or remove arbitrary {{&foo}}s.  Given the security properties of DTLS, none of the above can be tried without being fatally noticed by the endpoints.

A passive on-path attacker could potentially extrapolate useful knowledge about endpoints from the information encoded in a {{&foo}} (see also {{priv-cons}}).

Privacy Considerations {#priv-cons}
======================

The extent and consequences of metadata leakage from endpoints to path when using a certain {{&foo}} SHALL be assessed in the document that introduces this new {{&foo}}.  If needed, the document SHALL describe the relevant risk mitigations.

IANA Considerations {#iana-cons}
===================

This document defines a new IANA registry that, for each new {{&foo}}, shall provide its Type code-point.

Acknowledgements
================

TODO

--- back
