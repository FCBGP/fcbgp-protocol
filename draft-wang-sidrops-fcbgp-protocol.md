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
  github: "FCBGP/fcbgp-protocol"
  latest: "https://FCBGP.github.io/fcbgp-protocol/draft-wang-sidrops-fcbgp-protocol.html"

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
  RFC4271: # BGP protocol
  RFC4724: # Graceful Restart Mechanism for BGP
  RFC4760: # Multiprotocol Extensions for BGP-4
  RFC5656: # EC algo for secure shell transport layer
  RFC6480: # RPKI infrastructure
  RFC6482: # ROA profile
  RFC6483: # validation route origin using RPKI and ROA
  RFC6487: # A Profile for X.509 PKIX Resource Certificates
  RFC6793: # BGP: 4-octet AS number
  RFC7606: # Revised Error Handling for BGP UPDATE Messages
  RFC7947: # Internet Exchange BGP Route Server
  RFC8205: # BGPsec protocol
  RFC8208: # BGPsec Algorithms, Key Formats, and Signature Formats
  RFC8209: # BGPsec router certificate, CRL, CSR
  RFC8210: # RTR, version 1
  RFC8635: # Router Keying for BGPsec
  RFC9234: # BGP ROLE Capability and OTC path attr.

informative:
  RFC4272: # BGP Security Vulnerabilities Analysis
  RFC5065: # Autonomous System Confederations for BGP
  RFC5492: # Capabilities Advertisement with BGP-4
  RFC6472: # Recommendation for Not Using AS_SET and AS_CONFED_SET in BGP
  RFC6811: # BGP Prefix Origin Validation
  RFC7132: # Threat Model for BGP Path Security
  RFC7607: # Codification of AS 0 Processing
  RFC7908: # Route Leaks Definition and Classification
  RFC8206: # BGPsec Operational Considerations
  RFC8416: # Simplified Local Internet Number Resource Management with the RPKI (SLURM)
  ASPP: I-D.ietf-grow-as-path-prepending
  ASPA-Profile: I-D.ietf-sidrops-aspa-profile
  ASPA-Verification: I-D.ietf-sidrops-aspa-verification
  Deprecation-AS_SET-AS_CONFED_SET: I-D.ietf-idr-deprecate-as-set-confed-set # Deprecation of AS_SET and AS_CONFED_SET in BGP
  FC-ARXIV:
    title: "Secure Inter-domain Routing and Forwarding via Verifiable Forwarding Commitments"
    date: Sep. 23, 2023
    target: https://arxiv.org/abs/2309.13271


--- abstract

This document defines an extension, Forwarding Commitment BGP (FC-BGP), to the Border Gateway Protocol (BGP). FC-BGP provides security for the path of Autonomous Systems (ASs) through which a BGP UPDATE message passes. Forwarding Commitment (FC) is a cryptographically signed segment to certify an AS's routing intent on its directly connected hops. Based on FC, FC-BGP aims to build a secure inter-domain system that can simultaneously authenticate the AS_PATH attribute in the BGP UPDATE message and alleviate route leaks in the BGP routing system. The extension is backward compatible, which means a router that supports the extension can interoperate with a router that doesn't support the extension.


--- middle

# Introduction {#Introduction}

In a post-ROV (Route Origin Validation) era, route hijacks are largely mitigated but not eliminated, which fundamentally shifts the security focus up the stack. The problem space becomes less about "who owns the prefix" and more about "how routes propagate and whether that propagation is trustworthy and policy-compliant".

The FC-BGP mechanism described in this document aims to ensure that advertised routes in BGP {{RFC4271}} are authentic and alleviate the BGP route leaks.

FC-BGP accomplishes this by introducing a new optional, transitive, and extended-length path attribute called FC (Forwarding Commitment) to the BGP UPDATE message. This attribute can be used by an FC-BGP-compliant BGP speaker (hereafter referred to as an FC-BGP speaker) to generate, propagate, and validate BGP UPDATE messages to enhance security. In other words, when the BGP UPDATE message travels through an FC-BGP-enabled autonomous system (AS), it adds a new FC based on the AS order in AS_PATH. Subsequent ASs can then utilize the list of FCs in the BGP UPDATE message to ensure that the advertised path is consistent with the AS_PATH attribute. And as a complementary of {{RFC9234}}, it can also alleviate the BGP route leaks.

BGPsec is a path-level authentication approach described in {{RFC8205}}. It replaces the AS_PATH attribute, which is used to record the sequence of ASs that a BGP update has traversed, with the non-transitive BGPsec_Path attribute. However, when a peer does not support BGPsec, the BGPsec_Path attribute will be downgraded to the standard AS_PATH attribute, losing the security benefits BGPsec provides. In contrast, FC-BGP (Forwarding Commitment BGP) preserves the AS_PATH attribute and introduces an additional list of signed messages called Forwarding Commitments. Each Forwarding Commitment (FC) is a publicly verifiable code certifying the correctness of a three-hop pathlet. FC-BGP builds its path authentication based on these FCs.

FC-BGP and BGPsec offer different levels of security benefits in the case of partial deployment, even though they achieve the same security benefits when fully deployed. BGPsec tightly couples path authentication with the BGP path construction process, requiring each AS to iteratively verify the signatures of each prior hop before extending the authentication chain. Consequently, a single legacy AS that does not support BGPsec can break the authentication chain, preventing subsequent BGPsec-aware ASs from reviving the authentication process. As a result, in partial deployment scenarios, BGPsec is often downgraded to the legacy BGP protocol, losing its security benefits.

In contrast to BGPsec, FC-BGP treats partial deployability as a first-class citizen. It adopts a pathlet-driven authentication paradigm, in which the authenticity of an AS path can be incrementally built based on authenticated pathlets.  This design ensures that downstream FC-BGP-aware ASs can use the authenticated pathlets provided by upstream upgraded ASs, even if the full AS path traverses legacy ASs that do not support FC-BGP. By allowing the authentication of sub-paths, FC-BGP enables incremental deployment and provides security benefits to the FC-BGP-aware ASs, regardless of the deployment status of other ASs on the path {{DeploymentBenefitsAnalysis}}.

Similar to BGPsec, FC-BGP relies on RPKI to perform route origin validation {{RFC6483}}. Additionally, any FC-BGP speaker that wishes to process the FC path attribute along with BGP UPDATE messages MUST obtain a router certificate and store it in the RPKI repository. This certificate is associated with its AS number. The router key generation here follows {{RFC8208}} and {{RFC8635}}.

It is NOT RECOMMENDED that both BGPsec and FC-BGP be enabled simultaneously in a BGP network. However, a BGP UPDATE message could carry both a BGPsec_Path attribute and an FC path attribute, and an FC-BGP speaker that also supports BGPsec MUST process such a message properly. The fundamental rule for coexistence is that the FC segments MUST be generated and verified against the same representation of the path that the UPDATE message actually carries: the AS_PATH attribute when no BGPsec_Path attribute is present, or the AS_Path reconstructed from the BGPsec_Path attribute when one is present. A speaker MUST NOT generate FC segments against one representation and verify them against the other. See {{coexist_bgpsec}} for the three cases (FC-BGP only, BGPsec only, and both present).

## Requirements Language

{::boilerplate bcp14-tagged}

## Definitions, and Acronyms

The following terms are used with a specific meaning:

{: vspace="0"}

BGP neighbor:
: Also just 'neighbor'. Two BGP speakers that communicate using the BGP protocol are neighbors. It can be divided into iBGP neighbors and eBGP neighbors.

BGP speaker:
: A device, usually a router, exchanges routes with other BGP speakers using the BGP protocol.

BGP UPDATE:
: The message is generated with several path attributes to advertise routes.

iBGP:
: iBGP neighbor, internal BGP neighbor, or internal neighbor. Internal neighbors are in the same AS.

eBGP:
: eBGP neighbor, external BGP neighbor, or external neighbor. External neighbors are in different ASs.

Router:
: In this document, the router always refers to a BGP speaker.


In addition to the list above, the following terms are used in this document:

{: vspace="0"}

FC:
: Forwarding Commitment, i.e., an FC segment. It contains several fields and a digital signature that together certify a three-hop pathlet, identified by the triplet <Previous AS Number, Current AS Number, Nexthop AS Number>, and bind that pathlet to the AS_PATH attribute and the prefix of the route being advertised.

FCList:
: An ordered list of FC segments that protects the AS_PATH attribute in the BGP UPDATE message. The order of the FC segments follows the order of the AS numbers in the AS_PATH attribute: the FC segment added most recently is placed at the front of the FCList and corresponds to the AS number at the front of the AS_PATH (the ASN most recently prepended). The formal correspondence is defined as the FC-to-AS_PATH Mapping in {{fcmapping}}.

FC path attribute:
: The optional, transitive, extended-length path attribute defined in this document that carries the FCList.

FC-BGP UPDATE:
: A BGP UPDATE message that carries the FC path attribute.

FC-BGP speaker:
: A BGP speaker that enables the FC-BGP feature. It can generate, propagate, and validate FC-BGP UPDATE messages. See also FC-capable, FC-enabled, and FC-required below.

FC-capable:
: A BGP speaker that implements the processing of the FC path attribute as specified in this document.

FC-enabled:
: An FC-capable BGP speaker, or an AS, or a BGP session, for which the FC-BGP feature is actually activated and for which FC segments are being generated. An AS that is FC-capable is not necessarily FC-enabled on every BGP session or for every prefix.

FC-required:
: An AS for which, given a particular route propagation scenario and the deployment state defined in {{deployment}}, the protocol requires that an FC segment be present in the FC path attribute for the FC coverage of the path to be complete.

FC-to-AS_PATH Mapping:
: The deterministic correspondence between the FC segments in an FCList and the AS numbers in the AS_PATH attribute of the same BGP UPDATE message, as defined in {{fcmapping}}.

Prepending Count (PC):
: A 1-octet field in an FC segment that records the number of consecutive occurrences of the Current AS Number (CASN) in the AS_PATH attribute that this single FC segment represents, and that is covered by the FC segment's signature.

Valid:
: The validation result for an FC-BGP UPDATE message when all applicable protocol checks, including the FC-to-AS_PATH Mapping check and all signature verifications, have passed.

Invalid:
: The validation result for an FC-BGP UPDATE message when a protocol violation, an FC-to-AS_PATH Mapping failure, or a cryptographic verification failure has been detected.

Not Validated:
: An internal validation state indicating that the validation of an FC-BGP UPDATE message has been started but has not yet finished (for example, because validation has been deferred). It is not a security statement.

Unsupported:
: The validation result for an FC-BGP UPDATE message that contains an algorithm or a protocol feature that the FC-BGP speaker cannot process.

Incomplete:
: The validation result for an FC-BGP UPDATE message whose FC path attribute authenticates only part of the AS_PATH (for example, because the path traverses ASes that are not FC-required). Incomplete does not imply that the authenticated portion is invalid; see {{deployment}} and {{validation-states}}.

# FC-BGP Negotiation

FC-BGP does not need to negotiate with neighbors since it is considered a transitive path attribute within the BGP UPDATE message. BGP speakers that do not recognize the FC path attribute or do not support FC-BGP SHOULD still transmit the FC path attribute to their neighbors. As a result, there is no need to establish a new BGP capability as defined in {{RFC5492}}.

<!-- Jeffery: The FC state can be removed from BGP without detection in the current procedure. An RPKI-registered policy could address the validation case. -->
Because there is no capability negotiation, an FC-BGP speaker cannot learn directly from BGP whether a neighbor has activated FC-BGP on a session. The only public and verifiable signal is the existence of a valid RPKI router certificate for the neighbor's AS number {{RFC8209}}. This signal is used to determine whether the neighbor is FC-capable and, together with the deployment rules in {{deployment}}, whether the neighbor is FC-required for a given UPDATE. The existence of a router certificate does not, by itself, prove that the AS generated an FC segment for every UPDATE it sent. The FC Deployment and Coverage Semantics in {{deployment}} define the conditions under which the absence of an FC segment is a protocol violation, and the conditions under which it is merely a result of partial deployment.

# FC Deployment and Coverage Semantics {#deployment}

FC-BGP is designed for incremental deployment: a single BGP UPDATE message may traverse ASes that support FC-BGP and ASes that do not. This section defines the deployment concepts used by the rest of this document and the conditions under which the absence of an FC segment is a protocol violation rather than an expected consequence of partial deployment. The concepts FC-capable, FC-enabled, and FC-required are defined in {{Introduction}}; this section gives their operational semantics. The three concepts are distinct and MUST NOT be used interchangeably.

## Sources of Deployment State

The only public and verifiable signal of FC support is the existence of a valid RPKI router certificate for the relevant AS number {{RFC8209}}. Such a certificate establishes that the certificate holder is FC-capable: it demonstrates that the holder possesses the private key associated with the certificate and is able to generate FC segments. A valid router certificate does not, by itself, prove any of the following:

- that the AS is FC-enabled on a particular BGP session;
- that the AS generates FC segments for a particular prefix or address family;
- that the AS generated FC segments at the time the UPDATE message was transmitted;
- that an FC segment, once generated, was not removed by an intermediate AS;
- that the AS's routing behavior matches what its certificate would allow.

The FC-enabled state and the set of prefixes, address families, and sessions for which FCs are generated (the "coverage configuration") MAY be communicated out of band, for example by an operator agreement, a routing registry, or administrative configuration. Absent such information, a receiver MUST apply the default determination rules defined below.

## Determining Whether an AS Is FC-required

An AS is FC-required for a given route if and only if ALL of the following conditions hold:

1. The AS is FC-enabled on the BGP session over which the route is propagated (or, for the origin AS, over which the route was originated toward the next hop); and
2. The route falls within the coverage configuration of that AS, considering at least the prefix, the address family, the SAFI, and the propagation scenario; and
3. The AS is expected to propagate the FC path attribute, i.e., it is not an AS that is only forwarding the attribute as an optional transitive attribute without generating FCs.

A validating FC-BGP speaker MUST determine whether an AS on the AS_PATH attribute is FC-required as follows:

- An AS on the AS_PATH attribute that has a valid RPKI router certificate for its AS number is considered FC-required for a route unless local configuration or out-of-band information establishes that the AS is not FC-enabled for the session over which that AS received or forwarded the route, or that the route is outside the AS's coverage configuration.
- An AS on the AS_PATH attribute without a valid RPKI router certificate is not FC-required.

## FC Coverage Scenarios

The following scenarios determine how the absence of an FC segment is treated. For each scenario, the indicated handling MUST be followed.

| Scenario | Handling |
| --- | --- |
| 1. The AS is FC-required, and the corresponding FC segment is absent. | The FC-to-AS_PATH Mapping ({{fcmapping}}) cannot be completed. The UPDATE message is handled as 'Invalid' per {{error-matrix}}. |
| 2. The AS is FC-capable but not FC-required for this route (e.g., it is not FC-enabled on the session, or the route is outside its coverage configuration). | No violation. The absence is expected, and this AS is not part of the FC coverage of the route. |
| 3. The AS is not FC-capable (a legacy AS). | No violation. The AS cannot generate FC segments; this is the partial-deployment case. |
| 4. The FC path attribute is absent from the UPDATE message. | The UPDATE message is not an FC-BGP UPDATE message. See "FC Path Attribute Wholly Absent" below. |
| 5. The FC path attribute is present but covers only part of the AS_PATH attribute. | The covered portion is still valid; the validation state is 'Incomplete' (see {{validation-states}}). |

