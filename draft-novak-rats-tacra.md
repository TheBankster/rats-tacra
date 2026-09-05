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
   org:  Franhaufer Inst.
   email: Henk.Birkholz@ietf.contact

normative:
  RFC5280: PKIX
  RFC7030: EST
  RFC7519: JWT
  RFC8555: ACMEv2
  RFC9334: RATS
  RFC9711: EAT

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
  AR4SI:
    -: AR4SI
    target: https://datatracker.ietf.org/doc/draft-ietf-rats-ar4si/
    title: Attestation Results for Secure Interactions
    author:
      org: IETF RATS Working Group
...

--- abstract

There is a large class of "RATS-Unaware" Relying Parties (RUPs) that Attesters nevertheless need to interoperate with.
Existing deployed services, which precede the introduction of Remote Attestation, are often difficult to change/update in significant ways due to, among other reasons, organizational friction, technological inertia, and regulatory policies.
There are significant advantages if workloads can be incrementally updated in the trustworthiness of the platform, without disrupting their clients and servers.

This document details a proposed Architecture by which Remote Attestation utilized for providing Attesters with Identity Documents (keys or credentials) to authenticate to RUPs. The proposal is intended to work with common credential acquisition protocols and mechanisms such as EST {{!RFC7030}}, {{SPIRE}}, ACMEv2 {{RFC8555}} and many others.

Another important but separate goal is to encapsulate the Attester-side complexity of Remote Attestation and credential acquisition similar to how {{ENVOY}} does it. This allows Attesters to be implemented in a way that abstracts away the details of credential acquisition: both the protocols used and the Credential Acquisition Mechanisms employed, whether minting new (Enrollment), or requesting existing (Retrieval) credentials.

--- middle


# Introduction {#intro}

Success of a technology is ultimately measured by its adoption.
The RATS Architecture requires that RATS Relying Parties understand Attestation Results expressed using standards such as EAT {{!RFC9711}} and AR4SI {{AR4SI}}, execute Appraisal Policy for Attestation Results, and have trust in Verifiers.
Additionally, there is an unstated assumption present in the RATS Architecture that a change in Evidence may lead to a change in either the Attestation Results or Appraisal Policy for Attestation Results.

One key requirement for successful deployment of Remote Attestation-capable workloads is minimal blast radius.
When a workload is moved from a legacy to a remotely attestable (e.g. Trusted Execution) environment, including Intel SGX, AMD SEV-SNP,  ARM CCA, that workload can use Remote Attestation to obtain a stable and trustworthy Identity Document while its clients and servers do not notice anything different.

For that, a mechanism is required by means of which a Credential Broker, a Key Broker, or a Credential Authority takes on the role of RATS Relying Party.
This provides an intermediation between Attestation Results, expressed using formats such as EAT and AR4SI, and the RATS-Unaware Relying Parties whose authentication and authorization policies may precede the introduction of Remotely Attestable Workloads and remain static for long periods of time.

For the RATS-Unaware Relying Parties, these adoption barriers are eliminated, as these RUPs are capable of authenticating their clients utilizing appropriate Identity Documents.

In summary, rather than using Remote Attestation directly against the RUP, the Attester uses it to obtain from the RATS Relying Party a key, bearer token or proof-of-possession credential that is compatible with the RUP.
This document details an Architecture by which legacy Identity Document issuance mechanisms are replaced or modified such that functionally identical Identity Documents are issued, but with the additional prerequisite of successful Remote Attestation of the workloads in question.

## Reasons for RATS Unaware Relying Party Immutability

The most important and most common scenario addressed here is that of a workload that employs Remote Attestation but whose Relying Party has no capacity to process Attestation Results or execute Appraisal Policy for Attestation Results.
This RATS Unaware Relying Party is typically unable to make the corresponding changes for a number of reasons:

* It may be a compiled object or container provided by a third party
* Or it may be implemented in a language not easily changed or upgraded with new capabilities
* Or, as an extreme example, it could be an ancient COBOL program compiled into a WASM object, perhaps connected to the network via virtual paper-tape and virtual printer interfaces
* Further, such a system may require extensive and significant review by an authority before changes to the core algorithm can be made
* Or, finally, the reluctance to change may come from organizational friction within an enterprise where the remotely attesting workload is organizationally separate from its Relying Party and different priorities of different parts of organization prevent them moving in lockstep

In all of these cases, it is assumed that the remotely attesting workload can make the necessary changes to perform remote attestation, and that interoperability with the RUP will be preserved so long as the pre-shared key, bearer token, or proof-of-possesion credential obtained by the workload following Remote Attestation matches that expected by the RUP.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

Terms related to Trustworthy Workload Identity defined by the TWI SIG at the Confidential Computing Consortium {{TWISIGReq}} are hereby incorporated by reference.

