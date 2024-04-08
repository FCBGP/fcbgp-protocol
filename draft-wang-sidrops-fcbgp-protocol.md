---
title: "FC-BGP Protocol Specification"
abbrev: "FC-BGP"
category: std

docname: draft-wang-sidrops-fcbgp-protocol-latest
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
  RFC5492:
  RFC5656:
  RFC6482:
  RFC6487:
  RFC6793:
  RFC6811:
  RFC7607:
  RFC7947:
  RFC8205:
  RFC8208:
  RFC8209:
  RFC8210:
  RFC8635:
  ASPP: I-D.ietf-grow-as-path-prepending

informative:
  RFC4272:
  RFC6472:
  RFC6483:


--- abstract

<!-- TODO: Is FC-BGP proposed for AS_PATH security only?
Then the last sentence says it will process with the data plane.
It is better to revise this abstract to make it more accurate. -->
This document describes Forwarding Commitment BGP (FC-BGP). It is an extension to the Border Gateway Protocol (BGP) that provides security for the path of Autonomous Systems (ASs) through which a BGP UPDATE message passes. Forwarding Commitment (FC) is a cryptographically signed segment to certify an AS's routing intent on its directly connected hops. Based on FC, FC-BGP aims to build a secure inter-domain system that can simultaneously authenticate the AS_PATH attribute in the BGP UPDATE message and validate network forwarding on the data plane.


--- middle

# Introduction

The FC-BGP control plane mechanism described in this document aims to ensure that advertised routes in BGP {{RFC4271}} are authentic. FC-BGP accomplishes this by introducing a new optional, transitive, and extended length path attribute called FC (Forwarding Commitment) to the BGP UPDATE message. This attribute can be used by an FC-BGP-compliant BGP speaker (referred to hereafter as an FC-BGP speaker) to generate, propagate, and validate BGP UPDATE messages to enhance security. In other words, when the BGP UPDATE message travels through an FC-BGP-enabled AS, it adds a new FC based on the AS order in AS_PATH. Subsequent ASs can then utilize the list of FCs in the BGP UPDATE message to ensure that the advertised path is consistent with the AS_PATH attribute.

FC-BGP is fully compatible with BGP and provides more security benefits in case of partial deployment compared with BGPsec {{RFC8205}}, which we will prove at {{comparison}}. Similar to BGPsec, FC-BGP relies on RPKI to perform route origin validation {{RFC6483}}. Additionally, any FC-BGP speaker that wishes to process the FC path attribute along with BGP UPDATE messages MUST obtain a router certificate and store it in the RPKI repository. This certificate is associated with its AS number. The router key generation here follows {{RFC8208}} and {{RFC8635}}.

It is worth noting that the FC-BGP framework can be extended to verify data plane forwarding paths, ensuring that these paths honor the control plane BGP paths. However, the description of these mechanisms is outside the scope of this document.

## Requirements Language

{::boilerplate bcp14-tagged}

## Definitions, and Acronyms

The following terms are used with a specific meaning:

BGP neighbor:
: Also just 'neighbor'. Two BGP speakers that communicate using the BGP protocols are neighbors.

BGP speaker:
: A device exchanging routes with other BGP speakers using the BGP protocol.

BGP UPDATE:
: The message is generated with several path attributes to advertise routes.

Router:
: In this document, the router always refers to a BGP speaker.

In addition to the list above, the following terms are used in this document:

FC:
: Forwarding Commitment, i.e., FC segment. It contains several fields and a digital signature to protect the current path.

FCList:
: A list of FC segments to protect the whole AS path in the BGP UPDATE message. All FC-BGP-enabled BGP speakers SHOULD add their FCs to the BGP UPDATE message.

FC path attribute:
: The optional, transitive, extended length path attribute is defined in this document to obtain BGP security.

FC-BGP speaker:
: a BGP speaker that enables the FC-BGP feature.

