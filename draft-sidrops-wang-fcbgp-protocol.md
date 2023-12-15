---
title: "FC-BGP Protocol Specification"
abbrev: "FC-BGP"
category: std

docname: draft-sidrops-wang-fcbgp-protocol-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
# area: AREA
workgroup: sidrops
venue:
#  group: WG
#  type: Working Group
#  mail: WG@example.com
#  arch: https://example.com/WG
  github: "BasilGuo/fcbgp-protocol"
  latest: "https://BasilGuo.github.io/fcbgp-protocol/draft-sidrops-wang-fcbgp-protocol.html"

pi:    # can use array (if all yes) or hash here

toc: yes
sortrefs: yes  # defaults to yes
symrefs: yes

author:
  -
      fullname: Ke Xu
      org: Tsinghua University
      city: Beijing
      country: China
      email: xuke@tsinghua.edu.cn
  -
      fullname: Xiaoliang Wang
      org: Tsinghua University
      city: Beijing
      country: China
      email: wangxiaoliang0623@foxmail.com
  -
      fullname: Zhuotao Liu
      org: Tsinghua University
      city: Beijing
      country: China
      email: zhuotaoliu@tsinghua.edu.cn
  -
      fullname: Qi Li
      org: Tsinghua University
      city: Beijing
      country: China
      email: qli01@tsinghua.edu.cn
  -
      fullname: Jianping Wu
      org: Tsinghua University
      city: Beijing
      country: China
      email: jianping@cernet.edu.cn

normative:

informative:


--- abstract

This document describes Forwarding Commitment BGP (FC-BGP), an extension to the Border Gateway Protocol (BGP) that provides security for the path of Autonomous Systems (ASes) through which a BGP UPDATE message passes. Forwarding Commitment（FC）is a cryptographically signed code to certify an AS's routing intent on its directly connected hops. Based on FC, the goal of FC-BGP is to build a secure inter-domain system that can simultaneously authenticate the AS_PATH attribute in BGP-UPDATE and validate network forwarding on the data plane.


--- middle

# Introduction

The FC-BGP control plane mechanism described in this document is used to verify the authenticity of BGP advertised routes. Compared to BGPsec, the FC-BGP control plane mechanism is characterized by its support for partial deployment and its security guarantees in partial deployment scenarios.

FC-BGP extends the BGP Update message with a new optional, transitive, non-partial, extended path attribute called FC (Forwarding Commitment). All FC-BGP enabled ASes that the BGP Update message traverses will add the corresponding FC. Subsequent ASes can then use the FC list carried in the update message to verify that the advertised path is consistent with the AS Path attribute.

Similar to BGPsec, FC-BGP relies on RPKI to perform route origin validation. Additionally, any FC-BGP speaker that wishes to generate and propagate FC along with BGP Update messages must use a router certificate from RPKI that is associated with its AS number.

It is worth noting that FC-BGP also includes the ability to verify data plane forwarding paths. However, the description of these mechanisms is outside the scope of this document.


## Requirements Language

{::boilerplate bcp14-tagged}

# FC-BGP Negotiation

TODO: BGP-OPEN packet
TODO: no use BGP-AS_SET

# FC Attribute

FC exists in BGP UPDATE messages as a path attribute. It is an optional, transitive, non-partial, and extended-length path attribute. The format of FC path attribute is shown in {{figure2}}.