* Proof-of-Possession Credential: a credential that requires a private asymmetric signing key to sign statements using this credential
    * PKIX certificates {{RFC5280}}
    * WIMSE Workload Idenity Certificates (WICs)
    * WIMSE Workload Identity Tokens (WITs)
    * etc.
* Bearer Token Credential: a credential that does not require proof of possession of a secret to use:
    * Pre-shared symmetric key
    * API key
    * JSON Web Token JWT {{RFC7519}}
    * etc.
* Credential Type: a catch-all term for any concrete credential type from the two lists above
* Credential, a.k.a. Identity Document: an instance of a Credential Type
* Credential Acquisition Mechanism: one of any number of existing or future mechanisms for acquiring credentials, such as EST, SPIRE, ACMEv2, etc.
* Credential Acquisition System (CAS): a client-server Architecture comprising a CAS Client and a CAS Server which implements a Credential Acquisition Mechanism
* Credential Acquisition Mode: one of
    1. Credential Enrollment (minting new proof-of-possession credential), or
    2. Credential Retrieval (retrieving an existing, pre-provisioned credential of any Credential Type)
* Replica workloads: workloads that are functionally indistinguishable from the point of view of clients that authenticate to them or servers that they authenticate to; typical in "horizontal scale-out" scenarios where multiple identical workload instances are launched to handle the load in parallel


# Requirements

This proposal is a result of work by the Confidential Computing Consortium's Trustworthy Workload Identity (TWI) SIG (TODO: reference) which has published a set of Definitions {{TWISIGDef}} and Requirements {{TWISIGReq}}. The requirements published by the TWI SIG are deliberately high-level. The requirements specified here fully align with the TWI SIG requirements, while focusing on a portable and extensible implementation.

1. Supports mechanisms for minting (new) as well as retrieving (pre-existing) credentials; the workload does not know what type of credential it will be given (new or pre-existing) when it launches; the approproate mode is negotiated and utilized at runtime
2. Supports most current and future credential formats:
   * X.509 Certificates
   * WIMSE Workload Identity Certificates (WICs)
   * WIMSE Workload Identity Tokens (WITs)
   * Bearer tokens (JWTs, API Keys)
   * TPM 2.0 DAA Group Certificates (for Replica workloads)
   * Pre-shared keys
   * Future formats through architectural extensibility
3. Minimal trust boundary expansion of the Attester implementation
4. Supports, transparently to the Attester, most current and future Credential Acquisition Mechanisms:
   * EST (RFC 7030)
   * SPIRE
   * ACMEv2 (RFC 8555)
   * TPM 2.0 DAA Join protocol (based on TPM 2.0 AK Cert)
   * Future mechanisms through architectural extensibility