FC-BGP UPDATE:
: a BGP UPDATE message carries the FC path attribute.

# FC-BGP Negotiation

<!-- TODO -->
FC-BGP doesn't need to negotiate with neighbors since it is considered a transitive path attribute within the BGP UPDATE message. BGP speakers that do not recognize the FC path attribute or do not support FC-BGP, SHOULD still transmit the FC path attribute to their neighbors. As a result, there is no need to establish a new BGP capability as defined in {{RFC5492}}.

# FC Path Attribute

Unlike BGPsec, FC-BGP does not modify the AS_PATH. Instead, FC is enclosed in a BGP UPDATE message as an optional, transitive, and extended length path attribute. This document registers a new attribute type code for this attribute: TBD, see {{iana-considerations}} for more information.

The FC path attribute includes the digital signatures that protect the pathlet information. We refer to those update messages that contain the FC path attribute as "FC-BGP UPDATE messages". Although FC-BGP would not modify the AS_PATH path attribute, it is REQUIRED to never use the AS_SET or AS_CONFED_SET in FC-BGP according to {{RFC6472}}.

The format of the FC path attribute is shown in {{figure1}} and {{figure2}}. {{figure1}} shows the format of FC path attribute and {{figure2}} shows the FC segment format.

~~~~
 0                   1                   2                   3
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
: The current value is 0b11010000, representing the FC path attribute as optional, transitive, partial, and extended-length.

Type (1 octet):
: The current value is TBD. Refer to {{iana-considerations}} for more information.

FCList Length (2 octets):
: The value is the total length of the FCList in octets.

FCList (variable length):
: The value is a sequence of FC segments, in order. The definition of the FC segment format is shown in {{figure2}}. It does not conflict with the AS_PATH attribute. That means the FC-BGP path attribute and AS_PATH attribute can coexist in the same FC-BGP UPDATE message.

