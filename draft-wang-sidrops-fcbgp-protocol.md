---
title: "FC-BGP Protocol Specification"
abbrev: "FC-BGP"
category: std

docname: draft-wang-sidrops-fcbgp-protocol-00
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
  -
      name: Yangfei Guo
      org: Zhongguancun Laboratory
      city: Beijing
      country: China
      email: guoyangfei@zgclab.edu.cn

normative:
  RFC4271:
  RFC6793:
  RFC7607:
  RFC8205:
  RFC8208:
  RFC8635:

informative:
  RFC4272:
  RFC6472:


--- abstract

This document describes Forwarding Commitment BGP (FC-BGP), an extension to the Border Gateway Protocol (BGP) that provides security for the path of Autonomous Systems (ASes) through which a BGP UPDATE message passes. Forwarding Commitment（FC）is a cryptographically signed code to certify an AS's routing intent on its directly connected hops. Based on FC, FC-BGP aims to build a secure inter-domain system that can simultaneously authenticate the AS_PATH attribute in BGP UPDATE and validate network forwarding on the data plane.


--- middle

# Introduction

The FC-BGP control plane mechanism described in this document is to verify the authenticity of BGP advertised routes.  FC-BGP is fully compatible with BGP and provides more security benefits in case of partial deployment compared with BGPsec. 
FC-BGP extends the BGP UPDATE message with a new optional, transitive, and extended path attribute called FC (Forwarding Commitment). When the BGP UPDATE message traverses an FC-BGP enabled AS, it adds a new FC according to the AS order in AS_PATH. Subsequent ASes can then use the list of FCs in the UPDATE message to verify that the advertised path is consistent with the AS_PATH attribute.

Similar to BGPsec defined in {{RFC8205}}, FC-BGP relies on RPKI to perform route origin validation. Additionally, any FC-enabled BGP speaker that wishes to generate and propagate FC along with BGP UPDATE messages MUST use a router certificate from RPKI that is associated with its AS number. The router key generation here follows {{RFC8635}}.

It is worth noting that the FC-BGP framework can be extended to verify data plane forwarding paths, ensuring that these paths are honoring the control plane BGP paths. However, the description of these mechanisms is outside the scope of this document.


## Requirements Language

{::boilerplate bcp14-tagged}

# FC Attribute

Unlike BGPsec, FC-BGP does not modify the AS_PATH. Instead, FC is enclosed in a BGP UPDATE message as an optional, transitive, and extended length path attribute. Thus, it is unnecessary to negotiate this feature in the BGP OPEN message.

Although FC-BGP would not modify the AS_PATH path attribute, it is REQUIRED to ever use the AS_SET or AS_CONFED_SET in FC-BGP according to what {{RFC6472}} says.

The format of the FC path attribute is shown in {{figure1}}.

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

Flags (1 octet):
: The current value is 0b11010000, representing the FC attribute as optional, transitive, partial, and extended-length.

Type (1 octet):
: The current value is TBD, which is waiting for the IANA assignment. Refer to {{iana-considerations}}.

FCList Length (2 octets):
: The value is the total length of the FCList in bytes.

FCList (variable length):
: The value is a sequence of FCs, in order. The definition of the FC signed code format is shown in {{figure2}}.

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

In FC-BGP, all ASes MUST use 4-byte AS numbers. Existing 2-byte AS numbers are converted into 4-byte AS numbers by setting the two high-order octets of the 4-octet field to 0 {{RFC6793}}.

FC signed code includes the following parts.

Previous Autonomous System Number (PASN, 4 octets):
: The AS number of the previous AS. If the current AS has no previous AS hop, it MUST be filled with 0.

Current Autonomous System Number (CASN, 4 octets):
: The AS number of the current AS.

Nexthop Autonomous System Number (NASN, 4 octets):
: The AS number of the next hop AS to whom the current AS will send the BGP UPDATE message.

Subject Key Identifier (SKI, 20 octets):
: It exists in the RPKI router certificate, used to uniquely identify the public key for signature verification.

<!-- TODO: Different from BGPsec as each FC has its algorithm ID, there is no need to set more than one FC for each hop. -->
Algorithm ID (1 octet):
: The current assigned value is 1, indicating that SHA256 is used to hash the content to be signed, and ECDSA is used for signing. It follows the algorithm suite defined in {{RFC8208}} and its updates. As each FC has its Algorithm ID, so no need to worry about that one suddenly changing its algorithm suite.

Flags (1 octet):
: Its value MUST be 0. None of these bits are assigned values.

Signature Length (2 octets):
: It indicates the signature length in bytes.

