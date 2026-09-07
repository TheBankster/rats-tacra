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
area: "Security"
workgroup: "Remote ATtestation ProcedureS"
keyword:
 - trustworthy workload identity
 - remote attestation
 - credential enrollment
 - credential retrieval
venue:
  group: "Remote ATtestation ProcedureS"
  type: "Working Group"
  mail: "rats@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/rats/"
  github: "TheBankster/rats-tacra"
  latest: "https://TheBankster.github.io/rats-tacra/draft-novak-rats-tacra.html"

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
   org:  Fraunhofer SIT
   email: Henk.Birkholz@ietf.contact

normative:
  RFC9334: RATS

informative:
  RFC5280: PKIX
  RFC7030: EST
  RFC7519: JWT
  RFC8555: ACMEv2
  WIMSE: I-D.ietf-wimse-workload-creds
  CSR-ATTEST: I-D.ietf-lamps-csr-attestation
  INTERACTION-MODELS: I-D.ietf-rats-reference-interaction-models
  ATTESTATION-FRESHNESS: I-D.ietf-lamps-attestation-freshness
  DAA: I-D.ietf-rats-daa
  TWISIGCharter:
    target: https://github.com/confidential-computing/governance/blob/main/SIGs/TWI/TWI_Charter.md
    title: Trustworthy Workload Identity (TWI) Special Interest Group — Charter
    author:
      org: Confidential Computing Consortium Trustworthy Workload Identity SIG
  TWISIGDef:
    target: https://github.com/confidential-computing/twi/blob/main/TWI_Definitions.md
    title: Trustworthy Workload Identity (TWI) Special Interest Group — Definitions
    author:
      org: Confidential Computing Consortium Trustworthy Workload Identity SIG
  TWISIGReq:
    target: https://github.com/confidential-computing/twi/blob/main/TWI_Requirements.md
    title: Trustworthy Workload Identity (TWI) Special Interest Group — Requirements
    author:
      org: Confidential Computing Consortium Trustworthy Workload Identity SIG
  SPIFFE:
    target: https://github.com/spiffe/
    title: Secure Production Identity Framework for Everyone
    author:
      org: spiffe.io
  SPIRE:
    target: https://spiffe.io/docs/latest/spire-about/spire-concepts/
    title: SPIRE Concepts
    author:
      org: spiffe.io
  ENVOY:
    target: https://envoyproxy.io
    title: The Envoy Proxy
    author:
      org: Envoy
...

--- abstract

There is a large class of "RATS-Unaware" Relying Parties (RUPs) that Attesters nevertheless need to interoperate with.
Existing deployed services, which precede the introduction of Remote Attestation, are often difficult to change/update in significant ways due to, among other reasons, organizational friction, technological inertia, and regulatory policies.
There are significant advantages if workloads can be incrementally updated in the trustworthiness of the platform, without disrupting their clients and servers.

This document describes an architecture by which Remote Attestation is utilized for providing Attesters with Identity Documents (keys or credentials) to authenticate to RUPs. The proposal is intended to work with common credential acquisition protocols and mechanisms such as EST, SPIFFE/SPIRE, ACMEv2, and many others.

Another important but separate goal is to encapsulate the Attester-side complexity of Remote Attestation and credential acquisition similar to how Envoy does it. This allows Attesters to be implemented in a way that abstracts away the details of credential acquisition: both the protocols used and the Credential Acquisition Mechanisms employed, whether minting new (Enrollment), or requesting existing (Retrieval) credentials. Likewise, the choice between RATS Passport and Background Check models is made opaque to the Attester, further simplifying its development.

--- middle


# Introduction {#intro}

Success of a technology is ultimately measured by its adoption.
The RATS Architecture {{!RFC9334}} requires that RATS Relying Parties understand Attestation Results, execute Appraisal Policy for Attestation Results, and have trust in Verifiers.
A change in Evidence may lead to a change in either the Attestation Results or Appraisal Policy for Attestation Results.
However, it is common for authentication and authorization policies on Relying Parties to remain static for long periods of time. This is achieved by limiting which entities get to receive the credentials used for authentication and authorization, rather than have the Relying Party make complex decisions based on the credential's changing content.

One key requirement for successful deployment of Remote Attestation-capable workloads is minimal blast radius.
When a workload is moved from a legacy to a remotely attestable Trusted Execution Environment, that workload can use Remote Attestation to obtain a stable and trustworthy Identity Document while its clients and servers do not notice anything different.