~~~~
 0                   1                   2                   3
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
{: #figure2 title="Format of FC segment."}

In FC-BGP, all ASs MUST use 4-byte AS numbers in FC segments. Existing 2-byte AS numbers are converted into 4-byte AS numbers by setting the two high-order octets of the 4-octet field to 0 {{RFC6793}}.

FC segment includes the following parts. (See {{fcbgp-update}} for more details on populating these fields.)

Previous Autonomous System Number (PASN, 4 octets):
: The PASN is the AS number of the previous hop AS from whom the FC-BGP speaker receives the FC-BGP UPDATE message. If the current AS has no previous AS hop, it MUST be filled with 0. We would discuss more about it at {{security-considerations}}.

Current Autonomous System Number (CASN, 4 octets):
: The CASN is the AS number of the FC-BGP speaker that added this FC segment to the FC path attribute.

Nexthop Autonomous System Number (NASN, 4 octets):
: The NASN is the AS number of the next hop AS to whom the FC-BGP speaker will send the BGP UPDATE message.

Subject Key Identifier (SKI, 20 octets):
: The SKI in the RPKI router certificate is a unique identifier for the public key used for signature verification. If the SKI length exceeds 20 octets, it should retrieve the leftmost 20 octets.

<!-- TODO: Different from BGPsec as each FC segment has its algorithm ID, there is no need to set more than one FC segment for each hop. Hope so. -->
Algorithm ID (1 octet):
: The current assigned value is 1, indicating that SHA256 is used to hash the content to be signed, and ECDSA is used for signing. It follows the algorithm suite defined in {{RFC8208}} and its updates. Each FC segment has an Algorithm ID, so there is no need to worry about sudden changes in its algorithm suite. The key in FC-BGP uses the BGPsec Router Key, so its generation and management follow {{RFC8635}}.

<!-- TODO: Flags in BGPsec used to indicate the AS_CONFED_SEQUENCE. But the pCount field is omitted here. We may merge Flags field and pCount field. However, in BGPsec, setting the pCount field to a value greater than 1 has the same semantics as repeating an AS number multiple times in the AS_PATH of a non-BGPsec UPDATE message (e.g., for traffic engineering purposes). If 0, it means this is for IXP Route Server. And the leftmost bit in Flags is the Confed_Segment flag to indicate the BGPsec speaker that constructed this Secure_Path Segment is sending the UPDATE message to a peer AS within the same AS confederation [RFC5065]. I have no idea how to deal with Confed_Segment/AS confederation. -->
Flags (1 octet):
: The rightmost bit of the Flags field in {{figure2}} is the RouteServer flag (Flags-RS). The Flags-RS flag is set to 1 to indicate that the FC segment is added by a route server, but the AS number will never appear in the AS_PATH attribute.

Signature Length (2 octets):
: It only contains the length of the Signature field in octets, not including other fields.

<!-- TODO: I don't know if it should distinguish IPv4 and IPv6 in calc signature -->
Signature (variable length):
: The signature content and order are Signature=ECDSA(SHA256(PASN, CASN, NASN, Prefix, Prefix Length)), where the Prefix is the IP address prefix which is encapsulated in the BGP UPDATE, i.e. NLRI, and only one prefix is used each time. When hashing and signing, the full IP address and IP prefix length are used, i.e., IPv4 uses 4 octets and IPv6 uses 16 octets.

# FC-BGP UPDATE Messages {#fcbgp-update}

## Generation

This part defines the generation of the FC path attribute and the FC segment. An FC-BGP speaker SHOULD generate a new FC segment even a new FC path attribute when it decides to propagate a route to its external neighbors. For internal neighbors, the FC path attribute in the BGP UPDATE message remains unchanged.

<!-- TODO: Improve the last sentence. I don't know if it has a ruleset for the insertion position of optional transitive (FC) path attributes. Out of efficiencies, the FC path attribute should be placed as the last path attribute as there are several variable-length signatures. -->
To create the FC path attribute for the ongoing UPDATE message, the FC-BGP speaker follows a specific process. Firstly, it generates a new FC Segment, which contains the necessary information for the FC path. This new FC Segment is then added to the FCList of the outer format defined in {{figure1}}. It is important to note that if the FC-BGP speaker is not the origin AS and there is already an existing FC path attribute of the UPDATE message, it MUST prepend its new FC segment to the FCList of the existing FC path attribute like the insertion process of ASN in AS_PATH attribute. This allows the FC-BGP speaker to contribute to its own FC segment while maintaining the existing FC path information. Otherwise, if it is the source AS, the FC-BGP speaker generates the FC path attribute defined in {{figure1}} and inserts it into the UPDATE message.

There are three AS numbers in one FC segment as {{figure2}} shows. The populating of the Current AS number (CASN) within the FC segment is like the AS number in BGPsec, it MUST match the AS number in the Subject field of the RPKI router certificate that will be used to verify the FC segment constructed by this FC-BGP speaker (see {{Section 3.1.1 of RFC8209}} and {{RFC6487}}). The Previous AS number (PASN) is typically set to the AS number from which the UPDATE message receives. However, if the FC-BGP speaker is located in the origin AS, the PASN SHOULD be filled with 0. The Nexthop AS number (NASN) is set to the AS number of the peer to whom the route is advertised. So if there are several neighbors, the FC-BGP speaker should generate separate FCs for different neighbors. But it would never generate a new FC segment for the iBGP neighbor.

The Subject Key Identifier field (SKI) within the new FC segment is populated with the identifier found in the Subject Key Identifier extension of the RPKI router certificate associated with the FC-BGP speaker. This identifier serves as a crucial piece of information for recipients of the route advertisement. It enables them to identify the appropriate certificate to employ when verifying the signatures in FC segments attached to the route advertisement. This practice adheres to the guidelines outlined in {{RFC8209}}.

<!-- TODO: Flags or pCount
The following several paragraphs discuss these two fields in BGPsec. But, we don't need pCount field if it only means for multiple AS in AS_PATH. Because the AS_PATH attribute has remained in FC-BGP. We should separate Flags into different bits for different usages.
It needs more discussion for RS.
The lowest/rightmost one is representative of pCount in RS. The last sentence is not really. It is correct in BGPsec but not in FC-BGP. -->
Typically, the Flags field is set to 0.

A route server (RS) is a third-party brokering system that interconnects three or more BGP-speaking routers using eBGP in IXPs {{RFC7947}}. Typically, RS performs like a transit AS except that it does not insert its AS number to the AS_PATH attribute. The route server also can participate in the FC-BGP process. If the RS is an FC-BGP-enabled RS, it may choose to set the Flags-RS bit to 1 when it populates its FC segment. However, when the RS chooses to add its AS number to the AS_PATH attribute, the Flags-RS bit SHOULD be set to 0. If the RS is a non-FC-BGP RS, it propagates the FC-BGP UPDATE message directly. Anyway, the AS number of RS would be used in the FC segment no matter if it appears in the AS_PATH attribute.

There is an AS Path Prepending feature in BGP to deprioritize a route or alternate path, which will prepend the local AS number multiple times to the AS_PATH attribute {{ASPP}}. To minimize unnecessary processing load during the validation of FC segments, an FC-BGP speaker SHOULD NOT generate multiple consecutive FC segments with the same AS number. Instead, the FC-BGP speaker SHOULD aim to produce a single FC segment once, even if the intention is to achieve the semantics of prepending the same AS number multiple times.

<!-- TODO: It is difficult to guarantee its neighbors support multiple algorithm suites.
The following parts are copied from BGPsec. We may not need to consider AS migration, carefully.
The following definitions in BGPsec {{RFC8205}} also apply to this document. See {{RFC8206}} for a discussion of setting pCount to 0 to facilitate AS Number migration. Also, see Section 4.3 for the use of pCount=0 in the context of an AS confederation.  See Section 7.2 for operational guidance for configuring a BGPsec router for setting pCount=0 and/or accepting pCount=0 from a neighbor. -->
The Algorithm ID field is set to 1. FC-BGP only supports one algorithm suite in this document as BGPsec Algorithm defined in {{RFC8208}}. That is the signature algorithm used MUST be the Elliptic Curve Digital Signature Algorithm (ECDSA) with curve P-256 and the hash algorithm used MUST be SHA-256. If it transits from one algorithm suite to another, the FC-BGP speaker MUST place its router certificate in the RPKI repository first and then specify the Algorithm ID in the FC segment.

The Signature Length field is populated with the length (in octets) of the value in the Signature field.

The Signature field in the new FC segment is a variable length field. It contains a digital signature encapsulated in DER format that binds the prefix, its length, and triplet <PASN, CASN, NASN> to the RPKI router certificate corresponding to the FC-BGP speaker. The digital signature is computed as follows: Signature=ECDSA(SHA256(PASN, CASN, NASN, Prefix, Prefix Length))
<!-- TODO: this is not a good way to clarify the calculation of Signature. More details are needed, such as AFI, SAFI, and NLRI in Figure 8 in RFC8205. -->

The signatures within the FC segments of an FC-BGP UPDATE message ensure the protection of crucial information including the AS number of the neighbor involved in the message exchange. This information is explicitly included in the generated FC segment. Consequently, if an FC-BGP speaker intends to transmit an FC-BGP UPDATE message to multiple BGP neighbors, it MUST generate a distinct FC-BGP UPDATE message for each unique neighbor AS to whom the UPDATE message is being sent.

Indeed, an FC-BGP UPDATE message is REQUIRED to advertise a route to just one prefix. This is because if an FC-BGP speaker receives an UPDATE message containing multiple prefixes, it would be unable to construct a valid FC-BGP UPDATE message, including valid path signatures, with a subset of the received prefixes. To advertise routes to multiple prefixes, the FC-BGP speaker MUST generate individual FC-BGP UPDATE messages for each prefix. This ensures the proper construction of valid path signatures for each advertised prefix.
<!-- TODO: we don't use this rule though I don't know why BGPsec uses it. Additionally, an FC-BGP UPDATE message MUST use the MP_REACH_NLRI attribute {{RFC4760}} to encode the prefix. -->

All FC-BGP UPDATE messages MUST conform to BGP's maximum message size. If the resulting message exceeds the maximum message size, then the guidelines in {{Section 9.2 of RFC4271}} MUST be followed.

## Propagation

Any BGP speaker should propagate the optional transitive FC path attribute encapsulated in the FC-BGP UPDATE message, even though they don't support the FC-BGP feature. However, it is important to note that an FC-BGP speaker SHOULD NOT make any attestation regarding the validation state of the FC-BGP UPDATE message it receives. They MUST verify the FC path attribute themselves.

<!-- The last part is for iBGP and eBGP. -->
To incorporate or create a new FC segment for an FC-BGP UPDATE message using a specific algorithm suite, the FC-BGP speaker MUST possess an appropriate private key capable of generating signatures for that particular algorithm suite. Moreover, this private key MUST correspond to the public key found in a valid RPKI End Entity (EE) certificate. The AS number resource extension within this certificate should include the FC-BGP speaker's AS number as specified in {{RFC8209}}. It is worth noting that these new segments are only prepended to an FC-BGP UPDATE message when the FC-BGP speaker generates the UPDATE message for transmission to an external neighbor. In other words, this occurs when the AS number of the neighbor differs from the AS number of the FC-BGP speaker.

The RPKI allows the legitimate holder of IP address prefix(es) to issue a digitally signed object known as a Route Origin Authorization (ROA). This ROA authorizes a specific AS to originate routes for a particular set of prefixes {{RFC6482}}. It is anticipated that most Relying Parties (RPs) will combine FC-BGP with origin validation {{RFC6483}} and {{RFC6811}}. Therefore, it is strongly RECOMMENDED that an FC-BGP speaker only advertises a route for a given prefix in an FC-BGP UPDATE message if there is a valid ROA that authorizes the FC-BGP speaker's AS to originate routes for that specific prefix.

If an FC-BGP router receives a non-FC-BGP UPDATE message from an external neighbor, meaning that the origin BGP speaker does not support FC-BGP, the router processes the UPDATE message as a regular BGP UPDATE message. In this case, it SHOULD NOT add its own FC segment to the UPDATE message. On the other hand, when the FC-BGP speaker advertises routes belonging to its local AS and receives them from an internal neighbor, it MUST add its FC segment to the UPDATE message if it decides to propagate those routes. Furthermore, if the FC-BGP router receives an FC-BGP UPDATE message from a neighbor for a specific prefix and chooses to propagate that neighbor's route for the prefix, it MUST propagate the route as an FC-BGP UPDATE message containing the FC path attribute.

<!-- TODO: if an received UPDATE message contains AS_SET or AS_CONFED_SET, how can FC-BGP speaker processes that message. IN BGPSEC: the BGPsec speaker MUST remove any existing BGPsec_PATH in the received advertisement(s) for this prefix and produce a traditional (non-BGPsec) UPDATE message. What's more, in the mixed scenario, both AS_SET and FC path attributes are contained in one UPDATE message, how to process it. The second issue pertains to the aggregation of routes following the setting of the FC path attribute. -->

When an FC-BGP speaker sends an FC-BGP UPDATE message to an iBGP (internal BGP) neighbor, the process is straightforward. When the FC-BGP speaker originates a new route advertisement and sends it to an iBGP neighbor, it MUST NOT include the FC path attribute of the UPDATE message. In other words, the FC-BGP speaker omits the FC path attribute. Similarly, when an FC-BGP speaker decides to forward an FC-BGP UPDATE message to an iBGP neighbor, it MUST refrain from adding a new FC segment to the FC-BGP UPDATE message.

When an FC-BGP speaker receives an FC-BGP UPDATE message containing an FC path attribute (with one or more FC segments) from an (internal or external) neighbor, it may choose to propagate the route advertisement by sending it to its other (internal or external) neighbors. When sending the route advertisement to an internal FC-BGP-speaking neighbor, the FC path attribute SHALL NOT be modified. When sending the route advertisement to an external FC-BGP-speaking neighbor, the following procedures are used to form or update the FC path attribute.

## Processing Instructions for AS Confederation Members

<!-- TODO: need more study. -->
TBD.

# Processing a Received FC-BGP UPDATE Message

Upon receiving an FC-BGP UPDATE message from an external (eBGP) neighbor carrying the FC path attribute, an FC-BGP speaker will perform the following three steps:

1. Verify the AS_PATH and FC path attributes.
2. BGP best path selection.
3. Update the FC path attributes and continue advertising the BGP route.

Same as BGPsec, an FC-BGP speaker will wish to perform origin validation (see {{RFC6483}} and {{RFC6811}}) on an incoming FC-BGP UPDATE message, but such validation is independent of the validation described in this section.

The FC-BGP speaker that originates the FC-BGP UPDATE message only performs the third step. An FC-BGP speaker SHOULD no longer propagate an FC-BGP UPDATE message only after completing the first two steps. For the sake of discussion, we assume that AS 65537 receives an FC-BGP UPDATE message for prefix 192.0.2.0/24, with an originating AS of 65536 and a next hop of 65538. All three ASs support FC-BGP.

## Overview of FC-BGP Validation

The RPKI router certificates provide the data including the triplet <AS Number, Public Key, Subject Key Identifier> to verify the AS_PATH and FC path attributes. The recipient SHOULD have access to these RPKI router certificates.

Note that the FC-BGP speaker could perform the validation of RPKI router certificates on its own and extract the required data, or it could receive the same data from a trusted cache that performs RPKI validation on behalf of (some set of) FC-BGP speakers. (For example, the trusted cache could deliver the necessary validity information to the FC-BGP speaker by using the Router Key PDU (Protocol Data Unit) for the RPKI-Router protocol {{RFC8210}}.)

To validate an FC-BGP UPDATE message containing the FC path attribute, the recipient performs the validation steps specified in {{validation-algo}}. The validation procedure results in one of two states: 'Valid' and 'Not Valid'.

<!-- TODO: BGP route selection. -->
It is expected that the output of the validation procedure will be used as an input to BGP route selection. That said, the BGP route selection, and thus the handling of the validation states, is a matter of local policy and is handled using local policy mechanisms. Implementations SHOULD enable operators to set such local policy on a per-session basis.  (That is, it is expected that some operators will choose to treat FC-BGP validation status differently for UPDATE messages received over different BGP sessions.)

<!-- TODO -->
FC-BGP validation need only be performed at the eBGP edge. The validation status of an FC-BGP signed/unsigned UPDATE message MAY be conveyed via iBGP from an ingress edge router to an egress edge router via some mechanism, according to local policy within an AS. As discussed in {{fcbgp-update}}, when an FC-BGP speaker chooses to forward a (syntactically correct) FC-BGP UPDATE message, it SHOULD be forwarded with its FC path attribute intact (regardless of the validation state of the UPDATE message). Based entirely on local policy, an egress router receiving an FC-BGP UPDATE message from within its own AS MAY choose to perform its own validation.

## Validation Algorithm {#validation-algo}

The FC-BGP speaker in AS 65537, upon receiving an UPDATE message, retrieves the FC path attribute and extracts the FC list. It then finds the FC with CASN = 65536 and checks if NASN is equal to 65537. If so, it uses the SKI field to find the public key and calculates the signature using the algorithm specified in the Algorithm ID. If the calculated signature matches the signature in the message, then the AS-Path hop associated with the AS 65536 is verified. This process repeats for all FCs and AS-Paths in the FC list. If AS 65537 does not support FC-BGP, it simply forwards the BGP UPDATE to its neighbors when propagating this BGP route.

<!--
## BGP Best Path Selection

Based on the AS-Path verification, it is recommended that AS 65537 prioritize route selection as follows:

1. Local preference. Local preference is the highest priority, regardless of the verification results.

2. Full path validation. All AS hops in the AS-Path attribute from the source AS to the current AS have successfully passed the aforementioned FC validation.

3. Partial path validation. There is a contiguous AS subsequence in the AS-Path attribute starting from the source AS. All AS hops in the subsequence have successfully passed the FC validation. However, there is at least one AS between the last AS in the subsequence and the current AS whose associated FC is either missing or invalid. We denote the number of ASs included in the subsequence as N, and the total number of ASs from the source AS to the current AS as M. This path should be considered to be secure in the following two conditions: N = 1 and M <= 4 or N > 1 and M <= N + 3.

4. Shorter AS-Path. The current AS selects the route with the shorter AS_PATH.

5. Other attributes with lower priority than the AS-path length. The addition of FCs in this case should not affect path selection.
-->

## Update the FC path attributes and continue to advertise the BGP route

FC-BGP speakers need to generate different UPDATE messages for different neighbors. Each UPDATE announcement contains only one route prefix and cannot be aggregated. This is because different route prefixes may have different announcement paths due to different routing policies. Multiple aggregated route prefixes may cause FC generation and verification errors. When multiple route prefixes need to be announced, the FC-BGP speaker needs to generate different UPDATE messages for each route prefix.

When the AS-PATH uses AS_SEQUENCE in the BGP UPDATE, the FC-BGP function will not be enabled. In other cases, the FC-BGP speaker router will enable the FC-BGP function and update the FC path attribute after verifying AS-Path Attribute and selecting the preferable BGP path.
All FC-BGP UPDATE messages must comply with the maximum BGP message size. If the final message exceeds the maximum message size, then it must follow the processing of {{Section 9.2 of RFC4271}}.

The FC-BGP speaker in AS 65537 will encapsulate each prefix to be sent to AS 65538 in a single UPDATE message, add the FC path attribute, and sign the path content using its private key. Afterwards, AS65537 will prepend its own FC on the top of the FC List. The FC path attribute uses the message format shown in {{figure1}} and {{figure2}} and should be signed with the RPKI router certificate. When signing the FC attribute, the FC-BGP speaker computes the  SHA256 hash in the order of (PASN ( 0 if absent), CASN, NASN, IP Prefix Address, and IP Prefix Length) firstly. Afterward, the FC-BGP speaker should calculate the digest information Digest, sign the Digest with ECDSA, and then fill the Signature field and FC fields. At this point, the processing of FC path attributes by the FC-BGP speaker is complete. The subsequent processing of BGP messages follows the standard BGP process.

# Algorithms and Extensibility

TBD.

# Operations and Management Considerations

TBD.

# Comparison with BGPsec {#comparison}

## Similarities and Differences

TBD.

## Advantages and Disadvantages

TBD.

## Partial Deployment and Full Deployment

<!-- This could also be classed to advantages/disadvantags. If needed, you have the flexibility to modify the organization of this document. -->
TBD.

# Security Considerations {#security-considerations}

## Populating of 0 in PASN

<!-- TODO: prove that insertion of 0 does not include route loop/BGP dispute in security consideration. Question asked by Keyur. See [Discussions: IETF119](https://github.com/FCBGP/fcbgp-protocol/discussions/10) for more information. -->
TBD.

## MISC

The security considerations of {{RFC8205}} and {{RFC4272}} also apply to FC-BGP.


# IANA Considerations {#iana-considerations}

TBD: Wait for IANA to assign FC-BGP-UPDATE-PATH-ATTRIBUTE-TYPE.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.



