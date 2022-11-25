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
  RFC9260:


--- abstract

This document describes a method for adding encryption support to the
Stream Control Transmission Protocol (SCTP). SCTP Encryption Chunk is
intended to enable communications privacy for applications that use
SCTP as their transport protocol and allows applications to
communicate in a way that is designed to prevent eavesdropping and
detect tampering or message forgery.

The crypto chunk defined here in is one half of a complete
solution. Where a companion specification is required to define how
the content of the crypto chunk is encrypted, authenticated and
protected against replay, as well as how keymanagement is accomplised.

Applications using SCTP Encryption Chunk can use all transport
features provided by SCTP and its extensions.

--- middle

# Introduction {#introduction}

## Overview

   This document defines a encryption chunk for the Stream Control
   Transmission Protocol (SCTP), as defined in {{RFC9260}}.

   This specification defines the actual crypto chunk, how to enable
   it usage, how it interact with the SCTP association establishment
   to enable endpoint authentication, key-establishment and other
   features of the seperate cipher specfication.

   This specification is intended to be capable of enabling mutual
   authentication of endpoints, data confidentiality, data origin
   authentication, data integrity protection, and data replay
   protection for SCTP packets after SCTP association has been
   established. The exact properties will depend on the companion
   cipher specification used with the crypto chunk.

   Applications using SCTP Encryption Chunk can use all transport
   features provided by SCTP and its extensions. Due to its level of
   intergration as discussed in next section it will provide its
   security functions on all content of the SCTP packet, and will thus
   not impact the potential to utilize any SCTP functionalities or
   extensions.

## Protocol Overview

SCTP Encryption Chunk is defined as a method for secure and
confidential transfer for SCTP packets.  This is implemented inside
the SCTP protocol, in a sublayer between the SCTP Header handling and
the SCTP Chunk handling.  Once an SCTP packet has been received and
the SCTP Common Header has been validated, the SCTP Encrypted Chunk(s)
are being sent to the chosen decryption engine that will return the
plain SCTP payload containing the plain SCTP chunks, those chunks will
then be handled according to current SCTP protocol specification.

On the outgoing direction, once SCTP stack has created the plain SCTP
payload containing Control and/or Data chunks, that payload will be
sent to the Encryption Engine to be protected and the encrypted and
integrity tagged data will be encapsulated in a SCTP Encrypted chunk.

~~~~~~~~~~~ aasvg
+---------------------+
|                     |
|        ULP          |
|                     |
+---------------------+ <- User Level Messages
|                     |
| SCTP Chunks Handler |
|                     |
+---------------------+ <- SCTP Plain Payload
|                     |
|  Encryption Engine  |
|                     |
+---------------------+ <- SCTP Encrypted Payload
|                     |
| SCTP Header Handler |
|                     |
+---------------------+