For that, a mechanism is required by means of which a Secret Vault or a Credential Authority takes on the role of RATS Relying Party.
This provides an intermediation between Attestation Results and the RATS-Unaware Relying Parties whose authentication and authorization policies may precede the introduction of Remotely Attestable Workloads and remain static for long periods of time.

For the RATS-Unaware Relying Parties, these adoption barriers are eliminated, as these RUPs are capable of authenticating their clients utilizing Identity Document types they are already familiar with.

In summary, rather than using Remote Attestation directly against the RUP, the Attester uses it to obtain from the RATS Relying Party a key, bearer token or proof-of-possession credential that is compatible with the RUP.
This document details an Architecture by which legacy Identity Document issuance mechanisms are replaced or modified such that functionally identical Identity Documents are issued, but with the additional prerequisite of successful Remote Attestation of the workloads in question.

## Reasons for RATS Unaware Relying Party Immutability

The most important and most common scenario addressed here is that of a workload that employs Remote Attestation but whose Relying Party has no capacity to process Attestation Results or execute Appraisal Policy for Attestation Results.
This RATS Unaware Relying Party is typically unable to make the corresponding changes for a number of reasons:

* It may be a compiled object or container provided by a third party
* Or it may be implemented in a language not easily changed or upgraded with new capabilities
* Further, such a system may require extensive and significant review by an authority before changes to the core algorithm can be made
* Or, finally, the reluctance to change may come from organizational friction within an enterprise where the remotely attesting workload is organizationally separate from its Relying Party and different priorities of different parts of organization prevent them moving in lockstep

In all of these cases, it is assumed that the remotely attesting workload can make the necessary changes to perform remote attestation, and that interoperability with the RUP will be preserved so long as the pre-shared key, bearer token, or proof-of-possession credential obtained by the workload following Remote Attestation matches that expected by the RUP.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

* Proof-of-Possession Credential: a credential that requires a private asymmetric signing key to sign statements using this credential
    * PKIX certificates {{RFC5280}}
    * WIMSE Workload Identity Certificates (WICs) and Workload Identity Tokens (WITs) {{WIMSE}}
    * etc.
* Bearer Token Credential: a credential that does not require proof of possession of a secret to use:
    * Pre-shared symmetric key
    * API key
    * JSON Web Token JWT {{RFC7519}}
    * etc.
* Credential Type: a catch-all term for any concrete credential type from the two lists above
* Credential, a.k.a. Identity Document: an instance of a Credential Type
* Credential Acquisition Mechanism: one of any number of existing or future mechanisms for acquiring credentials, such as {{RFC7030}}, {{SPIFFE}}/{{SPIRE}}, {{RFC8555}}, etc.
* Credential Acquisition System (CAS): a client-server Architecture comprising a CAS Client and a CAS Server which implements a Credential Acquisition Mechanism
* Credential Acquisition Mode: one of
    1. Credential Enrollment (minting new proof-of-possession credential), or
    2. Credential Retrieval (retrieving an existing, pre-provisioned credential of any Credential Type)
* Replica workloads: workloads that are functionally indistinguishable from the point of view of clients that authenticate to them or servers that they authenticate to; typical in "horizontal scale-out" scenarios where multiple identical workload instances are launched to handle the load in parallel

Note: WITs are JWTs that require key confirmation. That makes them proof-of-possession credentials, whereas JWTs are used without key confirmation and are thus considered bearer tokens.

# Requirements

This proposal is a result of work by the Confidential Computing Consortium's Trustworthy Workload Identity (TWI) SIG {{TWISIGCharter}} which has published a set of Definitions {{TWISIGDef}} and Requirements {{TWISIGReq}}. The requirements published by the TWI SIG are deliberately high-level. The requirements specified here fully align with the TWI SIG requirements, while focusing on a portable and extensible implementation.

1. Supports mechanisms for enrolling (minting new) as well as retrieving (pre-existing) credentials; the workload knows what Credential Type it will need when it launches, but discovers whether it will have to retrieve an existing or enroll a new credential at runtime.
2. Supports most current and future credential formats:
   * X.509 Certificates
   * WIMSE Workload Identity Certificates (WICs)
   * WIMSE Workload Identity Tokens (WITs)
   * Bearer tokens (JWTs, API Keys)
   * TPM 2.0 DAA Group Certificates (for Replica workloads)
   * Pre-shared keys
   * Future formats through architectural extensibility