## FC Path Attribute Wholly Absent

If an UPDATE message carries no FC path attribute, there is no FC to validate. The FC-BGP validation state is 'Not Validated' (see {{validation-states}}), and the route MAY be processed as an ordinary BGP UPDATE message. An operator whose coverage policy requires FCs for the route MAY additionally apply a locally configured route policy (for example, to reject such routes). FC-BGP itself MUST NOT mark the route 'Invalid' solely because the attribute is absent, because absence is also what partial deployment produces when a legacy AS does not propagate the optional transitive attribute.

## Missing FC: Detectability and Attack

Whether the absence of an FC segment can be detected and whether it indicates an attack are two distinct questions:

- The absence is DETECTABLE when the FC-required determination above can be made independently of the FC path attribute in the received UPDATE message, i.e., from public signals and configuration.
- The absence INDICATES AN ATTACK, or at least a protocol violation by the FC-required AS, when the AS was FC-required for the route and the corresponding FC segment is absent (scenario 1 above).
- When the AS is not FC-required (scenarios 2 and 3 above), the absence is expected and MUST NOT be treated as an attack.

In particular, the mere presence of a valid router certificate does not prove that an FC should have been generated for a specific route. Therefore, when an AS is FC-capable but its FC-required status for the route cannot be established, the absence of an FC segment MUST NOT be treated as a violation.

## Security Guarantees per Deployment State

| Deployment state | Guarantee provided by this document |
| --- | --- |
| Fully deployed, i.e., all ASes on the AS_PATH attribute are FC-required and all corresponding FC segments are present and valid | The FC-to-AS_PATH Mapping is complete: every path node is bound to a valid FC segment. The route can be marked 'Valid'. |
| Partially deployed, i.e., some ASes on the AS_PATH attribute are not FC-required | The authenticated portion of the path is still verified. The route can be marked 'Incomplete'. The guarantees of {{security-considerations}} apply to the covered portions only. |
| No FC path attribute | 'Not Validated'. No FC-BGP guarantee applies. |

# FC Path Attribute {#fc-path-attribute}

Unlike BGPsec, FC-BGP does not modify the AS_PATH. Instead, FC is enclosed in a BGP UPDATE message as an optional, transitive, and extended-length path attribute. This document registers a new attribute type code for this attribute: TBD, see {{iana-considerations}} for more information.

The FC path attribute includes the digital signatures that protect the pathlet information. We refer to those update messages that contain the FC path attribute as "FC-BGP UPDATE messages". Although FC-BGP would not modify the AS_PATH path attribute, it is REQUIRED to never use the AS_SET or AS_CONFED_SET in FC-BGP according to {{RFC6472}} and {{Deprecation-AS_SET-AS_CONFED_SET}}.

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

{: vspace="0"}

Flags (1 octet):
: The current value is 0b11010000, representing the FC path attribute as optional, transitive, partial, and extended-length.

Type (1 octet):
: The current value is TBD. Refer to {{iana-considerations}} for more information.

FCList Length (2 octets):
: The value is the total length of the FCList in octets.

FCList (variable length):
: The value is a sequence of FC segments, in order. The order of the FC segments follows the order of the AS numbers in the AS_PATH attribute: the FC segment added most recently is at the front of the FCList and corresponds to the AS number at the front of the AS_PATH; see {{fcmapping}}. The FCList does not conflict with the AS_PATH attribute; the FC path attribute and the AS_PATH attribute can coexist in the same FC-BGP UPDATE message.

The FC path attribute and the FCList are variable length, and each FC segment contains a variable-length signature. To keep FC-BGP UPDATE messages within BGP's maximum message size {{RFC4271}} and to bound the resources required to parse and verify them, the following limits apply:

