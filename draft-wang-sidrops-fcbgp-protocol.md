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
  RFC6482:
  RFC6487:
  RFC6793:
  RFC6811:
  RFC7607:
  RFC8205:
  RFC8208:
  RFC8209:
  RFC8210:
  RFC8635:

informative:
  RFC4272:
  RFC6472:
  RFC6483:


--- abstract

This document describes Forwarding Commitment BGP (FC-BGP), an extension to the Border Gateway Protocol (BGP) that provides security for the path of Autonomous Systems (ASes) through which a BGP UPDATE message passes. Forwarding Commitment (FC) is a cryptographically signed code to certify an AS's routing intent on its directly connected hops. Based on FC, FC-BGP aims to build a secure inter-domain system that can simultaneously authenticate the AS_PATH attribute in BGP UPDATE and validate network forwarding on the data plane.


--- middle

# Introduction

The FC-BGP control plane mechanism described in this document is to verify the authenticity of BGP {{RFC4271}} advertised routes.  FC-BGP is fully compatible with BGP and provides more security benefits in case of partial deployment compared with BGPsec {{RFC8205}}, which we will prove at {{comparison}}.

<!-- TODO: the following paragraph can remove the last 2 sentences -->

FC-BGP extends the BGP UPDATE message with a new optional, transitive, and extended length path attribute called FC (Forwarding Commitment). This document also describes how an FC-BGP-compliant BGP speaker (referred to hereafter as an FC-BGP speaker) can generate, propagate, and validate BGP UPDATE messages containing this attribute to obtain security. In short, when the BGP UPDATE message traverses an FC-BGP-enabled AS, it adds a new FC according to the AS order in AS_PATH. Subsequent ASes can then use the list of FCs in the UPDATE message to verify that the advertised path is consistent with the AS_PATH attribute.

Similar to BGPsec, FC-BGP relies on RPKI to perform route origin validation. Additionally, any FC-BGP speaker that wishes to generate and propagate FC along with BGP UPDATE messages MUST use a router certificate from RPKI. This certificate is associated with its AS number. The router key generation here follows {{RFC8208}} and {{RFC8635}}.

It is worth noting that the FC-BGP framework can be extended to verify data plane forwarding paths, ensuring that these paths honor the control plane BGP paths. However, the description of these mechanisms is outside the scope of this document.


## Requirements Language

{::boilerplate bcp14-tagged}

# FC-BGP Negotiation

<!-- TODO -->

FC-BGP is NOT REQUIRED to negotiate with peers as it is defined as a transitive path attribute in the BGP UPDATE message. Therefore, there is no need to define a new BGP capability {{RFC5492}} here.

# FC Path Attribute

Unlike BGPsec, FC-BGP does not modify the AS_PATH. Instead, FC is enclosed in a BGP UPDATE message as an optional, transitive, and extended length path attribute. So, it isn't necessary to negotiate this capability in the BGP OPEN message.

This document registers a new attribute type code for this attribute: TBD, see {{iana-considerations}} for more information.

The FC path attribute includes the digital signatures that protect the pathlet information. We refer to those update messages that contain the FC path attribute as "FC-BGP UPDATE messages". Although FC-BGP would not modify the AS_PATH path attribute, it is REQUIRED to never use the AS_SET or AS_CONFED_SET in FC-BGP according to what {{RFC6472}} says.

The format of the FC path attribute is shown in {{figure1}} and {{figure2}}.

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
: The current value is 0b11010000, representing the FC path attribute as optional, transitive, partial, and extended-length.

Type (1 octet):
: The current value is TBD, which is waiting for the IANA assignment. Refer to {{iana-considerations}} for more information.

FCList Length (2 octets):
: The value is the total length of the FCList in octets.

FCList (variable length):
: The value is a sequence of FC segments, in order. The definition of the FC segment format is shown in {{figure2}}. It does not conflict with the AS_PATH attribute. They can coexist in the same FC-BGP UPDATE message.

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

FC segment includes the following parts. (See {{fcbgp-update}} for more details on populating these fields.)