~~~~~~~~~~~
{: #sctp-encryption-chunk-layering title="SCTP Encryption Chunk layering
in regard to SCTP and upper layer protocol"}

SCTP Encryption Engine performs protection operations on the whole
SCTP plain packet payload, i.e. all chunks after the SCTP common
header. Information protection is kept during the lifetime of the
Association and no information is sent in plain except than the
initial SCTP handshake, the SCTP common Header and the SCTP Encrypted
Chunk header.

SCTP Encryption Chunk capability is agreed by the peers at the initialization
of the SCTP Association, during that phase the peers exhange information
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

## PMTU considerations {#pmtu}

The addition of the Encryption to SCTP reduces the room for payload,
in order to cope with that when creating the payload of SCTP for
encryption the size of the Encrypted chunk header and plain text
expansion due to algorithm and any authentication tag needs to be
included in the calculation.

On the other hand, the Encryption engine needs to be informed about
the PMTU by removing from the value the sum of the common SCTP header
and the Encrypted chunk header.

# Conventions
The keywords "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
   "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
   "OPTIONAL", when they appear in this document, are to be interpreted
   as described in {{RFC2119}}.

# New Parameter Types

   This section defines the new parameter types that will be used to
   negotiate the authentication during association setup.  Table 1
   illustrates the new parameter types.

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

## Encrypted Association (CRYPT)

   This parameter is used to carry a random number of an arbitrary
   length.

~~~~~~~~~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|     Parameter Type = 0x80xx   |       Parameter Length        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|        Crypto Engines                                         |
|                               +-------------------------------+
|                               |            padding            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~
{: #sctp-encryption-chunk-init-options title="CRYPT Options"}

   Type: 1 byte (unsigned integer)
      This value MUST be set to 0x80xx.

   Length: 2 bytes (unsigned integer)
      This value holds the length of the Crypto Engines in bytes plus 8.

   Crypto Engines: n bytes (unsigned integer)
      This holds the list of crypto engines in order of preference.

   Padding: 0, 1, 2, or 3 bytes (unsigned integer)
      If the length of the Crypto Engines is not a multiple of 4 bytes, the sender
      MUST pad the chunk with all zero bytes to make the chunk 32-bit
      aligned.  The Padding MUST NOT be longer than 3 bytes and it MUST
      be ignored by the receiver.

# New Chunk Type

   This section defines the new chunk types that will be used to
   authenticate chunks.  Table 4 illustrates the new chunk type.

##  Encrypted Chunk (ENCRYPT)

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

##  Endpoint Authentication Chunk (EAUTH)

~~~~~~~~~~~ aasvg
+------------+-----------------------------------+
| Chunk Type | Chunk Name                        |
+------------+-----------------------------------+
| 0x0x       |  Endpoint Authentication (EAUTH)  |
+------------+-----------------------------------+
~~~~~~~~~~~
{: #sctp-encryption-chunk-newchunk-eauth title="EAUTH Chunk Type"}

   It should be noted that the EAUTH-chunk format requires the receiver
   to ignore the chunk if it is not understood and silently discard all
   chunks that follow.  This is accomplished (as described in {{RFC9260}}
   Section 3.2.) by the use of the upper bits of the chunk type.

   This chunk is used to hold the crypto engines list.

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
{: #sctp-encryption-chunk-newchunk-eauth-struct title="EAUTH Chunk Structure"}

  Type: 1 byte (unsigned integer)
      This value MUST be set to 0x80xx.

   Flags: 1 byte (unsigned integer)
      SHOULD be set to zero on transmit and MUST be ignored on receipt..

   Length: 2 bytes (unsigned integer)
      This value holds the length of the Crypto Engines in bytes plus 8.

   Crypto Engines: n bytes (unsigned integer)
      This holds the list of crypto engines in order of preference.

   Padding: 0, 1, 2, or 3 bytes (unsigned integer)
      If the length of the Crypto Engines is not a multiple of 4 bytes, the sender
      MUST pad the chunk with all zero bytes to make the chunk 32-bit
      aligned.  The Padding MUST NOT be longer than 3 bytes and it MUST
      be ignored by the receiver.

# Procedures

## Establishment of an Encrypted Association

An SCTP Endpoint acting as Client willing to create an Encrypted Association shall send
to the remote peer an INIT chunk containing the CRYPT parameter
(see {{sctp-encryption-chunk-newchunk-crypt}}) where the Crypto Engines
lists all the supported encryption engines, given in order of preference
(see {{sctp-encryption-chunk-init-options}}).

As alternative, an SCTP Endpoint acting as Server willing to support only Encrypted
Associations shall ignore any INIT chunk not containing the CRYPT parameter.

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
to the Server an EAUTH Chunk (see {{sctp-encryption-chunk-newchunk-eauth}})
containing the list of Encryption Engines previously sent in the CRYPT parameter
of the INIT chunk. The Server receiving the EAUTH chunk will compare the Encryption
Engines list with the one previously received in the INIT chunk, if they will be
exactly the same, with the same engine in the same position, it will reply to the
Client with an EAUTH chunk containing the chose Encryption Engine, otherwise it
will reply with an ABORT chunk.
When the Client will receive the EAUTH chunk, it will compare with the previous
chosen Encryption Engine and in case of mismatch with the one received previously
as CRYPT parameter in the INIT-ACK chunk, it will reply with ABORT, otherwise
it will discard it.

## Termination of an Encrypted Association

Besides the procedures for terminating an Association explained in {{RFC9260}},
the Encryption Engine SHOULD ask SCTP Host for terminating an Association
when having an internal error or by detecting a security violation.
The internal design of Encryption Engines and their capability is out of the
scope of the current document.

## Encrypted SCTP State Diagram

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
            |     |              |  CRYPT PENDING  | This state is
            |     |              +-----------------+ reached when
            |     |                       |          INIT/INIT-ACK has
            |     |                       |          CRYPT option
            |     |                       |
            |     |                       | [CRYPTO SETUP]
            |     |                       |-----------------
            |     |                       | send and receive
            |     |                       | encrypt engine hanshake
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
            |     |                       | EAUTH by means of
            |     |                       | ENCRYPT chunk
            |     |                       | in case of error
            |     |                       | send plain ABORT
            |     |                       v
            |     |               +---------------+
            |     |               |   VALIDATED   | From here on only
            |     |               +---------------+ ENCRYPT chunks
            |     |                       |         are sent as SCTP
            |     |                       |         payload and chunks
            |     +-----------------+     |         other than ENCRYPT
            +-----------------+     |     |         are silently
                              |     |     |         discarded
                              v     v     v
                            +---------------+
                            |  ESTABLISHED  |
                            +---------------+
                                    |
                                    |
                           /--------+--------\
       [SHUTDOWN]         /                   \
       -------------------|                   |
       check outstanding  |                   |
       DATA chunks        |                   |
                          v                   |
                 +----------------+           |
                 |SHUTDOWN-PENDING|           | receive SHUTDOWN
                 +----------------+           |------------------
                                              | check outstanding
                          |                   | DATA chunks
   No more outstanding    |                   |
   -----------------------|                   |
   send SHUTDOWN          |                   |
   start T2-shutdown timer|                   |
                          v                   v
                   +-------------+   +-----------------+
               (4) |SHUTDOWN-SENT|   |SHUTDOWN-RECEIVED| (5,6)
                   +-------------+   +-----------------+
                          |  \                |
   receive SHUTDOWN ACK   |   \               |
   -----------------------|    \              |
   stop T2-shutdown timer |     \             |
   send SHUTDOWN COMPLETE |      \            |
   delete TCB             |       \           |
                          |        \          | No more outstanding
                          |         \         |--------------------
                          |          \        | send SHUTDOWN ACK
   receive SHUTDOWN      -|-          \       | start T2-shutdown timer
   --------------------/  | \----------\      |
   send SHUTDOWN ACK      |             \     |
   start T2-shutdown timer|              \    |
                          |               \   |
                          |                |  |
                          |                v  v
                          |          +-----------------+
                          |          |SHUTDOWN-ACK-SENT| (7)
                          |          +-----------------+
                          |                   | (A)
                          |                   |receive SHUTDOWN COMPLETE
                          |                   |-------------------------
                          |                   | stop T2-shutdown timer
                          |                   | delete TCB
                          |                   |
                          |                   | (B)
                          |                   | receive SHUTDOWN ACK
                          |                   |-----------------------
                          |                   | stop T2-shutdown timer
                          |                   | send SHUTDOWN COMPLETE
                          |                   | delete TCB
                          |                   |
                          \    +---------+    /
                           \-->| CLOSED  |<--/
                               +---------+
~~~~~~~~~~~
{: #sctp-encryption-state-diagram title="SCTP State Diagram with Encryption"}


## Encrypted Data Chunk handling

With reference to the State Diagram as shown in {{sctp-encryption-state-diagram}},
the handling of Control Chunks, Data Chunks and Encrypted chunks follows the
rules defined belows:

- When the Association is in states CLOSED, COOKIE-WAIT, COOKIE-ECHOED and CRYPT PENDING,
any Control Chunk is sent plain. No DATA chunks shall be sent in these states and DATA
chunks received shall be silently discarded.

- ENCRYPT chunks MAY be bundled with COOKIE-ECHO and COOKIE-ACK

- When the Association is in states ENCRYPTED, ESTABLISHED, SHUTDOWN-PENDING, SHUTDOWN-SENT,
SHUTDOWN-RECEIVED, SHUTDOWN-ACK-SENT any Control Chunk as well as Data chunks will be
used for creating an SCTP payload that will be encrypted and the resul from encryption
will be the payload of an ENCRYPT chunk that will be the only one being sent as payload
of the SCTP packet.

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
Chunks or ENCRYPTED chunk can be handled.

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
reached. Suck packets are built with the SCTP Common header and only one
ENCRYPT chunk.