- The FCList Length MUST NOT exceed 4095 octets (leaving room for the BGP message header and other path attributes within BGP's maximum message size).
- A single FC segment MUST NOT exceed 1024 octets.
- The number of FC segments in an FCList MUST NOT exceed 128.

A speaker that receives an FC path attribute or an FC segment that exceeds any of these limits MUST handle the FC path attribute as a malformed attribute as specified in {{error-matrix}}.

~~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           Previous Autonomous System Number (PASN)            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|            Current Autonomous System Number (CASN)            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|            Nexthop Autonomous System Number (NASN)            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~                  Subject Key Identifier (SKI)                 ~
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| Algorithm ID  |      Flags    |       Signature Length        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|      PC       |                                              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~                          Signature                            ~
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~
{: #figure2 title="Format of FC segment."}

In FC-BGP, all ASs MUST use 4-byte AS numbers in FC segments. Existing 2-byte AS numbers are converted into 4-byte AS numbers by setting the two high-order octets of the 4-octet field to 0 {{RFC6793}}.

FC segment includes the following parts. (See {{fcbgp-update}} for more details on populating these fields.)

{: vspace="0"}

Previous Autonomous System Number (PASN, 4 octets):
: The PASN is the AS number of the previous hop AS from which the FC-BGP speaker receives the FC-BGP UPDATE message. If the current AS has no previous AS hop, it MUST be filled with 0. It will be discussed more at {{sec-three-asn}}.

Current Autonomous System Number (CASN, 4 octets):
: The CASN is the AS number of the FC-BGP speaker that added this FC segment to the FC path attribute.

Nexthop Autonomous System Number (NASN, 4 octets):
: The NASN is the AS number of the next hop AS to which the FC-BGP speaker will send the BGP UPDATE message.

Subject Key Identifier (SKI, 20 octets):
: The SKI in the RPKI router certificate is a unique identifier for the public key used for signature verification. If the SKI length exceeds 20 octets, it should retrieve the leftmost 20 octets.

<!-- TODO: Different from BGPsec as each FC segment has its algorithm ID, there is no need to set more than one FC segment for each hop. Hope so. -->

{: vspace="0"}

Algorithm ID (1 octet):
: The current assigned value is 1, indicating that SHA256 is used to hash the content to be signed, and ECDSA is used for signing. It follows the algorithm suite defined in {{RFC8208}} and its updates. Each FC segment has an Algorithm ID, so there is no need to worry about sudden changes in its algorithm suite. The key in FC-BGP uses the BGPsec Router Key, so its generation and management follow {{RFC8635}}.

<!-- TODO: Flags in BGPsec used to indicate the AS_CONFED_SEQUENCE. But the pCount field is omitted here. We may merge the Flags field and pCount field. However, in BGPsec, setting the pCount field to a value greater than 1 has the same semantics as repeating an AS number multiple times in the AS_PATH of a non-BGPsec UPDATE message (e.g., for traffic engineering purposes). If 0, it means this is for the IXP Route Server. And the leftmost bit in Flags is the Confed_Segment flag to indicate the BGPsec speaker that constructed this Secure_Path Segment is sending the UPDATE message to a peer AS within the same AS confederation [RFC5065]. I have no idea how to deal with Confed_Segment/AS confederation. -->

{: vspace="0"}

Flags (1 octet):
: Several flag bits. Bits are numbered from bit 7 (the most significant, leftmost bit) down to bit 0 (the least significant, rightmost bit) in {{figure2}}. The following flags are defined:

  - Bit 7 - Confed_Segment flag (Flags-CS). The Flags-CS flag is set to 1 to indicate that the FC-BGP speaker that constructed this FC segment is sending the UPDATE message to a peer AS within the same AS confederation {{RFC5065}}. That is, a sequence of consecutive Flags-CS flags appears in an FC-BGP UPDATE message whenever, in a non-FC-BGP UPDATE message, an AS_PATH segment of type AS_CONFED_SEQUENCE occurs. In all other cases, the Flags-CS flag is set to 0.

  - Bit 6 - Route_Server flag (Flags-RS). The Flags-RS flag is set to 1 to indicate that the FC segment was added by a route server whose AS number does not appear in the AS_PATH attribute. If the AS number of the route server is inserted into the AS_PATH attribute, the Flags-RS flag MUST be set to 0. See {{rs-processing}}.

  - Bit 5 - Provider_to_Customer flag (Flags-P2C). The Flags-P2C flag is set to 1 to indicate that the FC segment's issuer AS sends routes to its customer or RS-Client; see {{route-leak}}.

  - Bit 4 - Peer_to_Peer flag (Flags-P2P). The Flags-P2P flag is set to 1 to indicate that the FC segment's issuer AS sends routes to its peer; see {{route-leak}}.

The remaining bits (bits 3 to 0) of the Flags field are unassigned. They MUST be set to 0 by the sender and ignored by the receiver. New flags are registered in the FC Flags registry defined in {{iana-considerations}}.

Note that the behavior that was previously expressed by an AS_Path_Prepending flag (Flags-ASPP) is now expressed by the Prepending Count (PC) field defined below.

Signature Length (2 octets):
: The length, in octets, of the Signature field. It does not include any other field of the FC segment.

{: vspace="0"}

Prepending Count (PC, 1 octet):
: The number of consecutive occurrences of the Current AS Number (CASN) in the AS_PATH attribute that this single FC segment represents. In the normal case, in which an ASN appears once in the AS_PATH attribute, the PC MUST be set to 1. When the issuer AS uses AS Path Prepending and its ASN appears n (n greater than or equal to 2) consecutive times in the AS_PATH attribute {{ASPP}}, the issuer MUST generate a single FC segment with PC set to n rather than n FC segments. The PC field is covered by the FC segment's signature and MUST match the number of consecutive occurrences of the CASN in the AS_PATH attribute of the same BGP UPDATE message; see {{fcmapping}} and {{validation-steps}}.

{: vspace="0"}

Signature (variable length):
: A digital signature, in DER format, computed over the canonical FC Signature Input defined in {{sig-input}}. The signature is generated with the algorithm suite identified by the Algorithm ID field, using the private key that corresponds to the RPKI router certificate identified by the SKI field. For Algorithm ID 1, the suite is ECDSA over P-256 with SHA-256 {{RFC8208}}. The Signature Length field is set to the length of the encoded signature; when computing the signature, the value 0 is used for the Signature Length component of the canonical input as specified in {{sig-input}}.

## FC Signature Input Construction {#sig-input}

This section defines the canonical serialization of the data that is covered by the signature of an FC segment. The signature MUST be computed over exactly the byte string defined here, so that different implementations produce and verify identical inputs.

All fields are concatenated in the order listed below, and each integer field is encoded in network byte order (big-endian):

~~~~
FC-Signature-Input =
    Domain-Separation       (8 octets)
 || Protocol-Version        (1 octet)
 || PASN                    (4 octets)
 || CASN                    (4 octets)
 || NASN                    (4 octets)
 || SKI                     (20 octets)
 || Algorithm ID            (1 octet)
 || Flags                   (1 octet)
 || Canonical Sig Length    (2 octets, always 0)
 || PC                      (1 octet)
 || AFI                     (2 octets)
 || SAFI                    (1 octet)
 || Prefix                  (4 or 16 octets)
 || Prefix Length           (1 octet)
~~~~

The individual components are defined as follows:

Domain-Separation:
: 8 octets containing the ASCII encoding of the string "FC-BGP" followed by three 0x00 octets. The Domain-Separation string prevents a signature computed for FC-BGP from being valid for a different protocol that signs similar data.

Protocol-Version:
: 1 octet, set to 1. This is the version of the FC segment format defined in this document; see {{iana-considerations}} for version management.

PASN, CASN, NASN:
: The three AS numbers of the FC segment, each encoded as 4 octets as defined in {{figure2}}.

SKI:
: The Subject Key Identifier of the RPKI router certificate used for verification, encoded as 20 octets as defined in {{figure2}}.

Algorithm ID:
: 1 octet, the value of the Algorithm ID field.

Flags:
: 1 octet, the value of the Flags field.

Canonical Sig Length:
: 2 octets, fixed to 0. This is the canonical value of the Signature Length field used for signature computation. It is fixed so that the signer does not need to modify the wire-format field to compute the signature, and so that the same signature can be reproduced given the same FC segment content.

PC:
: 1 octet, the value of the Prepending Count field.

AFI and SAFI:
: The Address Family Identifier (2 octets) and Subsequent Address Family Identifier (1 octet) of the prefix being announced, as carried in the MP_REACH_NLRI attribute {{RFC4760}}. This document supports only unicast IPv4 (AFI 1, SAFI 1) and unicast IPv6 (AFI 2, SAFI 1). The same prefix announced under a different AFI or SAFI yields a different signature input and therefore a different signature, which prevents signatures from being replayed across address families.

Prefix:
: The network prefix, encoded in network byte order using the full address length: 4 octets for IPv4 and 16 octets for IPv6.

Prefix Length:
: 1 octet, the length of the prefix in bits.

The signature for Algorithm ID 1 is ECDSA over P-256 using SHA-256, as defined in {{RFC8208}}. The Signature field carries the ECDSA signature in DER format. Only the DER encoding produced by the algorithms of {{RFC8208}} is considered valid; a signature that is not valid DER MUST cause the FC segment to be handled as specified in {{error-matrix}}.

Test vectors for the FC Signature Input are provided in {{test-vectors}}.

# FC-BGP UPDATE Messages {#fcbgp-update}

## Generation

This part defines the generation of the FC path attribute and the FC segment. An FC-BGP speaker SHOULD generate a new FC segment or even a new FC path attribute when it propagates a route to its external neighbors. For internal neighbors, the FC path attribute in the BGP UPDATE message remains unchanged.

Because the FC path attribute carries several variable-length signatures, it SHOULD be placed as the last path attribute in the BGP UPDATE message.

The FC-BGP speaker follows a specific process to create the FC path attribute for the ongoing UPDATE message. First, it generates a new FC Segment containing the information described below and adds it to the FCList of the outer format defined in {{figure1}}. If the FC-BGP speaker is not the origin AS and an FC path attribute already exists in the UPDATE message, the speaker MUST prepend the new FC segment to the FCList, in the same way that an AS number is prepended to the AS_PATH attribute. This keeps the FCList ordered in the same direction as the AS_PATH attribute, so that the first FC segment in the FCList corresponds to the first AS number (the most recently prepended ASN) in the AS_PATH; see {{fcmapping}}. Otherwise, if the speaker is the origin AS, it generates the FC path attribute defined in {{figure1}} and inserts it into the UPDATE message.

There are three AS numbers in one FC segment, as {{figure2}} shows. The populating of the Current AS number (CASN) within the FC segment is like the AS number in BGPsec; it MUST match the AS number in the Subject field of the RPKI router certificate that will be used to verify the FC segment constructed by this FC-BGP speaker (see {{Section 3.1.1 of RFC8209}} and {{RFC6487}}). The Previous AS number (PASN) is typically set to the AS number from which the UPDATE message is received. However, if the FC-BGP speaker is located in the origin AS, the PASN SHOULD be filled with 0. The Nexthop AS number (NASN) is set to the AS number of the peer to whom the route is advertised. So if there are several neighbors, the FC-BGP speaker should generate separate FCs for different neighbors. But it would never generate a new FC segment for the iBGP neighbor.

The Subject Key Identifier field (SKI) within the new FC segment is populated with the identifier found in the Subject Key Identifier extension of the RPKI router certificate associated with the FC-BGP speaker. This identifier serves as a crucial piece of information for recipients of the route advertisement. It enables them to identify the appropriate certificate to employ when verifying the signatures in FC segments attached to the route advertisement. This practice adheres to the guidelines outlined in {{RFC8209}}.

Typically, the Flags field is set to 0 and the PC field is set to 1. The PC field is set to a value greater than 1 only when AS Path Prepending is used, as described below.

A route server (RS) is a third-party brokering system that interconnects three or more BGP-speaking routers using eBGP in IXPs {{RFC7947}}. Typically, an RS behaves like a transit AS except that it does not insert its AS number into the AS_PATH attribute. An RS can also participate in FC-BGP. If the RS is FC-enabled, it adds its FC segment with the Flags-RS bit set to 1 when its AS number does not appear in the AS_PATH attribute; when its AS number is inserted into the AS_PATH attribute, the Flags-RS bit MUST be set to 0. A non-FC RS propagates the FC-BGP UPDATE message directly without adding an FC segment. Note that the AS number of an RS is used in the FC segment whether or not it appears in the AS_PATH attribute; how the resulting FC-to-AS_PATH Mapping is established is defined in {{rs-processing}}.

Typically, the route server does not insert its ASN into the AS_PATH. Take {{fig-rs-ex}} as an example, where AS A advertises a BGP UPDATE to AS C and the RS connects AS A and AS B. When the RS supports FC-BGP, AS A adds its FC segment FC(NULL, A, RS), the RS adds FC(A, RS, B) with Flags-RS set to 1, and AS B adds FC(RS, B, C). If the RS does not support FC-BGP, FC(A, RS, B) is missing from the FCList; this is the partial-deployment case for Route Servers, and its treatment is defined in {{rs-processing}} and {{deployment}}.

~~~~~~
+--------+     +--------+     +--------+     +--------+
|  AS A  | --> |   RS   | --> |  AS B  | --> |  AS C  |
+--------+     +--------+     +--------+     +--------+
~~~~~~
{: #fig-rs-ex title="A network topology with a Router Server linking two ASs."}

AS Path Prepending is a traffic-engineering mechanism in BGP in which the local AS number is prepended to the AS_PATH attribute multiple times to deprioritize a route {{ASPP}}. To minimize the number of signatures, an FC-BGP speaker MUST NOT generate multiple consecutive FC segments whose CASN is the same. Instead, the speaker MUST generate a single FC segment and set the PC field to the number of consecutive occurrences of the CASN in the AS_PATH attribute. For example, if the path segment contributed by the local AS is "65001 65002 65002 65002 65003", the speaker that represents AS 65002 sets PC to 3. The PC value is covered by the FC segment's signature, so a downstream receiver can detect any modification of the number of consecutive occurrences; see {{fcmapping}} and {{validation-steps}}.

The Algorithm ID field for FC-BGP is set to 1. FC-BGP supports the same algorithm suite as BGPsec in this document, as defined in {{RFC8208}}: the signature algorithm MUST be the Elliptic Curve Digital Signature Algorithm (ECDSA) with curve P-256, and the hash algorithm MUST be SHA-256. Algorithm transitions are governed by {{algorithms-extensibility}}, and AS number migration is governed by {{asn-processing}}.

The Signature Length field is populated with the length, in octets, of the value in the Signature field.

The Signature field in the new FC segment is a variable-length field. It contains a digital signature in DER format that binds the pathlet identified by the triplet <PASN, CASN, NASN>, together with the PC, the SKI, the Algorithm ID, the Flags, and the prefix, to the RPKI router certificate of the FC-BGP speaker. The signature is computed over the canonical FC Signature Input defined in {{sig-input}}.


The signatures within the FC segments of an FC-BGP UPDATE message ensure the protection of crucial information, including the AS number of the neighbor involved in the message exchange. This information is explicitly included in the generated FC segment. Consequently, if an FC-BGP speaker intends to transmit an FC-BGP UPDATE message to multiple BGP neighbors, it MUST generate a distinct FC-BGP UPDATE message for each unique neighbor AS to whom the UPDATE message is being sent.

Indeed, an FC-BGP UPDATE message is REQUIRED to advertise a route to just one prefix. This is because if an FC-BGP speaker receives an UPDATE message containing multiple prefixes, it would be unable to construct a valid FC-BGP UPDATE message, including valid path signatures, with a subset of the received prefixes. To advertise routes to multiple prefixes, the FC-BGP speaker MUST generate individual FC-BGP UPDATE messages for each prefix. This ensures the proper construction of valid path signatures for each advertised prefix.
<!-- TODO: we don't use this rule, though I don't know why BGPsec uses it. Additionally, an FC-BGP UPDATE message MUST use the MP_REACH_NLRI attribute {{RFC4760}} to encode the prefix. -->

All FC-BGP UPDATE messages MUST conform to BGP's maximum message size. If the resulting message exceeds the maximum message size, then the guidelines in {{Section 9.2 of RFC4271}} MUST be followed.

## Propagation

Any BGP speaker should propagate the optional transitive FC path attribute encapsulated in the FC-BGP UPDATE message, even though they do not support the FC-BGP feature. However, it is important to note that an FC-BGP speaker SHOULD NOT make any attestation regarding the validation state of the FC-BGP UPDATE message it receives. They MUST verify the FC path attribute themselves.

<!-- The last part is for iBGP and eBGP. -->
To incorporate or create a new FC segment for an FC-BGP UPDATE message using a specific algorithm suite, the FC-BGP speaker MUST possess an appropriate private key capable of generating signatures for that particular algorithm suite. Moreover, this private key MUST correspond to the public key found in a valid RPKI End Entity (EE) certificate. The AS number resource extension within this certificate should include the FC-BGP speaker's AS number as specified in {{RFC8209}}. It is worth noting that these new segments are only prepended to an FC-BGP UPDATE message when the FC-BGP speaker generates the UPDATE message for transmission to an external neighbor. In other words, this occurs when the AS number of the neighbor differs from the AS number of the FC-BGP speaker.

The RPKI allows the legitimate holder of IP address prefix(es) to issue a digitally signed object known as a Route Origin Authorization (ROA). This ROA authorizes a specific AS to originate routes for a particular set of prefixes {{RFC6482}}. It is anticipated that most Relying Parties (RPs) will combine FC-BGP with origin validation {{RFC6483}} and {{RFC6811}}. Therefore, it is strongly RECOMMENDED that an FC-BGP speaker only advertises a route for a given prefix in an FC-BGP UPDATE message if there is a valid ROA that authorizes the FC-BGP speaker's AS to originate routes for that specific prefix.

If an FC-BGP router receives a non-FC-BGP UPDATE message from an external neighbor, meaning that the origin BGP speaker does not support FC-BGP, the router processes the UPDATE message as a regular BGP UPDATE message typically. In this case, it SHOULD NOT add its own FC segment to the UPDATE message. On the other hand, when the FC-BGP speaker advertises routes belonging to its local AS and receives them from an internal neighbor, it MUST add its FC segment to the UPDATE message if it decides to propagate those routes. Furthermore, if the FC-BGP router receives an FC-BGP UPDATE message from a neighbor for a specific prefix and chooses to propagate that neighbor's route for the prefix, it MUST propagate the route as an FC-BGP UPDATE message containing the FC path attribute.

<!-- The handling of received AS_SET / AS_CONFED_SET and of route aggregation is defined in {{aggregation}}. -->

When an FC-BGP speaker sends an FC-BGP UPDATE message to an iBGP (internal BGP) neighbor, the process is straightforward. When the FC-BGP speaker originates a new route advertisement and sends it to an iBGP neighbor, it MUST NOT include the FC path attribute of the UPDATE message. In other words, the FC-BGP speaker omits the FC path attribute. Similarly, when an FC-BGP speaker decides to forward an FC-BGP UPDATE message to an iBGP neighbor, it MUST refrain from adding a new FC segment to the FC-BGP UPDATE message.

When an FC-BGP speaker receives an FC-BGP UPDATE message containing an FC path attribute (with one or more FC segments) from an (internal or external) neighbor, it may choose to propagate the route advertisement by sending it to its other (internal or external) neighbors. When sending the route advertisement to an internal FC-BGP-speaking neighbor, the FC path attribute SHALL NOT be modified. When sending the route advertisement to an external FC-BGP-speaking neighbor, the following procedures are used to form or update the FC path attribute.

## iBGP Propagation and Validation Semantics {#ibgp-propagation}

Within an AS, iBGP is used to convey an FC-BGP route and its validation state from an ingress edge router to an egress edge router. The cryptographic validation of the FC path attribute and its propagation within the AS are distinct concerns; this section defines both, and the conditions under which a validation result remains valid inside an AS.

1. Attribute integrity. The FC path attribute MUST NOT be modified, re-ordered, or deleted when an FC-BGP UPDATE message is propagated over an iBGP session. No FC segment is added over iBGP, and an iBGP speaker MUST NOT re-sign or edit the FC path attribute before propagating it further. In particular, the FC path attribute SHALL NOT be modified when the route is sent to an internal FC-BGP-speaking neighbor (see {{fcbgp-update}}).

2. Revalidation. An iBGP speaker MUST NOT assume that a route received over iBGP has already been validated unless it can attribute a validation result to the identical FC path attribute, the identical AS_PATH attribute, the identical prefix, and an unchanged RPKI certificate state. An implementation MAY reuse a validation result under those conditions, as described in {{speedup-early-termination}}; otherwise it MUST re-run the validation of {{validation-steps}}. A validation result remains valid across multiple internal routers only while the FC path attribute, the AS_PATH attribute, the prefix, and the certificate state are unchanged.

3. Propagation of the validation state. The validation state ({{validation-states}}) MAY be conveyed to other routers within the AS by a mechanism local to the AS, or each internal router MAY re-run the validation itself; the protocol does not mandate a particular mechanism. A router MUST NOT report a validation state as authoritative for a route over iBGP if it did not itself complete the validation of the FC path attribute, or if the state was computed over inputs that are not known to be identical to the route it is propagating.

4. Route reflectors, confederations, and ordinary iBGP. Route reflectors and ordinary iBGP speakers MUST obey the same rules with respect to the FC path attribute: they MUST NOT modify, re-order, or delete the FC path attribute, and they MUST NOT add FC segments. The treatment of the eBGP sessions between AS Confederation members is defined separately in {{fcbgp-update}}; the Flags-CS checks of {{validation-steps}} apply to those sessions.

## Route Aggregation and Multi-Prefix UPDATE Messages {#aggregation}

Each FC-BGP UPDATE message announces exactly one route prefix. This is an explicit protocol limitation: because the Prefix and the Prefix Length are covered by the signature of every FC segment ({{sig-input}}), a single FCList cannot attest to multiple prefixes with different path semantics. An FC-BGP speaker MUST NOT include more than one route prefix in the NLRI of an FC-BGP UPDATE message that carries an FC path attribute. When multiple prefixes are to be announced, the FC-BGP speaker MUST generate a separate FC-BGP UPDATE message for each prefix, each with its own FC path attribute.

An UPDATE message that carries an FC path attribute and more than one NLRI cannot be bound to a single Prefix and Prefix Length and MUST be handled as malformed per {{error-matrix}}.

Route aggregation changes the prefix that is announced. Because the aggregated prefix has a new Prefix and Prefix Length, the existing FC segments cannot be reused for the aggregated route. An FC-BGP speaker that aggregates routes MUST either (a) generate a fresh FC path attribute for the aggregated route in accordance with {{fcbgp-update}}, or (b) remove the FC path attribute from the aggregated UPDATE message and propagate the route as an ordinary BGP UPDATE message. An FC-BGP speaker MUST NOT merge the FC path attributes of the aggregated UPDATE messages.

Because FC-BGP does not use AS_SET or AS_CONFED_SET ({{fc-path-attribute}}), an UPDATE message that contains both an FC path attribute and an AS_SET or AS_CONFED_SET segment in its AS_PATH attribute is malformed and MUST be handled per {{error-matrix}}. A speaker that receives an UPDATE message containing an AS_SET or AS_CONFED_SET MUST NOT add an FC segment to it. When propagating such a route, the speaker MUST either generate a fresh, valid FC path attribute for a route whose AS_PATH attribute contains only AS_SEQUENCE segments, or propagate the route without the FC path attribute.

## Processing Instructions for AS Confederation Members

<!-- TODO: need more study.
1. AS-CONFED-SET and AS-CONFED-SEQUENCE
-->

Members of AS Confederation {{RFC5065}} MUST additionally follow the instructions in this section for processing FC-BGP UPDATE messages.

When an FC-BGP speaker in an AS confederation receives an FC-BGP UPDATE message from a neighbor that is external to the confederation and chooses to propagate the UPDATE message within the confederation, it first adds an FC segment and the signature signed to its own Member-AS (i.e., the 'Current AS Number' is the FC-BGP speaker's Member-AS Number). In this internally modified UPDATE message, the newly added FC segment contains the public AS number (i.e., Confederation Identifier), and the segment's Confed_Segment flag is set to 1. The newly added signature is generated using a private key corresponding to the public AS number of the confederation. The FC-BGP speaker propagates the modified UPDATE message to its peers within the confederation. (Note: In this document, intra-Member-AS peering is regarded as iBGP, and inter-Member-AS peering is regarded as eBGP. The latter is also known as confederation-eBGP.)

Within a confederation, the verification of FC-BGP signatures added by other members of the confederation is optional. Note that if a confederation chooses not to verify digital signatures within the confederation, then FC-BGP is not able to provide any assurances about the integrity of the Member-AS Numbers placed in FC segments where the Confed_Segment flag is set to 1.

When a confederation member receives an FC-BGP UPDATE message from a peer within the confederation and propagates it to a peer outside the confederation, it needs to remove all of the FC segments added by confederation members when it removes all path segments of the AS_PATH with the type of AS_CONFED_SEQUENCE or AS_CONFED_SET. To do this, the confederation members propagated the route outside the confederation as follows:

1. Starting with the most recently added FC segment, remove FC segments whose Flags-CS bit is 1. Stop this process once an FC segment that has its Confed_Segment flags set to 0 is reached.
2. Add an FC segment containing, in the CASN field, the AS Confederation Identifier (the public AS number of the confederation).  Note that all fields other than the CASN field are populated.

Finally, as discussed above, an AS confederation MAY optionally decide that its members will not verify digital signatures added by members. In such a confederation, when an FC-BGP speaker runs the algorithm in {{validation-steps}}, the FC-BGP speaker, during the process of signature verifications, first checks whether the Confed_Segment flag in an FC segment is set to 1. If the flag is set to 1, the FC-BGP speaker skips verification for the corresponding signature and proceeds immediately to the next FC segment. It is an error when an FC-BGP speaker receives, from a neighbor who is not in the same AS confederation, an FC-BGP UPDATE message containing a Confed_Segment flag set to 1.

## Processing Instructions for BGP Route Leak Prevention {#route-leak}

The BGP routing system is susceptible to numerous vulnerabilities. The systemic vulnerability of the BGP routing system is known as "route leaks" {{RFC7908}}. There are 6 types of route leaks defined in {{RFC7908}}.

{{RFC9234}} can detect and prevent BGP route leaks by adding a new BGP OPEN Role capability and OTC transitive path attribute. However, it may be forged.

When using BGP route leak prevention with FC-BGP, it SHOULD tell the neighbor its BGP Role as {{Section 4 of RFC9234}}. If the peer's role is Customer or RS-Client, the Flags-P2C MUST be set to 1 on route advertisement. If the peer's role is peer, the Flags-P2P MUST be set to 1 on the route advertisement. Then, the route SHOULD subsequently go only to the Customers. It is worth noting that if the recently added FC Segment already set the Flags-P2C or Flags-P2P flags to 1, it MUST NOT be propagated to Providers, Peers, or RSes.

# Route Server Processing {#rs-processing}

A route server (RS) is a third-party brokering system that interconnects three or more BGP-speaking routers using eBGP at an Internet exchange point {{RFC7947}}. Unlike a transit AS, a route server typically does not insert its AS number into the AS_PATH attribute. This section defines the FC semantics in this case and how they relate to {{fcmapping}}.

## AS Numbers That Do Not Appear in the AS_PATH

The AS number of a route server can appear in an FC segment (as the Current AS Number) without appearing in the AS_PATH attribute. The following four combinations MUST be distinguished:

| Case | FC segment | Flags-RS |
| --- | --- | --- |
| Ordinary AS whose ASN appears in the AS_PATH attribute | Normal FC segment, CASN is the ASN | 0 |
| Route server whose ASN does NOT appear in the AS_PATH attribute | FC segment with CASN equal to the RS's ASN | 1 |
| Route server whose ASN IS inserted into the AS_PATH attribute | Normal FC segment, CASN is the RS's ASN | 0 |
| Ordinary AS whose ASN does not appear in the AS_PATH attribute (e.g., a transparent RS configured as such) | FC segment with Flags-RS set to 1 | 1 |

When the Flags-RS bit is set to 1, the FC segment does not consume any path node in the FC-to-AS_PATH Mapping ({{fcmapping}}); it bridges the two data-path hops that are adjacent in the AS_PATH attribute. The identity of the route server is authenticated by its RPKI router certificate, exactly as for any other AS.

## Route Server Neighbor Adjacency

For the FC segment added by a route server with Flags-RS set to 1:

- the Previous AS Number MUST be the AS number of the upstream sender (the AS from which the route server received the UPDATE message);
- the Nexthop AS Number MUST be the AS number of the downstream receiver (the AS to which the route server sends the UPDATE message);
- the Previous and Nexthop AS numbers MUST be adjacent in the AS_PATH attribute in the sense of {{fcmapping}} (the data-path hops that the route server bridges are exactly two consecutive path-node groups, or the receiver and the first path node).

When multiple customers share the same route server, each propagation leg is a separate data-path hop. A route server that is FC-enabled MUST generate a separate FC segment for each (upstream, downstream) pair, just as any FC-BGP speaker generates a distinct FC segment for each unique next hop (see {{fcbgp-update}}). The FC segments of different customers are therefore kept distinct by their PASN and NASN fields, and the Flags-RS bit does not by itself distinguish one customer from another.

## Non-FC Route Server

A route server that does not support FC-BGP propagates the FC-BGP UPDATE message without adding an FC segment (see {{fcbgp-update}}). A downstream receiver then observes a gap in the FCList where the route server's FC segment would be. Whether this is expected behavior or the result of the FC segment being deleted is determined by the FC-required determination of {{deployment}}:

- If the route server is not FC-required for the propagation leg (e.g., it has no valid router certificate, or it is not FC-enabled), the absence is expected and the FC coverage is 'Incomplete' ({{validation-states}}).
- If the route server is FC-required for the leg but its FC segment is absent, the FC-to-AS_PATH Mapping cannot be completed, and the UPDATE message is 'Invalid' per {{error-matrix}}.

Note that because a route server does not appear in the AS_PATH attribute, the receiver MUST NOT require an FC segment for the route server unless it can independently establish, per {{deployment}}, that the route server was FC-required for the leg. The Flags-RS bit alone does not create such a requirement.

# The FC-to-AS_PATH Mapping {#fcmapping}

The FCList and the AS_PATH attribute of an FC-BGP UPDATE message use the SAME order. In generation, the FC-BGP speaker prepends its new FC segment to the front of the FCList in the same way that it prepends its AS number to the front of the AS_PATH attribute (see {{fcbgp-update}}). Therefore:

- the first FC segment in the FCList corresponds to the first AS number in the AS_PATH attribute (the AS number most recently prepended, i.e., the one closest to the receiver);
- the last FC segment in the FCList corresponds to the last AS number in the AS_PATH attribute that is covered by the FCList (nearest the origin);
- validation walks the FCList from front to back while walking the AS_PATH attribute from front to back, i.e., from the receiver toward the origin.

The FC-to-AS_PATH Mapping is the deterministic correspondence between the FC segments in an FCList and the AS numbers in the AS_PATH attribute of the same FC-BGP UPDATE message. Given the same AS_PATH attribute, FCList, and certificate state, different implementations MUST obtain the same mapping and the same validation result.

FC-BGP requires that the AS_PATH attribute of an FC-BGP UPDATE message contain only AS_SEQUENCE segments; AS_SET and AS_CONFED_SET are not used (see {{fcbgp-update}}). For this section, the "path nodes" of an UPDATE message are the AS numbers obtained by concatenating the AS_SEQUENCE segments of the AS_PATH attribute in the order in which they appear, from the front (closest to the receiver) to the back (closest to the origin). Let P = [p0, p1, ..., p(m-1)] denote this sequence, where p0 is closest to the receiver and p(m-1) is the origin. Let F = [f0, f1, ..., f(k-1)] denote the FC segments of the FCList in the order in which they appear on the wire, where f0 is the most recently added FC segment.

The FC-to-AS_PATH Mapping holds for a pair (P, F) if and only if all of the following properties hold.

## Direction and Ordering

For any two FC segments f(i) and f(j) with i < j, the path nodes covered by f(i) MUST all be closer to the receiver than the path nodes covered by f(j). The FCList is never in the reverse order of the AS_PATH attribute. An FC segment that is out of order, or an FC segment whose group overlaps the group of another FC segment, is a Mapping failure.

## Data-Path Adjacency

The FC segments describe a chain of data-path hops, each of which is a three-hop pathlet <PASN, CASN, NASN>. Consecutive FC segments MUST be adjacent in this chain, where f(j) denotes the FC segment at wire position j, f(0) being the most recently added FC segment:

- For each j in [0, k-2], the Previous AS Number of f(j) MUST equal the Current AS Number of f(j+1): the issuer of f(j) received the UPDATE message from the issuer of f(j+1).
- For each j in [1, k-1], the Nexthop AS Number of f(j) MUST equal the Current AS Number of f(j-1): the issuer of f(j) sent the UPDATE message to the issuer of f(j-1).
- The Nexthop AS Number of f(0) MUST equal the AS number of the local FC-BGP speaker that validates the message; this is the receiver boundary (see below).

A violation of either adjacency rule is a Mapping failure. These two rules are the two directions of the same adjacency requirement; both MUST be checked so that a broken or spliced chain is detected.

## Receiver and Origin Boundaries

- The Nexthop AS Number of f0 MUST equal the AS number of the local FC-BGP speaker that validates the message (the AS number announced in the BGP OPEN of the session over which the UPDATE message was received); see {{sec-three-asn}}.
- The Previous AS Number of the FC segment whose group includes the origin path node MUST be 0 (AS 0), because there is no previous hop {{sec-three-asn}}.

## Coverage of Path Nodes

Each path node is covered by at most one FC segment. A compliant implementation MUST construct the mapping by processing F in wire order while walking P from front to back, as follows:

1. Set a cursor i over P to 0 and an index j over F to 0.
2. While j < k:

   a. If the Flags-RS bit of f(j) is 0 (a normal FC segment), let n be the Prepending Count (PC) of f(j). This FC segment MUST cover n consecutive path nodes starting at i:
      - If i + n is greater than m, or if i + n exceeds the end of P, the Mapping fails.
      - Every path node p(i) ... p(i+n-1) MUST equal the Current AS Number of f(j); otherwise the Mapping fails.
      - If two consecutive normal FC segments have the same Current AS Number (which is a non-canonical encoding, because they MUST have been merged into a single FC segment with PC greater than 1 per {{fcbgp-update}}), the Mapping fails.
      - Mark the nodes covered by f(j) and set i := i + n.

   b. If the Flags-RS bit of f(j) is 1 (an FC segment of a route server), the FC segment does not consume any path node; the route server's AS number does not appear in the AS_PATH attribute. This FC segment bridges two path-node groups (or, when the route server is the immediate neighbor, the receiver and the first path node):
      - If i is 0, then the Previous AS Number of f(j) MUST equal p0 and the Nexthop AS Number of f(j) MUST equal the AS number of the local FC-BGP speaker.
      - Otherwise, the Previous AS Number of f(j) MUST equal p(i) and the Nexthop AS Number of f(j) MUST equal p(i-1).
      - Any violation is a Mapping failure. No node is consumed.

3. After processing all FC segments:
   - If i is equal to m (every path node is covered) and every path node whose AS is FC-required ({{deployment}}) was covered by a normal FC segment, the Mapping is complete.
   - If i is less than m, the uncovered path nodes are the origin-ward tail of the path. If every uncovered path node is not FC-required ({{deployment}}), the Mapping is complete with respect to the FC coverage, and the FC coverage is partial. If any uncovered path node is FC-required, the corresponding FC segment is missing and the Mapping fails.

Any Mapping failure is a protocol violation. It is handled as 'Invalid' per {{error-matrix}} and MUST NOT be downgraded to a mere missing-FC case.

## Repeated Current AS Numbers

The handling of repeated Current AS Numbers is fully determined by the PC field and the rules above:

- Consecutive occurrences of the same AS number in the AS_PATH attribute (AS Path Prepending, {{ASPP}}) are expressed by a single FC segment whose PC equals the number of consecutive occurrences. The PC is covered by the FC segment's signature ({{sig-input}}) and MUST equal the number of consecutive occurrences, which is verified by the coverage walk above. A malicious change of the number of consecutive occurrences therefore either breaks the signature (if the PC is changed) or breaks the coverage walk (if consecutive occurrences are inserted or removed).
- Two consecutive normal FC segments with the same Current AS Number are a non-canonical encoding and are a Mapping failure.
- Non-consecutive occurrences of the same AS number in the FCList are a Mapping failure: a well-formed BGP path contains an AS number at most once (loop prevention, {{RFC4271}}), so a repeated CASN indicates an invalid AS_PATH attribute or a misplaced or replayed FC segment.

## Treatment on Generation

On generation, an FC-BGP speaker constructs the FCList so that the FC-to-AS_PATH Mapping holds by construction: it prepends each new FC segment to the front of the FCList in the same order in which the corresponding AS number is prepended to the AS_PATH attribute, and it sets the PASN, CASN, NASN, and PC fields according to the rules above (see {{fcbgp-update}}).

# Processing a Received FC-BGP UPDATE Message

## Overview

When receiving an FC-BGP UPDATE message from an external BGP neighbor carrying the FC path attribute, an FC-BGP speaker SHOULD first validate the message to determine the authenticity of the path information. Same as BGPsec, an FC-BGP speaker will wish to perform origin validation (see {{RFC6483}} and {{RFC6811}}) on an incoming FC-BGP UPDATE message, but such validation is independent of the validation described in this section.

After the validation, the FC-BGP speaker may want to send the FC-BGP UPDATE message to neighbors according to local route policies. Then it SHOULD update the FC path attributes and continue advertising the BGP route.

For the origin AS that launches the advertisement, the FC-BGP speaker only needs to generate the FC-BGP UPDATE message without the validation.

The FC-BGP speaker stores the router certificates in the RPKI repository, and any changes in the RPKI state can impact the validity of the UPDATE messages. That means the validity of FC-BGP UPDATE messages relies on the current state of the RPKI repository. When an FC-BGP speaker becomes aware of a change in the RPKI state, such as through an RPKI validating cache using the RTR protocol (as specified in {{RFC8210}}), it is REQUIRED to rerun validation on all affected UPDATE messages stored in its Adj-RIB-In {{RFC4271}}. For instance, if a specific RPKI router certificate becomes invalid due to expiration or revocation, all FC-BGP UPDATE messages containing an FC segment with an SKI matching the SKI in the affected certificate must be reassessed to determine their current validity. If the reassessment reveals a change in the validity state of an UPDATE message, the FC-BGP speaker, depending on its local policy, SHOULD rerun the best path selection process. This allows for the appropriate handling of the updated information and ensures that the most valid and suitable paths are chosen for routing purposes.

## Validation {#Validation}

When verifying the authenticity of an FC-BGP UPDATE message, information from the RPKI router certificates is utilized. The RPKI router certificates provide the data, including the triplet <AS Number, Public Key, Subject Key Identifier>, to verify the AS_PATH and FC path attributes. As a prerequisite, the recipient MUST have access to these RPKI router certificates.

<!-- Jeffery: The FC state can be removed from BGP without detection in the current procedure. RPKI RPKI-registered policy could address the validation case. -->
Note that the existence of a valid RPKI router certificate for an AS makes that AS FC-capable; it does not, by itself, make the AS FC-enabled on a particular BGP session or FC-required for a particular route (see {{deployment}}). Whether the absence of an FC segment is a protocol violation is determined by the FC-required determination of {{deployment}}, and the resulting handling is defined in {{fcmapping}} and {{error-matrix}}. The validation process MUST ensure that a malicious on-path AS cannot remove FC segments without detection when the corresponding AS is FC-required.

<!-- TODO: this is the original BGPsec description, except replacing BGPsec with FC-BGP. Anything to update? -->
Note that the FC-BGP speaker could perform the validation of RPKI router certificates on its own and extract the required data, or it could receive the same data from a trusted cache that performs RPKI validation on behalf of (some set of) FC-BGP speakers. (For example, the trusted cache could deliver the necessary validity information to the FC-BGP speaker by using the Router Key PDU (Protocol Data Unit) for the RPKI-Router protocol {{RFC8210}}.)

<!-- TODO: If it is reasonable to separate the validation steps and the description. It seems that here is the overview, but {{validation-steps}} describes the validation steps. -->
The recipient validates an FC-BGP UPDATE message containing the FC path attribute and obtains exactly one of the validation states defined in {{validation-states}}: 'Valid', 'Invalid', 'Not Validated', 'Unsupported', or 'Incomplete'. We will describe the validation procedure in {{validation-steps}} in this document. The validation result will be used at BGP route selection, thus it will be discussed at {{BGP-route-selection}}.

As the FC-BGP UPDATE message is generated at the eBGP router, the FC-BGP validation needs only to be performed at the eBGP router. The iBGP route plays a crucial role in the FC-BGP UPDATE message propagation and distribution. The function of iBGP is to convey the validation status of an FC-BGP UPDATE message from an ingress edge router to an egress edge router within an AS. The specific mechanisms used to convey the validation status can vary depending on the implementation and local policies of the AS. By propagating this information through iBGP, the eBGP router and other routers within the AS can be aware of the validation status of the FC-BGP UPDATE messages and make routing decisions accordingly. As stated in {{fcbgp-update}}, when an FC-BGP speaker decides to forward a syntactically correct FC-BGP UPDATE message, it is RECOMMENDED to do so while preserving the FC path attribute. This recommendation applies regardless of the validation state of the UPDATE message.

Ultimately, the decision to forward the FC-BGP UPDATE message with the FC path intact and the choice to perform independent validation at the egress router are both determined by local policies implemented within the AS. Note that the decision to perform validation on the received FC-BGP UPDATE message is left to the discretion of the egress router, which is the router receiving the message within its own AS. The egress router has the freedom to choose whether or not it wants to independently validate the FC path attribute based on its local policy, even if the FC path attribute has already been validated by the ingress router. This additional validation performed at the egress router helps ensure the integrity and security of the received FC-BGP UPDATE message.

The rules governing whether an iBGP speaker may modify or delete the FC path attribute, whether validation MUST be re-run, how the validation state is conveyed within an AS, and how route reflectors and confederation member speakers behave, are defined uniformly in {{ibgp-propagation}}.

### Validation States {#validation-states}

The validation of an FC-BGP UPDATE message produces exactly one of the following states. A state is an output of the protocol; it is deliberately decoupled from route policy, which is the responsibility of the operator (see {{BGP-route-selection}}).

| State | Meaning |
| --- | --- |
| 'Valid' | All applicable protocol checks, including the FC-to-AS_PATH Mapping ({{fcmapping}}) and all signature verifications, have passed, and the FC coverage ({{deployment}}) is complete. |
| 'Invalid' | A protocol violation, an FC-to-AS_PATH Mapping failure, or a cryptographic verification failure has been detected. |
| 'Not Validated' | Validation has been started but has not yet finished (for example, because it has been deferred, see {{deferring-validation}}). It is an internal processing state, not a security statement, and it MUST NOT be reported as if validation had produced a definitive result. |
| 'Unsupported' | The UPDATE message contains an algorithm or a protocol feature that the FC-BGP speaker cannot process, and no other failure has been detected. It is not 'Invalid'. |
| 'Incomplete' | The FC coverage is partial: at least one AS on the AS_PATH attribute is not FC-required and therefore has no FC segment, while every FC segment that is present is valid. 'Incomplete' does not imply that the authenticated portion is invalid. |

The dominance relationship among the states is strict: 'Invalid' dominates all others; 'Unsupported' dominates 'Incomplete'; 'Incomplete' dominates 'Valid'. 'Not Validated' is only a pre-completion state and must not be stored as a final result.

The protocol defines the following behavior for each state. Whether a route in a given state is admitted to the candidate route set or becomes the best path is a matter of local policy under {{BGP-route-selection}}; the protocol only requires that the state be available to the route selection process and that the validation result be conveyable within the AS as described in {{fcbgp-update}}.

| State | Required and permitted route-selection behavior |
| --- | --- |
| 'Valid' | MAY be considered a candidate and MAY become the best path. |
| 'Invalid' | MUST NOT be treated as FC-BGP-valid. It MAY still be used by local policy (for example, if it is also propagated without an FC path attribute), but the state MUST remain visible as 'Invalid'. |
| 'Not Validated' | MAY be treated as an ordinary BGP route by local policy while validation is pending, but MUST NOT be reported as validated. |
| 'Unsupported' | MUST NOT be silently reclassified as 'Valid' or as an ordinary unsigned route merely because an algorithm or feature is unsupported; local policy decides its fate. |
| 'Incomplete' | MAY be used, with the understanding that only the authenticated portion of the path is guaranteed. |

### Error Handling {#error-matrix}

This section specifies how the different kinds of failures that can occur while processing an FC-BGP UPDATE message are handled. It distinguishes:

- errors in the encoding of the UPDATE message or of the FC path attribute, which are handled following RFC 7606 {{RFC7606}};
- failures of the security semantics, which produce the validation states defined in {{validation-states}}; and
- the absence of FC segments, which is governed by the deployment semantics of {{deployment}}.

In particular, a cryptographic verification failure MUST NOT be treated as an UPDATE syntax error, and an FC-to-AS_PATH Mapping failure MUST NOT be treated as a mere missing FC.

| Error type | Examples | Protocol handling |
| --- | --- | --- |
| Attribute structure damaged | FC path attribute flags/type/length inconsistent; FCList Length not matching the content; an FC segment shorter than the fixed fields | Follow RFC 7606 {{RFC7606}}: handle the attribute as malformed (e.g., treat-as-withdraw) and do not continue to parse it. |
| Length or field encoding illegal | Signature Length inconsistent with the Signature field; reserved Flags bits set to 1; a PC value that cannot be represented or that would exceed the FCList limits | Follow RFC 7606 {{RFC7606}} as a malformed attribute. |
| Signature verification failure | A signature does not verify under the identified certificate; a signature that is not valid DER | The FC segment is marked 'Invalid'; the UPDATE message becomes 'Invalid' per {{validation-states}}. Not an UPDATE syntax error. |
| FC-to-AS_PATH Mapping failure | FCList order reversed; triplet not data-path adjacent; PC not equal to the number of consecutive occurrences; two consecutive normal FCs with the same CASN; a non-consecutive repeated CASN | 'Invalid' per {{validation-states}}. Not a mere missing FC. |
| Missing FC | An FC-required ({{deployment}}) AS has no corresponding FC segment | If the AS is FC-required, the Mapping cannot be completed and the UPDATE message is 'Invalid'. If the AS is not FC-required, there is no violation and the coverage is 'Incomplete' (partial deployment). |
| Unsupported algorithm | An Algorithm ID that is not recognized; an unknown FC segment version | The affected FC segment(s) are not cryptographically verified. If no other failure is detected, the UPDATE message is 'Unsupported' per {{validation-states}}. It MUST NOT be silently downgraded to an ordinary unsigned BGP route. |
| Certificate missing or invalid | No valid RPKI router certificate for the CASN; the certificate is revoked or expired; the SKI is not found | An FC segment whose verifying certificate cannot be found or is invalid cannot be verified. If the certificate state is definitive (absent, revoked, or expired), the FC segment is marked 'Invalid'. If the certificate state is only temporarily unknown (e.g., RPKI data not yet synchronized), validation may be deferred to 'Not Validated' per local policy; see {{deployment}} and {{Validation}}. |
| RPKI state change | A certificate used by FC segments becomes revoked or expires, is replaced, or a new certificate appears | Revalidation of the affected UPDATE messages is triggered as specified in {{Validation}}; the state of the affected FC segments is reevaluated. |

All limits on the size of the FC path attribute, the FCList, and the FC segments are defined in {{fc-path-attribute}}; exceeding them is handled as a malformed attribute per this section.

### Validation Algorithm {#validation-steps}

This section specifies the concrete validation algorithm of FC-BGP UPDATE messages. A compliant implementation MUST have an FC-BGP UPDATE validation algorithm that behaves the same as the specified algorithm. This ensures consistency and security in validating FC-BGP UPDATE messages across different implementations and allows for interoperability between FC-BGP-enabled networks.

The validation of an FC-BGP UPDATE message proceeds in three phases:

1. syntax checking, in which the structure of the FC path attribute is verified and structural errors are handled per {{error-matrix}} using the RFC 7606 {{RFC7606}} procedures;
2. FC-to-AS_PATH Mapping construction, in which the correspondence defined in {{fcmapping}} is constructed and verified;
3. signature validation, in which each FC segment is verified cryptographically.

The result of validation is one of the states defined in {{validation-states}}: 'Valid', 'Invalid', 'Not Validated', 'Unsupported', or 'Incomplete'. Which state a given failure produces is defined by {{error-matrix}}.

First, the integrity of the FC-BGP UPDATE message MUST be checked. Both syntactical and protocol violation errors are checked. The FC path attribute MUST be present when an FC-BGP UPDATE message is received from an external FC-BGP neighbor and also when such an UPDATE message is propagated to an internal FC-BGP neighbor. The error checks specified in {{Section 6.3 of RFC4271}} are performed, except that for FC-BGP UPDATE messages the checks on the FC path attribute do not apply, and the following checks on the FC path attribute are performed instead:

1. Check that the FC path attribute is syntactically correct, including the length limits defined in {{fc-path-attribute}}. A structural error is handled per {{error-matrix}} (RFC 7606 {{RFC7606}}).
2. Construct the FC-to-AS_PATH Mapping of {{fcmapping}}. If the Mapping cannot be constructed (an order violation, a data-path adjacency violation, a receiver or origin boundary violation, a PC mismatch, a non-canonical or non-consecutive repeated CASN, or a missing FC segment for an FC-required AS per {{deployment}}), the check fails and the UPDATE message is marked 'Invalid' per {{error-matrix}}. A Mapping failure is a protocol violation and MUST NOT be downgraded to a mere absence.
3. If the UPDATE message was received from an FC-BGP neighbor that is not a member of the FC-BGP speaker's AS confederation, check to ensure that none of the FC segments contain a Flags field with the Confed_Segment flag set to 1. <!-- TODO: need more study of AS confederation -->
4. If the UPDATE message was received from an FC-BGP neighbor that is a member of the FC-BGP speaker's AS confederation, check to ensure that the FC segment corresponding to that peer contains a Flags field with the Flags-CS flag set to 1. See {{fcbgp-update}}.
5. If the UPDATE message was received from a neighbor that is not expected to set the Flags-RS bit to 1 (see {{rs-processing}}), check to ensure that the Flags-RS bit in the most recently added FC segment is equal to 0.
6. If the UPDATE message was received from a neighbor that is expected to set the Flags-RS bit to 1 (see {{rs-processing}}), check to ensure that the Flags-RS bit in the most recently added FC segment is equal to 1.
7. If the UPDATE message was received from a neighbor that is not expected to set the Flags-P2C bit or the Flags-P2P bit to 1 (see {{route-leak}}), check to ensure that the Flags-P2C bit and the Flags-P2P bit in the most recently added FC segment are both equal to 0.
8. If the UPDATE message was received from a neighbor that is expected to set the Flags-P2C bit or the Flags-P2P bit to 1 (see {{route-leak}}), check to ensure that the corresponding flag in the most recently added FC segment is equal to 1.

If any of the checks for the FC path attribute fail because of a structural or encoding error, the FC-BGP speaker MUST handle the FC path attribute as a malformed attribute following {{RFC7606}}, as specified in {{error-matrix}}. Failures that are protocol violations or security failures, such as a Mapping failure or a failed signature, are NOT UPDATE syntax errors: they produce the validation states defined in {{error-matrix}} and MUST NOT be collapsed into "treat-as-withdraw" as if the UPDATE message were malformed.

When an AS appears on the AS_PATH attribute and is FC-required for the UPDATE message per {{deployment}}, it MUST have an FC segment in the FC path attribute. The absence of such an FC segment is detected by the Mapping construction in check 2 above and is handled as 'Invalid' per {{error-matrix}}.

Then, the FC-BGP speaker iterates through the FC segments and validates the signatures. If the FC-BGP speaker encounters a signature corresponding to an algorithm suite indexed by an Algorithm ID that it does not support, that signature is not considered in the validation process for that FC segment. After completing the signature validation phase, the FC-BGP speaker determines the aggregate state of the UPDATE message as follows (see {{validation-states}}):

- If any FC segment is 'Invalid' or any check above fails, the UPDATE message is 'Invalid'.
- Otherwise, if at least one FC segment could not be verified because of an unsupported Algorithm ID or an unsupported protocol feature, the UPDATE message is 'Unsupported'. The presence of an unsupported algorithm MUST NOT cause the UPDATE message to be silently treated as an ordinary unsigned BGP route.
- Otherwise, if the FC coverage is partial because some ASes on the AS_PATH attribute are not FC-required ({{deployment}}), the UPDATE message is 'Incomplete'.
- Otherwise (every FC segment is valid and the FC coverage is complete), the UPDATE message is 'Valid'.

For each FC segment, the FC-BGP speaker processes FC-BGP UPDATE message validation with the following steps. As different FC segments are independent, it is RECOMMENDED to verify FC segments in parallel; see {{speedup-early-termination}}.

<!-- TODO: agree that this section is the FC-BGP analogue of the BGPsec signature validation; the differences are (a) each FC segment carries its own Algorithm ID, and (b) the input to the signature is the FC Signature Input of {{sig-input}}. -->

- Step 1: Locate the public key needed to verify the signature in the current FC segment. To do this, consult the valid RPKI router certificate data and look up all valid <AS Number, Public Key, Subject Key Identifier> triples in which the AS matches the Current AS Number (CASN) in the corresponding FC segment. Of these triples that match the AS number, check whether there is an Subject Key Identifier (SKI) that matches the value in the SKI field of the FC segment. If this check finds no such matching SKI value, then mark the entire FC segment as 'Invalid' and stop.
- Step 2: Construct the digest input for the current FC segment exactly as specified in {{sig-input}}, using the PASN, CASN, NASN, SKI, Algorithm ID, Flags, PC, AFI/SAFI, Prefix, and Prefix Length of the FC segment and of the UPDATE message. Note that if an FC-BGP speaker uses multiple AS numbers (e.g., the FC-BGP speaker is a member of an AS confederation), the AS number used for the CASN MUST be the AS number announced in the BGP OPEN message for the session over which the FC-BGP UPDATE message was received. All three AS numbers in one FC segment follow this rule.
- Step 3: Use the signature validation algorithm (for the given algorithm suite) to verify the signature in the current segment. That is, invoke the signature validation algorithm on the following three inputs: the value of the Signature field in the current FC segment, the digest constructed in Step 2 above, and the public key obtained from the valid RPKI data in Step 1 above. If the signature validation algorithm determines that the signature is invalid, then mark the entire FC segment as 'Invalid' and stop. If the signature validation algorithm determines that the signature is valid, then the FC segment is marked as 'Valid' and validation continues with the following FC segments.

When one FC Segment has set the Flags-P2C flag to 1, the subsequent FC segments added by the following ASes MUST all set the Flags-P2C flag to 1 in their corresponding FC segments. The Flags-P2C flag is set to 1 only when the role of its neighbor, to whom the propagator AS sends routes, is Customer or RS-Client. The Flags-P2P flag is set to 1 only when the role of its neighbor, to whom the propagator AS sends routes, is Peer.

The following ingress procedure applies to the processing of the Flags-P2C flag and the Flags-P2P flag on route receipt:

1. If a route, with the Flags-P2C flag in the recently added FC segment, is received from a Customer, a Peer, or an RS-Client, then it is a route leak and MUST be considered ineligible.
2. If a route is received from a Peer (i.e., remote AS with a Peer Role) and with the Flags-P2P flag in both of the two most recently consecutively added FC segments, then it is a route leak and MUST be considered ineligible.

# Implementations, Operations, and Management Considerations

## Algorithms and Extensibility {#algorithms-extensibility}

The content of Algorithm Suite Considerations defined in {{Section 6.1 of RFC8205}} and the content of Considerations for the SKI Size defined in {{Section 6.2 of RFC8205}} apply to FC-BGP, with the difference that each FC segment carries its own Algorithm ID field. This document registers the Algorithm ID 1, whose suite is ECDSA over P-256 with SHA-256, as defined in {{RFC8208}} (see {{iana-considerations}}).

The following rules govern algorithm registration, transition, and deprecation:

- An Algorithm ID MUST be registered with IANA before it is used in FC segments (see {{iana-considerations}}).
- An FC-BGP speaker MUST NOT generate an FC segment with an Algorithm ID for which it has not placed the corresponding router certificate in the RPKI repository.
- During a transition between algorithm suites, a speaker MAY generate FC segments with different Algorithm IDs for different FC segments, provided that the private key and certificate for each Algorithm ID are available. Each FC segment's Algorithm ID is covered by that FC segment's own signature ({{sig-input}}), so mixed-Algorithm-ID FC lists are cryptographically self-contained.
- An FC-BGP speaker that encounters an FC segment whose Algorithm ID it does not recognize MUST NOT verify that FC segment and MUST handle it per {{error-matrix}}: the UPDATE message becomes 'Unsupported' (or 'Invalid' if another failure is present) per {{validation-states}}. An unsupported algorithm MUST NOT cause the UPDATE message to be silently treated as an ordinary unsigned BGP route.
- An algorithm MUST NOT be removed from use without a documented transition plan. Before deprecating an algorithm, operators MUST stop generating FC segments under it, allow enough time for the remaining FC segments signed under it to expire from the network, and then remove the Algorithm ID from the registry.
- The FC Segment Protocol-Version in {{sig-input}} provides the version of the FC segment format; the handling of an unknown Protocol-Version follows {{error-matrix}}.

## Speedup and Early Termination of Signature Verification {#speedup-early-termination}

It is advantageous for an implementation to establish a parallel verification process for FC-BGP if the router's processor supports such operations. As each FC segment contains the integral data that needs to be verified, parallel verification can significantly enhance the efficiency and speed of the validation process. By utilizing parallel processing capabilities, an implementation can simultaneously verify multiple FC segments, thereby reducing the overall verification time. This is particularly beneficial in scenarios where the FC path attribute contains a substantial number of segments or in high-traffic networks with a large volume of FC-BGP UPDATE messages. Implementations that leverage parallel verification take advantage of the processing power available in modern router processors. This allows for more efficient and faster verification, ensuring that the FC-BGP UPDATE messages are promptly validated and routed accordingly.

However, it's important to note that the feasibility of parallel verification depends on the specific capabilities and constraints of the router's processor. Implementations SHOULD consider factors such as available resources, concurrency limitations, and the impact on overall system performance when implementing parallel verification processes. Overall, setting up a parallel verification process for FC-BGP, if feasible, can contribute to improved performance and responsiveness in validating FC segments, further enhancing the reliability and efficiency of the FC-BGP protocol.

During the validation of an FC-BGP UPDATE message, route processor performance speedup can be achieved by incorporating the following observations. These observations provide valuable insights into optimizing the validation process and reducing the workload on the route processor. One of the key observations is that the FC-BGP UPDATE message can be marked 'Valid' only if all FC segments are marked as 'Valid' in the validation steps. This means that if an FC segment is marked as 'Invalid' or 'Unsupported', there is no need to continue verifying the remaining unverified FC segments. This optimization can significantly reduce the processing time and workload on the route processor. Furthermore, when the FC-BGP UPDATE message is selected as the best path, the FC-BGP speaker appends its own FC segment, including the appropriate signature generated with the corresponding algorithm, to the FC path attribute. This ensures that the updated path is propagated correctly.

Additionally, an FC-BGP UPDATE message is 'Invalid' if at least one signature in an FC segment is invalid. Thus, the verification process for an FC segment can terminate early as soon as the first invalid signature is encountered. There is no need to continue validating the remaining signatures in that FC segment.

By incorporating these observations, an FC-BGP implementation can achieve significant performance improvements and reduce the computational burden on the route processor. It allows for more efficient validation of FC-BGP UPDATE messages, ensuring the integrity and security of the routing information while maximizing system resources.

## Signature Generation Load and Performance Metrics {#signature-count}

A single FC-BGP speaker generates one FC segment per (prefix, next-hop AS, propagation leg) for each route it propagates to external neighbors (see {{fcbgp-update}}). Because the Prefix and the Nexthop AS Number are covered by the signature ({{sig-input}}), the number of signatures generated by a speaker grows with the number of prefixes and with the number of distinct externally reachable next hops. This is a design property that bounds the amount of path information each signature covers; operators MUST account for this growth when dimensioning the control plane.

Where the path semantics are identical, an implementation MAY reuse a signature or a validation result: for the same prefix, the same next-hop AS, the same FCList, and the same set of FC fields, the signature computation is deterministic and MAY be cached and reused, provided that the reused signature is valid for the same FC Signature Input ({{sig-input}}). Batch or parallel verification, and caching of verification results, MUST NOT change the validation state produced by the algorithm of {{validation-steps}}; they are performance optimizations only.

To support capacity planning, FC-BGP implementations SHOULD expose, at a minimum, the following metrics:

- the number of FC-BGP UPDATE messages processed per second;
- the signature generation latency per UPDATE message;
- the signature verification latency for FCList of different lengths;
- the CPU utilization attributable to FC generation and FC verification;
- the additional octets contributed by the FC path attribute to the UPDATE message;
- the BGP convergence time with FC-BGP enabled; and
- the memory footprint growth with the size of the RIB and of the RPKI dependency index ({{rpkideps}}).

## Deferring Validation {#deferring-validation}

When an FC-BGP speaker receives an exceptionally large number of UPDATE messages simultaneously, though it can use parallel verification to speed up the validation, it can be beneficial to defer the validation of incoming FC-BGP UPDATE messages. The decision to defer the validation process may depend on the local policy of the FC-BGP speaker, taking into account factors such as available resources and system load.

By deferring the validation of these messages, the FC-BGP speaker can prioritize its processing power and resources to handle other critical tasks or ongoing operations. Deferring the validation allows the FC-BGP speaker to temporarily postpone the resource-intensive validation process until it can allocate sufficient resources to handle the influx of incoming messages effectively.

The implementation SHOULD provide visibility to the operator regarding the deferment of validation and the status of the deferred messages. This visibility enables the operator to have awareness of the deferred messages and understand the current state of the system. This information is crucial for monitoring and managing the FC-BGP speaker's behavior, ensuring that the operator can make informed decisions based on the system's status.

The validation of an FC-BGP UPDATE message proceeds through the following states:

~~~~
 Received
    |
    v
 Syntax Check
    |
    v
 Pending Validation
    |
    +------> Valid
    |
    +------> Invalid
    |
    +------> Unsupported
~~~~
{: #fig-val-state title="Validation state machine for an FC-BGP UPDATE message."}

The following rules apply while an UPDATE message is in the 'Pending Validation' state:

- A route in the 'Pending Validation' state MAY be admitted to the candidate route set as an ordinary BGP route by local policy, but MUST NOT be reported as validated (see {{validation-states}}).
- An FC-BGP speaker MAY forward a route in the 'Pending Validation' state, provided that it does not claim a definitive validation state for it.
- The time an UPDATE message MAY remain in the 'Pending Validation' state is bounded by local policy; on timeout, the implementation SHOULD re-evaluate the route and keep the 'Pending Validation' state visible to the operator.
- Validation MAY be cancelled (for example, when the UPDATE message is withdrawn, when its route is no longer in the candidate set, or on session reset). Cancellation MUST NOT leave the route reported with a definitive validation state; the affected routes MUST be re-queued or reported as 'Not Validated'.
- When the validation of a previously pending UPDATE message completes, the FC-BGP speaker SHOULD re-run the best path selection process for routes affected by the new state (see {{BGP-route-selection}}).
- When resources are exhausted, an FC-BGP speaker MAY rate-limit the validation of UPDATE messages received from specific neighbors, but MUST NOT silently treat unvalidated messages as validated (see {{validation-states}}).

## BGP Route Selection {#BGP-route-selection}

<!-- TODO: An expert says the ROV takes the highest level in BGP route selection. Need confirmation. -->

While FC-BGP does modify the BGP route selection result, it is not the primary intention of FC-BGP to modify the BGP route selection process itself. Instead, FC-BGP focuses on providing an additional layer of validation and verification for BGP UPDATE messages. The validation state produced by {{validation-steps}} (see {{validation-states}}) is an input to route selection; the protocol does not prescribe a single mapping from validation states to routing decisions. The per-state route-selection behavior that the protocol requires and permits is defined in {{validation-states}}.

However, the handling of FC-BGP validation states, as well as the integration of FC-BGP with the BGP route selection, is indeed a matter of local policy. FC-BGP implementations SHOULD provide mechanisms that allow operators to define and configure their own local policies on a per-session basis. This flexibility enables operators to customize the behavior of FC-BGP based on their specific requirements and preferences.

By allowing operators to set local policies, FC-BGP implementations empower them to control how the validation status of FC-BGP UPDATE messages influences the BGP route selection process. Operators may choose to treat FC-BGP validation status differently for UPDATE messages received over different BGP sessions, based on their network's needs and security considerations.

To ensure consistency and interoperability, it is RECOMMENDED that FC-BGP implementations treat the priority of FC-BGP UPDATE messages at the same level as Route Origin Validation (ROV). This means that the validation status of FC-BGP UPDATE messages should be considered alongside other route selection criteria, such as path attributes, AS path length, and local preference.

## Non-deterministic Signature Algorithms

The non-deterministic nature of many signature algorithms can introduce variations in the signatures produced, even when signing the same data with the same key. This means that if an FC-BGP router receives two FC-BGP UPDATE messages from the same peer, for the same prefix, with the same FC path attribute except the signature fields, the signature fields MAY differ when using a non-deterministic signature algorithm. Note that if the sender caches and reuses the previous signature, the two sets of signature fields will not differ. This applies specifically to deterministic signature algorithms, where the signature fields between the two UPDATE messages MUST be identical.

Considering these observations, an FC-BGP implementation MAY incorporate optimizations in the UPDATE validation processing. These optimizations can take advantage of the non-deterministic nature of signature algorithms to reduce computational overhead. For example, if an FC-BGP router has already validated an FC segment and its corresponding signature in a previous UPDATE message from the same peer, it may choose to cache and reuse the previous validation result. This can help avoid redundant computations for subsequent UPDATE messages with the same FC path attribute and SKIs, as long as the sender does not generate new signatures.

By incorporating such optimizations, an implementation can reduce the computational load and processing time needed for validating FC-BGP UPDATE messages. However, it is important to ensure that the implementation adheres to the requirements and specifications of the FC-BGP protocol while considering the performance benefits of these optimizations.

## AS Number Processing {#asn-processing}

This section defines how FC-BGP handles AS number changes and special cases, and which cases remain inside the FC-BGP protection scope. The process for Private AS Numbers used in BGPsec speakers defined in {{Section 7.5 of RFC8205}} applies, with the additions below.

### CASN, Certificate, Session, and AS_PATH Identity

The Current AS Number (CASN) of an FC segment is the identity on which the whole validation of that FC segment rests. The following MUST all refer to the same AS:

- the CASN of the FC segment;
- the AS number in the Subject field of the RPKI router certificate used to verify the FC segment (see {{fcbgp-update}} and {{RFC8209}}); and
- for the FC segment of the immediate neighbor, the AS number announced in the BGP OPEN message of the session over which the FC-BGP UPDATE message was received (see {{validation-steps}}).

### AS Number Migration

When an AS migrates from one AS number to another (e.g., an AS number change following a merger), FC segments signed under the old AS number cannot be validated against a router certificate for the new AS number, and the FC-to-AS_PATH Mapping ({{fcmapping}}) would not be satisfiable for FCs whose CASN is the old AS number. The migrating AS MUST therefore publish a router certificate for the new AS number, generate all subsequent FC segments with the new AS number as the CASN, and stop using the old AS number in new FC segments before traffic is moved to the new AS number. This document does not define a mechanism analogous to the pCount=0 convention of BGPsec {{RFC8205}} {{RFC8206}}; a receiver MUST reject an FC segment whose CASN does not have a corresponding valid router certificate (see {{error-matrix}}). <!-- TODO: decide whether an explicit migration mechanism (e.g., a "PC=0"-like placeholder) is needed in a future revision. -->

### Private AS Numbers

Private AS numbers may appear in FC segments. The removal or replacement of private AS numbers from the AS_PATH attribute (e.g., by an edge router that strips them per local policy) changes the AS_PATH attribute of the UPDATE message. Because the FC-to-AS_PATH Mapping binds the FCList to the AS_PATH attribute, the FC path attribute MUST NOT be propagated in an UPDATE message whose AS_PATH attribute has been modified after the FCList was constructed. A speaker that modifies the AS_PATH attribute in a way that is not reflected in the FCList MUST either generate a fresh FC path attribute consistent with the new AS_PATH attribute or remove the FC path attribute from the UPDATE message before propagation.

### Repeated and Changed ASNs

- Consecutive repetitions of an AS number in the AS_PATH attribute are represented by the PC field of a single FC segment and are protected as described in {{fcmapping}}.
- Non-consecutive repetitions of an AS number in the AS_PATH attribute indicate an invalid AS_PATH attribute or a loop, and the corresponding Mapping failures are handled per {{fcmapping}} and {{error-matrix}}.
- AS Confederation membership changes are governed by {{fcbgp-update}}: on entering a confederation, an FC segment for the Confederation Identifier with the Confed_Segment flag set to 1 is added, and on leaving a confederation the confederation members' FC segments are removed and replaced.

### Certificate and ASN Mismatch

If the CASN of an FC segment does not correspond to any valid router certificate for that AS number (for example, because the certificate was issued for a different AS number), the FC segment cannot be verified and is handled per {{error-matrix}}. This covers the case where a router certificate and the AS number actually used in the BGP session are inconsistent.

### Supported and Out-of-Scope Cases

FC-BGP protects AS numbers that appear in the AS_PATH attribute of the UPDATE message. AS numbers that are not FC-required ({{deployment}}) are outside the protection scope for the route in question. Private AS numbers and AS numbers in AS Confederation are protected only while they remain in the AS_PATH attribute (or in the FCList under the Flags-RS or Flags-CS rules) and only for as long as the corresponding AS is FC-required.

## Robustness Considerations for Accessing RPKI Data

As there is a mature RPKI to Router protocol {{RFC8210}}, the implementation is REQUIRED to use this protocol to access the RPKI data. The content defined in {{Section 7.6 of RFC8205}} also applies here.

## Dependency Handling on RPKI State Change {#rpkideps}

When the state of a router certificate changes, the validity of every FC segment that references that certificate can change. To make revalidation on RPKI state change scalable, an FC-BGP speaker SHOULD maintain a dependency index keyed by the Subject Key Identifier (SKI) and, where a SKI is shared, by the Current AS Number (CASN) of the FC segments:

- a key (SKI, CASN) maps to the set of FC segments whose SKI and CASN fields match; and
- each such FC segment maps to the FC-BGP UPDATE messages that contain it.

When the state of a router certificate changes, the FC-BGP speaker MUST:

1. locate the (SKI, CASN) entries affected by the certificate change;
2. identify the FC segments that reference the affected certificate through those entries;
3. locate the FC-BGP UPDATE messages that contain those FC segments;
4. re-run the signature validation and, if needed, the FC-to-AS_PATH Mapping for the affected UPDATE messages ({{validation-steps}}); and
5. update the FC validation state and, if the state of an UPDATE message changed, re-run the best path selection process according to local policy (see {{BGP-route-selection}}).

The following certificate state changes have distinct semantics:

- Revocation: the certificate MUST no longer be used to verify FC segments; affected FC segments become 'Invalid' ({{validation-states}}).
- Expiration: same as revocation once the certificate's validity period has ended.
- Replacement (a new certificate for the same AS number): new FC segments MUST be signed with the new certificate; FC segments signed under the replaced certificate are handled as signatures whose key is no longer valid, per {{error-matrix}}.
- Key rollover: until the new key is valid and the old key has expired or been revoked, the same AS MAY have FC segments signed with either key; the SKI identifies which key to use ({{fcbgp-update}}).

## Graceful Restart

During Graceful Restart (GR), restarting and receiving FC-BGP speakers MUST follow the procedures specified in {{RFC4724}} for restarting and receiving BGP speakers, respectively. In particular, the behavior of retaining the forwarding state for the routes in the Loc-RIB {{RFC4271}} and marking them as stale, as well as not differentiating between stale routing information and other information during forwarding, will be the same as the behavior specified in {{RFC4724}}.

## Robustness of Secret Random Number in ECDSA

As both FC-BGP and BGPsec use ECDSA, the content of Robustness of Secret Random Number in ECDSA defined in {{Section 7.8 of RFC8205}} applies here.

<!-- TODO: check and rewrite the following parts copied from BGPsec -->

## Incremental/Partial Deployment Considerations {#comparison}

The core difference between FC-BGP and BGPsec is that BGPsec is a path-level authentication approach, whereas FC-BGP is a pathlet-driven authentication approach.

In design, FC-BGP does not modify the AS_PATH attribute. It defines a new transitive path attribute to transport the FC segments so that the legacy ASs can forward this attribute to their peers. Thus, FC-BGP is natively compatible with BGP and supports partial deployment. It differs from BGPsec, which replaces the AS_PATH attribute with a new Secure_Path information of the BGPsec_Path attribute.

As for incremental/partial deployment considerations, in Section 5.1.1 of {{FC-ARXIV}}, we have proved that the adversary cannot forge a valid AS path when FC-BGP is universally deployed. Section 5.1.2 of {{FC-ARXIV}} analyzes the benefits of FC-BGP in the case of partial deployment. The results show that FC-BGP provides more benefits than BGPsec in partial deployment. As a result, attackers are forced to pretend to be at least two hops away from the destination AS, which reduces the probability of successful path hijacks.

## Co-existence with BGPsec {#coexist_bgpsec}

BGPsec and FC-BGP are two different mechanisms for authenticating the path of a BGP UPDATE message. This section defines their behavior when they are enabled on the same speaker or on neighboring speakers. Three cases are distinguished:

1. Only FC-BGP is in use:
   : The FC path attribute is generated and validated against the AS_PATH attribute, as defined in {{fcbgp-update}} and {{fcmapping}}. The BGPsec_Path attribute is absent.
2. Only BGPsec is in use:
   : The BGP speaker follows {{RFC8205}}; the FC path attribute is absent and no FC processing is performed.
3. Both BGPsec and FC-BGP are in use:
   : The BGP speaker processes the BGPsec UPDATE message first, then processes the FC-BGP UPDATE message, as described below. It is NOT RECOMMENDED that both be enabled together except during a migration period.

When both mechanisms are enabled on the same speaker, the BGPsec_Path attribute is the authoritative path representation: the BGP speaker SHOULD generate the FC segments from the BGPsec_Path and MP_REACH_NLRI attributes, and SHOULD validate the FC segments against the BGPsec_Path and MP_REACH_NLRI attributes, of the same UPDATE message, rather than against the AS_PATH attribute. The receiver MUST use the same path representation (BGPsec_Path or AS_PATH) for both generation and validation; using one representation at generation and the other at validation results in an FC-to-AS_PATH Mapping failure ({{fcmapping}}) or a signature mismatch and MUST be treated as 'Invalid' per {{error-matrix}}.

When the BGPsec validation of an UPDATE message fails, the UPDATE message is 'Invalid' for BGPsec. The FC-BGP validation of the same UPDATE message MAY still be performed; however, the FC path attribute does not repair the BGPsec failure. Whether an UPDATE message whose BGPsec validation failed may still be selected on the basis of the FC-BGP validation result is a matter of local policy under {{BGP-route-selection}}; the protocol does not provide a default that silently ignores the BGPsec failure. In all cases, a BGPsec-enabled speaker that propagates a route whose BGPsec_Path attribute it removed because BGPsec validation failed MUST follow the rules of {{RFC8205}} and MUST NOT attach an FC path attribute that was validated against the removed BGPsec_Path attribute.

In summary, an implementation SHOULD be able to process a coexistence UPDATE message; the priority is that the FC segments and the path representation used for their generation and validation are always consistent, and that a BGPsec failure is never silently overridden by FC-BGP validation.

# Security Considerations {#security-considerations}

## Security Guarantees

This section states the security guarantees that FC-BGP provides and the conditions (preconditions) under which each guarantee holds. Each guarantee is conditional: it holds only when the stated preconditions are met. All of the guarantees below are control-plane guarantees about the FC path attribute; none of them, by itself, constrains the data plane.

| Security goal | Preconditions | Protocol mechanism |
| --- | --- | --- |
| AS identity authentication | The signer holds a valid RPKI router certificate for the AS; the FC segment verifies under that certificate; the CASN matches the certificate subject and, for the immediate neighbor, the session's BGP OPEN AS number ({{asn-processing}}). | Signature validation ({{validation-steps}}); RPKI router certificates ({{RFC8209}}). |
| Pathlet (path segment) integrity | The full FC Signature Input of {{sig-input}} is covered by the signature and the signature is valid DER; the FC segment is in its correct position in the FCList. | {{sig-input}}; {{fcmapping}}. |
| AS_PATH consistency | The FC-to-AS_PATH Mapping of {{fcmapping}} completes: FCList order, data-path adjacency, PC, and coverage all hold. | {{fcmapping}}. |
| Origin authorization | FC-BGP validation is combined with route origin validation (ROV) ({{RFC6811}}); a valid ROA authorizes the origin AS for the prefix. | {{fcbgp-update}}; {{RFC6811}}. |
| Route leak detection | The BGP Role capability {{RFC9234}} or equivalent out-of-band role information is present, and the Flags-P2C / Flags-P2P rules of {{route-leak}} are applied with correct validation. | {{route-leak}}. |
| Data-plane path consistency | NOT guaranteed by FC-BGP alone. Control-plane FC signatures do not prove that the actual data-plane forwarding path matches the announced AS_PATH. Claiming this guarantee requires additional mechanisms (for example, forwarding-path attestation) and additional trust assumptions, which are outside the scope of this document. | None in this document. |

FC-BGP allows an AS to independently prove its BGP routing decisions with publicly verifiable cryptography commitments, based on which any on-path AS can verify the authenticity of a BGP path. The security benefits of FC-BGP are not binary, i.e., secure or non-secure; they increase monotonically with the deployment rate of FC-BGP, that is, the percentage of ASes that are upgraded to support FC-BGP. In partial deployment, the FC coverage is reflected in the 'Incomplete' validation state ({{validation-states}}), so that the authenticated portion of a path can be distinguished from the unauthenticated portion.

## Mitigation of Denial-of-Service Attacks

<!-- This is also mentioned at IETF 118 by Keyur: you may think about decentralizing this to solve different kinds of DoS attacks.
https://github.com/FCBGP/fcbgp-protocol/discussions/8 for more information. -->
The FC-BGP UPDATE process, due to its involvement in numerous cryptographic operations, becomes vulnerable to Denial-of-Service (DoS) attacks targeting FC-BGP speakers. This section addresses the mitigation strategies tailored for the specific DoS threats the FC-BGP protocol poses. To prevent the Denial-of-Service (DoS) attacks faced by the FC-BGP control plane mechanism, there is no need to put in more effort than BGPsec.

To reduce the impact of DoS attacks, FC-BGP speakers SHOULD employ an UPDATE validation algorithm that prioritizes inexpensive checks (such as syntax checks) before proceeding to more resource-intensive operations (like signature verification). The validation algorithm described in {{validation-steps}} is designed to sequence checks in order of likely expense, starting with less costly operations. However, the actual cost of executing these validation steps can vary across different implementations, and the algorithm in {{validation-steps}} may not offer the optimal level of DoS protection for all cases.

Moreover, the transmission of UPDATE messages with the FC path attribute, which entails a multitude of signatures, is a potential vector for denial-of-service attacks. To counter this, implementations of the validation algorithm must cease signature verification immediately upon encountering an invalid signature. This prevents prolonged sequences of invalid signatures from being exploited for DoS purposes. Additionally, implementations can further mitigate such attacks by limiting validation efforts to only those UPDATE messages that, if found to be valid, would be chosen as the best path. In other words, if an UPDATE message includes a route that would be disqualified by the best path selection process for some reason (such as an excessively long AS path), it is OPTIONALLY to determine its FC-BGP validity status.

## Route Server

When the Route Server populates its FC Segment into the FC path attribute, it is secure as the path is fully deployed.

When the Route Server fails to insert the FC Segment, no matter whether its ASN is listed in the AS path, it is considered a partial deployment, which poses a risk of path forgery.


## Additional Security Considerations

### Three AS Numbers {#sec-three-asn}

<!-- TODO: prove that insertion of 0 does not include route loop/BGP dispute in security consideration. Question asked by Keyur. See [Discussions: IETF119](https://github.com/FCBGP/fcbgp-protocol/discussions/10) for more information. -->

An FC segment contains only partial path information, and FCs in the FCList are independent. To prevent BGP Path Splicing attacks, we propose to use the triplet <Previous AS Number, Current AS Number, Nexthop AS Number> to locate the pathlet information.

But if there is no previous hop, i.e., this is the origin AS that tries to add its FC segment to the BGP UPDATE message, the Previous AS Number SHOULD be populated with 0. But, carefully, AS 0 SHOULD only be used in this case.

In the context of BGP {{RFC4271}}, to detect an AS routing loop, it scans the full AS path (as specified in the AS_PATH attribute) and checks that the autonomous system number of the local system does not appear in the AS path. As outlined in {{RFC7607}}, Autonomous System 0 was listed in the IANA Autonomous System Number Registry as "Reserved - May be used to identify non-routed networks". So, there should be no AS 0 in the AS_PATH attribute of the BGP UPDATE message. Therefore, AS 0 could be used to populate the PASN field when there are no previous AS hops in the AS path.

### MISC

For a discussion of the BGPsec threat model and related security considerations, please see {{RFC7132}}. The security considerations of {{RFC4272}} also apply to FC-BGP.

## Threat Model {#threat-model}

This section presents a systematic threat model for FC-BGP. It complements the BGPsec threat model of {{RFC7132}}; the security considerations of {{RFC4272}} also apply to FC-BGP. For each threat, the table states the assumed attacker capability, the attacker's goal, whether FC-BGP can detect the attack, the deployment conditions required for detection, the behavior when detection fails, and the mechanisms that complement FC-BGP.

| Threat | Attacker capability | FC-BGP detection | Required deployment | Behavior if not detected | Complementary mechanism |
| --- | --- | --- | --- | --- | --- |
| Malicious origin AS | The origin AS signs (or does not sign) an FC segment for a prefix it does not own. | No. FC-BGP does not prove origin authenticity: an FC segment for an unauthorized prefix still verifies. | Full FC-BGP on the origin and the path, plus ROV. | The route is accepted as 'Valid' even though the origin is not authorized. | Origin validation (ROV) {{RFC6811}} with ROAs. |
| Malicious intermediate AS | The AS lies about its position on the path; it removes or re-orders FC segments. | Yes, if the tampered AS is FC-required: the FC-to-AS_PATH Mapping ({{fcmapping}}) fails, or the signatures do not verify, and the UPDATE message becomes 'Invalid'. | The tampered AS or an adjacent hop is FC-required and FC-enabled, and FC coverage is complete. | A path with a missing or misplaced FC for an FC-required AS is accepted. | BGP loop prevention; RPKI-based neighbor validation; operator monitoring. |
| Malicious downstream AS | The AS advertises a path that it did not receive. | Yes, if the receiver enforces FC-required semantics: the NASN of the most recent FC segment does not equal the receiver's AS number ({{fcmapping}} receiver boundary). | FC enforcement by the receiver and complete coverage. | A forged path with an out-of-place boundary FC is accepted. | Neighbor verification; RPKI; route policies. |
| FC removal (FC stripping) | An on-path AS removes FC segments that it cannot forge. | Yes, if the removed AS was FC-required: the Mapping shows a gap and the UPDATE message is 'Invalid' ({{deployment}}, {{fcmapping}}). | The removed AS is FC-required and its FC presence can be independently established. | The path is treated as 'Incomplete' (partial deployment) and accepted with partial coverage. | RPKI router certificates; operational verification of coverage. |
| AS_PATH modification | An on-path AS adds, removes, or re-orders AS numbers. | Yes, for additions or re-ordering of covered ASes: Mapping or signature failure. For the removal of an AS, see 'FC removal' above. | FC coverage of the modified portion. | The modified path is accepted. | {{fcmapping}}; route monitoring. |
| Replay of old FC segments | An attacker replays a previously valid FCList with an older path or older certificate state. | Partially. A replayed FC whose PASN / CASN / NASN no longer match the current AS_PATH and certificate state fails the Mapping or signature validation. | Fresh certificate state at the receiver; complete coverage. | A replayed FC is accepted if the path and certificates are unchanged. | Certificate time validity; RPKI state tracking ({{rpkideps}}). |
| Route leak | An AS advertises a route learned from a peer or provider to customers, or reverses roles. | Yes, when the BGP Role capability {{RFC9234}} is present and the Flags-P2C / Flags-P2P rules of {{route-leak}} are enforced. | Role capability exchange and enforcement of {{route-leak}}. | The leaked route is selected. | BGP Roles {{RFC9234}}; OTC; ASPA. |
| Malicious route server | A route server inserts, omits, or alters an FC segment with the Flags-RS bit set to 1. | Yes. The RS's FC is authenticated by its own router certificate, and the PASN / NASN adjacency and Flags-RS handling are checked ({{rs-processing}}). | The RS is FC-enabled and the receiver enforces the Flags-RS checks. | A forged or omitted RS FC is treated as partial deployment. | {{rs-processing}}; IXP operator verification. |
| Certificate revocation / private key exposure | An attacker uses a revoked or compromised key. | Yes, when revocation information is fresh: the affected certificate becomes invalid and the affected FC segments become 'Invalid' ({{rpkideps}}). | Timely RPKI state propagation. | Signatures under the exposed key continue to verify until the certificate is withdrawn. | RPKI; key management {{RFC8635}}. |
| Algorithm downgrade | An attacker selects an unsupported Algorithm ID that the receiver cannot verify. | The FC segment is marked 'Unsupported'; the UPDATE message MUST NOT be silently treated as ordinary and unauthenticated ({{validation-states}}). | Receiver-side policy for 'Unsupported' routes. | The route is treated as ordinary if local policy so chooses. | Algorithm registration and deprecation ({{algorithms-extensibility}}). |
| DoS / resource exhaustion | An attacker floods a speaker with FC-BGP UPDATE messages that require expensive verification. | The protocol sequences checks by cost and allows early termination, deferral, and rate limiting ({{validation-steps}}, {{deferring-validation}}). | Implementation limits and operator configuration. | Excessive CPU or memory use; legitimate UPDATE messages are delayed. | {{validation-steps}}; {{deferring-validation}}; rate limiting; the limits of {{fc-path-attribute}}. |


# IANA Considerations {#iana-considerations}

TBD. Wait for IANA to assign FC-BGP-UPDATE-PATH-ATTRIBUTE-TYPE.

TBD. Regist Flags. The leftmost bit is the Confed_Segment flag, and the second highest/leftmost bit is the Route_Server flag in this document.

TBD. A new OID should be assigned for keys used in FC-BGP.

AS number 0 is used here to populate the PASN in an FC segment where there is no previous hop for an AS, i.e., the origin AS when adding the FC segment to the FC-BGP UPDATE message.

--- back

# Attachment

## Comparison to Other Technologies

### BGPsec

For basic comparison, please see {{Introduction}}.

#### Deployment Benefits Analysis {#DeploymentBenefitsAnalysis}

One of the core differences between FC-BGP and BGPsec is the partial deployment scenario. It is difficult for FC-BGP and BGPsec to make an entire deployment, which is an evolution process.

The propagation of the BGP UPDATE message can be simplified as a line. Take the following propagation path as an example.

~~~~~~
AS(1) --- AS(2) -...- AS(k-1) ---- AS(k) -...- AS(n)
                             \               /
                              \--- AS(m) ---/
~~~~~~
{: #fig-deploy-topo title="An BGP UPDATE propagation path example."}

In this propagation path, the settings are:
- A path from AS(1) to AS(n) with N-1 hops, where FC-BGP is deployed consecutively from AS(1) to AS(k-1). AS(k) is the first legacy AS that doesn't enable FC-BGP.
- An upgraded AS(n) located after AS(k) tries to validate its received BGP path.
- A compromised AS(m) intents to hijack the traffic from AS(n) to AS(1).

Suppose the distance between AS(m), i.e., the compromised AS, and AS(n) is L hops. The best choice for AS(m) is to pretend to be a neighbor of AS(k) and construct a fake path AS(1)-AS(2)-...-AS(k)-AS(m)-...-AS(n) with length `K-1+1+L` hops. AS(m) can successfully hijack the traffic only if `N-1>K-1+1+L`, which implies that has to be smaller than `N-L-1`.

In the full path deployment scenario, i.e., `K=N+1`, FC-BGP and BGPsec have the same path protection rate. However, in the partial path deployment scenario, i.e., `K<N-L-1`, FC-BGP can protect more paths than BGPsec.

The conclusion can be drawn that FC-BGP provides strictly more security benefits than BGPsec in partial/incremental deployment.

Under full deployment, FC-BGP and BGPsec achieve comparable security benefits as both are capable of preserving the authenticity and immutability of the AS_PATH attribute within BGP UPDATE messages. From a security guarantee perspective, FC-BGP establishes a cryptographic commitment and propagates this commitment across the entire network. Through this mechanism, each BGP router can verify that it has received routes from the preceding AS and has forwarded them to the subsequent AS in the path. This approach ensures security through modular pathlets, which are incrementally validated and aggregated to construct a fully authenticated end-to-end routing path. In contrast, BGPsec does not incorporate such a commitment-based framework, relying instead on cryptographic path validation at each AS hop without modular decomposition of path security.

### ASPA

The ASPA {{ASPA-Profile}} {{ASPA-Verification}} mechanism is designed to solve the problem of route leak with RPKI ASPA signed objects and AS_PATH attribute. It is an off-path mechanism with a lightweight cryptographic cost for BGP routers. However, it does not protect the AS_PATH attribute. Thus, ASPA and FC-BGP are complementary technologies.

### Only to Customer (OTC) Attribute

OTC is a route-leak detection and prevention mechanism. However, the OTC value itself is not protected. It can be forged. With the signature, FC-BGP can protect the Flags-P2C flag and the Flags-P2P flag. So the FC-BGP route leak prevention mechanism is complementary to the OTC attribute.

## Test Vectors {#test-vectors}

This section specifies the interoperability test cases for FC-BGP validation. Each row gives the input, the expected validation state (per {{validation-states}}), the expected route behavior, and the error handling (per {{error-matrix}}). Implementers SHOULD reproduce these cases in their test suites. Concrete byte-level vectors, including the wire encoding of the FC path attribute, the exact FC Signature Input of {{sig-input}}, and the digital signatures, are published together with the implementation described in {{implementation-status}}.

| Case | Input | Expected validation state | Expected route behavior | Error handling |
| --- | --- | --- | --- | --- |
| 1. Legal IPv4 FC | AS_PATH = [65001, 65002]; FCList complete and cryptographically valid; IPv4 unicast; PC = 1 | 'Valid' | MAY be admitted to the candidate route set per local policy | None |
| 2. Legal IPv6 FC | Same as case 1, but with an IPv6 unicast NLRI (AFI 2, SAFI 1) | 'Valid' | MAY be admitted to the candidate route set per local policy | None |
| 3. FC order reversed | FCList is in the reverse order of the AS_PATH attribute | 'Invalid' | MUST NOT be treated as FC-BGP-valid | Mapping failure per {{fcmapping}} and {{error-matrix}}; not a syntax error |
| 4. Missing FC for an FC-required AS | An AS that is FC-required ({{deployment}}) has no corresponding FC segment | 'Invalid' | MUST NOT be treated as FC-BGP-valid | Mapping failure; MUST NOT be downgraded to a mere absence |
| 5. Missing FC for a legacy AS | An AS without a valid router certificate has no FC segment | 'Incomplete' | MAY be used as an ordinary route, with the authenticated portion only guaranteed | No violation; partial deployment |
| 6. Non-canonical repeated CASN | Two consecutive normal FC segments with the same CASN and PC = 1 | 'Invalid' | MUST NOT be treated as FC-BGP-valid | Mapping failure per {{fcmapping}} |
| 7. AS Path Prepending, PC matches | AS_PATH = [65001, 65002, 65002, 65003]; a single FC segment for AS 65002 with PC = 2 | 'Valid' | MAY be admitted to the candidate route set per local policy | None |
| 8. AS Path Prepending, PC mismatch | The PC field does not equal the number of consecutive occurrences of the CASN | 'Invalid' | MUST NOT be treated as FC-BGP-valid | Mapping failure or signature failure per {{error-matrix}} |
| 9. PASN, CASN, or NASN tampered | An AS number field is modified after signing | 'Invalid' | MUST NOT be treated as FC-BGP-valid | Signature failure per {{error-matrix}} |
| 10. Prefix or Prefix Length tampered | The NLRI is modified after signing | 'Invalid' | MUST NOT be treated as FC-BGP-valid | Signature failure per {{error-matrix}} |
| 11. Signature Length inconsistent | The Signature Length field does not match the length of the Signature field | Malformed attribute | Treated as a malformed attribute | Per RFC 7606 {{RFC7606}} via {{error-matrix}} |
| 12. Invalid DER signature | The Signature field is not valid DER | 'Invalid' | MUST NOT be treated as FC-BGP-valid | Cryptographic failure; not a syntax error |
| 13. Unsupported Algorithm ID | The FC segment carries an Algorithm ID the receiver does not recognize | 'Unsupported' | MUST NOT be silently downgraded to an ordinary unsigned BGP route | Per {{error-matrix}} |
| 14. Certificate revoked or expired | The router certificate referenced by an FC segment is revoked or expired | 'Invalid' after revalidation | Best-path selection rerun per local policy | Revalidation per {{Validation}} and {{error-matrix}} |
| 15. Route Server FC | An FC segment with the Flags-RS bit set to 1 bridges two path nodes per {{rs-processing}} | 'Valid' if the FC coverage is complete, otherwise 'Incomplete' | MAY be admitted to the candidate route set per local policy | None |
| 16. iBGP propagation | The FC path attribute is propagated unchanged over iBGP | State unchanged; not revalidated | Validation status conveyed within the AS | None |
| 17. AS Confederation | FC segments with the Confed_Segment flag are processed per {{fcbgp-update}} | Per the confederation rules of {{fcbgp-update}} | FCList reconstructed at the confederation boundary | Per {{fcbgp-update}} and {{error-matrix}} |
| 18. Aggregation or AS_PATH modification | The AS_PATH attribute is changed after the FCList was constructed | 'Invalid', or the FC path attribute is removed before propagation | Regenerate a fresh FC path attribute or remove it | Per {{fcmapping}} and {{asn-processing}} |
| 19. Over-long FCList | The FCList Length exceeds the limits of {{fc-path-attribute}} | Malformed attribute | Treated as a malformed attribute | Per RFC 7606 {{RFC7606}} via {{error-matrix}} |
| 20. FC-BGP and BGPsec coexisting | The UPDATE message carries both a BGPsec_Path attribute and an FC path attribute | Per {{coexist_bgpsec}} | The FC segments are generated and verified against the same path representation | Per {{coexist_bgpsec}} |

The byte-level test vectors for IPv4, IPv6, AS Path Prepending, invalid DER, and tampered fields are derived from the canonical FC Signature Input defined in {{sig-input}} and MUST be reproducible from the wire encoding of the FC path attribute and of the UPDATE message alone. Any two conforming implementations MUST compute the same FC Signature Input and MUST obtain the same validation state for a given input.

## Implementation Status {#implementation-status}

We implement the FC-BGP mechanism with FRR version 9.0.1. The implementation includes verifying the FC path attribute upon receiving BGP UPDATE messages and adding and signing the FC path attribute when sending BGP UPDATE messages. The development and testing of this implementation were conducted on Ubuntu 22.04 with OpenSSL 3.X installed.

GitHub repository: https://github.com/fcbgp/fcbgp-implementation.

## An Example

~~~~~~
AS(65536) --> AS(65537) --> AS(65538)
                      \
                       \--> AS(65539)
~~~~~~
{: #fig-atta-ex title="An FC-BGP UPDATE propagation example."}

It is important to note that this example only introduces important steps here and see {{Validation}} for details. What's more,  here ASs are global AS and not RS or AS Confederation. No ASPP is taken into consideration.

For the sake of discussion, we assume that AS 65537 receives an FC-BGP UPDATE message for prefix 192.0.2.0/24 from AS 65536 and will send the route to AS 65538 and AS 65539 as {{fig-atta-ex}} shows. An FC-BGP speaker SHOULD propagate an FC-BGP UPDATE message to downstream ASs only after completing the validation and best route path selection.

When the FC-BGP speaker in AS 65536 plans to propagate routes to its downstream AS 65537, it fills the FC path attribute first. The PASN is 0/NULL, the CASN is 65536, and the NASN is 65537. Then it calculates the signature, fills the FC, and forms the FC path attribute and FC-BGP UPDATE message. Then, it sends the UPDATE message out.

When receiving an UPDATE message from AS 65536, the FC-BGP speaker in AS 65537 retrieves the FC path attribute and extracts the FC list. It then finds the FC with `CASN == 65536` as the AS_PATH has only one AS(65536) and checks whether PASN is 0 as well as NASN is 65537. If so, it uses the SKI field to find the public key and calculates the signature using the algorithm specified in the Algorithm ID. If the calculated signature matches the signature in the FC segment, then the AS-Path hop associated with the AS 65536 is verified. This process repeats for all FCs and AS-Paths in the FC list if it has other ASs in AS_PATH. However, if AS 65537 does not support FC-BGP, the BGP speaker of AS 65537 simply forwards the BGP UPDATE to its neighbors when propagating this FC-BGP route without validating the FC path attribute.

FC-BGP speakers need to generate different UPDATE messages for different neighbors. Each UPDATE announcement contains only one route prefix and cannot be aggregated. This is because different route prefixes may have different announcement paths due to different routing policies. Multiple aggregated route prefixes may cause FC generation and verification errors. When multiple route prefixes need to be announced, the FC-BGP speaker needs to generate different UPDATE messages for each route prefix. Thus, the FC-BGP speaker of AS 65537 generates different UPDATE messages for AS 65538 and AS 65539 separately. The biggest difference is that the NASN is 65538 for AS 65538 and 65539 for AS 65539 in the FC segment generated by AS 65537.

Take AS 65538 as the next hop. The FC-BGP speaker in AS 65537 will encapsulate each prefix to be sent to AS 65538 in a single UPDATE message, add the FC path attribute, and sign the path content using its private key to fill a new FC segment. The FC path attribute and the FC segment use the message format shown in {{figure1}} and {{figure2}} separately. When signing, the FC-BGP speaker constructs the canonical FC Signature Input defined in {{sig-input}} (which covers the Domain-Separation, Protocol-Version, PASN, CASN, NASN, SKI, Algorithm ID, Flags, canonical signature length, PC, AFI/SAFI, Prefix, and Prefix Length), computes the SHA-256 hash over that byte string, signs the digest with ECDSA, and then fills in the Signature field and the other FC fields. After that, AS 65537 prepends its own FC on top of the FC List. At this point, the processing of FC path attributes by the FC-BGP speaker is complete. The subsequent processing of BGP messages follows the standard BGP process.

## Acknowledgments
{:numbered="false"}

<!-- It is better to update this part gradually with the completion of this document. -->

The authors would like to thank Keyur Patel, Jeffery Hass, Randy Bush, Maria Matejka, Tobias Fiebig, Nan Geng, Tom Strickx, Susan Hares, Rüdiger Volk, Jun Zhang, Kotikalapudi Sriram, John Scudder, Job Snijders, Russ Housley, and Andrew for their review and valuable comments.

