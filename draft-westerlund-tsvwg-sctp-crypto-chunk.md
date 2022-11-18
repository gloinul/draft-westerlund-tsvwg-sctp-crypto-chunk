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
This document describes the a methos for addying encryption
support for the Stream Control Transmission Protocol (SCTP).

SCTP Encryption Chunk provides communications privacy for applications that
use SCTP as their transport protocol and allows client/server
applications to communicate in a way that is designed to prevent
eavesdropping and detect tampering or message forgery.

Applications using SCTP Encryption Chunk can exploit all transport
features provided by SCTP and its extensions.

--- middle

# Introduction {#introduction}

## Overview

   This document defines a encryption chunk for the Stream Control
   Transmission Protocol (SCTP), as defined in {{RFC9260}}.

   This specification provides mutual authentication of endpoints,
   data confidentiality, data origin authentication, data integrity
   protection, and data replay protection of user messages for
   applications that use SCTP as their transport protocol.  Thus, it
   allows client/server applications to communicate in a way that is
   designed to give communications privacy and to prevent
   eavesdropping and detect tampering or message forgery. SCTP Encryption Chunk
   uses a guest encryption protocol for mutual authentication, integrity
   protection and confidentiality of user messages as well as SCTP
   Control chunks.

   Applications using SCTP Encryption Chunk can use all transport
   features provided by SCTP and its extensions. SCTP Encryption Chunk supports:

   * preservation of message boundaries.

   * a large number of unidirectional and bidirectional streams.

   * ordered and unordered delivery of SCTP user messages.

   * the partial reliability extension as defined in {{RFC3758}}.

   * the dynamic address reconfiguration extension as defined in
      {{RFC5061}}.

   * User messages of any size.

   The method described in this document poses no further requirements
   on SCTP.

## Protocol Overview

SCTP Encryption Chunk protection is defined as a method for secure
and confidential data transfer extensiont for SCTP.
This is implemented inside the SCTP protocol, in a sublayer between the
SCTP Header handling and the SCTP Chunk handling.
Once an SCTP packet has been received and the SCTP Common Header has been
validated, the SCTP Encrypted Chunk(s) are being sent to the chosen decryption
engine that will return the plain SCTP payload containing the plain SCTP chunks,
those chunks will be handled in the legacy SCTP protocol.

On the outgoing direction, once SCTP stack has created the plain SCTP payload
containing Control and/or Data chunks, that payload will be sent to the Encryption
Engine for being encrypted and the encrypted data will be encapsulated in a
SCTP Encrypted chunk.

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

SCTP Encryption Chunk performs protection operations on the whole SCTP
payload. Information protection is kept during the lifetime of the
Association and no information is sent in plain except than the initial
SCTP handshake, the SCTP common Header and the SCTP Encrypted Chunk header.
Mutual Authentication of the endpoint is performed before any ULP data
chunk is sent.