3. Minimal trust boundary expansion of the Attesting Environment
4. Supports, transparently to the Attester, most current and future Credential Acquisition Mechanisms:
   * EST (RFC 7030)
   * SPIFFE/SPIRE
   * ACMEv2 (RFC 8555)
   * TPM 2.0 DAA Join protocol (based on TPM 2.0 AK Cert) {{DAA}}
   * Future mechanisms through architectural extensibility
5. Supports Workloads utilizing different Credential Acquisition Mechanisms per-target
6. Supports both Background Check and Passport RATS modes, indistinguishably from the PoV of the Attester
7. Supports most current and future RATS Verifiers, Identity Providers, and Secret Vaults, transparently to the Attester
8. Cannot assume that Workload has independent network access
9. Compatible with all existing and future Confidential Computing platforms meeting minimum requirements around secure cryptography and evidence generation
10. Restricts visibility of fetched secrets to the Attester, excluding the CAS Client and CAS Server

## Required Modifications to Existing Credential Acquisition Mechanisms

It is not a goal, and, at any rate, it is not possible, to leave credential acquisition protocols and mechanisms (EST, SPIFFE/SPIRE, etc.) unmodified. These mechanisms currently do not support Remote Attestation, for the following reasons:
1. Remote Attestation is typically a two-phase process:
    1. The Attester requests and obtains a challenge, also sometimes referred to as "freshness", from the Verifier {{INTERACTION-MODELS}} {{ATTESTATION-FRESHNESS}}
    2. The Attester responds to the Verifier's challenge with Evidence, which is how it demonstrates its security, possession cryptographic key material, and capabilities
2. None of the existing broadly deployed Credential Acquisition Mechanisms support this challenge-response sequences, but all appear extensible to accommodate such changes without a lot of additional effort, and without risking backwards compatibility.
3. The credentials that these mechanisms return are typically visible in plaintext to the control plane (the CAS Client and the CAS Server), whereas it is a common requirement to keep the knowledge of authentication secrets to the Attesters and a small number of trusted services, such as key vaults and HSMs.


# Architecture