<!-- TODO: prove that insertion of 0 does not include route loop/BGP dispute in security consideration. Question asked by Keyur. See [Discussions: IETF119](https://github.com/FCBGP/fcbgp-protocol/discussions/10) for more information. -->
Previous Autonomous System Number (PASN, 4 octets):
: The PASN is the AS number of the previous hop AS from whom the FC-BGP speaker receives the FC-BGP UPDATE message. If the current AS has no previous AS hop, it MUST be filled with 0. We would discuss more about it at {{security-considerations}}.

Current Autonomous System Number (CASN, 4 octets):
: The CASN is the AS number of the FC-BGP speaker that added this FC segment to the FC path attribute.

Nexthop Autonomous System Number (NASN, 4 octets):
: The NASN is the AS number of the next hop AS to whom the FC-BGP speaker will send the BGP UPDATE message.

Subject Key Identifier (SKI, 20 octets):
: The RPKI router certificate contains the SKI, a unique identifier for the public key used for signature verification. If the length of the SKI exceeds 20 octets, it MUST fetch the initial 20 octets.

<!-- TODO: Different from BGPsec as each FC segment has its algorithm ID, there is no need to set more than one FC segment for each hop. Hope so. -->
Algorithm ID (1 octet):
: The current assigned value is 1, indicating that SHA256 is used to hash the content to be signed, and ECDSA is used for signing. It follows the algorithm suite defined in {{RFC8208}} and its updates. Each FC segment has an Algorithm ID, so there is no need to worry about sudden changes in its algorithm suite. The key in FC-BGP uses the BGPsec Router Key, so its generation and management follow {{RFC8635}}.

<!-- TODO: Flags in BGPsec used to indicate the AS_CONFED_SEQUENCE. But the pCount field is omitted here. We may merge Flags field and pCount field together. However, in BGPsec, setting the pCount field to a value greater than 1 has the same semantics as repeating an AS number multiple times in the AS_PATH of a non-BGPsec UPDATE message (e.g., for traffic engineering purposes). -->
Flags (1 octet):
: Its value MUST be 0. None of these bits are assigned values in this document.

Signature Length (2 octets):
: It only contains the length of the Signature field in octets, not including other fields.

Signature (variable length):
: The signature content and order are Signature=ECDSA(SHA256(PASN, CASN, NASN, Prefix, Prefix Length)), where the Prefix is the IP address prefix which is encapsulated in the BGP UPDATE, i.e. NLRI, and only one prefix is used each time. When hashing and signing, the full IP address and IP prefix length are used, i.e., IPv4 uses 4 octets and IPv6 uses 16 octets.

# FC-BGP UPDATE Messages {#fcbgp-update}

The information protected by the signature on an FC-BGP UPDATE message includes the AS number of the peer from/to whom the UPDATE message is being received/sent. And it explicitly adds them to the generated FC segment. Therefore, if an FC-BGP speaker wishes to send an FC-BGP UPDATE message to multiple BGP peers, it MUST generate a separate FC-BGP UPDATE message for each unique peer AS to whom the UPDATE message is sent.

An FC-BGP UPDATE message MUST advertise a route to only a single prefix. This is because an FC-BGP speaker receiving an UPDATE message with multiple prefixes would be unable to construct a valid FC-BGP UPDATE message (i.e., valid path signatures) containing a subset of the prefixes in the received UPDATE. If an FC-BGP speaker wishes to advertise routes to multiple prefixes, it MUST generate a separate FC-BGP UPDATE message for each prefix.
<!-- TODO: we don't use this rule: Additionally, an FC-BGP UPDATE message MUST use the MP_REACH_NLRI attribute {{RFC4760}} to encode the prefix. -->

<!-- The last part is for iBGP and eBGP. -->
To create or add a new signature to an FC-BGP UPDATE message with a given algorithm suite, the FC-BGP speaker MUST possess a private key suitable for generating signatures for this algorithm suite. Additionally, this private key MUST correspond to the public key in a valid RPKI End Entity (EE) certificate whose AS number resource extension includes the FC-BGP speaker's AS number {{RFC8209}}.  Note also that the new signatures are only added to an FC-BGP UPDATE message when an FC-BGP speaker is generating an UPDATE message to send to an external peer (i.e. when the AS number of the peer is not equal to the FC-BGP speaker's own AS number).

