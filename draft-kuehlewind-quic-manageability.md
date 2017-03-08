---
title: Manageability of the QUIC Transport Protocol
docname: draft-kuehlewind-quic-manageability-00
date:
category: info

ipr: trust200902
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
  -
    ins: M. Kuehlewind
    name: Mirja Kuehlewind
    org: ETH Zurich
    email: mirja.kuehlewind@tik.ee.ethz.ch
    street: Gloriastrasse 35
    city: 8092 Zurich
    country: Switzerland
  -
    ins: B. Trammell
    name: Brian Trammell
    org: ETH Zurich
    email: ietf@trammell.ch
    street: Gloriastrasse 35
    city: 8092 Zurich
    country: Switzerland

normative:
  RFC2119:
  I-D.ietf-quic-transport:
  I-D.ietf-quic-tls:

informative:
  RFC7011:
  IPIM:
    title: In-Protocol Internet Measurement (arXiv preprint 1612.02902)
    author:
      - 
        ins: M. Allman
      -
        ins: R. Beverly
      -
        ins: B. Trammell
    url: https://arxiv.org/abs/1612.02902
    date: 2016-12-09
  Ding2015:
    title: TCP Stretch Acknowledgments and Timestamps - Findings and Impliciations for Passive RTT Measurement (ACM Computer Communication Review)
    author:
      - 
        ins: H. Ding
      -
        ins: M. Rabinovich
    url: http://www.sigcomm.org/sites/default/files/ccr/papers/2015/July/0000000-0000002.pdf
    date: 2015-07 
  I-D.ietf-quic-http:
  I-D.trammell-plus-statefulness:
  draft-kuehlewind-quic-applicability:
    title: Applicability of the QUIC Transport Protocol
    docname: draft-kuehlewind-quic-applicability-00
    date: 2017-03-08
    author:
      -
        ins: M. Kuehlewind
        name: Mirja Kuehlewind
        org: ETH Zurich
        email: mirja.kuehlewind@tik.ee.ethz.ch
        street: Gloriastrasse 35
        city: 8092 Zurich
        country: Switzerland
      -
        ins: B. Trammell
        name: Brian Trammell
        org: ETH Zurich
        email: ietf@trammell.ch
        street: Gloriastrasse 35
        city: 8092 Zurich
        country: Switzerland

--- abstract

This document discusses manageability of the QUIC transport protocol, focusing
on caveats impacting network operations involving QUIC traffic. Its intended
audience is network operators, as well as implementors of the QUIC transport
protocol and applications using it.

--- middle

# Introduction

QUIC {{I-D.ietf-quic-transport}} is a new transport protocol currently under
development in the IETF quic working group, focusing on support of semantics as
needed for HTTP/2 {{I-D.ietf-quic-http}} such as stream-multiplexing to avoid
head-of-line blocking. Based on current deployment practices, QUIC is
encapsulated in UDP and encrypted by default. This means the version of QUIC
that is currently under development will integrate TLS 1.3 {{I-D.ietf-quic-tls}}
to encrypt all payload data and most header information. Given QUIC is an
end-to-end transport protocol, all information in the protocol header, even that
which can be inspected, is is not meant to be mutable by the network, and will
therefore be integrity-protected to the extent possible.

This document provides guidance for network operation and management of QUIC
traffic. This includes guidance on how to interpret and utilize information that
is exposed by QUIC to the network as well as explaining requirement and
assumptions that the QUIC protocol design takes toward the expected network
treatment. It also discusses how common network management practices will be
impacted by QUIC.

Of course, network management is not a one-size-fits-all endeavour: practices
considered necessary or even mandatory within enterprise networks with certain
compliance requirements, for example, would be impermissible on other networks
without those requirements. This document therefore does not make any specific
recommendations as to which practices should or should not be applied; for each
practice, it describes what is and is not possible with the QUIC transport
protocol as defined. 

QUIC is at the moment very much a moving target. This document refers the state
of the QUIC working group drafs as well as to changes under discussion, via
issues and pull requests in GitHub current as of the time of writing.

## Notational Conventions

The words "MUST", "MUST NOT", "SHOULD", and "MAY" are used in this document.
It's not shouting; when these words are capitalized, they have a special meaning
as defined in {{RFC2119}}.

# Features of the QUIC Wire Image

In this section, we discusses those aspects of the QUIC transport protocol that
have an impact on the design and operation of devices that forward QUIC packets.
Here, we are concerned primarily with QUIC's unencrypted wire image, which we
define as the information available in the packet header in each QUIC packet,
and the dynamics of that information. Since QUIC is a versioned protocol,
everything about the header format can change except the mechanism by which a
receiver can determine whether and where a version number is present, and the
meaning of the fields used in the version negotiation process. This document is
focused on the protocol as presently defined in {{I-D.ietf-quic-transport}} and
{{I-D.ietf-quic-tls}}, and will change to track those documents.