5. Supports Workloads utilizing different Credential Acquisition Mechanisms per-target
6. Supports both Background Check and Passport RATS modes, indistinguishably from the PoV of the Attester
7. Supports most current and future RATS Verifiers, Identity Providers, and Key/Credential Stores, transparently to the Attester
8. Cannot assume that Workload has independent network access (i.e., its only way of communicating with the outside world for purposes of credential acquisition are the platform's Confidential Computing-specific Application Binary Interface (ABI) and the Credential Acquisition Interface, defined later in this document)
9. Compatible with all existing and future Confidential Computing platforms meeting minimum requirements around secure cryptography and evidence generation
10. Restricts visibility of fetched secrets to the Attester, excluding the CAS Client and CAS Server

## Required Modifications to Existing Credential Acquisition Mechanisms

It is not a goal, and, at any rate, it is not possible, to leave credential acquisition protocols and mechanisms (EST, SPIRE, etc.) unmodified. These mechanisms currently do not support Remote Attestation, for the following reasons:
1. Remote Attestation is typically a two-phase process:
    1. The Attester requests and obtains a challenge, also sometimes referred to as "nonce" or "freshness" from the Verifier
    2. The Attester responds to the Verifier's challenge with Evidence, which is how it demonstrates its security, possession cryptographic key material, and capabilities
2. None of the existing broadly deployed Credential Acquisition Mechanisms support this challenge-response sequences, but all appear extensible to accommodate such changes without a lot of additional effort, and without risking backwards compatibility.
3. The credentials that these mechanisms return are typically visible in plaintext to the control plane (the CAS Client and the CAS Server), whereas it is a common requirement to keep the knowledge of authentication secrets to the Attesters and a small number of trusted services, such as key vaults and HSMs.


# Architecture

In the text that follows, numbers in the format [Req #] refer to the corresponding numbered items in the list of Requirements in the opening section of this document.

~~~~ ascii-art
{::include tacra_architecture.txt}
~~~~
{: #fig-tacra title="TACRA architecture"}

This Architecture assumes the existence of a “Credential Acquisition System” (CAS), such as EST, SPIRE, ACMEv2, etc., that comprises a client and a server. The CAS Client is presumed to be running on the Attester’s system, but outside the Attester’s TEE. The CAS Server is a remote service invoked by the CAS Client over the CAS protocol, which must remain opaque to the Attester.

The Attester (the workload) runs inside a TEE. It obtains credentials by calling the CAS Client Proxy, which insulates it from all the details of platform-specific and protocol-specific aspects and services involved in Remote Attestation and credential enrollment/retrieval. The CAS Client Proxy can be a statically or dynamically linked library, an Envoy-style sidecar, or any other mechanism, but it must run inside the Attester's TEE. The Attester communicates with the CAS Client Proxy via a simple Credential Acquisition Interface (CAI) defined later in this document.

For the purposes of credential acquisition, the CAS Client Proxy interacts with the outside world on behalf of the Attester via two channels:

1. With the underlying hardware platform to utilize its TEE-specific functions, such as generating keys and obtaining evidence, via the platform-specific plugin, and
2. With the Credential Acquisition Client, via the well-defined Credential Acquisition API (CAAPI), also outlined later in this document.

These being the only two communication mechanisms needed to interact with the outside world, no network or storage stack are needed by the Attester [Req 8]. The server side of CAAPI is part of the Credential Acquisition Client. There can be as many Credential Acquisition Client implementations as there are Credential Acquisition Mechanisms [Req 4]: EST Client, SPIRE Agent, etc. There is no restriction against multiple Credential Acquisition Mechanisms collectively serving the same Attester, with different mechanisms utilized for different targets [Req 5].

The Credential Acquisition System controls which Credentials Types and which Credential Acquisition Mechanisms (enrollment, retrieval) can be provisioned to the Attester for any Attester-supplied target, without the Attester’s knowledge or involvement [Req 1]. If a Credential Type specified by the Attester is unavailable due to Credential Acquisition System limitations, an error will result. The Attester discovers what is avaiable by trial and error, but typically it is an administrative error to pair an Attester with a Credential Acquisition System that is unable to supply it with the type of credential it requires.

The Credential Acquisition Server implements the server side of the corresponding Credential Acquisition Mechanism and interacts with the RATS Verifier, the Identity Provider (e.g., a Certificate Authority for minting new certificates) and the Key/Credential store for fetching existing keys or credentials, on the Attester’s behalf [Req 7]. The Credential Types supported by this Architecture are limited only by what the Credential Acquisition System can support [Req 2].

This arrangement shields the Attester developers from having to know the details of the platform on which the Attester runs [Req 9]. It restricts the unavoidable expansion of the Attester TCB to the smallest possible amount [Req 3]. There is no difference, from the standpoint of the Attester, whether the RATS Passport or Background Check model is being used [Req 6].

Under the covers and opaquely to the Attester, the CAS Client Proxy discovers and utilizes one of two Credential Acquisition Modes: Enrollment and Retrieval. Enrollment corresponds to minting new proof-of-possession credentials, and Retrieval is used to fetch preshared keys, bearer tokens and shared proof-of-possession credentials (e.g., for Replica workloads). In both cases, the associated secrets remain opaque to the CAS at all times [Req 10].

* Enrollment: the CAS Client Proxy generates and includes alongside Evidence a CSR. It is possible to include Evidence in the CSR, or vice versa: include the CSR in Evidence. The details of how this is decided at runtime are TBD (TODO: discuss).
* Retrieval: the CAS Client Proxy generates an asymmetric encryption key CEK and includes CEKpub in Evidence. The resulting secrets are encrypted to CEKpub.


# Credential Acquisition Interface (CAI)

The Credential Acquisition Interface can be implemented using any mechanism suitable for local communication, including but not limited to statically linked calls and Protobuf/gRPC. Here only the high-level description is provided. The CAI consists of a single Acquire-Credential API, outlined below.

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


# Credential Acquisition API (CAAPI)

The Credential Acquisition API can be implemented using any mechanism suitable for interprocess communication, including but not limited to Protobuf/gRPC. Here only the high-level description is provided.

## Initiate-Credential-Acquisition

Initiates the credential acquisition process by obtaining a challenge from a Verifier.

Patameters:
* Target name, e.g., the server URI to which the Attester wishes to authenticate
* Credential Type the Attester plans to use with this Target

Returns:
* On success:
    * Credential acquisition mechanism (enroll or retrieve)
    * Verifier challenge
    * Other TBD pertinent information, such as supported ciphers, etc. (TODO: define)
* On failure: enumerated erason for failure, such as:
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
* Target name, e.g., the server URI to which the Attester wishes to authenticate
* Credential Type the Attester plans to use with this Target
* Evidence matching the previously returned Verifier challenge
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
* Target name, e.g., the server URI to which the Attester wishes to authenticate
* Credential Type the Attester plans to use with this Target
* Evidence matching the previously returned Verifier challenge
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


# Security Considerations {#security}

This specification supports but discourages the use of bearer token credentials. They are supported in the interest of maximizing compatibility.

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
