---
docname: draft-westerlund-tsvwg-sctp-crypto-chunk-latest
title: Stream Control Transmission Protocol (SCTP) CRYPTO Chunk
abbrev: SCTP CRYPTO Chunk
obsoletes:
cat: std
ipr: trust200902
wg: TSVWG
area: Transport
submissiontype: IETF  # also: "independent", "IAB", or "IRTF"

venue:
  group: Transport Area Working Group (tsvwg)
  mail: tsvwg@ietf.org
  github: gloinul/draft-westerlund-tsvwg-sctp-crypto-chunk

author:
-
   ins:  M. Westerlund
   name: Magnus Westerlund
   org: Ericsson
   email: magnus.westerlund@ericsson.com
-
   ins: J. Preuß Mattsson
   name: John Preuß Mattsson
   org: Ericsson
   email: john.mattsson@ericsson.com
-
   ins: C. Porfiri
   name: Claudio Porfiri
   org: Ericsson
   email: claudio.porfiri@ericsson.com

informative:
  RFC8446:

  I-D.westerlund-tsvwg-sctp-crypto-dtls:
    target: "https://datatracker.ietf.org/doc/draft-westerlund-tsvwg-sctp-crypto-dtls/"
    title: "Datagram Transport Layer Security (DTLS) in the Stream Control Transmission Protocol (SCTP) CRYPTO Chunk"
    author:
      -
       ins:  M. Westerlund
       name: Magnus Westerlund
       org: Ericsson
       email: magnus.westerlund@ericsson.com
      -
       ins: J. Preuß Mattsson
       name: John Preuß Mattsson
       org: Ericsson
       email: john.mattsson@ericsson.com
      -
       ins: C. Porfiri
       name: Claudio Porfiri
       org: Ericsson
       email: claudio.porfiri@ericsson.com
    date: June 2023


normative:
  RFC2119:
  RFC4895:
  RFC5061:
  RFC8126:
  RFC8174:
  RFC9260:

--- abstract

This document describes a method for adding cryptographic protection
to the Stream Control Transmission Protocol (SCTP). The SCTP CRYPTO
chunk defined in this document is intended to enable communications
privacy for applications that use SCTP as their transport protocol and
allows applications to communicate in a way that is designed to
prevent eavesdropping and detect tampering or message forgery.

The CRYPTO chunk defined here in is one half of a complete
solution. Where a companion specification is required to define how
the content of the CRYPTO chunk is protected, authenticated, and
protected against replay, as well as how key management is
accomplished.

Applications using SCTP CRYPTO chunk can use all transport
features provided by SCTP and its extensions but with some limitations.

--- middle

# Introduction {#introduction}

   This document defines a CRYPTO chunk for the Stream Control
   Transmission Protocol (SCTP), as defined in {{RFC9260}}.

   This specification defines the actual CRYPTO chunk. How to enable
   it usage, how it interacts with the SCTP association establishment
   to enable endpoint authentication, key-establishment, and other
   features require a separate protection engine specification.

   This specification is intended to be capable of enabling mutual
   authentication of endpoints, data confidentiality, data origin
   authentication, data integrity protection, and data replay
   protection for SCTP packets after the SCTP association has been
   established. The exact properties will depend on the companion
   specification defining the protection engine used with the CRYPTO
   chunk. The protection engine specification might be based on an
   existing security protocol.

   Applications using SCTP CRYPTO chunk can use all transport features
   provided by SCTP and its extensions. However, there can be some
   limitations or additional requirements for them to function such as
   those noted for SCTP restart and use of Dynamic Address
   Reconfiguration, see {{sec-asconf}} and {{sec-restart}}. Due to its
   level of integration as discussed in next section it will provide
   its security functions on all content of the SCTP packet, and will
   thus not impact the potential to utilize any SCTP functionalities
   or extensions.

# Overview

## Protocol Overview {#protocol-overview}

The CRYPTO chunk is defined as a method for secure and confidential
transfer for SCTP packets.  This is implemented inside the SCTP
protocol, in a sublayer between the SCTP common header handling and
the SCTP chunk handling.  Once an SCTP packet has been received and
the SCTP common header has been used to identify the SCTP association,
the CRYPTO chunk is sent to the chosen protection engine that will
return the SCTP payload containing the unprotected SCTP chunks, those
chunks will then be handled according to current SCTP protocol
specification. {{sctp-Crypto-chunk-layering}} illustrates the CRYPTO
chunk layering in regard to SCTP and the Upper Layer Protocol (ULP).