In the text that follows, numbers in the format [Req #] refer to the corresponding numbered items in the list of Requirements in the opening section of this document.

~~~~ ascii-art
{::include tacra_architecture.txt}
~~~~
{: #fig-tacra title="TACRA architecture"}

This Architecture assumes the existence of a “Credential Acquisition System” (CAS), such as EST, SPIFFE/SPIRE, ACMEv2, etc., that comprises a client and a server. The CAS Client is presumed to be running on the Attester’s system, but outside the Attester’s TEE. The CAS Server is a remote service invoked by the CAS Client over the CAS protocol. Which CAS protocol is used MUST remain opaque to the Attester.

The Attester (the workload) runs inside a TEE. It obtains credentials by invoking the Credential Acquisition Interface (CAI), defined in this document. CAI insulates the Attester from all the details of platform-specific and protocol-specific aspects and services involved in Remote Attestation and credential enrollment/retrieval. The CAI MAY be a statically or dynamically linked library, an Envoy-style sidecar {{ENVOY}}, or any other mechanism, but it MUST run inside the Attester's TEE.

CAI interacts with the outside world on behalf of the Attester via two channels:

1. With the underlying hardware platform to utilize its TEE-specific functions, such as generating keys and obtaining evidence, via the platform-type-specific plugin, and
2. With the Credential Acquisition System Client, via the well-defined Credential Acquisition API (CAAPI), also outlined later in this document.

These being the only two communication mechanisms needed to interact with the outside world, no network or storage stack are needed by the Attester [Req 8]. The server side of CAAPI is part of the Credential Acquisition Client. There can be as many Credential Acquisition Client implementations as there are Credential Acquisition Mechanisms [Req 4]: EST Client, SPIRE Agent, etc. There is no restriction against multiple Credential Acquisition Mechanisms collectively serving the same Attester, with different mechanisms utilized for different targets [Req 5]. Existing CAS Clients are extended to support Remote Attestation via dedicated CAS Client Plug-ins.

The Credential Acquisition System controls which Credentials Types and which Credential Acquisition Mechanisms (enrollment, retrieval) can be provisioned to the Attester for any Attester-supplied target, without the Attester’s knowledge or involvement [Req 1]. If a Credential Type specified by the Attester is unavailable due to Credential Acquisition System limitations, an error will result. It is an administrative error to pair an Attester with a Credential Acquisition System that is unable to supply it with the Credential Type it requires.

The Credential Acquisition Server implements the server side of the corresponding Credential Acquisition Mechanism and interacts with the RATS Verifier, the Identity Provider (e.g., a Certificate Authority for minting new certificates) and the Secret Vault for fetching existing keys or credentials, on the Attester’s behalf [Req 7]. The Credential Types supported by this Architecture are limited only by what the Credential Acquisition System can support [Req 2]. Existing Credential Acquisition Servers are extended to support Remote Attestation via dedicated CAS Server Plug-ins. The CAS Server is not the RATS Relying Party in this Architecture: the Certificate Authority or the Secret Vault play that role.

This arrangement shields the Attester developers from having to know the details of the platform on which the Attester runs [Req 9]. It restricts the unavoidable expansion of the Attesting Environment to the smallest possible amount [Req 3]. There is no difference, from the standpoint of the Attester, whether the RATS Passport or Background Check model is being used [Req 6].

Under the covers and opaquely to the Attester, the Credential Acquisition Interface discovers and utilizes one of two Credential Acquisition Modes: Enrollment and Retrieval. Enrollment corresponds to minting new proof-of-possession credentials, and Retrieval is used to fetch preshared keys, bearer tokens and shared proof-of-possession credentials (e.g., for Replica workloads). In both cases, the associated secrets remain opaque to the CAS at all times [Req 10] even if the credential, such as an X.509 certificate, is public and can be returned in plaintext.

* Enrollment: the Credential Acquisition Interface generates and includes alongside Evidence a CSR. It is possible to include Evidence in the CSR, or vice versa: include the CSR in Evidence. The details of how this is decided at runtime are TBD (TODO: discuss, with reference to {{CSR-ATTEST}}).
* Retrieval: the Credential Acquisition Interface generates an asymmetric encryption key CEK and includes CEKpub in Evidence. The resulting secrets are encrypted to CEKpub, ensuring that only the Attester in possession of CEKpri can decrypt them.

## Summary of RATS Roles

| TACRA Component | RATS Role | Remarks |
| :--- | :--- | :--- |
| Attester | Attester | Attesting Environment extended and complemented by CAI, CAAPI, Platform Plug-in |
| CAI | Part of Attester | Library or Sidecar assisting Attester in obtaining credentials |
| Platform Plug-in | Part of Attester | Invoked by CAI to perform platform-specific Remote Attestation and key generation operations |
| CAAPI Client | Part of Attester | Invoked by CAI to communicate with CAS Client |
| CAS Client | None: Conduit only | CAS Client extended by Remote Attestation Plug-in |
| CAS Server | None: Conduit only | CAS Server extended by Remote Attestation Plug-in |
| Secret Vault | RATS Relying Party | Invoked in the Enrollment variant of this architecture; MUST encrypt retrieved results to CEKpub |
| Certificate Authority | Relying Party | Invoked in the Enrollment variant of this architecture |
| Verifier | Verifier | No changes in RATS Verifier role or implementation |
| RATS-Unaware Relying Party | None | No changes in RUP role or implementation |
{: #tab-rats-roles title="RATS roles in TACRA"}


# Credential Acquisition API (CAAPI)

The Credential Acquisition API allows the Attester to communicate with the Credential Acquisition System. These APIs are invoked by the Credential Acquisition Interface, covered in the next section. CAAPI can be implemented using any mechanism suitable for interprocess communication, including but not limited to Protobuf/gRPC. Here only the high-level description is provided.

## Initiate-Credential-Acquisition

Initiates the credential acquisition process by obtaining Freshness and validating that the indicated Target name and Credential Type are supported by the CAS.

Parameters:

* Target name, e.g., the server URI to which the Attester wishes to authenticate
* Credential Type the Attester plans to use with this Target

Returns:

* On success:
    * Credential acquisition mechanism: "enroll" or "retrieve"
    * (optional) Freshness handle (see {{INTERACTION-MODELS}})
    * Other TBD pertinent information, such as supported ciphers, etc. (TODO: define)
* On failure: enumerated reason for failure, such as:
    * Invalid Target
    * Unsupported Target
    * Invalid Credential Type
    * Server error: failed remote attestation
    * Server error: permission failure
    * Server error: server unavailable; try again later
    * Server error: server unreachable; try again later
    * etc. (TBD)

## Enroll-Credential

Enrolls (mints) a new proof-of-possession credential.

Parameters:

* Target name matching that of the corresponding Initiate-Credential-Acquisition call
* Credential Type matching that of the corresponding Initiate-Credential-Acquisition call
* Evidence, bound to the previously returned Freshness, if any
* CSR matching the Evidence (TODO: discuss CSR-to-Evidence binding/relationship)

Returns:

* On success: plaintext newly enrolled (minted) credential or the requested type
* On failure: enumerated reason for failure, such as:
    * Invalid Target
    * Unsupported Target
    * Invalid Credential Type
    * Server error: failed remote attestation
    * Server error: permission failure
    * Server error: remote attestation failure
    * Server error: server unavailable; try again later
    * Server error: server unreachable; try again later
    * etc. (TBD)

## Retrieve-Credential

Retrieves (fetches pre-existing) credential.

Parameters:

* Target name matching that of the corresponding Initiate-Credential-Acquisition call
* Credential Type matching that of the corresponding Initiate-Credential-Acquisition call
* Evidence, bound to the previously returned Freshness, if any
* CEKpub matching the Evidence (TODO: discuss CEK-to-Evidence binding/relationship)

Returns:

* On success: wrapped (encrypted to CEKpub) credential or the requested type
* On failure: enumerated reason for failure, such as:
    * Invalid Target
    * Unsupported Target
    * Invalid Credential Type
    * Server error: failed remote attestation
    * Server error: permission failure
    * Server error: remote attestation failure
    * Server error: server unavailable; try again later
    * Server error: server unreachable; try again later
    * etc. (TBD)

## CAAPI Invocation Sequence

The caller (normally the Credential Acquisition Interface) first decides which Target it wishes to authenticate to, and using which Credential Type. CAAPI offers no facilities for this, so this must be decided out of band.

The typical invocation flow is:

1. CAAPI: Initiate-Credential-Acquisition(Target, Credential Type)
    * Returns optional Freshness and the Credential Acquisition Mode for this Target and Credential Type
2. Platform Plug-in: Generate Keys and the corresponding Evidence:
    * CSK, the Certificate Signing Key, and the corresponding CSR, for credential enrollment or
    * CEK, the Credential Encryption Key, for credential retrieval
3. CAAPI: Depending on which Credential Acquisition Mode is returned, either
    * Enroll-Credential(Target, Credential Type, CSR, Evidence)
    * Retrieve-Credential(Target, Credential Type, CEKpub, Evidence)


# Credential Acquisition Interface (CAI)

The Credential Acquisition Interface can be implemented using any mechanism suitable for local communication, including but not limited to statically linked calls and Protobuf/gRPC. Here only the high-level description is provided. The CAI consists of a single Acquire-Credential API, outlined below. The Acquire-Credential implementation follows the recommended CAAPI invocation sequence.

## Acquire-Credential

Orchestrates an opaque-to-Attester process by which the Attester acquires a credential that it would need to authenticate to a given Target utilizing the given Credential Type.

Parameters:

* Target name, e.g., the server URI to which the Attester wishes to authenticate
* Credential Type the Attester plans to use with this Target

Returns:

* On success: newly acquired credential or the requested type
* On failure: enumerated reason for failure, such as:
    * Invalid Target
    * Unsupported Target
    * Invalid Credential Type
    * Server error: failed remote attestation
    * Server error: permission failure
    * Server error: remote attestation failure
    * Server error: server unavailable; try again later
    * Server error: server unreachable; try again later
    * etc. (TBD)


# Security Considerations {#security}

This specification supports but discourages the use of bearer token credentials. They are supported in the interest of maximizing compatibility. While the specification takes care to deliver bearer token credentials to the Attester securely, subsequent usage, such as using them in authentication against the RUP, still risks leaking them.

(TODO: Mention the following:)

* CAS Server is not necessarily trusted with plaintext secrets and how to keep such secrets opaque to it
* Secure binding between Evidence and CSR for Enrollment mode
* Authenticating the CAS Client (the part outside the Attester) to the CAS Server


# IANA Considerations {#iana}

This document has no IANA actions.


--- back


# Acknowledgments
{:numbered="false"}

TODO acknowledge.