~~~~
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|      Flags    |      Type     |         FCList Length         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~                            FCList                             ~
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~
{: #figure1 title="Format of FC path attribute."}

FC path attribute includes the following parts:

* Flags (1 byte): The current value is 0b11010000, which represents optional, extended-length, transitive.

* Type (1 byte): The current value is 0x40, or 64, which is unassigned in the IANA.

* FCList Length (2 bytes): The value is the total length of the FCList in bytes.

* FCList (variable length): The value is a sequence of FCs, in order. The definition of the FC format is shown in {{figure2}}.

~~~~
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                Previous Autonomous System Number              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                 Current Autonomous System Number              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                 Nexthop Autonomous System Number              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~                     Subject Key Identifier                    ~
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| Algorithm ID  |      Flags    |       Signature Length        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~                          Signature                            ~
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~
{: #figure2 title="Format of FC."}

FC includes the following parts:

* Previous Autonomous System Number (PASN): 4 bytes, the AS number of the previous AS. If the current AS has no previous hop AS, it is 0.

* Current Autonomous System Number (CASN): 4 bytes, the AS number of the current AS.

* Nexthop Autonomous System Number (NASN): 4 bytes, the AS number of the next hop AS that the current AS will send the BGP Update Message to.

* Subject Key Identifier (SKI): 20 bytes, which exists in the RPKI router certificate, used to uniquely identify the public key for signature verification.

* Algorithm ID: 1 byte, the current value is 1, indicating that SHA256 is used to hash the content to be signed, and ECDSA is used for signing.

* Flags: 1 byte, the current value is 0.

* Signature Length: 2 bytes, indicating the signature length in bytes.

* Signature: variable length, the signature content, and order are Signature=ECDSA(SHA256(PASN, CASN, NASN, Prefix)), where the Prefix is the IP address prefix which is encapsulated in the BGP UPDATE, and only one prefix is used each time.

# Processing a Received FC-BGP UPDATE Message
Upon receiving a BGP update message carrying FC path attributes, an AS will perform the following three steps:

1. Verify the AS-Path attribute.
2. BGP best path selection.
3. Update the FC path attributes and continue to advertise the BGP route.
Note that an AS that originates a BGP update message carrying FC path attributes will only perform the third step. An AS that no longer propagates a BGP update will only complete the first two steps. For the sake of discussion, we assume that AS 65537 receives an FC-BGP update message that contains a declaration for the prefix 192.0.2.0/24, with an originating AS of 65536 and a next hop of 65538. All three ASes support FC-BGP.

## Verify AS-Path Attribute
An FC-BGP speaker in AS 65537, upon receiving an update message, retrieves the FC path attribute and extracts the FC list. It then finds the FC with CASN = 65536 and checks if NASN is equal to 65537. If so, it uses the SKI field to find the public key and calculates the signature using the algorithm specified in the Algorithm ID. If the calculated signature matches the signature in the message, then the AS-Path hop of AS 65536 is verified. This process is repeated for all FCs and AS-Paths in the FC list. If AS 65537 does not support FC-BGP, it simply forwards the BGP UPDATE.

## BGP Best Path Selection
Based on the AS-Path verification, it is recommended that AS 65537 prioritize route selection as follows:

1. Local preference. Local preference is the highest priority, regardless of the verification results.

2. Full path validation. All AS hops in the AS-Path attribute from the source AS to the current AS have successfully passed FC validation.

3. Partial path validation. There is a contiguous AS subsequence in the AS-Path attribute starting from the source AS. All AS hops in the subsequence have successfully passed FC validation. However, there is at least one AS between the last AS in the subsequence and the current AS that is missing or fails the FC validation. We denote the number of ASes included in the subsequence as N, and the total number of ASes from the source AS to the current AS as M. At this time, N and M need to meet the following requirements: if N = 1, then M <= 4; if N > 1, then M <= N + 3. Under these conditions, the route path can still be guaranteed to be safe from hijacking. For detailed analysis, see xxx.

4. Shorter AS-Path. The current AS selects the route with the shorter AS-Path.

5. Other attributes with lower priority than the AS-Path of the shooter. The addition of FC does not affect the order of route selection in this section, so it will not be discussed further.

## Update the FC path attributes and continue to advertise the BGP route
In FC-BGP, FC protects the route prefix and part of the AS-Path information in the UPDATE message. Since FC contains path information, the UPDATE messages received by different BGP peers are different. FC-BGP speakers need to generate different UPDATE messages for different peers. Each UPDATE announcement contains only one route prefix and cannot be aggregated. This is because different route prefixes may have different announcement paths due to different routing policies. Multiple aggregated route prefixes will cause FC generation and verification errors. When multiple route prefixes need to be announced, the FC-BGP speaker needs to generate different UPDATE messages for each route prefix.

When the AS-PATH uses AS_SEQUENCE in the BGP UPDATE, the FC-BGP function will not be enabled. In other cases, the FC-BGP speaker router will enable the FC-BGP function and update the FC path attribute after completing the above two steps, and continue to announce to other neighbors.

All FC-BGP UPDATE messages must comply with the maximum BGP message size. If the final message exceeds the maximum message size, then it must follow the processing of section 9.2 of RFC4271.

The FC-BGP speaker in AS 65537 will encapsulate each prefix to be sent to AS 65538 in a single UPDATE message, add the FC path attribute, and sign the path content using its private key. Afterwards, AS65537 will insert its own FC at the top of the FC List. The FC path attribute uses the message format shown in {{figure1}} and {{figure2}} for filling and packaging. FC signature uses RPKI router certificate. When signing, first perform SHA256 hash in the order of (PASN, CASN, NASN, IP Prefix Address, and IP Prefix Length). When PASN is not present, the field is 0. 0 is reserved by IANA. For specific uses, please refer to {{RFC7607}}. Calculate the digest information Digest, sign the Digest with ECDSA, fill the result into the Signature field of the FC message, and fill in the FC fields at the same time. After the FC-BGP speaker is filled, it sends the BGP update message according to the standard BGP processing flow.

# Security Considerations

The security considerations of {{RFC8205}} and {{RFC4272}} also apply to ASPAs.


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
