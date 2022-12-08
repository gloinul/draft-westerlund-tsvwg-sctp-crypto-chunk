---
docname: draft-westerlund-tsvwg-sctp-crypto-chunk-latest
title: Stream Control Transmission Protocol (SCTP) Encryption Chunk
abbrev: SCTP Crypto Chunk
obsoletes:
cat: std
ipr: trust200902
wg: TSVWG
area: Transport
submissiontype: IETF  # also: "independent", "IAB", or "IRTF"


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
   RFC3758:
   RFC5061:

normative:
  RFC2119:
  RFC8174:
  RFC9260:

  IANA-SCTP-PARAMETERS:
    target: <http://www.iana.org/assignments/sctp-parameters>
    title: Stream Control Transmission Protocol (SCTP) Parameters

--- abstract

This document describes a method for adding encryption support to the
Stream Control Transmission Protocol (SCTP). SCTP Encryption Chunk is
intended to enable communications privacy for applications that use
SCTP as their transport protocol and allows applications to
communicate in a way that is designed to prevent eavesdropping and
detect tampering or message forgery.

The crypto chunk defined here in is one half of a complete
solution. Where a companion specification is required to define how
the content of the crypto chunk is encrypted, authenticated, and
protected against replay, as well as how key management is accomplished.

Applications using SCTP Encryption Chunk can use all transport
features provided by SCTP and its extensions.

--- middle

# Introduction {#introduction}

## Overview

   This document defines an encryption chunk for the Stream Control
   Transmission Protocol (SCTP), as defined in {{RFC9260}}.

   This specification defines the actual crypto chunk, how to enable
   it usage, how it interacts with the SCTP association establishment
   to enable endpoint authentication, key-establishment, and other
   features of the separate cipher specification.

   This specification is intended to be capable of enabling mutual
   authentication of endpoints, data confidentiality, data origin
   authentication, data integrity protection, and data replay
   protection for SCTP packets after SCTP association has been
   established. The exact properties will depend on the companion
   cipher specification used with the crypto chunk.

   Applications using SCTP Encryption Chunk can use all transport
   features provided by SCTP and its extensions. Due to its level of
   integration as discussed in next section it will provide its
   security functions on all content of the SCTP packet, and will thus
   not impact the potential to utilize any SCTP functionalities or
   extensions.

## Protocol Overview {#protocol-overview}

SCTP Encryption Chunk is defined as a method for secure and
confidential transfer for SCTP packets.  This is implemented inside
the SCTP protocol, in a sublayer between the SCTP Header handling and
the SCTP Chunk handling.  Once an SCTP packet has been received and
the SCTP Common Header has been validated, the SCTP Encrypted Chunk(s)
are being sent to the chosen decryption engine that will return the
plain SCTP payload containing the plain SCTP chunks, those chunks will
then be handled according to current SCTP protocol specification.

~~~~~~~~~~~ aasvg
+---------------------+
|                     |
|        ULP          |
|                     |
+---------------------+ <- User Level Messages
|                     |
| SCTP Chunks Handler |
|                     |
+-------------+-------+-----------------+ <- SCTP Plain Payload
|  Encryption |    Encryption Engine    |
|    Chunk    +-------------------------+
|   Handler   | Encryption KEY Handler  |
+-------------+-------+-----------------+ <- SCTP Encrypted Payload
|                     |
| SCTP Header Handler |
|                     |
+---------------------+