The RPKI enables the legitimate holder of IP address prefix(es) to issue a signed object, called a Route Origin Authorization (ROA), that authorizes a given AS to originate routes to a given set of prefixes (see {{RFC6482}}).  It is expected that most Relying Parties (RPs) will utilize BGPsec in tandem with origin validation (see {{RFC6483}} and {{RFC6811}}). Therefore, it is RECOMMENDED that an FC-BGP speaker only originates an FC-BGP UPDATE message advertising a route for a given prefix if there exists a valid ROA authorizing the FC-BGP speaker's AS to originate routes to this prefix.

<!-- TODO: the original BGP speaker does not enable FC-BGP -->
If an FC-BGP router has received only a non-FC-BGP UPDATE message, it processes the UPDATE message as an FC-BGP UPDATE message. It SHOULD add its own FC segment to the UPDATE message whether received from internal or external peers. Conversely, if an FC-BGP router has received an FC-BGP UPDATE message from a peer for a given prefix and it chooses to propagate that peer's route for the prefix, then it MUST propagate the route as an FC-BGP UPDATE message containing the FC path attribute.

As an optional transitive path attribute, any BGP speaker should propagate the FC-BGP UPDATE message even though they don't support the FC-BGP feature. Any FC-BGP cannot attest to the validation state of the FC-BGP UPDATE message it received.

<!-- TODO: if an received UPDATE message contains AS_SET or AS_CONFED_SET, how can FC-BGP speaker processes that message. IN BGPSEC, the BGPsec speaker MUST remove any existing BGPsec_PATH in the received advertisement(s) for this prefix and produce a traditional (non-BGPsec) UPDATE message. What's more, in the mixed scenario, both AS_SET and FC path attributes are contained in one UPDATE message, how to process it. The second issue pertains to the aggregation of routes following the setting of the FC path attribute. -->

The case where the FC-BGP speaker sends an FC-BGP UPDATE message to an iBGP (internal BGP) peer is quite simple.  When originating a new route advertisement and sending it to an iBGP peer, the FC-BGP speaker omits the FC path attribute.  When an FC-BGP speaker chooses to forward an FC-BGP UPDATE message to an iBGP peer, it MUST NOT add a new FC segment to the FC-BGP UPDATE message.

All FC-BGP UPDATE messages MUST conform to BGP's maximum message size. If the resulting message exceeds the maximum message size, then the guidelines in {{Section 9.2 of RFC4271}} MUST be followed.

When an FC-BGP speaker receives an FC-BGP UPDATE message containing an FC path attribute (with one or more FC segments) from an (internal or external) peer, it may choose to propagate the route advertisement by sending it to its other (internal or external) peers. When sending the route advertisement to an internal FC-BGP-speaking peer, the FC path attribute SHALL NOT be modified. When sending the route advertisement to an external FC-BGP-speaking peer, the following procedures are used to form or update the FC path attribute.

To generate the FC path attribute on the ongoing UPDATE message, the FC-BGP speaker first generates a new FC Segment and adds it to the FCList of the outer format defined in {{figure1}}. Note that if the FC-BGP speaker is not the origin AS and there is an existing FC path attribute, then the FC-BGP speaker prepends its new FC segment (placed in first position) onto the FCList of the existing FC path attribute.

The Current AS number (CASN) in this FC segment MUST match the AS number in the Subject field of the RPKI router certificate that will be used to verify the digital signature constructed by this BGPsec speaker (see {{Section 3.1.1 of RFC8209}} and {{RFC6487}}).

The Subject Key Identifier field in the new FC segment is populated with the identifier contained in the Subject Key Identifier extension of the RPKI router certificate corresponding to the FC-BGP speaker {{RFC8209}}.  This Subject Key Identifier will be used by recipients of the route advertisement to identify the proper certificate to use in verifying the signature.

<!-- TODO: Flags or pCount
The following several paragraphs discuss these two fields in BGPsec.
But, we don't need pCount field if it only means for multiple AS in AS_PATH.
Because the AS_PATH attribute has remained in FC-BGP.
We should separate Flags into different bits for different usages.
The lowest/rightmost one is representative of pCount in RR. -->

To prevent unnecessary processing load in the validation of FC segments, an FC-BGP speaker SHOULD NOT produce multiple consecutive FC segments with the same AS number. This means that to achieve the semantics of prepending the same AS number k times, an FC-BGP speaker SHOULD produce a single FC segment once.

