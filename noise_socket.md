---
title:      'The Noise Socket Protocol'

---


1. Introduction
================

Noise Socket is a secure transport layer protocol that is much simplier than TLS,
easier to implement and does not require certificates. Only raw public keys can
be used to establish secure connection.

It is useful in IoT, as TLS replacement in microservice architecture, messaging
and other cases where TLS looks overcomplicated.


It is based on the [Noise protocol framework](http://noiseprotocol.org) which
internaly uses only symmetric ciphers, hashes and DH to do secure handshakes.

2. Overview 
============ 
Noise Socket describes how to compose and parse handshake and transport messages, do versioning and negotiation.
There is only one mandatory pattern that must be present in any first handshake message: [Noise_XX](http://noiseprotocol.org/noise.html#interactive-patterns).
Noise_XX allows any combination of authentications (client, server, mutual, none) by using null
public keys (i.e. sending a public key of zeros if you don't want to authenticate).

Other patterns may be supported by concrete implementations, for example Noise_IK can be used for 0-RTT if client knows server's public key. But at least one Noise_XX message must be included first in any first message

Traffic in Noise Socket is split into packets each less than or equal to 65535 bytes (2^16 - 1) which allows for easy parsing and memory management.


3. Packet structure
---------------------------
There are 2 types of packets: 
 - Handshake messages
 - Transport packets
 
Each packet is prepended by 2 bytes big-endian length.


4. Handshake packets
---------------------------

The handshake process consists of set of messages which client and server send to each other. First two of them have a specific packet structure

In the **First handshake message** client offers server a set of sub-messages, each of which corresponds to a concrete [Noise protocol](http://noiseprotocol.org/noise.html#protocol-names)

The total amount of messages is stored in the first byte right after length, thus cannot exceed 255.

Each handshake sub-message contains following fields:
   - 1 byte length of the following string, indicating the ciphersuite/protocol used, i.e. message type (Tl)
   - L bytes string indicating message type (T)
   - 2 bytes big-endian length of following Noise message
   - **Noise message**

**Noise message** is received by calling **WriteMessage** on the corresponding [HandshakeState](http://noiseprotocol.org/noise.html#the-handshakestate-object)

First handshake message full structure:
 - 2 bytes big-endian length of following data
 - 1 byte handshake messages count (N)
  - Repeat N times:
   - 1 byte length of the following string, indicating the protocol used, i.e. message type (Tl)
   - L bytes string indicating message type (T)
   - 2 bytes big-endian length of following message
   - **Noise message**
 
In the **Second packet** server responds to client with the handshake message index it chose and the response itself.
Second packet full structure:
 - 2 bytes big-endian length of following message
 - 1 byte index of the message that responder responds to
 - **Noise message**

After client gets server response there's no longer need in extra transport fields, so all following packets have the following structure:

All subsequent messages full structure:
 - 2 bytes big-endian length of following message
 - **Noise message**
 
 
A total of 3 packets is needed to implent Noise_XX handshake.


3. Versioning and negotiation
---------------------------
For extensibility we'd like the ability for the client to offer
multiple Noise initial handshake messages, and for the server to
choose one:

First NoiseSocket message:
 - 2 bytes big-endian length of following data (N)
 - Repeat:
   - 1 byte length of the following string, indicating the ciphersuite/protocol used, i.e. message type (Tl)
   - L bytes string indicating message type (T)
   - 2 bytes big-endian length of following message
   - <Noise message>

Second NoiseSocket message:
 - 1 byte index of the message that responder responds to
 - 2 bytes big-endian length of following message
 - **Noise message**

All subsequent messages:
 - 2 bytes big-endian length of following message
 - **Noise message**

4. Prologue
---------------------------
The Noise "prologue" is calculated as a concatenation of all T strings of the first message along




EXAMPLES:

 * Suppose the client wants to offer a new protocol version.  It
simply appends the new version initial message to the first initial
message, and sends them both in the initial NoiseSocket message.  The
server responds to the highest version it recognizes.

 * Suppose the client wants to offer three new ciphers 
but reuse the version 0 DH. The client can send version indicators
for 1-3, each followed by a zero-length message, reusing the version 0
message's ephemeral public value.

 * Suppose we want to extend NoiseSocket to support 0-RTT via the
Noise Pipe design.  A version 1 could be used to indicate an IK
handshake attempt, reusing the version 0 message's ephemeral public
key, but containing the additional encrypted fields from the initial
IK message.  The server would respond to either version 0 (XX or XX
fallback) or version 1 (IK).


Naming
-------
A name like "NoiseSocket_25519_ChaChaPoly_BLAKE2b" would translate
into a Noise protocol name in the obvious way:

Noise_XX_25519_ChaChaPoly_BLAKE2b

For hybrid forward secrecy, adding the "hfs" transformation to get
"NoiseSockethfs_25519+NewHope..." is ugly.  Maybe we can improve
naming by saying that a name with a pair of public-key algorithms
defaults to the "hfs" transform, unless a different transform is
explicitly specified?  Then we get:

NoiseSocket_25519+NewHope_AESGCM_SHA256 ->

NoiseXX_25519+NewHope_AESGCM_SHA256