## QUIC Packet Header Structure {#public-header}

The QUIC packet header is under active development; see section 5 of {{I-D.ietf-quic-transport}} for the present header structure, and https://github.com/quicwg/base-drafts/pull/361 for one current proposed redesign.

In any case, the following information may be exposed in the packet header:

- version number: The version number is present during version negotiation.
- connection ID: The connection ID identifies the connection associated with a QUIC packet, for load-balancing and NAT rebinding purposes; see {{rebinding}}.
- packet number: Every packet has an associated packet number; this packet number increases with each packet, and the least-significant bits of the packet number are present on each packet; see {{packet-numbers}}.
- public reset indication: Public reset packets expose the fact that a connection is being torn down to devices along the path. The applicability of public reset is currently under discussion; see https://github.com/quicwg/base-drafts/issues/353 and https://github.com/quicwg/base-drafts/pull/20.
- key phase: To support 0-RTT session establishment, QUIC uses two key phases; the key phase of each packet must be exposed to support efficient reception.
- additional flags: Additional flags for diagnostic use are also under consideration; see https://github.com/quicwg/base-drafts/issues/279.

## Integrity Protection of the Wire Image {#wire-integrity}

All information in the QUIC header, even that exposed in the packet header, is
integrity protected. Therefore, devices on path MUST NOT change QUIC packet
headers, as alteration of header information would cause packet drop due to a
failed integrity check at the receiver.

## Connection ID and Rebinding {#rebinding}

The connection ID in the QUIC packer header is used to allow routing of QUIC
packets at load balancers on other than five-tuple information, ensuring that
related flows are appropriately balanced together; and to allow rebinding of a
connection after one of the endpoint's addresses changes - usually the client's,
in the case of the HTTP binding. The connection ID is proposed by the server
during connection establishment. A flow might change one of its IP addresses but
keep the same connection ID, as noted in {{public-header}}, and the connection
ID may change during a connection as well; see section 6.3 of
{{I-D.ietf-quic-transport}}. See also
https://github.com/quicwg/base-drafts/issues/349 for ongoing discussion of the
Connection ID.

## Packet Numbers

The packet number field is always present in the QUIC packet header. The packet
number exposes the least significant 32, 16, or 8 bits of an internal packet
counter per flow direction that increments with each packet sent. This packet
counter is initialized with a random 31-bit initial value at the start of a
connection.

Unlike TCP sequence numbers, this packet number increases with every packet,
including those containing only acknowledgment or other control information.
Indeed, whether a packet contains user data or only control information is
intentionally left unexposed to the network.

While loss detection in QUIC is based on packet numbers, congestion
control by default provides richer information than vanilla TCP does.
Especially, QUIC does not rely on duplicated ACKs, making it more tolerant of
packet re-ordering.

# Specific Network Management Tasks

In this section, we address specific network management and measurement
techniques and how QUIC's design impacts them.

## Stateful Treatment of QUIC Traffic {#statefulness}

Stateful network devices such as firewalls use exposed header information to
support state setup and tear-down. {{I-D.trammell-plus-statefulness}} provides a
general model for in-network state management on these devices, independent of
transport protocol. Features already present in QUIC may be used for state
maintenance in this model. Here, there are two important goals: distinguishing
valid QUIC connection establishment from other traffic, in order to establish
state; and determining the end of a QUIC connection, in order to tear that state
down.

1-RTT connection establishment, using a TLS handshake on stream 0, is detectable
using heuristics similar to those used to detect TLS over TCP. 0-RTT connection
establishment, however, provides no particular heuristic for differentiation
from random background traffic at this time.

Exposure of connection shutdown is currently under discussion; see
https://github.com/quicwg/base-drafts/issues/353 and
https://github.com/quicwg/base-drafts/pull/20.

## Measurement of QUIC Traffic {#measurement}

Passive measurement of TCP performance parameters is commonly deployed in access
and enterprise networks to aid troubleshooting and performance monitoring
without requiring the generation of active measurement traffic. The availability of 

The presence of packet numbers on most QUIC packets allows the trivial one-sided
estimation of packet loss and reordering between the sender and a given
observation point. However, since retransmissions are not identifiable as such,
loss between an observation point and the receiver cannot be reliably estimated.

The lack of any acknowledgement information or timestamping information in the
QUIC packet header makes running passive estimation of latency via round trip
time (RTT) impossible. RTT can only be measured at connection establishment
time, and only when 1-RTT establishment is used.

