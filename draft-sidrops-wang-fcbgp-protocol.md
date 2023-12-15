![image](https://github.com/BasilGuo/fcbgp-protocol/assets/42705918/65172209-7a42-4a35-8bd2-ce95312facbf)---
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
  - RFC5492

informative:


--- abstract

This document describes Forwarding Commitment BGP (FC-BGP), an extension to the Border Gateway Protocol (BGP) that provides security for the path of Autonomous Systems (ASes) through which a BGP UPDATE message passes. Forwarding Commitment（FC）is a cryptographically signed code to certify an AS's routing intent on its directly connected hops. Based on FC, the goal of FC-BGP is to build a secure inter-domain system that can simultaneously authenticate the AS_PATH attribute in BGP-UPDATE and validate network forwarding on the data plane.


--- middle

# Introduction

TODO

## Requirements Language

{::boilerplate bcp14-tagged}

# FC-BGP Negotiation

In the running process of BGP, it should first negotiate the FC-BGP capability {{RFC5492}}. This document defines the BGP capability that allows a BGP speaker to advertise to a neighbor the ability to send or receive FC-BGP UPDATE messages containing the FC path attribute.

This capability has capability code TBD.

The capability length for this capability MUST be set to 3.

The three octets of the capability value are specified as follows.

~~~~~~
  0   1   2  3   4      5  6  7
+--------------------------------+
|   Version   | Dir |  Reserved  |
+--------------------------------+
|                                |
+------------- AFI --------------+
|                                |
+--------------------------------+
~~~~~

Version:
: The first octet's first four bits (bit 0, 1, and 3). It defines the version of FC-BGP for which the BGP speaker is advertising support. This document only defines version 0 and this field MUST be set to 0. Other versions may be set in  future documents. An FC-BGP speaker MAY advertise support for multiple versions of FC-BGP by including multiple versions of the FC-BGP capability in its BGP OPEN message.

Dir:
: The fifth bit (bit 4) of the first octet. It defines the direction of this .

Reserved:
: The last three bits (bits 5, 6, and 7) of the first octet.

AFI:
: The last two octets.






# FC Attribute

TODO: FC path attribute format: optional, transitive, non-partial, extended

# Processing a Sending FC-BGP UPDATE Message

# Processing a Received FC-BGP UPDATE Message



# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
