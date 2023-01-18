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

normative:
  RFC2119:
  RFC8126:
  RFC8174:
  RFC9260:

  IANA-SCTP-PARAMETERS:
    target: https://www.iana.org/assignments/sctp-parameters/sctp-parameters.xhtml
    title: Stream Control Transmission Protocol (SCTP) Parameters

--- abstract

This document describes a method for adding cryptographic protection
to the Stream Control Transmission Protocol (SCTP). The SCTP CRYPTO
chunk defined in this document is intended to enable communications
privacy for applications that use SCTP as their transport protocol and
allows applications to communicate in a way that is designed to
prevent eavesdropping and detect tampering or message forgery.

The CRYPTO chunk defined here in is one half of a complete
solution. Where a companion specification is required to define how
the content of the CRYPTO chunk is encrypted, authenticated, and
protected against replay, as well as how key management is
accomplished.

Applications using SCTP CRYPTO chunk can use all transport
features provided by SCTP and its extensions.

--- middle

# Introduction {#introduction}

   This document defines a CRYPTO chunk for the Stream Control
   Transmission Protocol (SCTP), as defined in {{RFC9260}}.

   This specification defines the actual CRYPTO chunk. How to enable
   it usage, how it interacts with the SCTP association establishment
   to enable endpoint authentication, key-establishment, and other
   features of the separate specification.

   This specification is intended to be capable of enabling mutual
   authentication of endpoints, data confidentiality, data origin
   authentication, data integrity protection, and data replay
   protection for SCTP packets after the SCTP association has been
   established. The exact properties will depend on the companion
   cipher specification used with the CRYPTO chunk.

   Applications using SCTP CRYPTO chunk can use all transport
   features provided by SCTP and its extensions. Due to its level of
   integration as discussed in next section it will provide its
   security functions on all content of the SCTP packet, and will thus
   not impact the potential to utilize any SCTP functionalities or
   extensions.

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
+---------------------+
|                     |
|        ULP          |
|                     |
+---------------------+ <-- User Level Messages
|                     |
| SCTP Chunks Handler | +-- SCTP Unprotected Payload
|                     |/
+---------------------+    +--------------------+
|        CRYPTO       +--->| Protection Engine  |
|        Chunk        |    +--------------------+
|       Handler       |<---+   Key Management   |
+---------------------+    +--------------------+
|                     |\
| SCTP Header Handler | +-- SCTP Protected Payload
|                     |
+---------------------+
~~~~~~~~~~~
{: #sctp-Crypto-chunk-layering title="CRYPTO chunk layering
in regard to SCTP and ULP" artwork-align="center"}

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
exchange information about the protection engine availability. Once
the peers have agreed on what protection to use, the SCTP hosts start
sending SCTP Encrypted chunks containing the initialization
information related to the protection engine including the endpoint
validation. This is depending on the chosen protection engine thus is
not being detailed in the current specification.

When the endpoint authentication has been completed, the association
is meant to be initialized and the ULP is informed about that, from
this time on it's possible for the ULPs to exchange data.

CRYPTO chunks will never be retransmitted, retransmission is
implemented by SCTP host at chunk level as in the legacy.  Duplicated
CRYPTO chunks, whenever they will be accepted by the protection
engine, will result in duplicated SCTP chunks and will be handled as
duplicated chunks by SCTP host in the same way a duplicated SCTP packet
with those SCTP chunks would have been.


## Protection Engines Considerations {#protection-engines}

The protection engine, independently from the security
characteristics, needs to be capable working on an unreliable
transport mechanism same as UDP and have own KEY handler capability.

SCTP CRYPTO chunk directly exploits the protection engine by
requesting encryption and decryption of a buffer, in particular the
encrypted buffer shall never exceed the SCTP payload size thus
protection engine shall be aware of the PMTU (see {{pmtu}}).

A protection engine may use KEYs, in this case the KEY Management
part of the protection engine is the set of data and procedures
that take care of KEY distribution, verification, and update.

KEY Management of protection engine SHOULD exploit SCTP CRYPTO chunk for
handshaking, in that case any packet being exchanged between
protection engine peers shall be transported as payload of Encrypted
chunk (see {{crypto-chunk}}).

KEY Management MAY use other mechanism than what provided by SCTP CRYPTO
chunks, in any case the mechanism for KEY Management MUST be specified
in the Cipher Specification document for that specific
protection engine.

Out-of-band communication between protection engines MAY exploit the
Flags byte provided by the ENCRYPT chunk header (see
{{sctp-Crypto-chunk-newchunk-crypt-struct}}).

Details of the use of Flags, if different from what described in the
current document, MUST be specified in the Cipher Specification
document for that specific protection engine.

An example of protection engine can be DTLS.

## SCTP CRYPTO Chunk Buffering and Flow Control {#buffering}

protection engine and SCTP are asynchronous, meaning that the
protection engine may deliver the decrypted SCTP Payload to the SCTP
Host without respecting the reception order.  It's up to SCTP Host to
reorder the chunks in the reception buffer and to take care of the
flow control according to what specified in {{RFC9260}}. From SCTP
perspective the CRYPTO chunk is part of the transport network.

Even though the above allows the implementors to adopt a
multithreading design of the protection engines, the actual
implementation should consider that out-of-order handling of SCTP
chunks is not desired and may cause false congestions and
retransmissions.

## PMTU Considerations {#pmtu}

The addition of the CRYPTO chunk to SCTP reduces the room for payload, in
order to cope with that when creating the payload of SCTP for
protection the size of the CRYPTO chunk header and plain text
expansion due to algorithm and any authentication tag needs to be
included in the calculation.

On the other hand, the protection engine needs to be informed about
the PMTU by removing from the value the sum of the common SCTP header
and the Encrypted chunk header. This implies that SCTP can propagate
the computed PMTU at run time specifically. The way protection engine
provides the primitive for PMTU communication shall be part of the
Cipher Specification.

From SCTP perspective, if there is a maximum size of plain text data
that can be protected by the protection engine that must be
communicated to SCTP. As such a limit will limit the PMTU for SCTP to
the maximums plain text plus CRYPTO chunk and algorithm overhead plus
the SCTP common header.

## Congestion Control Considerations {#congestion}

The SCTP mechanism for handling congestion control does depend on
successful data transfer for enlarging or reducing the congestion
window CWND (see {{RFC9260}} section 7.2).

It may happen that protection engine discards packets due to internal
checks or because it has detected a malicious attempt.

The protection engine shall not interfere with the SCTP congestion
control mechanism, this basically means that from SCTP perspective
the congestion control is exactly the same as how specified
in {{RFC9260}}.

## ICMP Considerations {#icmp}

SCTP implementation will be responsible for handling ICMP messages and
their validation as specified in {{RFC9260}} Section 10. This includes
for SCTP packets sent by the protection engines key management
function. However, valid ICMP errors or information may indirectly be
provided to the protection engine, such as an update to PMTU values
based on packet to big ICMP messages.


# Conventions

{::boilerplate bcp14}


# New Parameter Type {#new-parameter-type}

This section defines the new parameter type that will be used to
negotiate the use of the CRYPTO chunk and encryption engines during
association setup. {{sctp-Crypto-chunk-init-parameter}} illustrates
the new parameter type.

| Parameter Type | Parameter Name |
| 0x80xx | Protected Association (PROTECTED) |
{: #sctp-Crypto-chunk-init-parameter title="New INIT Parameter" cols="r l"}

Note that the parameter format requires the receiver to ignore the
parameter and continue processing if the parameter is not understood.
This is accomplished (as described in {{RFC9260}}, Section 3.2.1.)  by
the use of the upper bits of the parameter type.

## Protected Association Parameter {#crypt-parameter}

This parameter is used to carry the list of proposed protection
engines and the chosen protection engine during INIT/INIT-ACK
handshake.

~~~~~~~~~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|     Parameter Type = 0x80xx   |       Parameter Length        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|        Protection Engines                                     |
|                               +-------------------------------+
|                               |            padding            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~
{: #sctp-Crypto-chunk-init-options title="Protected association parameter" artwork-align="center"}

   Type: 2 byte (unsigned integer)
      This value MUST be set to 0x80xx.

   Length: 2 bytes (unsigned integer) This value holds the length of
      the Protection Engines field in bytes plus 4.

   Protection Engines: n 16-bit words (unsigned integer)
      This holds the list of protection engines in order of preference.
      Each protection engine is specified by a 16-bit word.

   Padding: 0, or 2 bytes (unsigned integer) If the length of the
      Protection Engines field is not a multiple of 4 bytes, the sender MUST
      pad the chunk with all zero bytes to make the chunk 32-bit
      aligned.  The Padding MUST NOT be longer than 2 bytes and it
      MUST be ignored by the receiver.

RFC-Editor Note: Please replace 0x08xx with the actual parameter type
value assigned by IANA and then remove this note.


# New Chunk Types {#new-chunk-types}

##  CRYPTO Chunk {#crypto-chunk}

This section defines the new chunk type that will be used to
transport protected SCTP payload.
{{sctp-Crypto-chunk-newchunk-crypt}} illustrates the new chunk type.

| Chunk Type | Chunk Name |
| 0x0x | CRYPTO Chunk |
{: #sctp-Crypto-chunk-newchunk-crypt title="CRYPTO Chunk Type" cols="r l"}

RFC-Editor Note: Please replace 0x0x with the actual chunk type value
assigned by IANA and then remove this note.

It should be noted that the CRYPTO chunk format requires the receiver
to ignore the chunk if it is not understood and silently discard all
chunks that follow in the same SCP packet.  This is accomplished
(as described in {{RFC9260}} Section 3.2.) by the use of the upper
bits of the chunk type.

The CRYPTO chunk is used to hold the protected payload of a plain SCTP packet.

~~~~~~~~~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| Type = 0x0x   |   Flags       |             Length            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
|                            Payload                            |
|                                                               |
|                               +-------------------------------+
|                               |           Padding             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~
{: #sctp-Crypto-chunk-newchunk-crypt-struct title="CRYPTO Chunk Structure" artwork-align="center"}

   Type: 1 byte (unsigned integer)
      This value MUST be set to 0x0x for all CRYPTO chunks.

   Flags: 1 byte (unsigned integer)
      This is used by the protection engine and ignored by SCTP.

   Length: 2 bytes (unsigned integer)
      This value holds the length of the Payload in bytes plus 4.

   Payload: n bytes (unsigned integer)
      This holds the encrypted data.

   Padding: 0, 1, 2, or 3 bytes (unsigned integer)
      If the length of the Payload is not a multiple of 4 bytes, the sender
      MUST pad the chunk with all zero bytes to make the chunk 32-bit
      aligned.  The Padding MUST NOT be longer than 3 bytes and it MUST
      be ignored by the receiver.

##  INIT Option Validation Chunk (PVALID ) {#pvalid-chunk}

This section defines the new chunk types that will be used to validate
the negotiation of the protection engine selected for CRYPTO
chunk.  {{sctp-Crypto-chunk-newchunk-pvalid-chunk}} illustrates the new
chunk type.

| Chunk Type | Chunk Name |
| 0x0x | INIT Option Validation (PVALID ) |
{: #sctp-Crypto-chunk-newchunk-pvalid-chunk title="PVALID Chunk Type" cols="r l"}

It should be noted that the PVALID chunk format requires the receiver
to ignore the chunk if it is not understood and silently discard all
chunks that follow in the same SCP packet.  This is accomplished
(as described in {{RFC9260}} Section 3.2.) by the use of the
upper bits of the chunk type.

The PVALID chunk is used to hold the protection engines list.

~~~~~~~~~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| Type = 0xXX   |   Flags=0     |             Length            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                      Protection Engines                       |
|                                                               |
|                                                               |
|                               +-------------------------------+
|                               |           Padding             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~
{: #sctp-Crypto-chunk-newchunk-PVALID -struct title="PVALID Chunk Structure" artwork-align="center"}

  Type: 1 byte (unsigned integer)
      This value MUST be set to 0xXX.

   Flags: 1 byte (unsigned integer)
      SHOULD be set to zero on transmit and MUST be ignored on receipt.

   Length: 2 bytes (unsigned integer)
      This value holds the length of the Protection Engines field in bytes plus 4.

   Protection Engines: n words (unsigned integer) This holds the list
      of protection engines in order of preference.  Each Protection
      engine is specified by a 16-bit word. This field MUST be
      identical to the content of the CRYPT parameter ({{crypt-parameter}}) Protection
      Engines field that the endpoint sent in the handshake.

   Padding: 0, or 2 bytes (unsigned integer)
      If the length of the Protection Engines field is not a multiple of 4 bytes, the sender
      MUST pad the chunk with all zero bytes to make the chunk 32-bit
      aligned.  The Padding MUST NOT be longer than 3 bytes and it MUST
      be ignored by the receiver.

RFC-Editor Note: Please replace 0xXX with the actual chunk type value
assigned by IANA and then remove this note.


# Error Handling {#error_handling}

This specification introduces a new set of error causes that are to be
used when SCTP endpoint detects a faulty condition. The special case is
when the error is detected by the protection engine that may provide
additional information.

## Mandatory PROTECTED Option Missing (ENOPROTECTED) {#enoprotected}

When a client SCTP endpoint sends an INIT chunk that doesn't contain the
PROTECTED association parameter towards an SCTP endpoint that only
accepts protected associations, the server endpoint will raise a
Missing Mandatory Parameter error. The ERROR chunk will contain
the cause code 'Missing Mandatory Parameter' (2) (see {{RFC9260}}
section 3.3.10.7) and PROTECTED in the missing param Information field.

~~~~~~~~~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|        Cause Code = 2         |         Cause Length          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                 Number of missing params = N                  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|            PROTECTED          |     Missing Param Type #2     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|    Missing Param Type #N-1    |     Missing Param Type #N     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~
{: #sctp-Crypto-init-chunk-missing-protected title="ERROR Missing PROTECTED Paramater" artwork-align="center"}

Cause Length is equal to the number of missing parameters N + 8 * 2
according to {{RFC9260}}, section 3.3.10.2.

## Error in the Protection Engine List (EPROTLIST) {#eprotlist}

If list of protection engines contained in the INIT signal doesn't
contain at least an entry that fits the list of protection engines
at the server, the server will reply with an ERROR chunk with ECRYPTO
cause code (specified in {{sctp-Crypto-new-error-causes}}) and additional
Cause-Specific EPROTLIST.

## Error During KEY Handshake (EHANDSHAKE) {#ekeyhandshake}

If the protection engine specifies that KEY handling is implemented
inband it may happen that the procedure has errors. In such case an
ERROR chunk will be sent with ECRYPTO cause code (specified in
{{sctp-Crypto-new-error-causes}}) and additional Cause-Specific
EHANDSHAKE. The error MAY contain further Cause-Specific Informations
according to the Cipher Specification.

## Error in Protection Engines Validation (EVALIDATE) {#evalidate}

An error may occur during protection engine Validation (see
{encrypted-state}).
In such case an ERROR chunk will be sent with ECRYPTO cause code
(specified in {{sctp-Crypto-new-error-causes}}) and additional
Cause-Specific EVALIDATE.

## Timeout During KEY Handshake or Validation (ETMOUT) {#etmout}

Whenever a T-valid timeout occurs, the SCTP Host will send an ERROR
chunk with ETMOUT cause (specified in
{{sctp-Crypto-new-error-causes}}). Depending on the state, and
additional Cause-Specific code will be added.
If the protection engine specifies that KEY handling is implemented
inband and the T-valid timeout occurs during the handshake
the Cause-Specific code to use is EHANDSHAKE.
If the T-valid timeout occurs during the Validation, the
Cause-Specific code to use is EVALIDATE.

## Critical Error from Protection Engine {#eengine}

Protection engine MAY inform local SCTP Host about errors,
in such case it's to be defined in the Cipher Specification document.
When an Error in the protection engine compromises the
protection mechanism, the protection engine shall stop processing
data in such a way that the local SCTP Host will not be able sending
or receiving any chunk for the specified Association.
This will cause the Association to be closed by legacy
timer-based mechanism. Since the Association protection is
compromised no further data will be sent and the remote peer
will also experience timeout on the Association.

## Non-critical Error in the Protection Engine {#non-critical-errors}

A non-critical error in the Pprotection engine means that the
protection engine is capable of recovering without the need
of the whole Association to be restarted.

From SCTP perspective, a non-critical error will be perceived
as a temporary problem in the transport and will be handled
with retransmissions and SACKS according to {{RFC9260}}.

When the protection engine will experience a non-critical error,
an ERROR chunks SHALL NOT be sent. This way non-critical errors
are handled and how the protection engine will recover from
these errors is being described in the Cipher Specifications.

## New Error Causes {#new_errors}

This section lists the new error cause that are to be used.
They require additional lines of the "Error Cause Codes" table in
SCTP-parameters {{IANA-SCTP-PARAMETERS}}:

| Value | Cause Code | Reference |
| TBA1 | ECRYPTO | RFC-To-Be |
{: #sctp-Crypto-new-error-causes title="New Error Causes" cols="r l l"}

The following {{sctp-Crypto-new-local-error-causes}} defines the error
causes that don't need IANA specification as they are subordinate to
ECRYPTO.

| Value | Cause Code | Reference |
| TBA2 | EHANDSHAKE | RFC-To-Be |
| TBA3 | EVALIDATE | RFC-To-Be |
| TBA4 | ETMOUT | RFC-To-Be |
{: #sctp-Crypto-new-local-error-causes title="New Local Error Causes" cols="r l l"}


# Encrypted SCTP State Diagram {#state-diagram}

The {{sctp-Crypto-state-diagram}} shows the changes versus the SCTP
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
         |     |              +-----------------+
         |     |              |  CRYPT PENDING  | If INIT/INIT-ACK
         |     |              +--------+--------+ has CRYPT option
         |     |                       |          start T-valid
         |     |                       |          timer.
         |     |                       |
         |     |                       | [CRYPTO SETUP]
         |     |                       |-----------------
         |     |                       | send and receive
         |     |                       | encrypt engine handshake
         |     |                       | by means of ENCRYPT chunks
         |     |                       | in case of error
         |     |                       | send plain ABORT
         |     |                       v
         |     |              +-----------------+
         |     |              |    ENCRYPTED    |
         |     |              +--------+--------+
         |     |                       |
         |     |                       | [ENDPOINT VALIDATION]
         |     |                       |------------------------
         |     |                       | send and receive
         |     |                       | PVALID by means of
         |     |                       | ENCRYPT chunk
         |     |                       | in case of error
         |     |                       | send plain ABORT
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
association Establishment state machine.

### CRYPT PENDING {#crypt-pending-state}

The presence of CRYPT option in INIT or INIT-ACK chunk makes the State
Machine entering CRYPT PENDING state instead of ESTABLISHED.

When entering CRYPT PENDING state, a T-valid timer is started that
will cover the whole validation time including the in-band KEY
handling.  It's up to the implementor to take care of the value for
the timer also related to the time needed for KEY handshake of each
protection engine.

If KEY handling is in-band, the protection engine will start the
handshake with its peer and in case of failure or T-valid
timeout, it will generate an ERROR chunk and an ABORT chunk.  The
ERROR handling follows what specified in {{ekeyhandshake}}.  When
Handshake has been successfully completed, the association state
machine will enter ENCRYPTED state.

If KEY handling is out-of-band, after starting T-valid timer the SCTP
association will enter ENCRYPTED state.

### ENCRYPTED {#encrypted-state}

The association state machine can only reach ENCRYPTED state from
CRYPT PENDING state (see {{crypt-pending-state}}). When entering into
ENCRYPTED state the T-valid timer is running and the protection engine
has completed the KEY handshake so that encrypted data can be sent to
the peer.

From this time on, only ENCRYPT chunks can be sent to the remote peer
and any other type of plain text SCTP chunks coming from the remote
peer will be silently discarded.

In ENCRYPTED state the association initiating SCTP Endpoint (Client)
MUST validate the INIT sent CRYPT parameter, thus the Client will send
a PVALID chunk that will contain exactly the same list of Protection
Engines as previously sent in CRYPT option of INIT chunk and in the
same order.

When the Server will receive PVALID , it will compare the list of
protection engines with the list received in the INIT chunk, if they
are identical it will reply to the Client with a PVALID chunk
containing the Protection Engine previously sent as CRYPT option in
INIT-ACK chunk, it will clear the T-valid timer and will move into
ESTABLISHED state.

If the lists of Protection Engines don't match, it will generate an
ERROR chunk and an ABORT chunk. ERROR CAUSE will indicate EVALIDATE
meaning that an error has been happening during VALIDATION of SCTP
Endpoints.

After sending PVALID , the Client will wait for the Server to reply
with the PVALID confirmation. The Client will compare the Protection
Engine received from the Server, if the value is the same it will
clear the T-valid timer and move into ESTABLISHED state.  If the
chosen Protection Engines don't match, it will generate an ERROR chunk
and an ABORT chunk. ERROR CAUSE will indicate EVALIDATE meaning that
an error has been happening during VALIDATION of SCTP Endpoints.

If T-valid timer expires either at Client or Server, it will generate
an ERROR chunk and an ABORT chunk.  The ERROR handling follows what
specified in {{etmout}}.

### Consideration on T-valid

The timer T-Valid supervises initializations that depend on how
the handshake is specified for the Protection Engine and also on
the characteristics of the transport network.
This specification recommends a default value of 30 seconds for
T-valid. This value is superseded by what recommended in the
Cipher Specification for each Protection Engine.

# Procedures {#procedures}

## Establishment of an Encrypted Association {#establishment-procedure}

An SCTP Endpoint acting as Client willing to create an Encrypted
association shall send to the remote peer an INIT chunk containing the
CRYPT parameter (see {{sctp-Crypto-chunk-newchunk-crypt}}) where the
Protection Engines lists all the supported Protection Engines, given
in order of preference (see {{sctp-Crypto-chunk-init-options}}).

As alternative, an SCTP Endpoint acting as Server willing to support
only Encrypted associations shall consider INIT chunk not containing
the PROTECTED parameter as an error, thus it will reply with an ERROR
chunk according to what specified in {{enoprotected}} indicating that the
mandatory CRYPT option is missing.

An SCTP Endpoint acting as Server, when receiving an INIT chunk with
CRYPT parameter, will search the list of Protection Engines for a
common choice and will reply with INIT-ACK containing the CRYPT
parameter with the chosen Protection Engine. When the Server cannot
find a supported Protection Engine, it will reply with ABORT and
ERROR according to {{eprotlist}}.

When Client and Server have agreed on an Encrypted association by
means of handshaking INIT/INIT-ACK with a common Protection Engine,
only Control chunks and Encrypted chunks will be accepted. Any Data
chunk being sent on an Encrypted association will be silently
discarded.

After completion of initial handshake, that is after COOKIE-ECHO and
COOKIE-ACK, the Protection Engine shall initialize itself by
transferring its own data as Payload of the ENCRYPT chunk (see
{{sctp-Crypto-chunk-newchunk-crypt-struct}}) if necessary.  At
completion of Protection Engine initialization, the setup of the
Encrypted association is complete and from that time on only ENCRYPT
chunks will be exchanged.  Any other type of plain text chunks will be
silently discarded.

After completion of Encrypted association initialization, the Client
MUST send to the Server a PVALID chunk (see
{{sctp-Crypto-chunk-newchunk-pvalid-chunk}}) containing the list of
Protection Engines previously sent in the CRYPT parameter of the INIT
chunk. The Server receiving the PVALID chunk will compare the
Protection Engines list with the one previously received in the INIT
chunk, if they will be exactly the same, with the same Protection
engine in the same position, it will reply to the Client with a
PVALID chunk containing the chose Protection Engine, otherwise it will
reply with an ABORT chunk.  When the Client will receive the PVALID
chunk, it will compare with the previous chosen Protection Engine and
in case of mismatch with the one received previously as CRYPT
parameter in the INIT-ACK chunk, it will reply with ABORT, otherwise
it will discard it.

## Termination of an Encrypted Association {#termination-procedure}

Besides the procedures for terminating an association explained in
{{RFC9260}}, the Protection Engine SHALL ask SCTP Host for
terminating an association when having an internal error or by
detecting a security violation, using the procedure described
in {{eengine}}.  The internal design of Protection
Engines and their capability is out of the scope of the current
document.

# Encrypted Data Chunk Handling {#encrypted-data-handling}

With reference to the State Diagram as shown in
{{sctp-Crypto-state-diagram}}, the handling of Control chunks, Data
chunks and Encrypted chunks follows the rules defined below:

- When the association is in states CLOSED, COOKIE-WAIT, COOKIE-ECHOED
and CRYPT PENDING, any Control chunk is sent plain. No DATA chunks
shall be sent in these states and DATA chunks received shall be
silently discarded.

- When the association is in states ENCRYPTED and in general in a
state different than CLOSED, COOKIE-WAIT, COOKIE-ECHOED and CRYPT
PENDING, any Control chunk as well as Data chunks will be used to
create an SCTP payload that will be encrypted by the Protection Engine
and the result from that encryption will be the used as payload of an
ENCRYPT chunk that will be the only chunk of the SCTP packet to be
sent.

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
{: #sctp-Crypto-encrypt-chunk-states-1 title="SCTP packet before ENCRYPTED state" artwork-align="center"}

The diagram shown in {{sctp-Crypto-encrypt-chunk-states-1}} describes
the structure of an SCTP packet being sent or received when the
association has not reached the ENCRYPTED state yet. In this case only
Control chunks or ENCRYPT chunk can be handled.  Only one ENCRYPT
chunk can be sent in a SCTP packet.

~~~~~~~~~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         Common Header                         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         CRYPTO Chunk                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~
{: #sctp-Crypto-encrypt-chunk-states-2 title="SCTP packet after ENCRYPTED state" artwork-align="center"}

The diagram shown in {{sctp-Crypto-encrypt-chunk-states-2}} describes
the structure of an SCTP packet being sent after the ENCRYPTED state
has been reached. Suck packets are built with the SCTP common
header. Only one CRYPTO chunk can be sent in a SCTP packet.

## Encrypted Data Chunk Transmission {#data-sending}

When the association state machine (see {{sctp-Crypto-state-diagram}})
has reached the CRYPT PENDING state, it MAY handle KEY handshake
inband depending on how the specification for the chosen Protection
Engine has been defined.  In such case, the CRYPTO chunk Handler will
receive plain Control chunks from the SCTP chunk Handler and ENCRYPT
chunks from the Protection Engine.  Plain Control chunks and ENCRYPT
chunks CANNOT be bundled within the same SCTP packet.

When the association state machine (see {{sctp-Crypto-state-diagram}})
has reached the ENCRYPTED state, the CRYPTO chunk Handler will receive
Control chunks and Data chunks from the SCTP chunk Handler as a
complete SCTP Payload with maximum size limited by PMTU reduced by the
dimension of the SCTP common header and the ENCRYPT chunk header.

That plain payload will be sent to the Protection Engine in use for
that specific association, the Protection Engine will return an
encrypted payload with maximum size PMTU reduced by the dimension of
the SCTP common header and the ENCRYPT chunk header.

Depending on the specification for the chosen Protection Engine, when
forming the ENCRYPT chunk header the CRYPTO chunk Handler may set the
Flags (see {{sctp-Crypto-chunk-newchunk-crypt-struct}}).

An SCTP packet containing an SCTP ENCRYPT Chunk SHALL be delivered
without delay and SCTP bundling is NOT PERMITTED.

## Encrypted Data Chunk Reception {#data-receiving}

When the association state machine (see {{sctp-Crypto-state-diagram}})
has reached the CRYPT PENDING state, it MAY handle KEY handshake
inband depending on how the specification for the chosen Protection
Engine has been defined.  In such case, the CRYPTO chunk Handler will
receive plain Control Chunks and ENCRYPT chunks from the SCTP Header
Handler.  ENCRYPT chunks will be forwarded to the Protection Engine
whilst plain Control chunks will be forwarded to SCTP Chunk Handler.
During CRYPT PENDING state, plain Control chunks and ENCRYPT chunks
CANNOT be bundled within the same SCTP packet.

When the association state machine (see {{sctp-Crypto-state-diagram}})
has reached the ENCRYPTED state, the CRYPTO Chunk Handler will receive
ENCRYPT chunks from the SCTP Header Handler.  Payload from ENCRYPT
chunks will be forwarded to the Protection Engine in use for that
specific association for decryption, the Protection Engine will return
a plain SCTP Payload.  The Plain SCTP Payload will be forwarded to
SCTP Chunk Handler that will split it in separated chunks and will
handle them according to {{RFC9260}}.

Depending on the specification for the chosen Protection Engine, when
receiving the ENCRYPT chunk header the CRYPTO Cchunk Handler may handle
the Flags (see {{sctp-Crypto-chunk-newchunk-crypt-struct}}) according
to that specification.

Meta data belonging to the SCTP packet received SHALL be tied to the
relevant chunks and forwarded transparently to the SCTP Host.

### SCTP Header Handler

The SCTP Header Handler is responsible for correctness of the SCTP
common header, it receives the SCTP packet from the lower transport
layer, discriminates among associations and forwards the payload and
relevant data to the SCTP protection engine for handling.

In the opposite direction it creates the SCTP common header and fills
it with the relevant information for the specific association and
delivers it towards the lower transport layer.


# IANA Considerations {#IANA-Consideration}

This document defines one new registry in the Stream Control
Transmission Protocol (SCTP) Parameters group that IANA maintains for
the protection engine identifiers. It also adds registry entries into
several other registries in the Stream Control Transmission Protocol
(SCTP) Parameters group:

*  Two new SCTP Chunk Types

*  One new SCTP Chunk Parameter Type

*  Two new SCTP Error Cause Codes


## Protection Engine Identifier Registry

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


## SCTP Chunk Types

In the Stream Control Transmission Protocol (SCTP) Parameters group's
"Chunk Types" registry, IANA is requested to add the two new entries
depicted below in in {{iana-chunk-types}} with a reference to this
document. The registry at time of writing was available at:
https://www.iana.org/assignments/sctp-parameters/sctp-parameters.xhtml#sctp-parameters-1

| ID Value | Chunk Type | Reference |
| TBA5 | Crypto Chunk (CRYPTO) | RFC-To-Be |
| TBA6 | Endpoint Validation Chunk (PVALID ) | RFC-To-Be |
{: #iana-chunk-types title="New Chunk Types Registered" cols="r l l"}


## SCTP Chunk Parameter Types

In the Stream Control Transmission Protocol (SCTP) Parameters group's
"Chunk Parameter Types" registry, IANA is requested to add the new
entry depicted below in in {{iana-chunk-parameter-types}} with a
reference to this document. The registry at time of writing was
available at:
https://www.iana.org/assignments/sctp-parameters/sctp-parameters.xhtml#sctp-parameters-2

| ID Value | Chunk Parameter Type | Reference |
| TBA7 | Encrypted Association (CRYPT) | RFC-To-Be |
{: #iana-chunk-parameter-types title="New Chunk Type Parameters Registered" cols="r l l"}


## SCTP Error Cause Codes

In the Stream Control Transmission Protocol (SCTP) Parameters group's
"Error Cause Codes" registry, IANA is requested to add the three new
entries depicted below in in {{iana-error-cause-codes}} with a
reference to this document. The registry at time of writing was
available at:
https://www.iana.org/assignments/sctp-parameters/sctp-parameters.xhtml#sctp-parameters-24

| ID Value | Error Cause Codes | Reference |
| TBA8 | Protection Engine Error | RFC-To-Be |
| TBA9 | Crypto Chunk Parameter Missing | RFC-To-Be |
{: #iana-error-cause-codes title="Error Cause Codes Parameters Registered" cols="r l l"}


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