~~~~~~~~~~~ aasvg
+---------------+ +--------------------+
|               | | Protection Engine  |  Keys
|      ULP      | |                    +-------------.
|               | |   Key Management   |              |
+---------------+-+---+----------------+              |
|                     |                 \    User     |
|                     |                  +-- Level    |
| SCTP Chunks Handler |                      Messages |
|                     |                               |
|                     | +-- SCTP Unprotected Payload  |
|                     |/                              |
+---------------------+    +---------------------+    |
|        CRYPTO       |    | Protection Engine   |    |
|        Chunk        |<-->|                     |<--'
|       Handler       |    | Protection Operator |
+---------------------+    +---------------------+
|                     |\
| SCTP Header Handler | +-- SCTP Protected Payload
|                     |
+---------------------+
~~~~~~~~~~~
{: #sctp-Crypto-chunk-layering title="CRYPTO Chunk Layering
in Regard to SCTP and ULP" artwork-align="center"}

Use of the CRYPTO chunk is defined per SCTP association and a SCTP
association uses a single protection engine. Different associations
within the same SCTP endpoint may use or not use the CRYPTO chunk, and
different associations may use different protection engines.

On the outgoing direction, once the SCTP stack has created the
unprotected SCTP packet payload containing control and/or DATA chunks,
that payload will be sent to the protection engine to be
protected. The format of the protected payload depends on the
protection engine but the unprotected payload will typically be
encrypted and integrity tagged before being encapsulated in a CRYPTO
chunk.

The SCTP protection engine performs protection operations on the whole
unprotected SCTP packet payload, i.e., all chunks after the SCTP
common header. Information protection is kept during the lifetime of
the association and no information is sent unprotected except than the
initial SCTP handshake, the SCTP common Header, the SCTP CRYPTO chunk
header and the SHUTDOWN-COMPLETE chunk.

SCTP CRYPTO chunk capability is agreed by the peers at the
initialization of the SCTP association, during that phase the peers
exchange information about the protection engines available. Once the
peers have agreed on what protection to use, the SCTP endpoints start
sending Protection Engine's payloads in SCTP DATA chunks containing
the initialization information related to the protection engine
including key agreement and endpoint authentication. This is depending
on the chosen protection engine thus is not being detailed in the
current specification.

When the endpoint authentication has been completed, the association
is meant to be initialized and the ULP is informed about that, from
this time on it's possible for the ULPs to exchange data.

CRYPTO chunks will never be retransmitted, retransmission is
implemented by SCTP endpoint at chunk level as in the legacy.  Duplicated
CRYPTO chunks, whenever they will be accepted by the protection
engine, will result in duplicated SCTP chunks and will be handled as
duplicated chunks by SCTP endpoint in the same way a duplicated SCTP packet
with those SCTP chunks would have been.


## Protection Engines Considerations {#protection-engines}

The protection engine, independently from the security
characteristics, needs to be capable working on an unreliable
transport mechanism same as UDP in regards to the payloads of the
CRYPTO chunk, and have its own key management capability. SCTP is
capable of providing reliable transport of key-management messages.

SCTP CRYPTO chunk directly exploits the protection engine by
requesting protection and unprotection of a buffer, in particular the
protection buffer should never exceed the possible SCTP packet size
thus protection engine needs to be aware of the PMTU (see {{pmtu}}).

The key management part of the protection engine is the set of data
and procedures that take care of key distribution, verification, and
update. For key management the Protection Engines uses SCTP DATA
chunks identified with a dedicated Payload Protocol Identifier. The
protection engine can specify if the transmission of any key-managment
messages are non-reliable or reliable transmitted by SCTP.

During initialization, that is before Association reaches the
ESTABLISHED state (see {{state-diagram}}), inband Key Management use
DATA chunks that SHALL use the Protection Engine PPID (see
{{iana-payload-protection-id}}). These DATA chunks SHALL be sent
unprotected by the protection engine as no keys have been established
yet. As soon as the SCTP Association reaches PROTECTED state, any
protection engine that uses inband key management, i.e. sent using
SCTP DATA chunks with the Protection Engine PPID, will have their
message protected inside SCTP CRYPTO chunk protected with the
currently established key.

Key management MAY use other mechanism than what provided by SCTP CRYPTO
chunks, in any case the mechanism for key management MUST be detailed
in the specification for that protection engine.

The protection engines MAY exploit the Flags byte provided by the
CRYPTO chunk header (see {{sctp-Crypto-chunk-newchunk-crypt-struct}})
for its needs. Details of the use of Flags, if different from what described in the
current document, MUST be specified in the Protection Engine Specification
document for that specific protection engine.

The SCTP common header is assumed to be implicitly protected by the
protection engine. This protection is based on the assumption that
there will be a one-to-one mapping between SCTP association and
individually established security contexts. If the protection engine
does not meet that assumption further protection of the common header
is likely required.

An example of protection engine can be DTLS as specified in
{{I-D.westerlund-tsvwg-sctp-crypto-dtls}}.

## SCTP CRYPTO Chunk Buffering and Flow Control {#buffering}

Protection engine and SCTP are asynchronous, meaning that the
protection engine may deliver the decrypted SCTP Payload to the SCTP
endpoint without respecting the reception order.  It's up to SCTP
endpoint to reorder the chunks in the reception buffer and to take
care of the flow control according to what specified in
{{RFC9260}}. From SCTP perspective the CRYPTO chunk processing is part
of the transport network.

Even though the above allows the implementors to adopt a
multithreading design of the protection engines, the actual
implementation should consider that out-of-order handling of SCTP
chunks is not desired and may cause false congestions and
retransmissions.


## PMTU Considerations {#pmtu}

The addition of the CRYPTO chunk to SCTP reduces the room for payload,
due to the size of the CRYPTO chunk header and plain text expansion
due to ciphering algorithm and any authentication tag.  Thus, the SCTP
layer creating the plain text payload needs to know about the overhead
to adjust its target payload size appropriately.

On the other hand, the protection engine needs to be informed about
the PMTU by removing from the value the sum of the common SCTP header
and the CRYPTO chunk header. This implies that SCTP can propagate
the computed PMTU at run time specifically. The way protection engine
provides the primitive for PMTU communication shall be part of the
protection engine specification.

From SCTP perspective, if there is a maximum size of plain text data
that can be protected by the protection engine that must be
communicated to SCTP. As such a limit will limit the PMTU for SCTP to
the maximum plain text plus CRYPTO chunk and algorithm overhead plus
the SCTP common header.

## Congestion Control Considerations {#congestion}

The SCTP mechanism for handling congestion control does depend on
successful data transfer for enlarging or reducing the congestion
window CWND (see {{RFC9260}} section 7.2).

It may happen that protection engine discards packets due to internal
checks or because it has detected a malicious attempt. As those
packets do not represent what the peer sent, it is acceptable to
ignore them, although in-situ modification on the path of a packet
resulting in discarding due to integrity failure will leave a gap, but
has to be accepted as part of the path behavior.

The protection engine shall not interfere with the SCTP congestion
control mechanism, this basically means that from SCTP perspective
the congestion control is exactly the same as how specified
in {{RFC9260}}.

## ICMP Considerations {#icmp}

The SCTP implementation will be responsible for handling ICMP messages
and their validation as specified in {{RFC9260}} Section 10. This
means that the ICMP validation needs to be done in relation to the
actual sent SCTP packets with the CRYPTO chunk and not the unprotected
payload. However, valid ICMP errors or information may indirectly be
provided to the protection engine, such as an update to PMTU values
based on packet to big ICMP messages.

## Path Selection Considerations {#multipath}

When an Association is multihomed there are multiple paths between Endpoints.
The selection of the specific path to be used at a certain time belongs
to SCTP protocol that will decide according to {{RFC9260}}.
The Protection Engine shall not influence the path selection algorithm,
actually the Protection Engine will not even know what path is being used.

## Dynamic Address Reconfiguration Considerations  {#sec-asconf}

When using Dynamic Address Reconfiguration {{RFC5061}} in an SCTP
association using CRYPTO Chunk the ASCONF chunk is protected, thus it
needs to be unprotected first, furthermore it MAY come from an unknown
IP Address.  In order to properly address the ASCONF chunk to the
relevant Association for being unprotected, Destination Address,
Source and Destination ports and VTag shall be exploited. If the
combination of those parameters is not unique the implementor MAY
choose to send the Crypto Chunk to all Associations that fit with the
parameters in order to find the right one. The association will
attempt de-protection operations on the crypto chunk, and if that is
successful the ASCONF chunk can be processed.

The recommendation {{RFC5061}} specifies (section 4.1.1 of
{{RFC5061}}) that ASCONF message are required to be sent authenticated
with SCTP-AUTH {{RFC4895}}. For SCTP associations using Crypto Chunks,
when the Protection Engine provides strong Authentication such for
instance in case of DTLS, results in the use of redundant mechanism
for Authentication with both SCTP-AUTH and the Crypto Chunk. We
recommend to amend {{RFC5061}} for including Crypto Chunks as
Authentication mechanism for ASCONF chunks.


## SCTP Restart Considerations  {#sec-restart}

This section deals with the handling of an unexpected INIT chunk during an
Association lifetime as described in {{RFC9260}} section 5.2 The introduction of
CRYPTO CHUNK opens for two alternatives depending on if the protection engine
preserves its state (crypto context) or not.

When the encryption engine can preserve the crypto context, meaning that
encrypted data belonging to the current Association can be encrypted and
decrypted, the request for SCTP Restart SHOULD use INIT chunk in CRYPTO chunk.

When the crypto context is not preserved, the SCTP Restart can only be
accomplished by means of plain text INIT.  This opens to a man-in-the-middle
attack where a malicious attacker may theoretcally generate an INIT chunk with
proper parameters and hijack the SCTP association.

### INIT chunk in CRYPTO chunk

If the crypto context has been preserved the peer aiming for a SCTP Restart can
still send CRYPTO chunks that can be processed by the remote peer.  In such case
the peer willing to restart the Association SHOULD send the INIT chunk in a
CRYPTO chunk and encrypt it.  At reception of the CRYPTO chunk containing INIT,
the receiver will follow the procedure described in {{RFC9260}} section 5.2.2
with the exception that all the chunks will be sent in CRYPTO chunks.

An endpoint supporting SCTP Association Restart and implementing Crypto Chunk
MUST accept receiving SCTP packets with a verification tag with value 0.  The
endpoint will attempt to map the packet to an association based on source IP
address, destination address and port. If the combination of those parameters is
not unique the implementor MAY choose to send the Crypto Chunk to all
Associations that fit with the parameters in order to find the right one. Note
that type of trial decrypting of the SCTP packets will increase the resource
consumption per packet with the number of matching SCTP associations.

Note that the Association Restart will update the verification tags for both
endpoints.  At the end of the unexpected INIT handshaking the receiver of INIT
chunk will trigger the creation of a new DTLS connection to be executed as soon
as possible to verify the peer identity.

### INIT chunk as plain text

If the crypto context isn't preserved the peer aiming for a SCTP Restart can
only perform an INIT in plain text. Supporting this option opens up the SCTP
association to an availability attack, where an capable attacker may be able to
hijack the SCTP association. Therefore an implementation should only support and
enable this option if restart is cruical.

To mount the attack the attacker needs to be able to process copies of packets
sent from the target endpoint towards its peer for the targeted SCTP
association. In addition the attacker needs to be able to send IP packets with
an source address of the target's peer. If the attacker can send an SCTP INIT
that appear to be from the peer, the target if allowing this option will
generate an INIT ACK back, and assuming the attacker succesfully completes the
restart handshake process the attack has managed to change the VTAG for the
association and the peer will no longer respond, leading to a SCTP assocaitons
failure.

A mitiagation an SCTP endpoint supporting Association Restart by means of plain
text INIT SHOULD support is the following. The endpoint when receiving an INIT
should send HEARTBEATs protected by CRYPTO CHUNK to its peer to validate that
the peer is unreachable. If the endpoint receive an HEARTBEAT ACK within a
reasonable time (at least a couple of RTTs) the restart INIT SHOULD be discarded
as the peer obviously can respond, and thus have no need for a restart. A
capable attacker can still succeed in its attack supressing the HEARTBEAT(s)
through packet filtering, congestion overload or any other method preventing the
HEARTBEATS or there ACKs to reach their destination. If it has been valdiated
that the peer is unreachable, the INIT chunk will trigger the procedure
described in {{RFC9260}} section 5.2.2

Note that the Association Restart will update the verification tags for both
endpoints.  At the end of the unexpected INIT handshaking the receiver of INIT
chunk SHALL trigger the creation of a new DTLS connection to be executed as soon
as possible.  Also note that failure in handshaking of a new DTLS connection is
considered a protocol violation and will lead to Association Abort (see
{{ekeyhandshake}}).


# Conventions

{::boilerplate bcp14}

# New Parameter Type {#new-parameter-type}

This section defines the new parameter type that will be used to
negotiate the use of the CRYPTO chunk and protection engines during
association setup. {{sctp-Crypto-chunk-init-parameter}} illustrates
the new parameter type.

| Parameter Type | Parameter Name |
| 0x80xx | Protected Association |
{: #sctp-Crypto-chunk-init-parameter title="New INIT/INIT-ACK Parameter" cols="r l"}

Note that the parameter format requires the receiver to ignore the
parameter and continue processing if the parameter is not understood.
This is accomplished (as described in {{RFC9260}}, Section 3.2.1.)  by
the use of the upper bits of the parameter type.

## Protected Association Parameter {#protectedassoc-parameter}

This parameter is used to carry the list of proposed protection
engines and the chosen protection engine during INIT/INIT-ACK
handshake.

~~~~~~~~~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|    Parameter Type = 0x80XX    |       Parameter Length        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
|                      Protection Engines                       |
|                                                               |
|                               +-------------------------------+
|                               |            Padding            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~
{: #sctp-Crypto-chunk-init-options title="Protected Association Parameter" artwork-align="center"}

{: vspace="0"}
Parameter Type: 16 bits (unsigned integer)
: This value MUST be set to 0x80XX.

Parameter Length: 16 bits (unsigned integer)
: This value holds the length of the Protection Engines field in
  bytes plus 4.

Protection Engines: variable length
: In the INIT chunk this holds the list of protection engines in descending order of preference,
  i.e. the most preferred comes first, and the least preferred last in
  this field. In the INIT-ACK chunk this holds a single chosen
  protection engine. Each protection engine is specified by a 16-bit
  unsigned integer.

Padding: 0 or 16 bits
: If the length of the Protection Engines field is not a multiple
  of 4 bytes, the sender MUST pad the chunk with all zero bytes
  to make the chunk 32-bit aligned. The Padding MUST NOT be longer
  than 2 bytes and it MUST be ignored by the receiver.

RFC-Editor Note: Please replace 0x08XX with the actual parameter type
value assigned by IANA and then remove this note.

# New Chunk Types {#new-chunk-types}

##  Crypto Chunk (CRYPTO) {#crypto-chunk}

This section defines the new chunk type that will be used to
transport protected SCTP payload.
{{sctp-Crypto-chunk-newchunk-crypt}} illustrates the new chunk type.

| Chunk Type | Chunk Name |
| 0x4X | Crypto Chunk (CRYPTO) |
{: #sctp-Crypto-chunk-newchunk-crypt title="CRYPTO Chunk Type" cols="r l"}

RFC-Editor Note: Please replace 0x4x with the actual chunk type value
assigned by IANA and then remove this note.

It should be noted that the CRYPTO chunk format requires the receiver
stop processing this SCTP packet, discard the unrecognized chunk and
all further chunks, and report the unrecognized chunk in an ERROR
chunk using the 'Unrecognized Chunk Type' error cause.  This is
accomplished (as described in {{RFC9260}} Section 3.2.) by the use of
the upper bits of the chunk type.

The CRYPTO chunk is used to hold the protected payload of a plain SCTP
packet.

~~~~~~~~~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Type = 0x4X  |  Chunk Flags  |         Chunk Length          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
|                            Payload                            |
|                                                               |
|                               +-------------------------------+
|                               |           Padding             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~
{: #sctp-Crypto-chunk-newchunk-crypt-struct title="CRYPTO Chunk Structure" artwork-align="center"}

{: vspace="0"}
Chunk Type: 8 bits (unsigned integer)
: This value MUST be set to 0x4X for all CRYPTO chunks.

Chunk Flags: 8 bits
: This is used by the protection engine and ignored by SCTP.

Chunk Length: 16 bits (unsigned integer)
: This value holds the length of the Payload in bytes plus 4.

Payload: variable length
: This holds the encrypted data.

Padding: 0, 8, 16, or 24 bits
: If the length of the Payload is not a multiple of 4 bytes, the sender
  MUST pad the chunk with all zero bytes to make the chunk 32-bit
  aligned.  The Padding MUST NOT be longer than 3 bytes and it MUST
  be ignored by the receiver.

##  Protected Association Parameter Validation Chunk (PVALID) {#pvalid-chunk}

This section defines the new chunk types that will be used to validate
the negotiation of the protection engine selected for CRYPTO chunk.
This to prevent down grade attacks on the negotiation of protection
engines. {{sctp-Crypto-chunk-newchunk-pvalid-chunk}} illustrates the
new chunk type.

| Chunk Type | Chunk Name |
| 0x4X | Protected Association Parameter Validation (PVALID) |
{: #sctp-Crypto-chunk-newchunk-pvalid-chunk title="PVALID Chunk Type" cols="r l"}

It should be noted that the PVALID chunk format requires the receiver
stop processing this SCTP packet, discard the unrecognized chunk and
all further chunks, and report the unrecognized chunk in an ERROR
chunk using the 'Unrecognized Chunk Type' error cause.  This is
accomplished (as described in {{RFC9260}} Section 3.2.) by the use of
the upper bits of the chunk type.

The PVALID chunk is used to hold the protection engines list.

~~~~~~~~~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Type = 0x4X  |   Flags = 0   |         Chunk Length          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
|                      Protection Engines                       |
|                                                               |
|                               +-------------------------------+
|                               |           Padding             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~
{: #sctp-Crypto-chunk-newchunk-PVALID -struct title="PVALID Chunk Structure" artwork-align="center"}

{: vspace="0"}
Chunk Type: 8 bits (unsigned integer)
: This value MUST be set to 0x4X.

Chunk Flags: 8 bits
: MUST be set to zero on transmit and MUST be ignored on receipt.

Chunk Length: 16 bits (unsigned integer)
: This value holds the length of the Protection Engines field in bytes plus 4.

Protection Engines: variable length
: This holds the list of protection engines where each protection engine is
  specified by a 16-bit unsigned integer. The field MUST be identical to the
  content of the Protected Association Parameter ({{protectedassoc-parameter}})
  Protection Engines field that the endpoint sent in the INIT or INIT-ACK chunk.

Padding: 0 or 16 bits
: If the length of the
  Protection Engines field is not a multiple of 4 bytes, the sender MUST
  pad the chunk with all zero bytes to make the chunk 32-bit
  aligned.  The Padding MUST NOT be longer than 2 bytes and it
  MUST be ignored by the receiver.

RFC-Editor Note: Please replace 0x4X with the actual chunk type value
assigned by IANA and then remove this note.

# Error Handling {#error_handling}

This specification introduces a new set of error causes that are to be
used when SCTP endpoint detects a faulty condition. The special case is
when the error is detected by the protection engine that may provide
additional information.

## Mandatory Protected Association Parameter Missing {#enoprotected}

When an initiator SCTP endpoint sends an INIT chunk that doesn't
contain the Protected Association parameter towards an SCTP endpoint
that only accepts protected associations, the responder endpoint SHALL
raise a Missing Mandatory Parameter error. The ERROR chunk will
contain the cause code 'Missing Mandatory Parameter' (2) (see
{{RFC9260}} section 3.3.10.7) and the protected association parameter
identifier {{protectedassoc-parameter}} in the missing param
Information field.

~~~~~~~~~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|        Cause Code = 2         |         Cause Length          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                 Number of missing params = N                  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   Protected Association ID    |     Missing Param Type #2     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|    Missing Param Type #N-1    |     Missing Param Type #N     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~
{: #sctp-Crypto-init-chunk-missing-protected title="ERROR Missing Protected Association Paramater" artwork-align="center"}

Note: Cause Length is equal to the number of missing parameters 8 + N
* 2 according to {{RFC9260}}, section 3.3.10.2. Also the Protection
Association ID may be present in any of the N missing params, no order
implied by the example in
{{sctp-Crypto-init-chunk-missing-protected}}.

## Error in Protection {#eprotect}

A new Error Type is defined for Crypto Chunk, it's used for any
error related to the Protection mechanism described in this
document and has a structure that allows detailed information
to be added as extra causes.

This specification describes some of the causes whilst the
Protection Engine Specification MAY add further Causes related to the
related Protection Engine.

When detecting an error, SCTP will send an ABORT chunk containing
the relevant Error Type and Causes.

~~~~~~~~~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|     Cause Code = TBA9         |         Cause Length          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         Extra Cause #1        |         Extra Cause #2        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|        Extra Cause #N-1       |         Extra Cause #N        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~
{: #sctp-eprotect-error-structure title="Error in Protection Cause Format" artwork-align="center"}

{: vspace="0"}
Casuse Code: 16 bits (unsigned integer)
: The SCTP Error Chunk Cause Code indicating "Error in Protection" is TBA9.

Cause Length: 16 bits (unsigned integer)
: Is for N extra Causes equal to  4 + N * 2

Extra Cause: 16 bits (unsigned integer)
: Each Extra Cause indicate an additional piece of information as part
  of the error. There MAY be zero to as many as can fit in the extra
  cause field in the ERROR Chunk (A maximum of 32764).

Editor's Note: Please replace TBA9 above with what is assigned by IANA.

Below a number of defined Error Causes are defined, additional causes
can be registered with IANA following the rules in {{IANA-Extra-Cause}}.

### No Supported Protection Engine {#eprotlist}

If list of protection engines contained in the INIT signal doesn't
contain at least an entry that fits the list of protection engines at
the responder, the responder will reply with an ABORT chunk with error
in protection cause code (specified in
{{eprotect}}) and the "No Supported Protection
Engine" extra cause code identifier 0x00.

### Error During Protection Handshake {#ekeyhandshake}

If the protection engine specifies a handshake for example for
authentication, and key management is implemented inband, it may happen
that the procedure has errors. In such case an ABORT chunk will be
sent with error in protection cause code (specified in
{{eprotect}}) and extra cause
"Error During Protection Handshake" identifier 0x01.

### Failure in Protection Engines Validation {#evalidate}

A Failure may occur during protection engine Validation (see
{{protected-state}}).  In such case an ABORT chunk will be sent with
error in protection cause code (specified in
{{eprotect}}) and extra cause "Failure in
Protection Engines Validation" identifier 0x02 to indicate this
failure.

### Timeout During Protection Handshake or Validation {#etmout}

Whenever a T-valid timeout occurs, the SCTP endpoint will send an
ABORT chunk with "Error in Protection" cause (specified in
{{eprotect}}) and extra cause "Timeout During
Protection Handshake or Validation" identifier 0x03 to indicate this
failure.  To indicate in which phase the timeout occurred an additional
extra cause code is added. If the protection engine specifies that key
management is implemented inband and the T-valid timeout occurs during
the handshake the Cause-Specific code to add is "Error During
Protection Handshake" identifier 0x01.  If the T-valid timeout occurs
during the protection association parameter validation, the extra
cause code to use is "Failure in Protection Engines Validation"
identifier 0x02.

## Critical Error from Protection Engine {#eengine}

Protection engine MAY inform local SCTP endpoint about errors, in such
case it's to be defined in the protection engine specification
document.  When an Error in the protection engine compromises the
protection mechanism, the protection engine may stop processing data
altogether, thus the local SCTP endpoint will not be able to send or
receive any chunk for the specified Association.  This will cause the
Association to be closed by legacy timer-based mechanism. Since the
Association protection is compromised no further data will be sent and
the remote peer will also experience timeout on the Association.

## Non-critical Error in the Protection Engine {#non-critical-errors}

A non-critical error in the protection engine means that the
protection engine is capable of recovering without the need
of the whole Association to be restarted.

From SCTP perspective, a non-critical error will be perceived
as a temporary problem in the transport and will be handled
with retransmissions and SACKS according to {{RFC9260}}.

When the protection engine will experience a non-critical error,
an ABORT chunk SHALL NOT be sent. This way non-critical errors
are handled and how the protection engine will recover from
these errors is being described in the Protection Engine Specifications.

# Protected SCTP State Diagram {#state-diagram}

The {{sctp-Crypto-state-diagram}} shows the changes of the SCTP
association state machine as described in {{RFC9260}} section 4.

~~~~~~~~~~~ aasvg
                                   .-------- (from any state)
                       .-----.    |
         receive INIT |       |   |    receive ABORT      [ABORT]
--------------------- |       v   v    --------------  or ----------
generate State Cookie |    +---------+ delete TCB         send ABORT
        send INIT ACK  '---+  CLOSED |                    delete TCB
                           +--+----+-+
                             /      \
                            /        \  [ASSOCIATE]
                           |          |-----------------
                           |          | create TCB
                           |          | send INIT
          receive valid    |          | start T1-init timer
          COOKIE  ECHO     |          v
      (1) -----------------|    +-----------+
          create TCB       |    |COOKIE-WAIT| (2)
          send COOKIE ACK  |    +-----+-----+
                           |          |
                           |          | receive INIT ACK
                           |          |-------------------
                           |          | send COOKIE ECHO
                           |          | stop T1-init timer
                           |          | start T1-cookie timer
                           |          v
                           |   +-------------+
                           |   |COOKIE-ECHOED| (3)
                           |   +------+------+
                           |          |
                           |          | receive COOKIE ACK
                           |          |-------------------
                           |          | stop T1-cookie timer
         +-----------------+-----+    |
         |     +-----------------)----+-----+
         |     |                 |          |
         |     |                 v          v
         |     |            +---------------------+
         |     |            |  PROTECTION PENDING | If INIT/INIT-ACK
         |     |            +----------+----------+ has Protected
         |     |                       |            Association
         |     |                       |            Parameter start
         |     |                       |            T-valid timer.
         |     |                       |
         |     |                       | [CRYPTO SETUP]
         |     |                       |-----------------
         |     |                       | send and receive
         |     |                       | protection engine handshake
         |     |                       |
         |     |                       v
         |     |           +----------------------+
         |     |           |       PROTECTED      |
         |     |           +-----------+----------+
         |     |                       |
         |     |                       | [ENDPOINT VALIDATION]
         |     |                       |------------------------
         |     |                       | send and receive
         |     |                       | PVALID by means of
         |     |                       | CRYPTO chunk.
         |     |                       |
         |     +-----------------+     |
         +-----------------+     |     |
                           |     |     |
                           v     v     v
                         +---------------+
                         |  ESTABLISHED  |
                         +---------------+
~~~~~~~~~~~
{: #sctp-Crypto-state-diagram title="SCTP State Diagram with Crypto" artwork-align="center"}

## New States {#new-states}

This section describes details on the amendment to the SCTP
association establishment state machine.

### PROTECTION PENDING {#protection-pending-state}

The presence of a Protected Association Parameter in the INIT or INIT-ACK
chunk makes the State Machine entering PROTECTION PENDING state instead
of ESTABLISHED.

When entering PROTECTION PENDING state, a T-valid timer is started
that will cover the whole validation time including the in-band key
establishment. The value of T-valid is dependent on the protection
engine and may also be further adjusted based if expected RTT values
are outside of the ones commonly occurring on the general Internet,
see {{t-valid-considerations}}.

If key establishment is in-band, the protection engine will start the
handshake with its peer and in case of failure or T-valid timeout, the
Crypto Chunk will generate an ABORT chunk.  The ERROR handling follows
what specified in {{ekeyhandshake}}.  When Handshake has been
successfully completed, the association state machine will enter
PROTECTED state.

The protection engine specification MUST specify when PROTECTED state
can be entered for each endpoint. If key establishment is out-of-band,
after starting T-valid timer the SCTP association will enter PROTECTED
state per protection engine specification when the necessary security
context is in place.

Whilst in PROTECTION PENDING state, only User Layer Protocol data
belonging to the Protection Engine will be handled, such data will be
transferred as SCTP DATA chunks with the Protection Engine PPID
(see {{iana-payload-protection-id}}) for the Protection Engine handshake.

### PROTECTED {#protected-state}

The association state machine can only reach PROTECTED state from
PROTECTION PENDING state (see {{protection-pending-state}}). When entering into
PROTECTED state the T-valid timer is running and the protection engine
has completed the key establishment so that protected data can be sent to
the peer.

Whilst in PROTECTION PENDING state, only User Layer Protocol data
belonging to the Protection Engine will be handled, such data will be
transferred as SCTP DATA chunks with the Protection Engine PPID
(see {{iana-payload-protection-id}}) for the Protection Engine handshake.

In PROTECTED state the association initiating SCTP Endpoint
(initiator) MUST validate the INIT sent protected association
parameter, thus the initiator will send a PVALID chunk that will
contain exactly the same list of Protection Engines as previously sent
in protected association parameter of INIT chunk and in the same order.

When the responder will receive PVALID, it will compare the list of
protection engines with the list received in the INIT chunk, if they
are identical it will reply to the initiator with a PVALID chunk
containing the Protection Engine previously sent as protected
association parameter in INIT-ACK chunk, it will clear the T-valid
timer and will move into ESTABLISHED state.

Once in ESTABLISHED state, only CRYPTO chunks can be sent to the remote peer
and any other type of plain text SCTP chunks coming from the remote
peer will be silently discarded with the exception of SHUTDOWN-COMPLETE chunk.

If the lists of Protection Engines don't match, it will generate an
ABORT chunk. ERROR CAUSE will indicate "Failure in
Protection Engines Validation" and the SCTP association will be
terminated.

After sending PVALID, the initiator will wait for the responder to
reply with the PVALID confirmation. The initiator will compare the
Protection Engine received from the responder, if the value is the
same it will clear the T-valid timer and move into ESTABLISHED state.
If the chosen Protection Engines don't match, it will generate an
ABORT chunk. ERROR CAUSE will indicate "Failure in
Protection Engines Validation" that is critical.

If T-valid timer expires either at initiator or responder, it will generate
an ABORT chunk.  The ERROR handling follows what
specified in {{etmout}}.

### Considerations on key management {#key-management-considerations}

When the Association is in PROTECTION PENDING state,
in-band key management shall exploit SCTP DATA chunk with the Protection Engine
PPID (see {{iana-payload-protection-id}}) that will be sent unencrypted.

When the Association is in PROTECTED, ESTABLISHED or in any of the states that can
be reached after ESTABLISHED state, in-band key management shall exploit
SCTP DATA chunk that will be protected by the Protection Engine and
encapsulated in CRYPTO chunks.

In-band key management shall use a dedicated Payload Protocol Identifier
assigned by IANA and defined in the specific Protection Engine Specification.

### Consideration on T-valid {#t-valid-considerations}

The timer T-Valid supervises initializations that depend on how
the handshake is specified for the Protection Engine and also on
the characteristics of the transport network.

This specification recommends a default value of 30 seconds for
T-valid. This value is expected to be superseded by recommendations in
the Protection Engine Specification for each Protection Engine.

# Procedures {#procedures}

## Establishment of a Protected Association {#establishment-procedure}

An SCTP Endpoint acting as initiator willing to create a protected
association shall send to the remote peer an INIT chunk containing the
Protected Association parameter (see {{protectedassoc-parameter}})
where all the supported Protection Engines are listed, given in
descending order of preference (see
{{sctp-Crypto-chunk-init-options}}).

As alternative, an SCTP Endpoint acting as responder willing to
support only protected associations shall consider INIT chunk not
containing the Protected Association parameter as an error, thus it
will reply with an ABORT chunk according to what specified in
{{enoprotected}} indicating that for this endpoint mandatory protected
association parameter is missing.

An SCTP Endpoint acting as responder, when receiving an INIT chunk
with protected association parameter, will search the list of
protection engines for the most preferred commonly supported choice
and will reply with INIT-ACK containing the protected association
parameter with the chosen protection engine. When the responder cannot
find a supported protection engine, it will reply with ABORT containing
Error in Protection with the extra cause code for "No Supported Protection
Engine" ({{eprotlist}}).

When initiator and responder have agreed on a protected association by
means of handshaking INIT/INIT-ACK with a common protection engine,
only control chunks, CRYPTO chunks and DATA Chunks with the Protection
Engine PPID will be accepted. Any other DATA chunk being sent on a Protected
association will be silently discarded.

After completion of initial handshake, that is after COOKIE-ECHO and
COOKIE-ACK, the Protection Engine shall initialize itself by
transferring its own data as payload of the DATA chunk if necessary.  At
completion of Protection Engine initialization, the setup of the
Protected association is complete and from that time on only CRYPTO
chunks will be exchanged.  Any plain text chunks will be silently
discarded.

After completion of protected association initialization, the
initiator MUST send to the responder a PVALID chunk (see
{{sctp-Crypto-chunk-newchunk-pvalid-chunk}}) containing the list of
Protection Engines previously sent in the protected association
parameter of the INIT chunk. The responder receiving the PVALID chunk
will compare the Protection Engines list with the one previously
received in the INIT chunk, if they are exactly the same, with the
same Protection engine in the same position, it will reply to the
initiator with a PVALID chunk containing the chosen Protection Engine,
otherwise it will reply with an ABORT chunk. If the association was
not aborted the protected association is considered successfully
established.

When the initiator receive the PVALID chunk, it will compare with the
previous chosen Protection Engine and in case of mismatch with the one
received previously in the protected association parameter in the
INIT-ACK chunk, it will reply with ABORT, otherwise the protected
association is successfully established.

## Termination of a Protected Association {#termination-procedure}

Besides the procedures for terminating an association explained in
{{RFC9260}}, the protection engine SHALL ask SCTP endpoint for
terminating an association when having an internal error or by
detecting a security violation, using the procedure described
in {{eengine}}.  The internal design of Protection
Engines and their capability is out of the scope of the current
document.

# Protected Data Chunk Handling {#protected-data-handling}

With reference to the State Diagram as shown in
{{sctp-Crypto-state-diagram}} and Figure 3 of {{RFC9260}}, the
handling of Control chunks, Data chunks and Crypto chunks follows the
rules defined below:

- When the association is in states CLOSED, COOKIE-WAIT, and
COOKIE-ECHOED, any Control chunk is sent unprotected (i.e. plain
text). No DATA chunks shall be sent in these states and DATA chunks
received shall be silently discarded.

- When the association is in state PROTECTION PENDING, any Control
chunk is sent unprotected (i.e. plain text). No DATA chunks, except
than the one used by the Protection Engine for handshake, thus
being identified with the Protection Engine PPID
(see {{iana-payload-protection-id}}), SHALL be
sent in these states and DATA chunks with different PPID being
received shall be silently discarded. DATA Chunks with the
Protection Engine PPID (see {{iana-payload-protection-id}})
shall be sent by the Protection Engine to establish its security context.

- When the association is in state PROTECTED, any SCTP chunk except
for CRYPTO chunks, will be used to create an SCTP payload
that will be encrypted by the Protection Engine and the result from
that encryption will be the used as payload for a CRYPTO chunk that
will be the only chunk in the SCTP packet to be sent. DATA chunks
received with PPID different than Protection engine PPID shall be
silently discarded. Note that User Data in PROTECTED
state are not used for creating DATA chunks, the only DATA chunk being
handled are the ones with Protection Engine PPID
(see {{iana-payload-protection-id}}) related to Protection Engine
handshake.

- When the association is in states ESTABLISHED and in the states for
association shutdown, i.e. SHUTDOWN-PENDING, SHUTDOWN-SENT,
SHUTDOWN-RECEIVED, SHUTDOWN-ACK-SENT as defined by {{RFC9260}}, any
SCTP chunk including DATA chunks, but excluding CRYPTO chunk, will be
used to create an SCTP payload that will be encrypted by the
Protection Engine and the result from that encryption will be the used
as payload for a CRYPTO chunk that will be the only chunk in the SCTP
packet to be sent. DATA chunks are accepted and handled according
to section 4 of {{RFC9260}}.

~~~~~~~~~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         Common Header                         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           Chunk #1                            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                            . . .                              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           Chunk #n                            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~
{: #sctp-Crypto-encrypt-chunk-states-1 title="Plain Text SCTP Packet" artwork-align="center"}

The diagram shown in {{sctp-Crypto-encrypt-chunk-states-1}} describes
the structure of any plain text SCTP packet being sent or received
when the association has not reached the PROTECTED state yet. SCTP
packet as depicted in {{sctp-Crypto-encrypt-chunk-states-2}} may also
be sent in PROTECTION PENDING state and in any later state of the
association.

~~~~~~~~~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         Common Header                         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         CRYPTO Chunk                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~
{: #sctp-Crypto-encrypt-chunk-states-2 title="Protected SCTP Packets" artwork-align="center"}

The diagram shown in {{sctp-Crypto-encrypt-chunk-states-2}} describes
the structure of an SCTP packet being sent after the PROTECTED state
has been reached. Such packets are built with the SCTP common
header. Only one CRYPTO chunk can be sent in a SCTP packet.

## Protected Data Chunk Transmission {#data-sending}

When the association state machine (see {{sctp-Crypto-state-diagram}})
has reached the PROTECTION PENDING state, it MAY perform protection
engine key management inband depending on how the specification for the
chosen Protection Engine has been defined.  In such case, the CRYPTO
chunk Handler will receive plain control and DATA chunks from the SCTP chunk
handler.

When the association state machine (see {{sctp-Crypto-state-diagram}})
has reached the PROTECTED state, the CRYPTO chunk handler will receive
control chunks and DATA chunks from the SCTP chunk handler as a
complete SCTP payload with maximum size limited by PMTU reduced by the
size of the SCTP common header and the CRYPTO chunk overhead.

That plain payload will be sent to the protection engine in use for
that specific association, the protection engine will return an
encrypted payload.

Depending on the specification for the chosen protection engine, when
forming the CRYPTO chunk header the CRYPTO chunk handler MAY set the
chunk header flags (see {{sctp-Crypto-chunk-newchunk-crypt-struct}}).

An SCTP packet containing an SCTP CRYPTO chunk SHALL be delivered
without delay and SCTP bundling SHALL NOT be performed.

## Protected Data Chunk Reception {#data-receiving}

When the association state machine (see {{sctp-Crypto-state-diagram}})
has reached the PROTECTION PENDING state, it MAY handle key management
inband depending on how the specification for the chosen protection
engine has been defined.  In such case, the CRYPTO chunk handler will
receive plain control chunks and DATA chunks with Protection Engine
PPID from the SCTP Header Handler. Those plain control chunks will be
forwarded to SCTP chunk handler.

When the association state machine (see {{sctp-Crypto-state-diagram}})
has reached the PROTECTED state, the CRYPTO chunk handler will receive
CRYPTO chunks from the SCTP Header Handler.  Payload from CRYPTO
chunks will be forwarded to the protection engine in use for that
specific association for decryption, the protection engine will return
a plain SCTP Payload.  The plain SCTP payload will be forwarded to
SCTP Chunk Handler that will split it in separated chunks and will
handle them according to {{RFC9260}}.

Depending on the specification for the chosen protection engine, when
receiving the CRYPTO chunk header the CRYPTO Chunk Handler MAY handle
the Flags (see {{sctp-Crypto-chunk-newchunk-crypt-struct}}) according
to that specification.

Meta data belonging to the SCTP packet received SHALL be tied to the
relevant chunks and forwarded transparently to the SCTP endpoint.

### SCTP Header Handler

The SCTP Header Handler is responsible for correctness of the SCTP
common header, it receives the SCTP packet from the lower transport
layer, discriminates among associations and forwards the payload and
relevant data to the SCTP protection engine for handling.

In the opposite direction it creates the SCTP common header and fills
it with the relevant information for the specific association and
delivers it towards the lower transport layer.


# IANA Considerations {#IANA-Consideration}

This document defines two new registries in the Stream Control
Transmission Protocol (SCTP) Parameters group that IANA
maintains. Theses registries are for the protection engine identifiers
and extra cause codes for protection related errors. It also adds
registry entries into several other registries in the Stream Control
Transmission Protocol (SCTP) Parameters group:

*  Two new SCTP Chunk Types

*  One new SCTP Chunk Parameter Type

*  One new SCTP Error Cause Codes

*  One new SCTP Payload Protocol Identifier

## Protection Engine Identifier Registry {#iana-protection-engines}

IANA is requested to create a new registry called "CRYPTO Chunk
Protection Engine Identifiers". This registry is part of the Stream
Control Transmission Protocol (SCTP) Parameters grouping.

The purpose of this registry is to enable identification of different
protection engines used by the CRYPTO chunk when performing the SCTP
handshake and negotiating support. Entries in the registry requires a
protection engine name, a reference to the specification for the
protection engine, and a contact. Each entry will be assigned by IANA
a unique 16-bit unsigned integer identifier for their protection
engine. Values 0-65534 are available for assignment. Value 65535 is
reserved for future extension. The proposed general form of the
registry is depicted below in {{iana-protection-engine-identifier}}.

| ID Value | Name | Reference | Contact |
| 0-65534 | Available for Assignment | RFC-To-Be | |
| 65535 | Reserved | RFC-To-Be | Authors |
{: #iana-protection-engine-identifier title="Protection Engine Identifier Registry" cols="r l l l"}

New entries are registered following the Specification Required policy
as defined by {{RFC8126}}.

## Protection Error Cause Codes Registry {#IANA-Extra-Cause}

IANA is requested to create a new registry called "Protection Error
Cause Codes". This registry is part of the Stream Control Transmission
Protocol (SCTP) Parameters grouping.

The purpose of this registry is to enable identification of different
protection related errors when using CRYPTO chunk and a protection
engine.  Entries in the registry requires a Meaning, a reference to
the specification defining the error, and a contact. Each entry will
be assigned by IANA a unique 16-bit unsigned integer identifier for
their protection engine. Values 0-65534 are available for
assignment. Value 65535 is reserved for future extension. The proposed
general form of the registry is depicted below in
{{iana-protection-error-cause}}.

| Cause Code | Meaning | Reference | Contact |
| 0 | Error in the Protection Engine List | RFC-To-Be | Authors |
| 1 | Error During Protection Handshake | RFC-To-Be | Authors|
| 2 | Failure in Protection Engines Validation | RFC-To-Be | Authors |
| 3 | Timeout During KEY Handshake or Validation | RFC-To-Be | Authors |
| 4-65534 | Available for Assignment | RFC-To-Be | Authors |
| 65535 | Reserved | RFC-To-Be | Authors |
{: #iana-protection-error-cause title="Protection Error Cause Code" cols="r l l l"}

New entries are registered following the Specification Required policy
as defined by {{RFC8126}}.

## SCTP Chunk Types

In the Stream Control Transmission Protocol (SCTP) Parameters group's
"Chunk Types" registry, IANA is requested to add the two new entries
depicted below in in {{iana-chunk-types}} with a reference to this
document. The registry at time of writing was available at:
https://www.iana.org/assignments/sctp-parameters/sctp-parameters.xhtml#sctp-parameters-1

| ID Value | Chunk Type | Reference |
| TBA6 | Crypto Chunk (CRYPTO) | RFC-To-Be |
| TBA7 | Protected Association Parameter Validation (PVALID) | RFC-To-Be |
{: #iana-chunk-types title="New Chunk Types Registered" cols="r l l"}


## SCTP Chunk Parameter Types

In the Stream Control Transmission Protocol (SCTP) Parameters group's
"Chunk Parameter Types" registry, IANA is requested to add the new
entry depicted below in in {{iana-chunk-parameter-types}} with a
reference to this document. The registry at time of writing was
available at:
https://www.iana.org/assignments/sctp-parameters/sctp-parameters.xhtml#sctp-parameters-2

| ID Value | Chunk Parameter Type | Reference |
| TBA8 | Protected Association | RFC-To-Be |
{: #iana-chunk-parameter-types title="New Chunk Type Parameters Registered" cols="r l l"}


## SCTP Error Cause Codes

In the Stream Control Transmission Protocol (SCTP) Parameters group's
"Error Cause Codes" registry, IANA is requested to add the new
entry depicted below in in {{iana-error-cause-codes}} with a
reference to this document. The registry at time of writing was
available at:
https://www.iana.org/assignments/sctp-parameters/sctp-parameters.xhtml#sctp-parameters-24

| ID Value | Error Cause Codes | Reference |
| TBA9 | Protection Engine Error | RFC-To-Be |
{: #iana-error-cause-codes title="Error Cause Codes Parameters Registered" cols="r l l"}


## SCTP Payload Protocol Identifier

In the Stream Control Transmission Protocol (SCTP) Parameters group's
"Payload Protocol Identifiers" registry, IANA is requested to add the new
entry depicted below in in {{iana-payload-protection-id}} with a
reference to this document. The registry at time of writing was
available at:
https://www.iana.org/assignments/sctp-parameters/sctp-parameters.xhtml#sctp-parameters-25

| ID Value | SCTP Payload Protocol Identifier | Reference |
| TBA10 | Protection Engine Protocol Identifier | RFC-To-Be |
{: #iana-payload-protection-id title="Protection Engine Protocol Identifier Registered" cols="r l l"}


# Security Considerations {#Security-Considerations}

All the security and privacy considerations of the security protocol
used as the protection engine applies.

## Privacy Considerations

Using a security protocol in the SCTP CRYPTO chunk might lower the
privacy properties of the security protocol as the SCTP Verification
Tag is an unique identifier for the association.

## Downgrade Attacks {#Downgrade-Attacks}

The CRYPTO chunk provides a mechanism for preventing downgrade attacks
that detects downgrading attempts between protection engines and
terminates the association. The chosen protection engine is the same
as if the peers had been communicating in the absence of an attacker.

The protection engine initial handshake is verified before the
association is set as ESTABLISHED, thus no user data are sent before
validation.

The downgrade protection is only as strong as the weakest of the
supported protection engines as an active attacker can trick the
endpoints to negotiate the weakest protection engine and then
modify the weakly protected CRYPTO chunks to deceive the endpoints
that the negotiation of the protection engines is validated. This
is similar to the downgrade protection in TLS 1.3 specified in
Section 4.1.3. of {{RFC8446}} where downgrade protection is not
provided when TLS 1.2 with static RSA is used. It is RECOMMENDED
to only support a limited set of strongly profiled protection
engines.

# Requirements towards the protection engines {#requirements}

This section specifies what is to be specified in the description
of a protection engine.

 * Define how to protect the plain text set of chunks and encapsulate
   them in the CRYPTO Chunk payload.

 * Can define its usage of the 8-bit chunk Flags field in the CRYPTO
   chunk

 * Is required to register the defined protection engine(s) with IANA
   per {{iana-protection-engines}}.

 * Detail the state transition between PROTECTION PENDING and
   PROTECTED state (see {{state-diagram}}).

# Acknowledgments

   The authors thank Michael Tüxen for his invaluable comments
   helping to cope with Association Restart, ASCONF handling and
   restructuring the Protection Engine architecture.
