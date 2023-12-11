---
title: "FC-BGP Protocol Specification"
abbrev: "FC-BGP"
category: std

docname: draft-sidrops-wang-fcbgp-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
# area: AREA
# workgroup: WG Working Group
venue:
#  group: WG
#  type: Working Group
#  mail: WG@example.com
#  arch: https://example.com/WG
  github: "BasilGuo/fcbgp-framework"
  latest: "https://BasilGuo.github.io/fcbgp-framework/draft-sidrops-wang-fcbgp.html"

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
      fullname: Zhuotao liu
      org: Tsinghua University
      city: Beijing
      country: China
      email: zhuotaoliu@tsinghua.edu.cn
  -
      fullname: Li Qi
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

This document defines a standard profile for the framework of Forwarding Commitment BGP (FC-BGP). Forwarding Commitment（FC）is a cryptographically signed code to certify an AS's routing intent on its directly connected hops. Based on FC, the goal of FC-BGP is to build a secure inter-domain system that can simultaneously authenticate AS_PATH attribute in BGP-UPDATE and validate network forwarding on the dataplane.


--- middle

# Introduction

## Conventions and Definitions

{::boilerplate bcp14-tagged}

# FC-BGP Negotiation

TODO: BGP-OPEN packet
TODO: no use BGP-AS_SET

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
