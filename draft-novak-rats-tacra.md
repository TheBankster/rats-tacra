---
title: Trustworthy Acquisition of Credentials via Remote Attestation
abbrev: TACRA
category: info

docname: draft-novak-rats-tacra-latest
submissiontype: IETF
number:
date:
# consensus: true
v: 3
area: Security
workgroup: RATS Working Group
keyword:
 - trustworthy workload identity
 - remote attestation
 - credential enrollment
 - credential retrieval
venue:
  group: RATS
  type: Working Group
  mail: rats@ietf.org
  arch: https://example.com/WG
  github: TheBankster/rats-twi-um
  # latest: https://example.com/LATEST

author:
 - ins: M. Novak
   name: Mark Novak
   org: J.P. Morgan Chase & Co.
   email: mark.f.novak@jpmchase.com

 - ins: M. Richardson
   name: Michael Richardson
   org: Sandelman Software Works
   email: mcr+ietf@sandelman.ca

 - ins: H. Birkholz
   name: Henk Birkholz
   org:  Franhaufer Inst.
   email: Henk.Birkholz@ietf.contact

normative:
  RFC7030: EST
  RFC9334: RATS

informative:
  TWISIGDef:
    -: TWISIGDef
    target: https://github.com/confidential-computing/twi/blob/main/TWI_Definitions.md
    title: Trustworthy Workload Identity (TWI) Special Interest Group — Definitions
    author:
      org: Confidential Computing Consortium Trustworthy Workload Identity SIG
  TWISIGReq:
    -: TWISIGReq
    target: https://github.com/confidential-computing/twi/blob/main/TWI_Requirements.md
    title: Trustworthy Workload Identity (TWI) Special Interest Group — Requirements
    author:
      org: Confidential Computing Consortium Trustworthy Workload Identity SIG
  SPIRE:
    -: SPIRE
    target: https://github.com/spiffe/spire
    title: The SPIFFE Runtime Environment
    author:
      org: spiffe.io
  ENVOY:
    -: ENVOY
    target: https://envoyproxy.io
    title: The Envoy Proxy
    author:
      org: Envoy

--- abstract

There is a large class of "RATS-Unaware" Relying Parties (RUPs) that Attesters nevertheless need to interoperate with.
Existing deployed services, which precede the introduction of Remote Attestation, are often difficult to change/update in significant ways due to, among other reasons, organizational friction, technological inertia, and regulatory policies.
Yet there are significant advantages if workloads can be incrementally updated in the trustworthiness of the platform, without disrupting their clients and servers.

This document details a proposed architecture by which Remote Attestation utilized for providing Attesters with Identity Documents (keys or credentials) to authenticate to RUPs. The proposal is intended to work with common credential acquisition protocols and mechanisms such as EST {{!RFC7030}}, SPIRE, and many others.
Another important but separate goal is to encapsulate the Attester-side complexity of Remote Attestation and credential acquisition similar to how {{!ENVOY}} does it.

--- middle

# Introduction {#intro}

Success of a technology is ultimately measured by its adoption.
The RATS Architecture requires that RATS Relying Parties understand Attestation Results expressed using standards such as EAT and AR4SI, execute Appraisal Policy for Attestation Results, and have trust in Verifiers.
Additionally, there is an unstated assumption present in the RATS Architecture that a change in Evidence may lead to a change in either the Attestation Results or Appraisal Policy for Attestation Results.

One key requirement for successful deployment of Remote Attestation-capable workloads is minimal blast radius.
When a workload is moved from a legacy to a remotely attestable (e.g. Trusted Execution) environment, including Intel SGX, AMD SEV-SNP,  ARM TrustZone, that workload can use Remote Attestation to obtain a stable and trustworthy Identity Document while its clients and servers do not notice anything different.

For that, a mechanism is required by means of which a Credential Broker, a Key Broker, or a Credential Authority takes on the role of RATS Relying Party.
This provides an intermediation between Attestation Results, expressed using formats such as EAT and AR4SI, and the RATS-Unaware Relying Parties whose authentication and authorization policies may precede the introduction of Remotely Attestable Workloads and remain static for long periods of time.

For the RATS-Unaware Relying Parties, these adoption barriers are eliminated, as these RUPs are capable of authenticating their clients utilizing appropriate Identity Documents.
This includes shared symmetric keys, bearer tokens, credentials including PKIX certificates {{!RFC5280}}, JWTs {{!RFC7515}}, or WIMSE WITs {{!I-D.ietf-wimse-workload-creds}}.
In this world, the Attester uses Remote Attestation to obtain from the RATS Relying Party a key, token or credential that is compatible with the RUP.

This document details an architecture by which legacy Identity Document issuance mechanisms are replaced with identical Identity Documents issued, but with the additional prerequisite of successful Remote Attestation of the workloads in question.

# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Security Considerations {#security}

TODO Security


# IANA Considerations {#iana}

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
