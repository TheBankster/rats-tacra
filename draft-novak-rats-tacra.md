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
  github: "TheBankster/rats-twi-um"
  latest: "https://TheBankster.github.io/rats-twi-um/draft-novak-rats-tacra.html"

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
There are significant advantages if workloads can be incrementally updated in the trustworthiness of the platform, without disrupting their clients and servers.

This document details a proposed architecture by which Remote Attestation utilized for providing Attesters with Identity Documents (keys or credentials) to authenticate to RUPs. The proposal is intended to work with common credential acquisition protocols and mechanisms such as EST {{!RFC7030}}, SPIRE, and many others.

Another important but separate goal is to encapsulate the Attester-side complexity of Remote Attestation and credential acquisition similar to how {{!ENVOY}} does it. This allows Attesters to be implemented in a way that abstracts away the details of credential acquisition: both the protocols used and the credential acquisition mechanisms employed, whether minting new (Enrollment), or requesting existing (Retrieval) credentials.

--- middle

# Introduction {#intro}

Success of a technology is ultimately measured by its adoption.
The RATS Architecture requires that RATS Relying Parties understand Attestation Results expressed using standards such as EAT and AR4SI, execute Appraisal Policy for Attestation Results, and have trust in Verifiers.
Additionally, there is an unstated assumption present in the RATS Architecture that a change in Evidence may lead to a change in either the Attestation Results or Appraisal Policy for Attestation Results.

One key requirement for successful deployment of Remote Attestation-capable workloads is minimal blast radius.
When a workload is moved from a legacy to a remotely attestable (e.g. Trusted Execution) environment, including Intel SGX, AMD SEV-SNP,  ARM CCA, that workload can use Remote Attestation to obtain a stable and trustworthy Identity Document while its clients and servers do not notice anything different.

For that, a mechanism is required by means of which a Credential Broker, a Key Broker, or a Credential Authority takes on the role of RATS Relying Party.
This provides an intermediation between Attestation Results, expressed using formats such as EAT and AR4SI, and the RATS-Unaware Relying Parties whose authentication and authorization policies may precede the introduction of Remotely Attestable Workloads and remain static for long periods of time.

For the RATS-Unaware Relying Parties, these adoption barriers are eliminated, as these RUPs are capable of authenticating their clients utilizing appropriate Identity Documents.
Identity Documents, also known as Credential Types, are any of:
* Pre-shared symmetric keys
* Bearer tokens, e.g., API Keys, JSON Web Tokens JWTs {{!RFC7515}}), and
* "Proof-of-possession" credentials, such as PKIX certificates {{!RFC5280}}, WIMSE Workload Idenity Certificates (WICs), or WIMSE Workload Identity Tokens (WITs) {{!I-D.ietf-wimse-workload-creds}}

In summary, rather than using Remote Attestation directly against the RUP, the Attester uses it to obtain from the RATS Relying Party a key, bearer token or proof-of-possession credential that is compatible with the RUP.
This document details an architecture by which legacy Identity Document issuance mechanisms are replaced with identical Identity Documents issued, but with the additional prerequisite of successful Remote Attestation of the workloads in question.

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

# Requirements

This proposal is a result of work by the Confidential Computing Consortium's Trustworthy Workload Identity (TWI) SIG (TODO: reference) which has published a set of Definitions (TODO: reference) and Requirements (TODO: reference). The requirements published by the TWI SIG are high-level and deliberately not implementation-centric. The requirements specified here fully align with the TWI SIG requirements, while focusing on a portable and extensible implementation.

1. Supports mechanisms for minting (new) as well as retrieving (pre-existing) credentials; the workload does not know what type of credential it will be given (new or pre-existing) when it launches; it discovers this at runtime
2. Supports most current and future credential formats:
   * X.509 Certificates
   * WIMSE Workload Identity Certificates (WICs)
   * WIMSE Workload Identity Tokens (WITs)
   * Bearer tokens (JWTs, API Keys)
   * TPM 2.0 DAA Group Certificates (for replica workloads)
   * Pre-shared keys
   * Future formats through architectural extensibility