Note that adding packet number echo (as in
https://github.com/quicwg/base-drafts/pull/367 or
https://github.com/quicwg/base-drafts/pull/368) to the public header would allow
passive RTT measurement at on-path observation points. For efficiency purposes,
this packet number echo need not be carried on every packet, and could be made
optional, allowing endpoints to make a measurability/efficiency tradeoff; see
section 4 of {{IPIM}}. Note further that this facility would have significantly
better measurability characteristics than sequence-acknowledgement-based RTT
measurement currently available in TCP on typical asymmetric flows, as adequate
samples will be available in both directions, and packet number echo would be
decoupled from the underlying acknowledgment machinery; see e.g. {{Ding2015}}

Note in-network devices can inspect and correlate connection IDs for partial
tracking of mobility events.

## DDoS Detection and Mitigation

For enterprises and network operators one of the biggest management challenges
is dealing with Distributed Denial of Service (DDoS) attacks. Some network
operators offer Security as a Service (SaaS) solutions that detect attacks by
monitoring, analyzing and filtering traffic. These approaches generally utilize
network flow data {{RFC7011}}. If any flows pose a threat, usually they are
routed to a "scrubbing environment" where the traffic is filtered, allowing the
remaining "good" traffic to continue to the customer environment.

This type of DDoS mitigation is fundamentally based on tracking state for flows
(see {{statefulness}}), and classifying flows as legitimate or DoS traffic. The
QUIC packet header currently has limited information to support this
classification, especially when the DDoS traffic consists of legitimate QUIC
packets.

In addition, the use of a connection ID to support connection migration renders
5-tuple based filtering ineffective, and requires more state to be maintained by
DDoS defense systems. QUIC's version negotiation feature, if used to support
many simultaneously deployed versions of QUIC with different packet header
layouts, will further complicate fast inspection and discrimination of
legitimate from illegitimate QUIC packets in-network.

0-RTT connection establishment further complicates the identification of valid
QUIC traffic and the extraction of association and confirmation signals as per
{{I-D.trammell-plus-statefulness}}. IoT devices acting as servers will pose a
particular risk in this context.

<!-- need to discuss more before this goes into the draft.

## Radio Access Network Tuning and Channel Optimization

[EDITOR'S NOTE: per ACCORD, there seems to be some disconnect as to whether
anyone is actually using, or planning to use, multiple-bearer solutions for
Internet-bound traffic. If not, this section is moot.]

3GPP mobile networks support Traffic Flow Templates that allow specific flows,
identified by 5-tuples, to traverse different data bearers with different
characteristics based on how they are configured. [EDITOR'S NOTE: a reference
here would be nice]. Specifically, in scheduled networks, there is an inherent
balancing act among absolute data rate, latency, and jitter. Today's 3GPP mobile
networks balance the needs of different types of traffic (e.g. the high bit rate
download that is unaffected by jitter vs real time communications that are lower
bit rate but require minimal jitter or latency impact) by performing traffic
inspection to classify flows, assigning them to bearers best configured for the
respective type of traffic.

[EDITOR'S NOTE: I'm not sure what the problem is here, since applicability says
"don't do this". Would anyone ever use different QUIC streams like this? There
is of course the further problem of classifying encrypted traffic.]

As noted in section 4 of {{draft-kuehlewind-quic-applicability}}, QUIC supports
stream multiplexing within the same connection; i.e. on the same 5-tuple. This provides
several benefits, however at same time it prevents a 3GPP mobile network from
working as previously described, because  there is no way to de-multiplex the
traffic into multiple data bearers. For a 3GPP mobile network operator this
limits the ability to tune the network efficiently based on traffic
classification. This can become even more challenging when different
applications will be able to rely on QUIC as a transport protocol. This
particular scenario is not to be confused with different QoS requirements for
the flows as those can be addressed be the client by initiating different
connections.

-->

# IANA Considerations

This document has no actions for IANA. 

# Security Considerations

Supporting manageability of QUIC traffic inherently involves tradeoffs with the
confidentiality of QUIC's control information; this entire document is therefore
security-relevant.

Some of the properties of the QUIC header used in network management are
irrelevant to application-layer protocol operation and/or user privacy. For
example, packet number exposure (and echo, as proposed in this document), as
well as connection establishment exposure for 1-RTT establishment, make no
additional information about user traffic available to devices on path.

At the other extreme, supporting current traffic classification methods that
operate through the deep packet inspection (DPI) of application-layer headers
are directly antithetical to QUIC's goal to provide confidentiality to its
application-layer protocol(s); in these cases, alternatives must be found.


# Acknowledgments

This work is partially supported by the European Commission under Horizon 2020
grant agreement no. 688421 Measurement and Architecture for a Middleboxed
Internet (MAMI), and by the Swiss State Secretariat for Education, Research,
and Innovation under contract no. 15.0268. This support does not imply
endorsement.