<!-- TODO The last sentence is not really. It is correct in BGPsec but not in FC-BGP.
It needs more discussion for RR. -->
A route server (RR) that participates in the BGP control plane but does not act as a transit AS in the data plane may choose to set Flags to 1. This option enables the route server to participate in FC-BGP and obtain the associated security guarantees without increasing the length of the AS path. However, when a route server sets the Flags value to 1, it still inserts its AS number into the FC segment, as this information is needed to validate the signature added by the route server.

<!-- TODO: the following parts are copied from BGPsec. We may not need to consider AS migration, carefully.
The following definitions in BGPsec {{RFC8205}} also apply to this document. See {{RFC8206}} for a discussion of setting pCount to 0 to facilitate AS Number migration. Also, see Section 4.3 for the use of pCount=0 in the context of an AS confederation.  See Section 7.2 for operational guidance for configuring a BGPsec router for setting pCount=0 and/or accepting pCount=0 from a peer.-->

In FC-BGP, it only supports one algorithm suite defined in {{RFC8208}}. If it transits from one algorithm suite to another, the FC-BGP speaker MUST place its router certificate in the RPKI repository first and then specify the Algorithm ID in the FC segment.

The Signature field in the new FC segment contains a digital signature that binds the prefix and <PASN, CASN, NASN> to the RPKI router certificate corresponding to the FC-BGP speaker. The digital signature is computed as follows: Signature=ECDSA(SHA256(PASN, CASN, NASN, Prefix, Prefix Length))
<!-- TODO: this is not a good way to clarify the calculation of Signature. More details are needed, such as AFI, SAFI, and NLRI in Figure 8 in RFC8205. -->

<!-- TODO: 4.3. Processing Instructions for Confederation Members. -->

# Processing a Received FC-BGP UPDATE Message

Upon receiving an FC-BGP UPDATE message from an external (eBGP) peer carrying the FC path attribute, an FC-BGP speaker will perform the following three steps:

1. Verify the AS_PATH and FC path attributes.
2. BGP best path selection.
3. Update the FC path attributes and continue advertising the BGP route.

Same as BGPsec, an FC-BGP speaker will wish to perform origin validation (see {{RFC6483}} and {{RFC6811}}) on an incoming FC-BGP UPDATE message, but such validation is independent of the validation described in this section.

The FC-BGP speaker that originates the FC-BGP UPDATE message only performs the third step. An FC-BGP speaker SHOULD no longer propagate an FC-BGP UPDATE message only after completing the first two steps. For the sake of discussion, we assume that AS 65537 receives an FC-BGP UPDATE message for prefix 192.0.2.0/24, with an originating AS of 65536 and a next hop of 65538. All three ASes support FC-BGP.

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

3. Partial path validation. There is a contiguous AS subsequence in the AS-Path attribute starting from the source AS. All AS hops in the subsequence have successfully passed the FC validation. However, there is at least one AS between the last AS in the subsequence and the current AS whose associated FC is either missing or invalid. We denote the number of ASes included in the subsequence as N, and the total number of ASes from the source AS to the current AS as M. This path should be considered to be secure in the following two conditions: N = 1 and M <= 4 or N > 1 and M <= N + 3.

4. Shorter AS-Path. The current AS selects the route with the shorter AS_PATH.

5. Other attributes with lower priority than the AS-path length. The addition of FCs in this case should not affect path selection.
-->

## Update the FC path attributes and continue to advertise the BGP route

FC-BGP speakers need to generate different UPDATE messages for different peers. Each UPDATE announcement contains only one route prefix and cannot be aggregated. This is because different route prefixes may have different announcement paths due to different routing policies. Multiple aggregated route prefixes may cause FC generation and verification errors. When multiple route prefixes need to be announced, the FC-BGP speaker needs to generate different UPDATE messages for each route prefix.

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

<!-- This could also be classed to advantages/disadvantags.
    If needed, you have the flexibility to modify the organization of this document. -->

TBD.

# Security Considerations {#security-considerations}

The security considerations of {{RFC8205}} and {{RFC4272}} also apply to FC-BGP.


# IANA Considerations {#iana-considerations}

TBD: Wait for IANA to assign FC-BGP-UPDATE-PATH-ATTRIBUTE-TYPE.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.