3. Minimal trust boundary expansion of the Attester implementation
4. Supports, transparently to the Attester, most current and future credential acquisition mechanisms:
   * EST (RFC 7030)
   * SPIRE
   * ACMEv2 (TODO: Add RFC #)
   * TPM 2.0 DAA Join protocol (based on TPM 2.0 AK Cert)
   * Future mechanisms through architectural extensibility
5. Supports Workloads utilizing different credential acquisition mechanisms per-target
6. Supports both Background Check and Passport RATS modes, indistinguishably from the PoV of the Attester
7. Supports most current and future RATS Verifiers, Identity Providers, and Key/Credential Stores, transparently to the Attester
8. Cannot assume that Workload has independent network access (i.e., its only way of communicating with the outside world for purposes of credential acquisition are the platform's Confidential Computing-specific Application Binary Interface (ABI) and the Credential Acquisition Interface, defined later in this document)
9. Compatible with all existing and future Confidential Computing platforms meeting minimum requirements around secure cryptography and evidence generation
10. Restricts visibility of fetched secrets to the Attester, excluding the CAS Client and Server

## Required Modifications to Existing Credential Acquisition Mechanisms

It is not possible, and neither is it a goal, to leave credential acquisition protocols and mechanisms (EST, SPIRE, etc.) unmodified. These protocols currently do not support Remote Attestation, for the following reasons:
1. Remote Attestation is typically a two-phase process:
    1. The Attester requests and obtains a challenge, also sometimes referred to as "nonce" or "freshness" from the Verifier
    2. The Attester responds to the Verifier's challenge with Evidence, which is how it demonstrates it security and capabilities
2. None of the existing broadly deployed credential mechanisms support this challenge-response sequences, but all appear extensible to accommodate such changes without a lot of additional effort, or risking backwards compatibility.
3. The credentials that these mechanisms return are typically visible in plaintext to the control plane, whereas it is a common requirement to keep the knowledge of authentication secrets to the Attesters and a small number of trusted services, such as key vaults and HSMs.

# Architecture

In the text that follows, numbers in the format [Req #] refer to the corresponding numbered requirement in the list of Requirements in the opening section of this document.

{::include tacra_architecture.txt}

This architecture assumes the existence of a “Credential Acquisition System” (CAS), such as EST, SPIRE, ACMEv2, etc., that comprises a client and a server. The CAS Client is presumed to be running on the Attester’s system, but outside the Attester’s TEE. The CAS Server is a remote service invoked by the CAS Client over the CAS protocol, which must remain opaque to the Attester.

The Attester (the workload) runs inside a TEE. It obtains credentials by calling the in-proc CAS Client Proxy, which can be a statically linked library or an Envoy-style sidecar (TODO: add Envoy reference). The CAS Client Proxy interacts with the outside world via two channels:

1. With the underlying hardware platform to utilize its TEE-specific functions, such as generating keys and obtaining evidence, via the platform-specific plugin [Req 9], and
2. With the Credential Acquisition Client, via the well-defined Credential Acquisition Interface

These being the only two communication mechanisms needed to interact with the outside world, no network or storage stack are needed by the Attester [Req 8]. The server side of the CAAPI interface is part of the CAS Server Proxy. The CAS Server Proxy is hosted by the Credential Acquisition Client. There can be as many Credential Acquisition Client implementations as there are credential acquisition mechanisms [Req 4]: EST Client, SPIRE Agent, etc. There is no restriction against multiple credential acquisition mechanisms collectively serving the same Attester, with different mechanisms utilized for different targets [Req 5].

The CAS Server Proxy performs the discovery of which credentials types and which credential acquisition mechanisms (enrollment, retrieval) can be provisioned to the Attester for any Attester-supplied target, and automatically picks and utilizes the corresponding mechanism and credential type, without the Attester’s knowledge or involvement [Req 1]. If a credential type specified by the Attester is unavailable due to the Credential Acquisition System limitations, an error may result.

The Credential Acquisition Server implements the server side of the corresponding credential acquisition mechanism and interacts with the RATS Verifier, the Identity Provider (e.g., a Certificate Authority for minting new certificates) and the Key/Credential store for fetching existing keys or credentials, on the Attester’s behalf [Req 7]. The credential types supported by this architecture are limited only by what the Credential Acquisition System can support [Req 2].

This arrangement shields the Attester developers from having to know the details of the platform on which the attester runs. It limits the additional size of the Attester TCB to the smallest possible amount [Req 3]. It enables the Credential Acquisition Client to execute any credential acquisition mechanism, without the Attester’s knowledge [Req 4]. There is no difference, from the standpoint of the Attester, whether the RATS Passport or Background Check model is being used [Req 6].

Under the covers and opaquely to the Attester, the CAS Client Proxy discovers and utilizes one of two credential acquisition modes: Enrollment and Retrieval. Enrollment corresponds to minting new proof-of-possession credentials, and Retrieval is used to fetch bearer tokens and shared proof-of-possession credentials (e.g., for replica workloads). In both cases, the associated secrets remain opaque to the CAS at all times [Req 10].

* Enrollment: the CAS Client Proxy generates and includes alongside Evidence a CSR. It is possible to include Evidence in the CSR, or vice versa: include the CSR in Evidence. The details of how this is decided at runtime are TBD.
For Retrieval, the CAS Client Proxy generates and includes in Evidence an asymmetric encryption key called the Credential Wrapping Key or CWK. The resulting secrets are encrypted to this CWK.

# Credential Acquisition Interface

The Credential Acquisition Interface can be implemented using any mechanism suitable for local communication, including but not limited to Protobuf/gRPC. Here only the high-level description is provided.

## Initiate-Credential-Acquisition

Initiates the process of credential acquisition, stating which target – the RUP – the Attester intends to authenticate to, and credential type(s) it can work with for that target. Can be called any number of times. Results can expire as the underlying challenge can be time-limited. Results on success must be garbage-collected.

Multiple credential acquisitions can be initiated in parallel.

Parameters:
* (required) Target name, i.e., the server URIs to which the Attester wishes to authenticate
* (optional) List of expected credential types; if not specified, any credential type is acceptable for that target

Returns:
* On success: a Context handle, to be used in subsequent communications
* On failure: enumerated reason for failure, such as:
    * Invalid target
    * Unsupported target
    * Invalid credential type(s)
    * Unsupported credential type(s)
    * Server error: permission failure
    * Server error: server unreachable
    * Server error: server unavailable
    * Invalid target
    * Unsupported target
    * etc. (TBD)

## Obtain-Credential

Obtains a credential of a specified credential type, for a previously initiated credential acquisition.

Parameters:
* (required) Valid context handle from the Initiate-Credential-Acquisition call
    * A context handle can only be used in this call once
    * The credential will be obtained for the target specified in the Initiate-Credential-Acquisition call
* (required) Credential type of the credential that the Attester expects to receive

Returns:
* On success: newly acquired credential
* On failure: enumerated reason for failure, such as:
    * Expired context handle; the Attester must repeat the process by calling Initiate-Credential-Acquisition again
    * Reused context handle
    * Invalid context handle
    * Unsupported credential type
    * Server error: failed remote attestation
    * Server error: permission failure
    * Server error: server unavailable; try again later
    * Server error: server unreachable
    * etc. (TBD)

## Finalize-Credential-Acquisition

Called to free up resources. The supplied context handle is no longer valid after this call.

Parameter:
* (required) Context handle from the Initiate-Credential-Acquisition call

Returns:
* Nothing

# Security Considerations {#security}

TODO Security


# IANA Considerations {#iana}

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