Signature (variable length):
: The signature content and order are Signature=ECDSA(SHA256(PASN, CASN, NASN, Prefix)), where the Prefix is the IP address prefix which is encapsulated in the BGP UPDATE, and only one prefix is used each time. For hashing and signing, it uses the full IP address and IP prefix length. The full IP address uses 4 bytes for IPv4 and 16 bytes for IPv6.

# Processing a Received FC-BGP UPDATE Message

Upon receiving a BGP UPDATE message carrying FC path attributes, an AS will perform the following three steps:

1. Verify the AS-Path attribute.
2. BGP best path selection.
3. Update the FC path attributes and continue advertising the BGP route.

The AS that originates a BGP UPDATE message with the FC path attributes only performs the third step. An AS which no longer propagates a BGP UPDATE only completes the first two steps. For the sake of discussion, we assume that AS 65537 receives an FC-BGP UPDATE message for prefix 192.0.0/24, with an originating AS of 65536 and a next hop of 65538. All three ASes support FC-BGP.

## Verify the AS-Path Attribute

The FC-BGP speaker in AS 65537, upon receiving an UPDATE message, retrieves the FC path attribute and extracts the FC list. It then finds the FC with CASN = 65536 and checks if NASN is equal to 65537. If so, it uses the SKI field to find the public key and calculates the signature using the algorithm specified in the Algorithm ID. If the calculated signature matches the signature in the message, then the AS-Path hop associated with the AS 65536 is verified. This process repeats for all FCs and AS-Paths in the FC list. If AS 65537 does not support FC-BGP, it simply forwards the BGP UPDATE to its neighbors when propagating this BGP route. 

## BGP Best Path Selection

Based on the AS-Path verification, it is recommended that AS 65537 prioritize route selection as follows:

1. Local preference. Local preference is the highest priority, regardless of the verification results.

2. Full path validation. All AS hops in the AS-Path attribute from the source AS to the current AS have successfully passed the aforementioned FC validation.

3. Partial path validation. There is a contiguous AS subsequence in the AS-Path attribute starting from the source AS. All AS hops in the subsequence have successfully passed the FC validation. However, there is at least one AS between the last AS in the subsequence and the current AS whose associated FC is either missing or invalid. We denote the number of ASes included in the subsequence as N, and the total number of ASes from the source AS to the current AS as M. This path should be considered to be secure in the following two conditions: N = 1 and M <= 4 or N > 1 and M <= N + 3. 

4. Shorter AS-Path. The current AS selects the route with the shorter AS_PATH.

5. Other attributes with lower priority than the AS-path length. The addition of FCs in this case should not affect path selection. 

## Update the FC path attributes and continue to advertise the BGP route
FC-BGP speakers need to generate different UPDATE messages for different peers. Each UPDATE announcement contains only one route prefix and cannot be aggregated. This is because different route prefixes may have different announcement paths due to different routing policies. Multiple aggregated route prefixes may cause FC generation and verification errors. When multiple route prefixes need to be announced, the FC-BGP speaker needs to generate different UPDATE messages for each route prefix.

When the AS-PATH uses AS_SEQUENCE in the BGP UPDATE, the FC-BGP function will not be enabled. In other cases, the FC-BGP speaker router will enable the FC-BGP function and update the FC path attribute after verifying AS-Path Attribute and selecting the preferable BGP path. 
All FC-BGP UPDATE messages must comply with the maximum BGP message size. If the final message exceeds the maximum message size, then it must follow the processing of {{Section 9.2 of RFC4271}}.

The FC-BGP speaker in AS 65537 will encapsulate each prefix to be sent to AS 65538 in a single UPDATE message, add the FC path attribute, and sign the path content using its private key. Afterwards, AS65537 will prepend its own FC on the top of the FC List. The FC path attribute uses the message format shown in {{figure1}} and {{figure2}} and should be signed with the RPKI router certificate. When signing the FC attribute, the FC-BGP speaker computes the  SHA256 hash in the order of (PASN ( 0 if absent), CASN, NASN, IP Prefix Address, and IP Prefix Length) firstly. Afterwards, the FC-BGP speaker should calculate the digest information Digest, sign the Digest with ECDSA, and then fill the Signature field and FC fields. At this point, the processing of FC path attributes by the FC-BGP speaker is complete. The subsequent processing of BGP messages follows the standard BGP process.

# Security Considerations

The security considerations of {{RFC8205}} and {{RFC4272}} also apply to FC-BGP.


# IANA Considerations {#iana-considerations}

TBD: Wait for IANA to assign FC-BGP-UPDATE-PATH-ATTRIBUTE-TYPE.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.