~~~~~~~~~~~
{: #sctp-encryption-chunk-layering title="SCTP Encryption Chunk layering
in regard to SCTP and upper layer protocol"}

SCTP Encryption Chunk is defined on per Association. Different Associations
within the same SCTP Endpoint may use or not the SCTP Encryption Chunk
and different Associations exploiting SCTP Encryption Chunks may use
different Encryption Engines.

On the outgoing direction, once SCTP stack has created the plain SCTP
payload containing Control and/or Data chunks, that payload will be
sent to the Encryption Engine to be protected and the encrypted and
integrity tagged data will be encapsulated in a SCTP Encrypted chunk.

SCTP Encryption Engine performs protection operations on the whole
SCTP plain packet payload, i.e., all chunks after the SCTP common
header. Information protection is kept during the lifetime of the
Association and no information is sent in plain except than the
initial SCTP handshake, the SCTP common Header and the SCTP Encrypted
Chunk header.

SCTP Encryption Chunk capability is agreed by the peers at the initialization
of the SCTP Association, during that phase the peers exchange information
about the encryption engine availability. Once the peers have agreed on what
encryption to use, the SCTP hosts start sending SCTP Encrypted chunks
containing the initialization information related to the encryption engine
including the endpoint validation. This is depending on the chosen engine
thus is not being detailed in the current specification.

When the endpoint validation has been completed, the Association is meant
to be initialized and the ULP is informed about that, from this time on
it's possible for the ULPs to exchange data.

SCTP Encrypted Chunks will never be retransmitted, retransmission
is implemented by SCTP host at chunk level as in the legacy.
Duplicated SCTP Encrypted Chunks, whenever they will be accepted
by the encryption engine, will result in duplicated SCTP
chunks and will be handled as duplicated chunks by SCTP host.

Besides the legacy methods for Association termination, furthermore
it may be that the encryption engine goes in troubles so that it
doesn't guarantee security and requires to terminate the link,
in this case it should require the Association to be aborted.

## Encryption Engines Considerations {#engines}

The Encryption Engine, independently from the security characteristics,
needs to be capable working on an unreliable transport mechanism
same as UDP and have own KEY handler capability.

SCTP Crypto Chunk directly exploits the Encryption Engine by
requesting encryption and decryption of a buffer, in particular
the encrypted buffer shall never exceed the SCTP payload size
thus Encryption Engine shall be aware of the PMTU (see {{pmtu}}).

KEY Handling of Encryption Engine SHOULD exploit SCTP Crypto Chunk
for handshaking, in that case any packet being exchanged between
Encryption Engine peers shall be transported as payload of Encrypted
chunk (see {{encrypt}}).

KEY Handling MAY use other mechanism than what provided by SCTP
Crypto Chunks, in any case the mechanism for KEY Handling MUST
be specified in the Encryption Engine specification document for that
specific Encryption Engine.

Out-of-band communication between Encryption Engines MAY exploit
the Flags byte provided by the ENCRYPT chunk header
(see {{sctp-encryption-chunk-newchunk-crypt-struct}}).

Details of the use of Flags, if different from what described
in the current document, MUST be specified in the Encryption
Engine specification document for that specific Encryption Engine.

An example of Encryption Engine can be DTLS.

## SCTP Encryption Chunk Buffering and Flow Control {#buffering}

Encryption Engine and SCTP are asynchronous, meaning that the
Encryption Engine may deliver the decrypted SCTP Payload to
the SCTP Host without respecting the reception order.
It's up to SCTP Host to reorder the chunks in the reception
buffer and to take care of the flow control according to
what specified in {{RFC9260}}. From SCTP perspective the
Encryption Chunk is part of the transport network.

Even though the above allows the implementors to adopt a multithreading
design of the Encryption Engines, the actual implementation should
consider that out-of-order handling of SCTP chunks is not desired
and may cause false congestions and retransmissions.

## PMTU Considerations {#pmtu}

The addition of the Encryption to SCTP reduces the room for payload,
in order to cope with that when creating the payload of SCTP for
encryption the size of the Encrypted chunk header and plain text
expansion due to algorithm and any authentication tag needs to be
included in the calculation.

On the other hand, the Encryption engine needs to be informed about
the PMTU by removing from the value the sum of the common SCTP header
and the Encrypted chunk header.

From SCTP perspective, the maximum size of the encryption engine
payload, if limited, has to be considered as well. If such limit
exists, PMTU for SCTP has to be limited to the encryption engine
largest payload value plus the SCTP Common Header.

# Conventions


   The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
   "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
   "OPTIONAL" in this document are to be interpreted as described in
   BCP 14 {{RFC2119}} {{RFC8174}} when, and only when, they appear in all
   capitals, as shown here.

# New Parameter Type

This section defines the new parameter type that will be used to
negotiate the use of Encrypted Chunks during association setup.
{{sctp-encryption-chunk-init-parameter}} illustrates the new parameter type.

~~~~~~~~~~~ aasvg
+----------------+------------------------------------------------+
| Parameter Type | Parameter Name                                 |
+----------------+------------------------------------------------+
| 0x80xx         | Encrypted Association (CRYPT)                  |
+----------------+------------------------------------------------+
~~~~~~~~~~~
{: #sctp-encryption-chunk-init-parameter title="New INIT Parameter"}

Note that the parameter format requires the receiver to ignore the
parameter and continue processing if the parameter is not understood.
This is accomplished (as described in {{RFC9260}}, Section 3.2.1.)
by the use of the upper bits of the parameter type.

## Encrypted Association Parameter (CRYPT) {#crypt-parameter}

This parameter is used to carry the list of proposed Encryption Engines
and the chosen Encryption Engine during INIT/INIT-ACK handshake.

~~~~~~~~~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|     Parameter Type = 0x80xx   |       Parameter Length        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|        Encryption Engines                                     |
|                               +-------------------------------+
|                               |            padding            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~
{: #sctp-encryption-chunk-init-options title="CRYPT Options"}

   Type: 1 byte (unsigned integer)
      This value MUST be set to 0x80xx.

   Length: 2 bytes (unsigned integer)
      This value holds the length of the Crypto Engines in bytes plus 8.

   Encryption Engines: n words (unsigned integer)
      This holds the list of Encryption engines in order of preference.
      Each Encryption engine is specified by a 16bit word.

   Padding: 0, 1, 2, or 3 bytes (unsigned integer)
      If the length of the Crypto Engines is not a multiple of 4 bytes, the sender
      MUST pad the chunk with all zero bytes to make the chunk 32-bit
      aligned.  The Padding MUST NOT be longer than 3 bytes and it MUST
      be ignored by the receiver.

# New Chunk Type

##  Encrypted Chunk (ENCRYPT) {#encrypt}

This section defines the new chunk types that will be used to
transport encrypted SCTP payload.
{{sctp-encryption-chunk-newchunk-crypt}} illustrates the new chunk type.

~~~~~~~~~~~ aasvg
+------------+-----------------------------+
| Chunk Type | Chunk Name                  |
+------------+-----------------------------+
| 0x0x       |  Encrypted Chunk (ENCRYPT)  |
+------------+-----------------------------+
~~~~~~~~~~~
{: #sctp-encryption-chunk-newchunk-crypt title="ENCRYPT Chunk Type"}

It should be noted that the ENCRYPT-chunk format requires the receiver
to ignore the chunk if it is not understood and silently discard all
chunks that follow.  This is accomplished (as described in {{RFC9260}}
Section 3.2.) by the use of the upper bits of the chunk type.

This chunk is used to hold the encrypted payload of a plain SCTP packet.

~~~~~~~~~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| Type = 0x0x   |   Flags=0     |             Length            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
\                            Payload                            /
/                                                               \
|                               +-------------------------------+
|                               |           Padding             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~
{: #sctp-encryption-chunk-newchunk-crypt-struct title="ENCRYPT Chunk Structure"}

   Type: 1 byte (unsigned integer)
      This value MUST be set to 0x0x for all ENCRYPT-chunks.

   Flags: 1 byte (unsigned integer)
      This is used by the Encryption Engine and ignored by SCTP.

   Length: 2 bytes (unsigned integer)
      This value holds the length of the Payload in bytes plus 8.

   Payload: n bytes (unsigned integer)
      This holds the encrypted data.

   Padding: 0, 1, 2, or 3 bytes (unsigned integer)
      If the length of the Payload is not a multiple of 4 bytes, the sender
      MUST pad the chunk with all zero bytes to make the chunk 32-bit
      aligned.  The Padding MUST NOT be longer than 3 bytes and it MUST
      be ignored by the receiver.

##  Encryption Validation Chunk (EVALID)

This section defines the new chunk types that will be used to validate
the negotiation of the encryption engine selected for Encryption
Chunk.  {{sctp-encryption-chunk-newchunk-EVALID}} illustrates the new
chunk type.

~~~~~~~~~~~ aasvg
+------------+-----------------------------------+
| Chunk Type | Chunk Name                        |
+------------+-----------------------------------+
| 0x0x       |  INIT Option Validation (EVALID)  |
+------------+-----------------------------------+
~~~~~~~~~~~
{: #sctp-encryption-chunk-newchunk-EVALID title="EVALID Chunk Type"}

It should be noted that the EVALID-chunk format requires the receiver
to ignore the chunk if it is not understood and silently discard all
chunks that follow.  This is accomplished (as described in {{RFC9260}}
Section 3.2.) by the use of the upper bits of the chunk type.

This chunk is used to hold the Encryption engines list.

~~~~~~~~~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| Type = 0x0x   |   Flags=0     |             Length            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                      Encryption Engines                       |
\                                                               /
/                                                               \
|                               +-------------------------------+
|                               |           Padding             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~
{: #sctp-encryption-chunk-newchunk-EVALID-struct title="EVALID Chunk Structure"}

  Type: 1 byte (unsigned integer)
      This value MUST be set to 0x80xx.

   Flags: 1 byte (unsigned integer)
      SHOULD be set to zero on transmit and MUST be ignored on receipt.

   Length: 2 bytes (unsigned integer)
      This value holds the length of the Crypto Engines in bytes plus 8.

   Encryption Engines: n words (unsigned integer)
      This holds the list of Encryption engines in order of preference.
      Each Encryption engine is specified by a 16bit word.

   Padding: 0, 1, 2, or 3 bytes (unsigned integer)
      If the length of the Crypto Engines is not a multiple of 4 bytes, the sender
      MUST pad the chunk with all zero bytes to make the chunk 32-bit
      aligned.  The Padding MUST NOT be longer than 3 bytes and it MUST
      be ignored by the receiver.

# Error Handling {#error_handling}

There are addition reasons for the Association to misbehave, this specification
introduces a new set of causes that are to be used when SCTP Host detects
a faulty condition. The special case is when the error is detected by the
Encryption Engine that may provide additional information.

## Mandatory CRYPT option missing {#enocrypt}

When an SCTP host will receive an INIT chunk that doesn't contain the
CRYPT option towards an SCTP Endpoint that only accepts SCTP Crypto
protected Association, it will reply with an ERROR chunk containing
the informations 7 Invalid Mandatory Parameter (see {{RFC9260}}  section 3.3.10.7) and ENOCRYPT
(specified in {{sctp-encryption-new-error-causes}}).

## Error during KEY handshake {#ekeyhandshake}

If the encryption engine specifies that KEY handling is implemented inband
it may happen that the procedure has errors. In such case an ERROR chunk
will be sent with EENGINE cause (specified in {{sctp-encryption-new-error-causes}}).
It MAY be followed by an appropriate cause
according to the Encryption Engine specification.

## Error in Crypto Engines validation {#evalidate}

Whenever an error occur in Crypto Engine Validation (see {encrypted-state}),
SCTP Host will send an ERROR chunk with EVALIDATE cause
(specified in {{sctp-encryption-new-error-causes}}).

## Timeout during KEY handshake or validation {#etmovalidate}

Whenever a T-valit timeout occurs, the SCTP Host will send an ERROR
chunk with ETMOVALIDATE cause (specified in {{sctp-encryption-new-error-causes}}).

## Error from Encryption Engine {#eengine}

Encryption Engine MAY inform SCTP Host about errors,
in such cases an ERROR chunk sill be sent with EENGINE cause
(specified in {{sctp-encryption-new-error-causes}}).
It MAY be followed by an appropriate cause
according to the Encryption Engine specification.

## Non critical error in the Encryption Engine

A non critical error in the Encryption Engine means that the
Ecryption Engine is capable of recoverying without the need
of the whole Association to be restarted.

From SCTP perspective, a non critical error will be perceived
as a temporary problem in the transport and will be handled
with retransmissions and SACKS according to {{RFC9260}}.

When the Encryption Engine will experience a non critical error,
no ERROR chunks shall be sent.

## New Error Causes {#new_errors}

This section lists the new error cause that are to be used.
They require additional lines of the "CAUSE CODES" table in
SCTP-parameters {{IANA-SCTP-PARAMETERS}}:

~~~~~~~~~~~ aasvg
   VALUE            CAUSE CODE                               REFERENCE
  ------           -----------------                        ----------
   xxx (0xxxxx)     ENOCRYPT                                 nnnn
   xxx (0xxxxx)     EENGINE                                  nnnn
   xxx (0xxxxx)     EVALIDATE                                nnnn
   xxx (0xxxxx)     ETMOVALIDATE                             nnnn
~~~~~~~~~~~
{: #sctp-encryption-new-error-causes title="New Error Causes"}

# Encrypted SCTP State Diagram

The {{sctp-encryption-state-diagram}} shows the changes versus the SCTP Association state
machine as described in {{RFC9260}} section 4.

~~~~~~~~~~~ aasvg
                        -----          -------- (from any state)
                      /       \      /receive ABORT      [ABORT]
        receive INIT |         |    |--------------  or ----------
---------------------|         v    v    delete TCB     send ABORT
generate State Cookie \    +---------+                  delete TCB
        send INIT ACK   ---|  CLOSED |
                           +---------+
                             /      \
                            /        \  [ASSOCIATE]
                           |          |-----------------
                           |          | create TCB
                           |          | send INIT
          receive valid    |          | start T1-init timer
          COOKIE  ECHO     |          v
      (1) -----------------|    +-----------+
          create TCB       |    |COOKIE-WAIT| (2)
          send COOKIE ACK  |    +-----------+
                           |          |
                           |          | receive INIT ACK
                           |          |-------------------
                           |          | send COOKIE ECHO
                           |          | stop T1-init timer
                           |          | start T1-cookie timer
                           |          v
                           |   +-------------+
                           |   |COOKIE-ECHOED| (3)
                           |   +-------------+
                           |          |
                           |          | receive COOKIE ACK
                           |          |-------------------
                           |          | stop T1-cookie timer
         +-----------------+-----+    |
         |     +---------------- | ---+-----+
         |     |                 |          |
         |     |                 v          v
         |     |              +-----------------+
         |     |              |  CRYPT PENDING  | If INIT/INIT-ACK
         |     |              +-----------------+ has CRYPT option
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
         |     |              +-----------------+
         |     |                       |
         |     |                       | [ENDPOINT VALIDATION]
         |     |                       |------------------------
         |     |                       | send and receive
         |     |                       | EVALID by means of
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
{: #sctp-encryption-state-diagram title="SCTP State Diagram with Encryption"}

## New states

This section describes details on the amendment to the SCTP Association Establishment
state machine.

### CRYPT PENDING {#crypt-pending-state}

The presence of CRYPT option in INIT or INIT-ACK chunk makes the State Machine
entering CRYPT PENDING state instead of ESTABLISHED.

When entering CRYPT PENDING state, a T-valid timer is started that will
cover the whole validation time including the inband KEY handling.
It's up to the implementor to take care of the value for the timer
also related to the time needed for KEY handshake of each Encryption
Engine.

If KEY handling is inband, the Encryption Engine will start the handshake
with its peer and in case of failure or T-validation timeout,
it will generate an ERROR chunk and an ABORT chunk.
The ERROR handling follows what specified in {{ekeyhandshake}}.
When Handshake has been successfully completed, the Association state machine
will enter ENCRYPTED state.

If KEY handling is out-of-band, after starting T-valid timer the SCTP
Association will enter ENCRYPTED state.

### ENCRYPTED {#encrypted-state}

The Association state machine can only reach ENCRYPTED state from CRYPT PENDING
state (see {{crypt-pending-state}}). When entering into ENCRYPTED state
the T-valid timer is running and the Encryption Engine has completed the
KEY handshake so that encrypted data can be sent to the peer.

From this time on, only ENCRYPT chunks can be sent to the remote peer and
any other type of chunks coming from the remote peer will be silently discarded.

In ENCRYPTED state the SCTP Endpoints MUST validate the INIT/INIT-ACK parameters,
thus the Client will send an EVALID chunk that will contain exactly the same list
as Crypto Engines as previously sent in CRYPT option of INIT chunk and in the same order.

When the Server will receive EVALID, it will compare the list of Crypto Engines
with the list received in the INIT chunk, if they are identical it will reply
to the Client with an EVALID chunk containing the Crypto Engine previously
sent as CRYPT option in INIT-ACK chunk , it will clean the T-valid timer and
will move into ESTABLISHED state.
If the lists of Crypto Engines don't match, it will generate an ERROR chunk
and an ABORT chunk. ERROR CAUSE will indicate EVALIDATE meaning that an error
has been happening during VALIDATION of SCTP Endpoints.

After sending EVALID, the Client will wait for the Server to reply with the
EVALID confirmation. The Client will compare the Crypto Engine received from
the Server, if the value is the same it will clean the T-valid timer and
move into ESTABLISHED state.
If the chosen Crypto Engines don't match, it will generate an ERROR chunk
and an ABORT chunk. ERROR CAUSE will indicate EVALIDATE meaning that an error
has been happening during VALIDATION of SCTP Endpoints.

If T-valid timer expires either at Client or Server, it will generate an ERROR chunk
and an ABORT chunk.
The ERROR handling follows what specified in {{etmovalidate}}.


# Procedures

## Establishment of an Encrypted Association

An SCTP Endpoint acting as Client willing to create an Encrypted Association shall send
to the remote peer an INIT chunk containing the CRYPT parameter
(see {{sctp-encryption-chunk-newchunk-crypt}}) where the Crypto Engines
lists all the supported encryption engines, given in order of preference
(see {{sctp-encryption-chunk-init-options}}).

As alternative, an SCTP Endpoint acting as Server willing to support only Encrypted
Associations shall consider INIT chunk not containing the CRYPT parameter as an error,
thus it will reply with an ERROR chunk according to what specified in {{enocrypt}}
indicating that the mandatory CRYPT option is missing.

An SCTP Endpoint acting as Server, when receiving an INIT chunk with CRYPT parameter,
will search the list of Crypto Engines for a common choice and will reply with
INIT-ACK containing the CRYPT parameter with the chosen Crypto Engine. When the
Server cannot find a supported Crypto Engine, it will silently discard the INIT chunk.

When Client and Server have agreed on an Encrypted Association by means of handshaking
INIT/INIT-ACK with a common Encryption Engine, only Control Chunks and Encrypted Chunks
will be accepted. Any Data chunk being sent on an Encrypted Association will be silently
discarded.

After completion of initial handshake, that is after COOKIE-ECHO and COOKIE-ACK,
the Encryption Engine shall initialize itself by transferring its own data as Payload
of the ENCRYPT chunk (see {{sctp-encryption-chunk-newchunk-crypt-struct}}).
At completion of Encryption Engine initialization, the setup of the Encrypted
Association is complete and from that time on only ENCRYPT chunks will be exchanged.
Any other type of chunks will be silently discarded.

After completion of Encrypted Association initialization, the Client SHOULD send
to the Server an EVALID Chunk (see {{sctp-encryption-chunk-newchunk-EVALID}})
containing the list of Encryption Engines previously sent in the CRYPT parameter
of the INIT chunk. The Server receiving the EVALID chunk will compare the Encryption
Engines list with the one previously received in the INIT chunk, if they will be
exactly the same, with the same engine in the same position, it will reply to the
Client with an EVALID chunk containing the chose Encryption Engine, otherwise it
will reply with an ABORT chunk.
When the Client will receive the EVALID chunk, it will compare with the previous
chosen Encryption Engine and in case of mismatch with the one received previously
as CRYPT parameter in the INIT-ACK chunk, it will reply with ABORT, otherwise
it will discard it.

## Termination of an Encrypted Association

Besides the procedures for terminating an Association explained in {{RFC9260}},
the Encryption Engine SHOULD ask SCTP Host for terminating an Association
when having an internal error or by detecting a security violation.
The internal design of Encryption Engines and their capability is out of the
scope of the current document.

## Encrypted Data Chunk Handling

With reference to the State Diagram as shown in {{sctp-encryption-state-diagram}},
the handling of Control Chunks, Data Chunks and Encrypted chunks follows the
rules defined below:

- When the Association is in states CLOSED, COOKIE-WAIT, COOKIE-ECHOED and CRYPT PENDING,
any Control Chunk is sent plain. No DATA chunks shall be sent in these states and DATA
chunks received shall be silently discarded.

- When the Association is in states ENCRYPTED and in general in a state different
than CLOSED, COOKIE-WAIT, COOKIE-ECHOED and CRYPT PENDING, any Control Chunk as well as Data
chunks will be used for creating an SCTP payload that will be encrypted
by the Encryption Engine and the result from that encryption will be the
used as payload of an ENCRYPT chunk that will be the only chunk of the
SCTP packet to be sent.

~~~~~~~~~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         Common Header                         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           Chunk #1                            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                              ...                              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           Chunk #n                            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~
{: #sctp-encryption-encrypt-chunk-states-1 title="SCTP packet before ENCRYPTED state"}

The diagram shown in {{sctp-encryption-encrypt-chunk-states-1}} describes
the structure of an SCTP packet being sent or received when the Association
has not reached the ENCRYPTED state yet. In this case only Control
Chunks or ENCRYPT chunk can be handled.
Only one ENCRYPT chunk can be sent in a SCTP packet.

~~~~~~~~~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         Common Header                         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         ENCRYPT chunk                         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~
{: #sctp-encryption-encrypt-chunk-states-2 title="SCTP packet after ENCRYPTED state"}

The diagram shown in {{sctp-encryption-encrypt-chunk-states-2}} describes
the structure of an SCTP packet being sent after the ENCRYPTED state has been
reached. Suck packets are built with the SCTP Common header. Only one
ENCRYPT chunk can be sent in a SCTP packet.

### Encrypted Data Chunk Transmission

When the Association state machine (see {{sctp-encryption-state-diagram}}) has
reached the CRYPT PENDING state, it MAY handle KEY handshake inband depending
on how the specification for the chosen Encryption Engine has been defined.
In such case, the Encryption Chunk Handler will receive plain Control Chunks
from the SCTP Chunk Handler and ENCRYPT chunks from the Encryption Engine.
Plain Control chunks and ENCRYPT chunks CANNOT be bundled within the same SCTP packet.

When the Association state machine (see {{sctp-encryption-state-diagram}}) has
reached the ENCRYPTED state, the Encryption Chunk Handler will
receive Control Chunks and Data chunks from the SCTP Chunk Handler as
a complete SCTP Payload with maximum size limited by PMTU reduced
by the dimension of the SCTP Common Header and the ENCRYPT chunk
header.

That plain payload will be sent to the Encryption Engine in use for
that specific Association, the Encryption Engine will return an
encrypted payload with maximum size PMTU reduced
by the dimension of the SCTP Common Header and the ENCRYPT chunk
header.

Depending on the specification for the chosen Encryption Engine,
when forming the ENCRYPT chunk header the Encryption Chunk Handler
may set the Flags (see {{sctp-encryption-chunk-newchunk-crypt-struct}}).

### Encrypted Data Chunk Reception

When the Association state machine (see {{sctp-encryption-state-diagram}}) has
reached the CRYPT PENDING state, it MAY handle KEY handshake inband depending
on how the specification for the chosen Encryption Engine has been defined.
In such case, the Encryption Chunk Handler will receive plain Control Chunks
and ENCRYPT chunks from the SCTP Header Handler.
ENCRYPT chunks will be forwarded to the Encryption Engine whilst plain
Control chunks will be forwarded to SCTP Chunk Handler.
During CRYPT PENDING state, plain Control chunks and ENCRYPT chunks CANNOT
be bundled within the same SCTP packet.

When the Association state machine (see {{sctp-encryption-state-diagram}}) has
reached the ENCRYPTED state, the Encryption Chunk Handler will
receive ENCRYPT Chunks from the SCTP Header Handler.
Payload from ENCRYPT Chunks will be forwarded to the Encryption Engine
in use for that specific Association
for decryption, the Encryption Engine will return a plain SCTP Payload.
The Plain SCTP Payload will be forwarded to SCTP Chunk Handler that
will split it in separated chunks and will handle them according
to {{RFC9260}}.

Depending on the specification for the chosen Encryption Engine,
when receiving the ENCRYPT chunk header the Encryption Chunk Handler
may handle the Flags (see {{sctp-encryption-chunk-newchunk-crypt-struct}})
according to that specification.

### SCTP Header Handler

The SCTP Header Handler is responsible for correctness of the SCTP Header,
it receives the SCTP Packet from the lower transport layer,
discriminates among Associations and forwards the Payload and relevant
data to the SCTP Encrypt Handler for handling.

in the opposite direction it creates the SCTP Header and fills it with
the relevant information for the specific Association and delivers it towards
the lower transport layer.

# IANA Considerations {#IANA-Consideration}

   This document defines four registries that IANA maintains:

   *  through definition of additional chunk types,

   *  through definition of additional chunk flags,

   *  through definition of additional parameter types,

   *  through definition of additional cause codes within ERROR chunks,

   IANA needs to perform the following updates for the above five
   registries:

   *  In the "Chunk Types" registry, IANA has to add  with a reference to this
      document.

      -  Encrypted Chunk (ENCRYPT)
      -  Endpoint Authentication Chunk (EVALID)


   *  In the "Chunk Parameter Types" registry, IANA has to add  with a reference to this
      document.

      - Encrypted Association (CRYPT)

   *  In the "Chunk Flags" registry, IANA has to add  with a reference to this
      document.

      - To Be defined in the Encryption Engine specification documents

   *  In the "Error Cause Codes" registry, IANA has to add  with a reference to this
      document.

      - Error in encryption engine EENGINE
      - Error in Encryption Chunk Endpoint Validation EVALIDATE
      - Timeout during Encryption Chunk Validation ETMOVALIDATE

## Downgrade Attacks {#Downgrade-Attacks}

SCTP Encrypted Chunks provides a mechanism for preventing downgrade attacks
that detects downgrading attempts and terminates the Association.

The Encryption Engine initial handshake is verified before the Association
is set as ESTABLISHED, thus no user data are sent before validation.
