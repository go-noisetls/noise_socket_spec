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

All sizes are in big endian form.

3. Packet structure
---------------------------

- 2 bytes packet size (Ps)
- 2 bytes padding size(PDs)
- PDs size padding (PD)
- Actual data
- Optional MAC

```
=================PACKET==========
[Ps] | [PDs] [PD] [data] [MAC] |
      =============PAYLOAD=======
```

The payload is not encrypted during the handshake and encrypted afterwards.
Each payload starts with padding len. Padding len is mandatory, but the padding itself is optional. It is done to simplify packet parsing.
MAC is added only if the payload is encrypted.

4. Padding
---------------------------
 - Padding aligns payload to a certain predefined size to make packets indistinguishable from each other. For example, all packets can have size which is a multiple of 1024 or always be 10 kilobytes.

A sample algorithm to calculate padding considering all packet fields is the following:
```
Const
MaxPayloadSize = 65535

Input: 
paddingMultiplier, //ex: 1024
overheadSize, // 16 if MAC is added, 0 otherwise
dataSize // the size of plaintext data to be sent.


Start:

payloadSize = 2 + dataSize + overheadSize 

if paddingMultiplier > 0 {
		paddingSize = paddingMultiplier - payloadSize mod paddingMultiplier
		if payloadSize+paddingSize > MaxPayloadSize {
			paddingSize = MaxPayloadSize - payloadSize
		}
	}

totalPacketSize = 2 + paddingSize + payloadSize

dataOffset = 2 + 2 + paddingSize // where to place the data and MAC after it

```

2 is the size of size fields

5. Handshake packets
---------------------------

The handshake process consists of set of messages which client and server send to each other. First two of them have a specific payload structure

In the **First handshake message** client offers server a set of sub-messages, each of which corresponds to a concrete [Noise protocol](http://noiseprotocol.org/noise.html#protocol-names)

The amount of sub-messages is stored in a 1-byte value N, right after padding.

Each handshake sub-message contains following fields:
   - 1 byte length of the following string, indicating the ciphersuite/protocol used, i.e. message type (Tl)
   - L bytes string indicating message type (T)
   - 2 bytes big-endian length of following Noise message (Ml)
   - **Noise message** (M)

**Noise message** is received by calling **WriteMessage** on the corresponding [HandshakeState](http://noiseprotocol.org/noise.html#the-handshakestate-object)

First handshake message full structure:
```
=================PACKET=============================================================================
[2 bytes len] | [2 bytes padding len] [padding] ...N times... ([ 1 byte Tl] [T] [Ml] [M])
                ====================================PAYLOAD=========================================
```

 
In the **Second handshake message** server responds to client with the handshake message index it chose and the response itself.
Second packet structure:
 - 1 byte index of the message that responder responds to
 - **Noise message**
 
 
Second handshake message full structure:
 ```
=================PACKET=============================================================================
[2 bytes len] | [2 bytes padding len] [padding] [1 byte index] [handshake message]
                ====================================PAYLOAD=========================================
```

After client gets server response there's no longer need in extra transport fields, so all following packets have the following structure:

 ```
=================PACKET=============================================================================
[2 bytes len] | [2 bytes padding len] [padding] [handshake nessage]
                ====================================PAYLOAD=========================================
```
 
 
3 messages are needed to be sent and received to implement full Noise_XX handshake.


6. Data packets
---------------------

After handshake is complete and both [Cipher states](http://noiseprotocol.org/noise.html#the-cipherstate-object) are created, all following packets must be encrypted.

The maximum amount of plaintext data that can be sent in one packet is

```
65535 - 2(padding size) - 16 (mac size) = 65517 bytes
```

7. Re-keying
-------------------

...
